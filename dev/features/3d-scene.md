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
