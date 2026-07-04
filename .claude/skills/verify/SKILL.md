---
name: verify
description: Build/launch/drive recipe for verifying changes to this single-file bracket app (index.html) with Playwright.
---

# Verifying WC26 (single-file static app)

No build step. The whole app is `index.html`; serve it and drive it with
Playwright chromium (`executablePath: '/opt/pw-browsers/chromium'` in remote
sessions — the npm-installed browser revision usually doesn't match).

## Launch

- Serve with a tiny `http.createServer` that returns `index.html` for every
  path (or `python3 -m http.server`), e.g. on `localhost:8901`.
- The page renders **nothing** until its feed loads. Intercept with
  `page.route`:
  - `**/raw.githubusercontent.com/**` → fulfill with a fixture
    `{"matches":[...]}` — 16 objects `num: 73..88`, `team1`/`team2` from the
    ISO map in index.html, plus `date: "2026-06-28"`, `time: "12:00 UTC-4"`,
    `ground: "Atlanta"`. Give one match `score: {ft:[2,1]}` to get a winner
    flag.
  - `**/api.github.com/**` → fulfill `[]`.
  - `**/flagcdn.com/**` → fulfill a 1x1 PNG (flags render as circles; layout
    is unaffected).
- Wait for `.flag` (33 render with the fixture above).

## Flows worth driving

- Tooltip: tap/hover a `.flag` → `.tooltip.on` appears; its horizontal center
  must equal the flag's rect center; near-top flags (rect.top < 90 of the
  visible region) get `.below`. Capsule (`.cap`) tooltips anchor at the
  pointer after a 600 ms settled hover; find a hover point via
  `elementFromPoint` sweep (caps are SVG strokes).
- Mobile: context with `isMobile: true, hasTouch: true`; tap toggles the tip.
- Pinch zoom: `Emulation.setPageScaleFactor` + `Input.synthesizeScrollGesture`
  via CDP (gesture coords must lie inside the *zoomed* visual viewport).
  Tooltip position math is in document coords; the stage width must NOT
  change on resize events while zoomed.

## Gotchas

- The tooltip show/flip transition is 120 ms — wait ~250 ms before measuring
  its rect, or you'll read mid-transition geometry.
- `Playwright tap` auto-scrolls the target into view; read the flag's
  bounding box *after* tapping when the page is scrolled.
- Flags sit on the ring perimeter; the ring interior has no tappable flags,
  so aim pans/taps at the arc, not the center.
