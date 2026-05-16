# Lesson: Matching a Three.js Camera to the Dot Grid

## What we were trying to do
Make a Three.js GLB model look like it sits on the same ground plane as the dot grid,
with camera movement perfectly in sync.

---

## What didn't work

### PerspectiveCamera with FOV=35°
**Why it failed:** The dot grid uses PERSP=0.011 — almost no perspective distortion.
A 35° FOV camera has much stronger perspective, so the model's edges foreshorten
differently than the grid lines. The mismatch is obvious on the far edges of the model.
**Fix:** Switch to OrthographicCamera.

### PerspectiveCamera with small FOV (telephoto approximation)
**Why we abandoned it:** Orthographic is cleaner and mathematically exact. No reason
to approximate when Three.js has the right camera type.

### Elevation set to 34° (0.60 rad) by feel
**Why it failed:** The dot grid has a geometric constraint: it compresses vertical depth
by exactly 0.5 (the `0.5 * D` term). This maps to sin(φ) = 0.5 → φ = 30° exactly.
Using 34° created a visible shear — the model's ground contact didn't align with the grid.
**Fix:** BASE_EL = Math.PI / 6 (exactly 30°).

### hSkew multiplier as positive
**Why it failed:** Positive multiplier made the model rotate opposite to the dot grid.
Mouse right → dots shift right, model rotates left. Visually jarring.
**Fix:** Negate the hSkew multiplier: `BASE_AZ - c.hSkew * factor`.

### vShift in lookAt as negative (`c.vShift * -0.01`)
**Why it failed:** Mouse down → vShift positive → lookAt target below origin → camera
tilts down → model drifts UP. Exact opposite of what the dot grid does (which shifts
its entire draw origin downward).
**Fix:** Positive sign: `c.vShift * 0.01`.

### Object.assign to set DirectionalLight position
**Why it failed:** `Object.assign` replaces the position property reference entirely.
Three.js internals may hold the original Vector3 reference, breaking the light.
**Fix:** Always use `.position.set(x, y, z)` on lights and Object3D instances.

---

## What worked

- OrthographicCamera + BASE_EL = 30° (π/6): ground plane alignment is exact
- `window._dotCam` shared object polled each frame: simple, zero-latency, no events needed
- ShadowMaterial ground plane: shadow looks great immediately, no tuning needed
- Two-pass Box3 (scale first, then recompute for ground placement): model sits flush
- `model.rotation.y = Math.PI`: correct facing on first try once we knew camera was NE

---

## Tuning process for movement factors

Started aggressive, dialed down by feel:

| Parameter | Start | Final | Notes |
|-----------|-------|-------|-------|
| hSkew factor | 1.4 | 0.47 | 1.4 was ~3× too much |
| tilt factor  | 0.5  | 0.11 | 0.5 was ~4× too much |
| vShift factor | 0.012 | 0.01 | minor, sign was the real issue |

The dot grid's camera movement is subtle by design. Three.js camera must match
that subtlety — large angular changes look completely wrong.
