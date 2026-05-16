# Lesson: Matching a Three.js Camera to the Dot Grid

## What we were trying to do
Make a Three.js GLB model look like it sits on the same ground plane as the dot grid,
with camera movement perfectly in sync — both at rest and during zoom animations.

---

## What didn't work

### PerspectiveCamera with FOV=35°
**Why it failed:** The dot grid uses PERSP=0.011 — almost no perspective distortion.
A 35° FOV camera has much stronger perspective, so the model's edges foreshorten
differently than the grid lines.
**Fix:** Switch to OrthographicCamera (or PerspectiveCamera with low FOV like 28°).

### OrthographicCamera for zoom animation
**Why it failed:** Dollying an ortho camera just changes frustum size — there's no
perspective depth change. The "zooming into the car" feel requires a perspective camera.
**Fix:** PerspectiveCamera(28°). The dot grid handles the isometric appearance at rest;
Three.js doesn't need to be strictly orthographic.

### Elevation set to 34° (0.60 rad) by feel
**Why it failed:** The dot grid has a geometric constraint: it compresses vertical depth
by exactly 0.5 (the `0.5 * D` term). This maps to sin(φ) = 0.5 → φ = 30° exactly.
Using 34° created a visible shear — the model's ground contact didn't align with the grid.
**Fix:** BASE_EL = Math.PI / 6 (exactly 30°).

### hSkew multiplier as positive
**Why it failed:** Mouse right → dots shift right, model rotates left. Jarring.
**Fix:** Negate: `BASE_AZ - c.hSkew * factor`.

### vShift in lookAt as negative (`c.vShift * -0.01`)
**Why it failed:** Mouse down → model drifts UP. Exact opposite.
**Fix:** Positive: `c.vShift * 0.01`.

### Object.assign to set DirectionalLight position
**Why it failed:** `Object.assign` replaces the position property reference entirely.
Three.js internals hold the original Vector3 reference, breaking the light.
**Fix:** Always use `.position.set(x, y, z)` on lights and Object3D instances.

### Blending isometric camera params toward side-view targets
**Why it failed:** Blending hSkew/tilt/vShift doesn't produce a front-facing ground
perspective. The dots tilt and distort rather than transitioning to a real side view.
**Fix:** Full dual-projection blend. See dot-grid-reprojection.md.

---

## What worked

- PerspectiveCamera(28°) + BASE_EL = 30° (π/6): ground plane alignment is good
- `window._dotCam` shared object polled each frame: zero-latency, no events needed
- ShadowMaterial ground plane: shadow works immediately, no tuning needed
- Two-pass Box3 (scale first, then recompute for ground placement): model sits flush
- `model.rotation.y = Math.PI`: correct facing on first try once camera direction known
- Dual-projection blend with geometrically derived constants: dots move with car perfectly

---

## Tuning process for movement factors

Started aggressive, dialed down by feel:

| Parameter    | Start | Final | Notes                        |
|-------------|-------|-------|------------------------------|
| hSkew factor | 1.4   | 0.30  | 1.4 was ~5× too much         |
| tilt factor  | 0.5   | 0.11  | 0.5 was ~4× too much         |
| vShift factor | 0.012 | 0.01 | minor, sign was the real issue |

The dot grid's camera movement is subtle by design. Three.js camera must match
that subtlety — large angular changes look completely wrong.
