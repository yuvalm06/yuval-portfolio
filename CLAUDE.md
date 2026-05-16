# Portfolio — System Context for Claude

This is a single-file portfolio (`index.html`) built in the style of hut8.com.
Everything lives in one file: HTML structure, CSS, and JavaScript.

---

## Architecture overview

```
card (rounded inset frame, #EEECEA background)
 ├── #dot-canvas          ← animated perspective dot grid (Canvas 2D)
 ├── .distance-fog        ← CSS gradient overlay that fades dots into distance
 ├── .scene               ← 3D render / SVG objects sit here
 ├── .nav                 ← top bar: logo left, brand pill + hamburger right
 ├── .nav-overlay         ← full-screen slide-up nav menu
 ├── .identity            ← bottom-left: icon | title | layer label | description
 ├── .page-dots           ← right edge vertical dot indicators
 └── .section-nav         ← bottom-right prev/next buttons
```

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

## Adding a new 3D object / Blender render

### With a Blender PNG render (recommended)
Replace the scene div content:
```html
<div class="scene">
  <img class="scene-img" src="your-render.png" alt="Scene" />
</div>
```

Blender camera settings to match this system:
- Type: **Orthographic**
- Rotation: X = 60°, Z = 45°
- Material: Principled BSDF, roughness 1.0, off-white (#E8E6E1)
- Lighting: HDRI overcast, no direct lights
- Render: transparent background, 2680×1560px (2× card size)

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
2  → .distance-fog
3  → .scene
5  → .card::after (vignette)
8  → .hotspot
10 → .card::before (grain)
20 → .nav, .identity, .page-dots, .section-nav
50 → .nav-overlay
```

---

## Key CSS variables
```css
--bg:        #EEECEA   /* card background */
--ink:       #1A1916   /* dark text */
--ink-mid:   #4A4844
--ink-light: #8A8784
--pill-bg:   #2E2D2B   /* hotspot buttons */
--font:      'Instrument Sans', sans-serif
```

---

## What NOT to do
- Do not add `localStorage` or `sessionStorage` — not needed, everything is in-memory
- Do not split into multiple files unless explicitly asked — single-file is intentional
- Do not add a JS framework — vanilla JS only
- Do not change `PERSP`, `GS`, `CX`, `CY` constants without re-deriving all SVG object positions
- Do not apply `skewX` or `scaleY` to `.scene` — it warps the render. Translate only.
- Do not use `transition` on `.scene` — the lerp in the rAF loop handles smoothing
