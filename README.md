# Netball Match Timer

A single-page netball match clock: quarters, quarter/three-quarter time,
half time, pause/resume, catch-up, a custom pre-game countdown, and a
landscape "big clock" view. No build step, no dependencies — it's one
HTML file.

## Deploy to GitHub Pages

1. Create a new GitHub repo (public, so Pages works on the free tier).
2. Add `index.html`, `manifest.json`, and the `icons/` folder from this
   package to the repo root (keep the same relative layout).
3. Repo → **Settings → Pages** → under "Build and deployment", set
   **Source: Deploy from a branch**, branch **main**, folder **/ (root)**
   → Save.
4. Wait ~1 minute, then your URL is:
   `https://<your-username>.github.io/<repo-name>/`
5. Open that URL on your phone and **Add to Home Screen** (Safari:
   Share → Add to Home Screen; Chrome/Android: menu → Install app).
   This uses `manifest.json` to launch without browser chrome — the
   most reliable way to get a true fullscreen timer, especially on
   iPhone where the in-page Fullscreen API doesn't work at all.

Any time you want to change the app, edit `index.html`, commit, push —
Pages redeploys automatically.

## Features

- Q1–Q4 (15:00), Quarter Time / Three Quarter Time (2:00), Half Time
  (3:00) — all editable in Settings.
- Runs off the real clock, not a `setInterval` counter — safe to close
  the tab or lock the phone; it recalculates from elapsed wall-clock
  time when reopened, including catching up through missed period
  transitions.
- Pause / Resume (preserves remaining time) and **Catch Up to Now**
  (lets a forgotten pause eat into the clock instead).
- **Start at Custom Time** — enter a future clock time to count down to
  kickoff, or a time already passed (running late) to jump straight
  into the correct current period, backdated, with missed-period beeps
  suppressed for that catch-up.
- Settings page: per-period duration editing, beep volume slider, beep
  test buttons (bypass mute), reset to defaults.
- Mute toggle, fullscreen toggle, 30-second warning beep, and a
  4-pulse end-of-period beep, kitchen-timer style.
- Landscape: rotating the phone shows just the big countdown and a
  small period label ("Half Time", "Starting In", etc.) full-screen,
  with a small ✕ in the corner to exit fullscreen. Rotate back to
  portrait for all other controls.
- "Stop & Reset" (paused-only, two-tap confirm) and an exit warning if
  you try to navigate away mid-match.

## Storage

Settings and match progress are saved automatically, using
`window.storage` when available (Claude's own app/artifact viewer) or
`localStorage` as a fallback (any normal browser). Storage is scoped
per browser origin — hosting on a stable URL (Pages, rather than a
redownloaded file) is what makes this reliable; see the note above
about always using the same URL.

## Known limitations

- **Audio needs a tap first.** Browsers block sound until a user
  gesture creates it — beeps stay silent until you've tapped Start,
  Pause/Resume, the fullscreen button, or a test-beep button at least
  once per page load.
- **Auto-fullscreen on rotation is best-effort.** Browsers generally
  require a direct user gesture (not just an orientation change) to
  enter fullscreen, so it may not fire automatically every time — use
  the fullscreen button in the top bar as a reliable manual fallback.
  iOS Safari doesn't support the in-page Fullscreen API on iPhone at
  all regardless — installing via "Add to Home Screen" (see deploy
  steps above) is the real fix there, since it launches without
  browser chrome without needing the Fullscreen API.
- **The exit-confirmation dialog is best-effort too.** Modern browsers
  show their own generic "Leave site?" wording (custom text isn't
  allowed anymore), and some mobile browsers skip the prompt entirely
  on back-gesture or tab close.
- **Storage is per-device, per-browser.** It won't sync between your
  phone and a laptop, and clearing site data for the page resets it.
