# Dev Reference

A living knowledge base for this portfolio project. Every time a feature is built or
a significant decision is made, log it here. The goal is fast re-entry: know what worked,
what didn't, and why — without re-deriving it from scratch.

## Structure

```
dev/
  features/       — one file per major feature: how it works, key values, gotchas
  lessons/        — what failed and why, indexed by topic
  README.md       — this file
```

## Features built

- [dot-grid.md](features/dot-grid.md) — animated perspective ground plane
- [3d-scene.md](features/3d-scene.md) — Three.js WebGL model integrated with dot grid
- [zoom-animation.md](features/zoom-animation.md) — click-to-zoom into side-profile view
- [digital-page.md](features/digital-page.md) — Build 2.0 flip page: file manifest + isometric sheet stack
- [race-game.md](features/race-game.md) — easter egg: green steering-wheel button opens an A/D obstacle runner
- [compact-layout.md](features/compact-layout.md) — phones / portrait tablets: the 3D scene as a one-model-at-a-time swipe carousel

## Lessons

- [camera-matching.md](lessons/camera-matching.md) — matching Three.js camera to dot grid
- [glb-workflow.md](lessons/glb-workflow.md) — sourcing, loading, and placing GLB models
- [dot-grid-reprojection.md](lessons/dot-grid-reprojection.md) — deriving side-view projection constants

## Tooling

Puppeteer is installed at `/tmp/node_modules` for taking automated screenshots during iteration.
Run: `cd /tmp && node your_script.mjs`
Note: `/tmp` gets wiped on reboot — if the module is missing, either reinstall or use headless
Chrome directly, which works without any install:
`"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu --screenshot=out.png --window-size=W,H --force-device-scale-factor=2 <url>`
Server runs at port 3000 via `npx serve` from project root.

### Headless testing notes (Sept 2026, cloud sandbox)

- Playwright + Chromium are preinstalled there; `cdn.jsdelivr.net` may be blocked by
  the sandbox network policy — `npm install three@0.165.0` somewhere and
  `context.route('https://cdn.jsdelivr.net/npm/three@0.165.0/**', ...)` to the local copy.
- Software WebGL (SwiftShader) runs the full scene at ~0.5–0.7 fps. CSS transitions and
  the rAF state only advance between those long frames, so fixed sleeps lie: wait on
  state instead (`window._dotCam.zoomEase`, `document.elementFromPoint` landing inside
  the page you just opened) before clicking or asserting.
