# Lesson: Dot Grid Reprojection for Camera Transitions

## The problem

When animating the Three.js camera from isometric to side-view, the dot grid
needs to reproject simultaneously so the ground plane "moves" as one unified scene.
Just moving Three.js camera parameters (az, el, dist) while the dot grid stays
isometric creates a jarring disconnect — car rotates, dots stay put.

---

## What didn't work

### Blending isometric camera params (hSkew, tilt, vShift) toward side-view targets
**Why it failed:** These params drive the isometric A/D formula. There's no set of
hSkew/tilt/vShift values that produces a true front-facing side perspective. The dots
just tilt in weird ways, never looking like a real ground plane from the side.

### Scaling GS by BASE_DIST / dist
**Why it failed:** GS×2.25 pushed most dots off-screen (spacing became 49.5px).
The pts array was built with isometric screen bounds, so the side-view coverage
was nearly empty. The `reach` expansion helps but doesn't fix the scale mismatch.

### Using wrong SV_PERSP / SV_TILT / SV_CY constants (guessed values)
**Why it failed:** SV_CY = CY-120 put the "ground" at y=270, but under the zoomed
camera the world origin maps to y≈488. The dots appeared 200px too high — above
the car instead of under it. SV_PERSP=0.040 was close but not exact. SV_TILT=0.45
was 4× too large for a nearly-horizontal camera.

### Filtering pts array by isometric screen bounds
**Why it failed:** Dots off-screen in isometric view are often on-screen in side-view.
With the old filter, the side-view projection had almost no dots to draw.

---

## What worked

### Dual-projection blend
Compute two full independent projections per dot, then lerp between them by `ze`:
- **Isometric** (ze=0): standard A/D formula driven by mouse
- **Side-view** (ze=1): forward-facing perspective using gz as depth axis

This gives a geometrically correct transition between two real projections,
not a distorted interpolation within one projection.

### Decomposing A/D back to (gx, gz)
The dot grid stores A = gx-gz and D = gx+gz (diagonal grid coords). For the
side-view, you need the world axes:
```
gz = (D - A) / 2   ← world z, depth axis when camera is at +z
gx = (D + A) / 2   ← world x, horizontal axis
```

### Deriving constants from camera math
Don't guess SV_PERSP, SV_GS, SV_CY. Derive them from the Three.js camera:

1. **K** (world units per grid unit) from isometric formula match:
   `K = GS × 2 × BASE_DIST × cos(BASE_EL) / (0.707 × f × H) = GS × 12.69 / H`

2. **SV_GS** = K × f × H / (2 × ZOOM_DIST × cos(ZOOM_EL)) = **70.3** (constant)

3. **SV_PERSP** = K / (ZOOM_DIST × cos(ZOOM_EL)) = **35.1 / H**

4. **SV_CY** from where gz=0 maps under the zoomed camera with lookAt.y=0.5:
   `SV_CY = (1 + f × lookAtY / ZOOM_DIST) × H/2 = H × 0.6255`

5. **SV_TILT** ≈ 0.10 (matched to Three.js projection numerically)

### Removing the pts screen-space filter
`build()` now includes all dots within reach (no isometric screen filter).
The draw loop's per-frame bounds check handles culling:
```js
if (sx < -60 || sx > W + 60 || sy < -60 || sy > H + 60) continue;
```
Increased reach from `+8` to `+16` to cover side-view extremes.

### Puppeteer for visual iteration
Instead of manually clicking and eyeballing, use a Puppeteer script to capture
isometric, mid-transition, and zoomed screenshots. Read them with the Read tool.
Makes tuning constants an order of magnitude faster. See zoom-animation.md.

---

## Key insight

The side-view projection constants (SV_GS, SV_PERSP, SV_CY) must match the
Three.js camera exactly. They're derived from the same camera math:
focal length, distance, elevation, lookAt position. If any Three.js camera
constant changes (ZOOM_DIST, ZOOM_EL, lookAt.y), recalculate all three.

The formulas are in [zoom-animation.md](../features/zoom-animation.md).
