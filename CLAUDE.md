# Portfolio — System Context for Claude

This is a single-file portfolio (`index.html`) built in the style of hut8.com.
Everything lives in one file: HTML structure, CSS, and JavaScript.

Two layouts, chosen once in `<head>`: the **desktop** 3D scene (everything below),
and the **compact** layout for phones / portrait tablets (`html.compact`) — the same
scene as a one-project-at-a-time carousel: swipes truck the camera along a line of
featured models, the dot ground scrolling under them, with the project panel docked
as a sheet. See `dev/features/compact-layout.md`. Project copy lives in ONE place,
`window.PROJECT_INFO` at the top of the main script — every project in one shape
(team · role, its own image first, Problem → Approach → Result, dates · status, the
next step), strongest first: FSAE, steering wheel, Rivian, scooter, printer. Where a
scene model is a stand-in rather than the project's own CAD, its `model` note says so.
The Digital files live in `DIGITAL_INFO` (the digital page script); each file's visual
leads its points — app screenshots (`screens`) play in a phone or browser frame
(`frame`), see `dev/features/digital-page.md`. Visitor copy stays short: say a thing once (the panel or
doc, not also the caption), and let the image carry what it can.

Every view has an address (`#fsae`, `#digital/pursr`, `#about` …) so browser Back
steps out of a view and a project can be linked — the router at the end of the main
script. See `dev/features/addresses.md`.

---

## Architecture overview

```
card (rounded inset frame, #EEECEA background)
 ├── #dot-canvas          ← animated perspective dot grid (Canvas 2D)
 ├── .distance-fog        ← CSS gradient overlay that fades dots into distance
 ├── <canvas> (WebGL)     ← Three.js scene, injected by JS with inline styles (z 3)
 ├── .digital-page        ← the Digital page, flips in from the right (z 16); an open
 │                          file docks left, its sheet (or app screens in a device
 │                          frame) right
 ├── .race-page           ← race-game easter egg overlay (z 45), opened by the
 │                          green button on the steering wheel's face
 ├── .nav                 ← top bar (z 35): name + role line + status note left;
 │                          Resume · Contact · menu button right
 ├── .nav-overlay         ← full-screen slide-up nav menu
 ├── .identity            ← bottom-left: icon | title | layer label | description
 ├── .frame-marks         ← drawing-frame corner ticks + sheet note
 ├── .scene-labels        ← desktop: each project's number + name at its model
 │                          at rest; click one to open it (z 14)
 ├── .edge-tab ×2         ← thin side handles to flip to the other world (z 21)
 ├── .section-nav         ← bottom-right prev/next: projects 01→05, or the files
 │                          on Digital (never flips pages); past the last → wrap-up
 ├── .end-card            ← the wrap-up after either side's last project: the
 │                          other side, resume, contact (z 18, over .end-scrim z 17)
 ├── .m-stage             ← compact only: the box the carousel frames the model
 │                          into (the dot grid measures it) + loading note (z 14)
 ├── .m-pager             ← compact only: ‹ project dots › (z 21)
 └── .mode-switch         ← compact only: Hands-On | Digital switch (z 21)
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

- `CX = W * 0.50`, `CY = H * 0.55` — ground plane anchor (screen centre; plus the
  compact view offset `offX`/`offY`, which is 0 on desktop)
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
scale, sit on ground). Draco-compress it first (`gltf-transform draco`) and
load it through the shared `gltfLoader` — see the Draco section of
`dev/lessons/glb-workflow.md`; raw Meshy exports are 10-15x too heavy to ship.
A generated model (Meshy, a prop) is a stand-in: set the project's `model` note in
`PROJECT_INFO` so the panel says so, and lead with the real CAD / FEA image (`img`).

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
14 → .dim-line, .detail-callout, .scene-labels, .m-stage (compact only)
15 → .sprocket-panel, .project-panel
16 → .digital-page (scrim z3 / doc z6 / raised stack column z5 inside it)
17 → .end-scrim
18 → .end-card (the wrap-up)
20 → .identity, .frame-marks, .section-nav
21 → .edge-tab, .mode-switch + .m-pager (compact only)
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
  (identity/hint opacity, panels, dim line, scene labels) — the lerp handles smoothing
- Do not assume the 3D module has loaded: it runs after three.js arrives from the CDN, so
  every `window._…` hook it defines can be absent — guard calls from the main script
  (`window._x && window._x()`)
- Do not hardcode the side-view dot constants again: `side()` scales them with the zoom
  distance (compact frames its own, desktop stays exactly at ZD 8) — see
  `dev/lessons/dot-grid-reprojection.md`
- Do not duplicate project copy — edit `window.PROJECT_INFO` (main script)
- Do not add a page or overlay inside `.card` without adding it to the click guard at the
  top of the card click handler and to `window._sceneCover()` — its clicks bubble to the
  3D raycast and would zoom whatever model sits under the pointer
  (see "Covered-scene gating" in `dev/features/3d-scene.md`) — and without giving it an
  address in the router's `stateHash()` / `applyHash()`, or Back can't close it
- Do not put jargon in visitor copy ("Build 1.0", "Index", "Sheet", "Rev"): the two sides
  are Hands-On and Digital everywhere
