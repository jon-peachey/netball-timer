# Netball Match Timer — Claude Code project instructions

Single-page netball match clock. No build step, no framework, no
dependencies (fonts load from Google Fonts CDN at runtime; everything
else is one HTML file). Deployed on GitHub Pages at netball-timer.com.

Read `memory.md` before making non-trivial changes — it has the design
rationale and bug history behind decisions in this file that aren't
otherwise obvious from the code.

## File layout

- `index.html` — the entire app (HTML + CSS + JS, single file, no
  build step). This is the only file that changes for feature work.
- `manifest.json` — PWA manifest so "Add to Home Screen" launches
  without browser chrome (this is the real fix for iOS fullscreen,
  see memory.md).
- `icons/` — app icons referenced by the manifest and `<link>` tags in
  `index.html`. Regenerate with Pillow if the design changes; don't
  hand-edit PNGs.
- `README.md` — user-facing setup/deploy instructions. Keep in sync
  with any user-visible behavior change (features, limitations).
- `CLAUDE.md` / `memory.md` — this file and the project history, not
  deployed, not linked from `index.html`.

## Hard constraints — do not violate these

1. **Stay a single HTML file.** No bundler, no npm, no build step. If
   a feature seems to need a library, check it can load as a plain
   `<script>` tag with no module system, or don't use it.

2. **Never use `window.confirm()`, `window.prompt()`, or
   `window.alert()`.** These are blocked/silently no-op in sandboxed
   iframe contexts (Claude's own artifact viewer sandboxes without
   `allow-modals`). This bit us twice already (Stop & Reset
   confirmation, the original custom-time picker). Build inline UI
   instead — see the two-tap "Stop & Reset" pattern or the inline
   `<input type="time">` picker in `index.html` for the established
   approach.

3. **Never use `localStorage`/`sessionStorage` as the only storage
   path.** `window.storage` (Claude's artifact-viewer API) is the
   primary path; `localStorage` is a fallback only used when
   `window.storage` doesn't exist (i.e. running as a normal
   standalone page, which is the primary real-world use case now).
   See `storageGet`/`storageSet` in `index.html` — always go through
   those, never call either storage API directly.

4. **The clock logic is anchor-based, not `setInterval`-counted.**
   Elapsed time for the current period/pre-game slot is always
   `now - state.anchor`, recomputed fresh every render — never
   accumulate elapsed time in a variable that ticks down. This is
   what makes it safe to close the tab mid-match: on reload,
   `reconcileAndBeep()` replays the real-time backlog (including
   multiple missed period transitions) from the stored anchor. Any
   change to period/timing logic must preserve this property — test
   it by checking state persists correctly across a simulated
   multi-minute gap, not just a live tick.

5. **Don't re-run `renderAll()` (or anything that rebuilds the DOM
   subtree) on every tick if that subtree contains live user input**
   (the start-period `<select>`, the custom-time `<input>`, the
   settings duration inputs). Rebuilding wipes focus/selection
   mid-interaction. The tick loop currently skips rendering entirely
   while `!state.started` for exactly this reason — if you add new
   always-visible inputs, make sure the tick loop's render calls
   don't blow them away.

6. **Keep beep/mute/volume behavior consistent across three call
   sites**: the automatic 30s-warning and end-of-period beeps (must
   respect `muted`), the settings page's test buttons (must bypass
   `muted` — user-requested exception, see memory.md), and the
   silent catch-up pass on a backdated custom start (must suppress
   regardless of `muted`, via the separate `beepsSuppressed` flag).
   These are three different flags for a reason — don't collapse them
   into one.

## Conventions

- Palette, fonts, and spacing are defined as CSS custom properties at
  the top of `<style>`. Change the theme there, not by hand-editing
  individual color values throughout.
- Portrait is the full-featured layout; landscape (media query,
  `orientation: landscape` + `max-height: 600px`) intentionally shows
  **only** the big clock and a small period label — this was an
  explicit user requirement, not an oversight. Don't add controls
  back into the landscape view without checking with the user first.
- JS is plain, un-namespaced global functions/state (no classes, no
  modules) — matches the rest of the file, keep new code in the same
  style rather than introducing a different pattern partway through.

## Testing changes

There's no test suite. Before considering any change to `index.html`
done, at minimum:

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html','utf8');
const script = html.split('<script>')[1].split('</script>')[0];
new Function(script); // throws on syntax errors
console.log('SYNTAX OK');
"
```

This only catches syntax errors, not logic bugs — it caught several
real mistakes during development (a function declaration accidentally
deleted by a bad find-replace, orphaned code blocks) that would
otherwise have shipped silently. Also grep for duplicate/missing
function definitions after any large edit:

```bash
grep -c "function " index.html
```

For anything touching the clock/period/beep logic, manually reason
through: pause → resume, pause → catch-up, tab-closed-and-reopened
after 1 missed period, tab-closed-and-reopened after 3+ missed
periods, and a backdated custom start time. These are the cases that
have broken before.
