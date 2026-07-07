# Portfolio — System Context for Claude

This is a single-file portfolio (`index.html`) built in the style of hut8.com.
Everything lives in one file: HTML structure, CSS, and JavaScript.

---

## Architecture overview

```
card (rounded inset frame, #EEECEA background)
 ├── #dot-canvas          ← animated perspective dot grid (Canvas 2D)
 ├── .distance-fog        ← CSS gradient overlay that fades dots into distance
 ├── <canvas> (WebGL)     ← Three.js scene, injected by JS with inline styles (z 3)
 ├── .digital-page        ← Build 2.0 page, flips in from the right (z 16)
 ├── .race-page           ← race-game easter egg overlay (z 45), opened by the
 │                          green button on the steering wheel's face
 ├── .nav                 ← top bar: logo left, hamburger right (z 35, above intro)
 ├── .nav-overlay         ← full-screen slide-up nav menu
 ├── .identity            ← bottom-left: icon | title | layer label | description
 ├── .frame-marks         ← drawing-frame corner ticks + sheet note
 ├── .split-intro         ← front door: two equal halves (Hands-On / Digital),
 │                          click swipes to that world (z 30, shown until a pick)
 ├── .edge-tab ×2         ← thin side handles to flip to the other world (z 21)
 └── .section-nav         ← bottom-right prev/next (cycle projects / flip page)
```

See `dev/features/*.md` for how each major feature works.

---

## The dot grid system (most important)

All logic lives in a single IIFE at the bottom of the `<script>` block.

### Projection formula
The dots are projected from a flat 3D ground plane using a perspective camera:

```
For lattice point (gx, gz):
  A     = gx - gz                    ← "across" the plane
  D     = gx + gz                    ← "depth" into the plane
  denom = 1 - PERSP * D              ← perspective divide

  sx = CX + GS * (A + hSkew * D) / denom
  sy = drawCY + GS * 0.5 * tilt * D / denom
```

- `CX = W * 0.50`, `CY = H * 0.55` — ground plane anchor (screen centre)
- `GS = 22` — base grid spacing in px
- `PERSP = 0.011` — perspective strength (~25% compression at far edge)
- Near dots (D > 0, bottom of screen): spacing expands
- Far dots  (D < 0, top of screen):    spacing compresses
- Lines stay locally parallel — same 26.57° angle everywhere

### Camera state (live lerped values)
| Variable | What it does | Range |
|----------|-------------|-------|
| `hSkew`  | Horizontal yaw — adds `hSkew * D` cross-term to sx, rotating the ground plane. Camera direction only, position fixed. | ±0.09 |
| `tilt`   | Vertical angle — scales the `D` contribution to sy. Low = top-down, high = lower camera. | 0.78–1.22 |
| `vShift` | Vertical position — physically shifts `drawCY` up/down (horizon moves). | ±16px |

### Resting state
When cursor leaves: `normX = -0.5` (left edge), `normY = 0` (middle vertical).
So resting hSkew targets `-0.09` — the ground is slightly angled left-forward.

### Wave
```js
const wave = Math.sin((A + D) * FREQ_D * 0.5 - t) * 0.5 + 0.5;
```
Wave fronts run along the `gz` axis (upper-right → lower-left).
Wave travels upper-left → lower-right. Speed: `SPEED = 1.1` rad/sec.

### Scene / objects transform
The `.scene` div gets a pure `translate(X, Y)` — no skew or warp.
Scene objects appear undistorted while the ground plane beneath them deforms.
```js
const sceneX = hSkew * -45;   // ±4px horizontal
const sceneY = vShift * 0.25; // ±4px vertical
```

---

## Adding a new 3D object

### With a GLB model (how the current scene works)
The 3D scene is a Three.js WebGL canvas injected by the module script — see
`dev/features/3d-scene.md` and `dev/lessons/glb-workflow.md`. Add a GLB under
`renders/` and follow the existing loader pattern (clone materials, normalise
scale, sit on ground).

### With SVG geometry (for simple shapes)
All SVG objects must use the same projection:
```
proj(x, y, z):
  sx = x * 22 - z * 22
  sy = x * 11 + z * 11 - y * 16
```
Group at `translate(CX_offset, CY_offset)` where `CY_offset ≈ 375` (slightly above CY)
so objects sit on the ground at y=0.

Painter order (back → front): far elements first, near elements last.
Face shading:
- Top faces:       #EEECEA (lightest)
- Near/left faces: #D8D6D0 (medium)
- Far/right faces: #BCBAB4 (dark)
- Recesses:        #A8A6A0 (darkest)

Shadow system (3 layers):
1. `filter: blur(18px)`, opacity 0.09 — ambient halo
2. `filter: blur(4px)`,  opacity 0.14 — body contact
3. Contact ellipses per ground-touch point, `blur(4px)`, opacity 0.18–0.22

---

## CSS z-index stack
```
1  → #dot-canvas
2  → .distance-fog, .card::after (vignette)
3  → WebGL canvas (inline style)
10 → .card::before (grain)
14 → .dim-line, .detail-callout
15 → .sprocket-panel, .project-panel
16 → .digital-page (scrim z3 / doc z6 / raised stack column z5 inside it)
20 → .identity, .frame-marks, .section-nav
21 → .edge-tab
22 → .hover-tag
30 → .split-intro
35 → .nav
40 → .contact-page, .page-overlay (about / skills)
45 → .race-page (steering-wheel easter egg)
50 → .nav-overlay
```

---

## Key CSS variables
```css
--bg:        #EEECEA   /* card background */
--ink:       #1A1916   /* dark text */
--ink-mid:   #4A4844
--ink-light: #8A8784
--font:      'Instrument Sans', sans-serif
--mono:      'IBM Plex Mono', ui-monospace, monospace
```

---

## What NOT to do
- Do not add `localStorage` or `sessionStorage` — not needed, everything is in-memory
- Do not split into multiple files unless explicitly asked — single-file is intentional
- Do not add a JS framework — vanilla JS only
- Do not change `PERSP`, `GS`, `CX`, `CY` constants without re-deriving all SVG object positions
- Do not add CSS `transition` to elements whose style is driven per-frame by the rAF loops
  (identity/hint opacity, panels, dim line, hover tag) — the lerp handles smoothing
