# Feature: Three.js 3D Scene

**File:** `index.html` — `<script type="module">` block after the main script  
**Status:** Working  
**Library:** Three.js r165 via importmap (CDN)

---

## What it does

A transparent WebGL canvas overlaid on the dot grid. Loads a GLB model, preserves
the original Meshy AI materials (not replaced), casts a shadow onto a transparent
ground plane, and tracks the dot grid's camera state every frame so the model
appears to sit on the same ground. Supports click-to-zoom into a side-profile view.

---

## Setup

```html
<script type="importmap">
{
  "imports": {
    "three":         "https://cdn.jsdelivr.net/npm/three@0.165.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.165.0/examples/jsm/"
  }
}
</script>
<script type="module">
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
  import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
  import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
  import { OutlinePass } from 'three/addons/postprocessing/OutlinePass.js';
```

The renderer canvas is injected into `.card` via JS (not hardcoded in HTML):
```js
cv.style.cssText = 'position:absolute;inset:0;width:100%;height:100%;z-index:3;pointer-events:none;border-radius:14px;display:block;';
card.appendChild(cv);
```

**IMPORTANT:** Must be served over HTTP (not `file://`). GLTFLoader uses fetch,
which is blocked by the browser on file:// protocol.
Run: `npx serve` from the project directory. Port 3000 by default.

---

## Camera — PerspectiveCamera

Uses PerspectiveCamera(28°) — NOT OrthographicCamera. The ortho camera was correct
for static isometric alignment, but the zoom animation (dollying in) requires perspective
to create the depth-of-field feel. The dot grid handles the isometric appearance.

```js
const camera = new THREE.PerspectiveCamera(28, aspect, 0.1, 200);
```

**Why 30° elevation at rest:**
The dot grid projects vertical depth as `0.5 * D` (half the horizontal weight).
For a camera at azimuth 45°: `sin(φ) = 0.5` → `φ = 30°` exactly. This is BASE_EL.

---

## Camera constants

```js
const BASE_AZ   = Math.PI / 4;   // 45° azimuth — northeast, isometric diagonal
const BASE_EL   = Math.PI / 6;   // 30° elevation — geometric match to dot grid
const BASE_DIST = 18;            // resting distance

const ZOOM_AZ   = 0;             // side view — camera at +z, looking along -z
const ZOOM_EL   = 0.10;          // nearly horizontal (~6°)
const ZOOM_DIST = 8.0;           // zoomed in distance
```

---

## Camera tracking (tuned values, rest/isometric)

```js
const az   = BASE_AZ - c.hSkew * 0.30 * (1 - ease);
const el   = BASE_EL - (1.0 - c.tilt) * 0.11 * (1 - ease);
const dist = BASE_DIST + (ZOOM_DIST - BASE_DIST) * ease;

camera.position.set(
  Math.cos(el) * Math.sin(az) * dist,
  Math.sin(el) * dist,
  Math.cos(el) * Math.cos(az) * dist
);
camera.lookAt(0, c.vShift * 0.01 * (1 - ease) + 0.5 * ease, 0);
```

**Sign rules (hard-won):**
- `hSkew` multiplier must be **negative** — positive caused inverse horizontal rotation
- `vShift` in lookAt must be **positive** — negative caused inverse vertical drift
- `tilt` formula: `BASE_EL - (1.0 - tilt)` — mouse down raises elevation correctly
- Mouse factors fade out as `ease → 1` so zoom locks into clean side profile

---

## Zoom animation

```js
let zoomT   = 0;     // 0 = isometric, 1 = zoomed side view
let zoomDir = 0;     // +1 = zooming in, -1 = zooming out

// In RAF loop:
zoomT = clamp(zoomT + zoomDir * dt * 1.4, 0, 1);  // 0.7s transition
const ease = zoomT * zoomT * (3 - 2 * zoomT);       // smoothstep

// Share with dot grid:
if (window._dotCam) {
  window._dotCam.zoomEase  = ease;
  window._dotCam.zoomScale = BASE_DIST / dist;
}
```

