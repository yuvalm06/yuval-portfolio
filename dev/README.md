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

## Lessons

- [camera-matching.md](lessons/camera-matching.md) — matching Three.js camera to dot grid
- [glb-workflow.md](lessons/glb-workflow.md) — sourcing, loading, and placing GLB models
- [dot-grid-reprojection.md](lessons/dot-grid-reprojection.md) — deriving side-view projection constants

## Tooling

Puppeteer is installed at `/tmp/node_modules` for taking automated screenshots during iteration.
Run: `cd /tmp && node your_script.mjs`
Server runs at port 3000 via `npx serve` from project root.
