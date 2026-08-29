# Netball Match Timer

A single-page netball match clock: quarters, quarter/three-quarter time,
half time, pause/resume, catch-up, a custom pre-game countdown, and a
landscape "big clock" view. No build step, no dependencies — it's one
HTML file.

## Deploy to GitHub Pages

1. Create a new GitHub repo (public, so Pages works on the free tier).
2. Add `index.html` from this folder to the repo root.
3. Repo → **Settings → Pages** → under "Build and deployment", set
   **Source: Deploy from a branch**, branch **main**, folder **/ (root)**
   → Save.
4. Wait ~1 minute, then your URL is:
   `https://<your-username>.github.io/<repo-name>/`
5. Bookmark that URL or "Add to Home Screen" on your phone — use the
   same URL every time so match settings and progress persist correctly
   (see "Storage" below).

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
- **Start at Custom Time** — set a target clock time and it counts down
  to kickoff, then hands off into whichever period you'd selected.
- Settings page: per-period duration editing, beep volume slider, beep
  test buttons (bypass mute), reset to defaults.
- Mute toggle, 30-second warning beep, and a 4-pulse end-of-period
  beep, kitchen-timer style.
- Landscape: rotating the phone shows just the big countdown and a
  small period label ("Half Time", "Starting In", etc.) full-screen.
  Rotate back to portrait for all controls.
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
  Pause/Resume, or a test-beep button at least once per page load.
- **Auto-fullscreen on rotation is best-effort.** Some browsers only
  allow entering fullscreen from a direct user gesture, and iOS Safari
  doesn't support element fullscreen on iPhone at all. If it doesn't
  trigger automatically, the layout still works — you just won't get
  true fullscreen.
- **The exit-confirmation dialog is best-effort too.** Modern browsers
  show their own generic "Leave site?" wording (custom text isn't
  allowed anymore), and some mobile browsers skip the prompt entirely
  on back-gesture or tab close.
- **Storage is per-device, per-browser.** It won't sync between your
  phone and a laptop, and clearing site data for the page resets it.