Click detection: hit the model → zoom in; already zoomed → zoom out.
Hover outline disabled during zoom (gated on `ease < 0.05`).

---

## Hover outline (OutlinePass — two layers)

```js
// Layer 1: dark grey outer
const outlineGrey = new OutlinePass(size, scene, camera);
outlineGrey.edgeStrength  = 3.5;
outlineGrey.edgeThickness = 2.0;
outlineGrey.edgeGlow      = 0.0;
outlineGrey.visibleEdgeColor.set('#4a4844');

// Layer 2: white inner (thinner)
const outlinePass = new OutlinePass(size, scene, camera);
outlinePass.edgeStrength  = 2.0;
outlinePass.edgeThickness = 0.4;
outlinePass.edgeGlow      = 0.0;
outlinePass.visibleEdgeColor.set('#ffffff');
```

Composer order: RenderPass → outlineGrey → outlinePass.

---

## Shadow

Transparent `ShadowMaterial` plane at y=0 — invisible except where the model casts shadow.

```js
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(30, 30),
  new THREE.ShadowMaterial({ opacity: 0.18 })
);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
```

Shadow map: 1024×1024, `PCFSoftShadowMap`. Sun position: `(-4, 10, 5)`.

---

## Materials

Original Meshy AI materials are PRESERVED (not replaced with SVG palette).
The Meshy materials give a detailed dark look that contrasts well with the light bg.

```js
// Do NOT do this — it kills the Meshy look:
// child.material = new THREE.MeshStandardMaterial({ color: 0xD4D2CC, ... });
```

---

## Model loading

```js
new GLTFLoader().load('renders/Meshy_AI_Open_Wheel_Prototype__0515194205_generate.glb', gltf => {
  const model = gltf.scene;
  model.rotation.y = Math.PI;   // face toward viewer (camera is NE in isometric)

  // Normalise to 3 world units on longest axis
  const box = new THREE.Box3().setFromObject(model);
  const sz  = box.getSize(new THREE.Vector3());
  model.scale.setScalar(3.0 / Math.max(sz.x, sz.y, sz.z));

  // Sit flush on ground (two-pass — must rescale before re-centering)
  const box2 = new THREE.Box3().setFromObject(model);
  const ctr  = box2.getCenter(new THREE.Vector3());
  model.position.set(-ctr.x, -box2.min.y, -ctr.z);

  threeScene.add(model);
  model.traverse(child => { if (child.isMesh) modelMeshes.push(child); });
});
```

**Why two Box3 calls:** scale must be applied before recomputing bounds for ground placement.
**Why `rotation.y = Math.PI`:** camera is NE of origin; model's default forward is +z
(NW from camera), so it faces away. 180° flip corrects this.

---

## Side objects — Rivian R1S + scooter

Two additional GLBs flank the race car: `renders/rivian-r1s.glb` (right) and
`renders/scooter.glb` (left). Both clone materials with `transparent = true`.

**Rivian ground glue — FINAL mechanic (July 2026), the requirement it must
satisfy: "wheels shouldn't move away from a dot."**

The dot field is NOT static under the mouse: dots shear by `hSkew·D` and move
vertically with `tilt`/`vShift`, proportional to grid depth `D`. The race car
feels grounded for free because it sits at `D≈0` where dots barely move. The
Rivian sits at `D≈−9` where dots swing ~63px — AND, critically, the field is
NON-RIGID across its footprint: dots under the front wheel move up to ~30px
differently than under the rear. No rigid placement can keep all four wheels
on dots. The resolution:

- **Structure:** the model is recentred inside a pivot `Group` whose origin is
  the ground point under the visual centre (the GLB origin is arbitrary).
- **Calibration (load time):** for each of the 4 wheel ground points, project
  through a rest-pose camera (hSkew −0.09 → az `BASE_AZ + 0.027`, tilt 1,
  vShift 0) to px, then invert the dot projection at rest
  (`window._dotCam.invertGroundRest`) to get a lattice anchor `(A, D)` per
  wheel. Resolution-independent → resize-safe.
