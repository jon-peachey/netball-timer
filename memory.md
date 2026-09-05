# Project memory

Build history and reasoning for decisions in this codebase that
aren't obvious from reading `index.html` cold. Written for a future
Claude Code session (or human) picking this up without the original
conversation. See `CLAUDE.md` for the resulting hard rules.

## What this is

A netball match timer, originally built as a Claude.ai artifact,
now a standalone site deployed on GitHub Pages at netball-timer.com
(DNS via Cloudflare — registrar and DNS provider). Owner uses it
courtside, on a phone, sometimes with the phone rotated to landscape
and propped up.

## Core design: wall-clock anchoring, not a ticking counter

The clock never counts down an in-memory number. Every render
computes `elapsed = now - state.anchor` fresh. This was the
foundational decision and everything else follows from it:

- Closing the tab and reopening "just works" — reload, recompute
  from the stored anchor, done.
- Pause freezes `elapsed` by *not* touching `anchor`, storing
  `pauseStartedAt` instead. Resume shifts `anchor` forward by the
  pause duration (preserves remaining time). "Catch Up to Now"
  deliberately does *not* shift `anchor` — the stoppage time gets
  eaten into the countdown instead, which is the whole point of that
  button (user's original request: "advance the time to catch up if
  I forget to unpause").
- Reopening after being closed across *multiple* period boundaries
  (e.g. closed the tab for an hour) needed to replay the whole
  backlog, not just the current period. `reconcileAndBeep()` is a
  `while` loop for exactly this reason — it can walk through several
  period transitions (and fire their beeps) in one pass.

## Pre-game countdown ("Start at Custom Time")

Modeled as `state.periodIdx === -1`, a sentinel that isn't in the
`PERIODS` array. This was a deliberate choice over adding a fake
8th period to the array, because the user explicitly required it
**never show up as a period** on the progress rail or in the period
count — keeping it out of `PERIODS` made that true for free rather
than needing special-case filtering everywhere the rail is drawn.

`currentDurationMs()` and `advanceToNextSlot()` abstract over "is
this the pregame slot or a real period" so `reconcileAndBeep()`
doesn't need to know the difference — it just asks "how long is the
current slot" and "what's next."

Two entry paths from the time picker, added late and worth
preserving the distinction:
- **Future time** → normal audible countdown to kickoff (pregame
  slot, full beeps).
- **Past time** (user is running late) → skip the countdown
  entirely, backdate `anchor` to that clock time, and run
  `reconcileAndBeep()` once immediately with `beepsSuppressed = true`
  so it silently fast-forwards through whatever periods would already
  be over, without firing their beeps. This was a deliberate late
  addition — the *original* behavior on a past time was to assume
  "tomorrow," which was wrong for the actual use case (arriving late
  to a game already in progress).

## Three separate mute/suppression mechanisms — don't merge them

- `muted` — the user-facing toggle. Silences the real in-match 30s
  and end-of-period beeps.
- `bypassMute` param on `playBeep`/`scheduleTone` — used only by the
  settings page's "Test 30s Beep"/"Test End Beep" buttons. Explicit
  user decision: test buttons should make sound *even if muted*, so
  someone can verify/tune the sound without having to un-mute first.
- `beepsSuppressed` — a temporary flag set only around the backdated
  catch-up reconcile call above. Independent of the mute toggle
  entirely; it doesn't change `muted` and doesn't persist.

These look similar but answer different questions ("did the user ask
for silence," "should this specific button ignore that," "is this
one programmatic catch-up pass allowed to be noisy"). Collapsing them
into one flag will misfire one of the three cases.

## Storage: two backends, chosen automatically

`window.storage` is an API that only exists inside Claude's own
artifact viewer/app — it doesn't exist in a normal browser tab.
Early versions used it exclusively, which meant settings/progress
appeared to work fine while testing inside Claude, then silently did
nothing once the file was downloaded and opened directly in Chrome
(all failures were swallowed by try/catch, so this wasn't obvious
until the user reported it).

Fix: `hasAppStorage` is checked once at load; `storageGet`/
`storageSet` route to `window.storage` if present, else
`localStorage`. Never call either storage API directly elsewhere.

Separately diagnosed: even with the fallback, a *downloaded file*
reopened via Android's Downloads/Chrome flow can get a different
origin each time (changing `content://` URI), which resets
`localStorage` regardless of the fallback working correctly. This is
why the recommendation was to move to a hosted URL (GitHub Pages)
rather than keep redistributing the raw file — the fallback code was
correct, the delivery method was the actual problem.

## Sandboxed-iframe modal restrictions (bit us twice)

Claude's artifact viewer runs pages in a sandboxed iframe without
`allow-modals`. `window.confirm()` and `window.prompt()` are
silently blocked there — they don't throw, they just don't do
anything, which is a confusing failure mode ("the button doesn't
work" with no error).

Hit this twice:
1. The original "Stop & Reset" used `confirm()`. Replaced with an
   inline two-tap pattern (button label changes to "Tap again to
   confirm" for a few seconds).
2. The original custom-start-time design considered `window.prompt()`
   for entering a time. Went straight to an inline `<input
   type="time">` instead, based on lesson 1.

**Do not reintroduce `confirm`/`prompt`/`alert` anywhere in this
file.** This matters even for a standalone deployment, since the
same file may still be opened inside Claude's viewer.

## Fullscreen: three overlapping approaches, none sufficient alone

- Auto-fullscreen on rotation (`orientation` media query listener
  calling `requestFullscreen()`) — mostly doesn't fire in practice,
  because an orientation change isn't treated as a "user gesture" by
  most browsers, and the Fullscreen API requires one.
- Manual fullscreen toggle button (added next to mute) — works,
  because a click is a real gesture. Kept as the reliable fallback.
- Home-screen web app (`manifest.json` + Apple meta tags) — the
  actual fix for iPhone, where the in-page Fullscreen API doesn't
  work *at all*, regardless of gesture. "Add to Home Screen" launches
  without browser chrome without touching the Fullscreen API, so it
  sidesteps the platform gap entirely.

All three are kept simultaneously on purpose — none of them alone
covers every browser.

## Other bugs worth knowing happened (fixed, but instructive)

- **"Reset Defaults" was a no-op** for a while: it rebuilt "defaults"
  by reading the *current* (possibly already-customized) `PERIODS`
  durations instead of a snapshot taken before any settings were ever
  loaded. Fixed by capturing `ORIGINAL_DURATIONS` immediately after
  `PERIODS` is declared, before `loadSettings()` runs. Any future
  "reset to default" feature must snapshot state before user
  customization can touch it, not derive "default" from current
  state.
- **The settings panel could go blank in landscape**: landscape CSS
  hides everything with class `.panel` to declutter the courtside
  view, but the settings container also used that class. Fixed by
  forcing `showSettings = false` whenever orientation flips to
  landscape, rather than scoping the CSS more narrowly — simpler and
  matches the existing "settings needs portrait" intent.
- **Volume slider was calling `saveSettings()` (async storage write)
  on every `input` event** — dozens of writes per second while
  dragging. Split into `oninput` (update the live `volume` variable
  only) and `onchange` (persist once, on release).
- A find-replace once deleted a `function ensureAudio(){` declaration
  line while leaving its body in place, and separately once deleted a
  CSS rule (`.footnote`) as collateral damage from an unrelated edit.
  Neither broke the syntax check loudly — one left orphaned top-level
  statements that happened to still parse, the other just silently
  dropped styling. Worth re-reading the diff area around any
  multi-line `str_replace`, not just trusting that "syntax OK" means
  "nothing was lost."

## Known deliberate non-features

- No date picker for the custom start time — `<input type="time">`
  only handles a clock time within the current day. A past time
  always means "today, earlier"; there's no way to schedule
  "tomorrow morning" through that field. Not requested; would need a
  `datetime-local` input and different backdating logic if it ever
  is.
- No score tracking — explicitly declined by the user during initial
  requirements gathering (clock-only was chosen over clock+score).
- No settings sync across devices — storage is per-browser-origin by
  design (see Storage section); this was accepted as a known
  limitation rather than solved.
