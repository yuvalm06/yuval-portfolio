# Feature: Three.js 3D Scene

**File:** `index.html` — `<script type="module">` block after the main script  
**Status:** Working  
**Library:** Three.js r165 via importmap (CDN)

---

## What it does

A transparent WebGL canvas overlaid on the dot grid. Loads a GLB model, applies
SVG-palette matte materials, casts a shadow onto a transparent ground plane, and
tracks the dot grid's camera state every frame so the model appears to sit on the
same ground.

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
```

The renderer canvas is injected into `.card` via JS (not hardcoded in HTML):
```js
cv.style.cssText = 'position:absolute;inset:0;width:100%;height:100%;z-index:3;pointer-events:none;border-radius:14px;display:block;';
card.appendChild(cv);
```

**IMPORTANT:** Must be served over HTTP (not `file://`). GLTFLoader uses fetch,
which is blocked by the browser on file:// protocol.
Run: `npx serve` from the project directory.

---

## Camera — OrthographicCamera

Uses OrthographicCamera, not PerspectiveCamera. The dot grid is nearly orthographic
(PERSP = 0.011), so perspective cameras create visible mismatch.

**Why 30° elevation:**
The dot grid projects vertical depth as `0.5 * D` (half the horizontal weight).
For an orthographic camera at azimuth 45°: `screen_y / screen_x = sin(φ)`.
Setting `sin(φ) = 0.5` → `φ = 30°` exactly matches.

```js
const ORTHO_SIZE = 4.5;  // world units in half-height — controls model size
```

Resize handler updates frustum:
```js
camera.left   = -ORTHO_SIZE * aspect;
camera.right  =  ORTHO_SIZE * aspect;
camera.top    =  ORTHO_SIZE;
camera.bottom = -ORTHO_SIZE;
camera.updateProjectionMatrix();
```

---

## Camera tracking (tuned values)

```js
const BASE_AZ  = Math.PI / 4;   // 45° azimuth — northeast, matches isometric diagonal
const BASE_EL  = Math.PI / 6;   // 30° elevation — exact geometric match to dot grid
const CAM_DIST = 18;

const az = BASE_AZ - c.hSkew * 0.47;       // hSkew sign: negative = correct direction
const el = BASE_EL - (1.0 - c.tilt) * 0.11;

camera.position.set(
  Math.cos(el) * Math.sin(az) * CAM_DIST,
  Math.sin(el) * CAM_DIST,
  Math.cos(el) * Math.cos(az) * CAM_DIST
);
camera.lookAt(0, c.vShift * 0.01, 0);      // vShift sign: positive = correct direction
```

**Sign rules (hard-won):**
- `hSkew` multiplier must be **negative** — positive caused inverse horizontal rotation
- `vShift` in lookAt must be **positive** — negative caused inverse vertical drift
- `tilt` formula: `BASE_EL - (1.0 - tilt)` — mouse down raises elevation, model goes down ✓

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
Shadow camera frustum: ±6 units (covers the normalised model size of 3 units).

---

## Materials

All meshes get a single flat material matching the SVG palette:

```js
new THREE.MeshStandardMaterial({
  color: 0xD4D2CC,   // mid-tone between SVG's near (#D8D6D0) and far (#BCBAB4) faces
  roughness: 1.0,    // fully matte — no specular highlights
  metalness: 0.0,
})
```

Lighting creates face differentiation naturally:
- Ambient `0xEEECEA` intensity 1.6 — fills everything with card base tone
- Key light `(-4, 10, 5)` intensity 0.65 — top/left faces lighter
- Fill light `(5, 3, -2)` intensity 0.22 — prevents fully dark right faces

---

## Model loading

```js
new GLTFLoader().load('path/to/model.glb', gltf => {
  const model = gltf.scene;
  model.rotation.y = Math.PI;   // face toward viewer (camera is southeast)

  // Normalise to 3 world units on longest axis
  const box = new THREE.Box3().setFromObject(model);
  const sz  = box.getSize(new THREE.Vector3());
  model.scale.setScalar(3.0 / Math.max(sz.x, sz.y, sz.z));

  // Sit flush on ground
  const box2 = new THREE.Box3().setFromObject(model);
  const ctr  = box2.getCenter(new THREE.Vector3());
  model.position.set(-ctr.x, -box2.min.y, -ctr.z);

  threeScene.add(model);
}, undefined, err => console.error('[3D] GLB load error:', err));
```

**Why two Box3 calls:** scale must be applied before recomputing bounds for ground placement.
**Why `rotation.y = Math.PI`:** camera is northeast of origin; model's default forward
is toward +z (northwest from camera), so it faces away. 180° flip corrects this.