- **Per frame** (gated `ease < 0.4`; the SUV fades out during zoom): get each
  anchor's live px from `window._dotCam.projectGround` (includes the zoom
  blend), raycast px → world onto the y=0 plane, then solve a weighted 2D
  Procrustes (translation + yaw + uniform scale, XZ as complex numbers) mapping
  the wheels' rest local offsets onto those world targets. Apply to the pivot.
- **Weights are the trick:** the camera only sees the +z side of the car, so
  the two visible wheels have weight 1 and the two occluded wheels weight 0 —
  two points determine the similarity EXACTLY, so the visible contact points
  are glued to their dots with 0.00px error at every cursor position. The
  hidden wheels absorb the field's non-rigidity (up to ~33px — invisible).
- **Yaw is solved separately with the vertical state pinned to rest** (tilt 1,
  vShift 0, live hSkew) through a frozen rest-pose camera (`rivianRestCam`).
  The full fit's yaw picks up the field's vertical squash, making the SUV
  rotate on up/down cursor movement — user wanted rotation only on horizontal.
  Verified: yaw identical for top vs bottom cursor at the same x, regardless
  of y. `RIVIAN_YAW_DAMP = 0.18` scales it (user-tuned; full ground yaw is
  ~5° at the cursor edge, damped to ~0.9°).
- **The uniform scale is load-bearing but damped:** the local dot spacing
  genuinely stretches ±10% with the cursor; the exact fit scaled the SUV
  0.96–1.14, which the user read as "enlarges too much". `RIVIAN_SCALE_DAMP =
  0.2` keeps 20% of the fitted scale (range 0.99–1.03) at the cost of ≤5.6px
  visible-wheel drift at cursor extremes (sub-3px in most positions; dot
  spacing ~20px, so still visually on the dot). 0 = rigid size, 1 = exact glue.
- The geometry is fully derived from the dot projection; the only feel knobs
  are `RIVIAN_SCALE_DAMP` (0.2) and `RIVIAN_YAW_DAMP` (0.18), both user-tuned
  fractions of the derived values.

Verification: `window._rivianCheck()` returns per-wheel `{dot, wheel}` px pairs
(and `window._rivianCheck.scale`). Visible-wheel deltas must be ~0. Proven
0.00px across a 9-position cursor sweep + zoom round-trip (July 2026).

**Failed approaches — do not resurrect:**
1. *Screen-NDC lock* (freeze screen position): blows up during zoom, and dots
   shear beneath a screen-frozen object → reads as hovering.
2. *Scooter-style co-rotation* (orbit position + counter-rotate with az): also
   screen-fixes the object. Fine for the scooter (small `|D|`, tiny footprint),
   obvious slide at the Rivian's depth.
3. *Single-anchor lockstep + tuned parallax/yaw knobs*: pins one point, but the
   field's shear makes the wheels drift off their dots by up to 17px. A long
   knob-tuning detour (reversed/damped/zeroed parallax) all read as "moving too
   much" because ANY wheel-vs-dot slide is what the eye perceives as motion.
4. *Centre-line two-point pin without scale*: yaw follows the field but the
   ±10% local stretch has nowhere to go — wheels still drift ~10px.

**Zoom fade:** both side objects fade out as the zoom starts so the side view
features only the race car:

```js
const sideOpacity = Math.max(0, 1 - ease * 2.5);  // gone by ease ≈ 0.4
// applied to rivianMeshes + scooterMeshes; m.visible = sideOpacity > 0.01
```

**Rivian base position:** `(1.2 - ctr.x, -box2.min.y, -4.4 - ctr.z)`.
Screen-X in the isometric view is proportional to `x − z` — the original
`(2.0, −5.0)` gave `x−z = 7.0` which pushed it against / past the right card
edge on narrower windows. `(1.2, −4.4)` keeps depth (`x+z ≈ −3.2`) similar
while pulling it inward.
