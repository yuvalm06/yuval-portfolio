# Feature: Perspective Dot Grid

**File:** `index.html` — single IIFE at bottom of `<script>` block  
**Status:** Working

---

## What it does

Animated canvas-based ground plane using a custom perspective projection.
Dots are projected from a flat 3D grid using a perspective divide, then lit
by a travelling sine wave. Mouse movement drives three live camera values
that lerp smoothly each frame.

---

## Projection formula

```
A     = gx - gz            // "across" the plane (left-right on screen)
D     = gx + gz            // "depth" into the plane (near/far)
denom = 1 - PERSP * D      // perspective divide

sx = CX + GS * (A + hSkew * D) / denom
sy = drawCY + GS * 0.5 * tilt * D / denom
```

- Near dots (D > 0): sit at bottom of screen, spacing expands
- Far dots  (D < 0): sit at top, spacing compresses toward horizon
- PERSP = 0.011 → very subtle, nearly orthographic

---

## Camera state (lerped every frame)

| Variable | Effect | Range | Rest value |
|----------|--------|-------|------------|
| `hSkew`  | Horizontal yaw — adds `hSkew*D` cross-term to sx | ±0.09 | −0.09 (slight left) |
| `tilt`   | Vertical angle — scales D contribution to sy | 0.78–1.22 | 1.0 |
| `vShift` | Horizon position — shifts `drawCY` up/down | ±16px | 0 |

Lerp rate for all three: `0.05` per frame (smooth, ~60fps feel).

---

## Key constants

```js
GS      = 22      // grid spacing in px
PERSP   = 0.011   // perspective strength
DOT_R   = 1.5     // base dot radius
SPEED   = 1.1     // wave speed (rad/sec)
FREQ_D  = 0.18    // wave spatial frequency along depth
MOUSE_R = 105     // px radius of mouse influence on dots
CX      = W * 0.50
CY      = H * 0.55
```

---

## Sharing state with Three.js

The IIFE writes to `window._dotCam` every frame:

```js
window._dotCam = { hSkew: -0.09, tilt: 1, vShift: 0 }; // init
// in draw():
window._dotCam.hSkew  = hSkew;
window._dotCam.tilt   = tilt;
window._dotCam.vShift = vShift;
```

The Three.js loop reads this each frame. No events, no callbacks — just a shared
object polled at 60fps.

---

## Wave

```js
const wave = Math.sin((A + D) * FREQ_D * 0.5 - t) * 0.5 + 0.5;
```

Travels upper-left → lower-right. Fronts run along the gz axis.
