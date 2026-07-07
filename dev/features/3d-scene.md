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

### Outline pass cost (July 2026 fix)

OutlinePass renders real geometry twice per pass per frame: a depth pre-pass of
every NON-selected mesh (for hidden-edge classification) plus a mask render of
the selected meshes. With these GLBs (car ~26 MB, Rivian ~48 MB) that dropped
the loop to ~20 fps whenever the car glow was up — which read as "the mouse
parallax freezes over the car". Two surgical patches in `index.html` (search
`slimOutlinePass`):

1. **slimOutlinePass(pass)** — all three passes set
   `hiddenEdgeColor === visibleEdgeColor`, so occlusion classification never
   changes a pixel. The wrapper hides every mesh except the pass's own
   selection during `render()`, emptying the depth pre-pass. Pixel-identical.
2. **Shared mask** — grey and white always outline the same selection (the
   same `modelMeshes` array is assigned to both), so the white pass skips its
   depth + mask renders entirely and reuses the grey pass's mask buffers,
   running only its screen-space edge/blur/overlay chain. Falls back to a
   normal render if the selection references ever diverge. Replicates the
   r165 OutlinePass.render() internals — re-check if three.js is upgraded.

## Hover/click hit proxies (July 2026 fix)

Never raycast the raw GLB meshes — `Raycaster.intersectObjects` tests every
triangle once the ray enters a mesh's bounding sphere, and the per-frame hover
raycast stalled the render loop to 10–20 fps exactly while the cursor was over
a model (10 fps over the Rivian). This froze the dot-grid parallax over models.

Instead each model group gets an invisible `BoxGeometry` proxy child sized to
its world AABB at load (`addHitProxy(group, list)` in index.html): the raycaster
still hits `visible: false` meshes but the renderer and outline passes skip
them, and as a child it rides every animation (zoom, glue, detach) for free.
All hover checks AND the click handler intersect the `*Hitbox` lists
(carHitbox, rivianHitbox, …); the `*Meshes` lists remain for outlines/fades.

Gotchas:
- The car/scooter split parts share one position attribute (only the index
  differs), so `Box3.setFromObject` on a part returns the WHOLE model's box.
  The seat-module proxy is therefore built from its own triangle indices in
  the scooter loader — do the same for any future index-split part.
- Add the proxy at the END of the loader, after final scale/position, and
  never `Box3.setFromObject` a group at runtime after its proxy exists (a
  rotated proxy inflates the AABB).
- Hit shape is the box, not the silhouette — hover/click trigger slightly
  outside the model edge. Cursor and click stay consistent by design.

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

**Trunk rear wall (July 2026):** `renders/Rivian-rear-wall.glb` — a gold
accent panel inside the open cargo bay. Loaded NESTED in the Rivian's load
callback (not top-level) so it can parent to the pivot and inherit the ground
glue + featured blend. Key details:

- The GLB ships **no materials** → Three.js applies the glTF default, which is
  **fully metallic** and renders the colour as muddy olive. Fix: `metalness =
  0`, `roughness = 0.85`, colour `0xFFB800` (same gold as the sprocket).
- Sized to 0.95 units wide, placed at pivot-local `(0, 0.78, 1.15)` — floats
  a touch above the trunk floor (~0.7), partway into the cargo bay (rear
  opening is at z ≈ +2.25). User-tuned via screenshot iteration.
- Opacity = `rivianOpacity * 0.35` per frame (separate `rearWallMeshes` array,
  NOT in `rivianMeshes` — it must not join the click/hover raycast at rest).
- **Shares the yellow-outline channel with the sprocket:** the idle glint and
  hover outline (`outlineYellow`) are gated by `featured` — sprocket owns the
  channel on the car, rear wall on the Rivian, so they never conflict. Hover
  is active only when zoomed on the Rivian (`ease > 0.9`); no pointer cursor
  because clicking it doesn't open a detail view (click = zoom out).

**Z-axis printer (July 2026):** `renders/z-axis-printer.glb`
sits deeper in the scene behind the scooter. Started as a background prop;
now Project 04 — clickable/hoverable and featured like the others (see the
featured-zoom section). Loader mirrors the scooter's (clone materials
with `transparent = true`, normalise to 1.8 units, sit on ground). Base
position `(-4.5, 0.3)`: screen-X ∝ x−z = −4.8 sits it right of the scooter
(−5.25), toward the car; x+z = −4.2 pushes it deeper than the Rivian (−3.2).
Final per-frame mechanic — the two axes are governed DIFFERENTLY, and the
asymmetry is the point (measured at this spot, full cursor sweep: local dots
move ~37px horizontally but only ~7px vertically):
- **Horizontal: dot glue, no damping.** A lattice anchor is calibrated under
  `PRINTER_POS` at load (rest-pose camera + `invertGroundRest`, same recipe
  as the Rivian's wheel probes); per frame the printer's NDC-x follows
  `projectGround(anchor)` exactly. Damping the sway instead was tried twice
  (world-static ~14px, then 85%-damped ~2.6px) and the user said "way too
  much" BOTH times — with the dots sweeping 37px, any screen-stilled object
  visibly skates across the field, and the eye reads that relative slide as
  the object moving (induced motion). Gluing raises the ABSOLUTE swing to
  ~45px but zeroes the slide at the ground contact, which is what actually
  reads as "less movement". Do not "fix" this by damping x again.
- **Vertical: damped against a rest camera.** The camera's vertical tracking
  (el + lookAt vShift) pivots on the origin; the printer sits ~3 units
  beyond it and was levered ~18px. Project `PRINTER_POS` through the live
  camera and through `printerRestCam` (el/lookAt pinned to vertical rest, az
  pinned to the −0.09 resting hSkew), keep `1 − PRINTER_BOB_DAMP` (0.92 →
  1.9px) of the NDC-y delta. Damping is fine here because the local dots
  barely move vertically (~7px), so no skate.
The combined NDC point is raycast back onto y=0 for the new ground position.

**Print-action rig (July 2026):** the GLB is ONE fused mesh — no separate
carriage/belt nodes, no animation clips — so the "actively printing" effect
is a procedural overlay built at load in the printer loader:
- **The model is kept fully INTACT — do not cut the mesh.** A long
  iteration tried removing the A-frame gantry from the fused mesh
  (region-based triangle removal; it leans the wrong way for a belt
  printer) to replace it with procedural rails. Every variant — full-deck
  cut, surgical z-bands with a flywheel exception — read as broken or
  over-removed to the user, who finally asked to "maintain the entire
  model". The A-frame stays and serves as the visual gantry.
- **Belt surface:** striped CanvasTexture on a thin box, running the whole
  bed up to the drive machinery (x −0.935..0.52 — the housings at x 0.6+
  sit above belt height, so it cannot extend further); `texture.offset.x`
  scrolls with the belt speed.
- **Cross beam (X axis):** a z-spanning bar with truck blocks that rides
  UP the model's own A-frame front legs — the driver keeps it on the
  legs' front-face line (`x = 0.16 + (y + 0.15) / 1.47`, measured from
  the mesh) at the head's height. No procedural rails.
- **Occlusion — `workZ = −0.25`:** at the featured yaw (PI − 0.4), local
  −z faces the viewer, and at the belt's centreline the A-frame's near
  leg hides the head entirely. The whole print line (gears, head sweep
  centre, fall, hole) is pulled to z −0.25 so the head works in the
  frame's visible window. (First attempt moved it to +0.05 — wrong
  direction, +z faces AWAY at this yaw.)
- **45° print head:** carriage plate → finned heatsink (5 disc fins) →
  heatbreak → heater block → nozzle cone, all in hardware grey 0x5a5a57
  matched to the model body (user vetoed the earlier darker shade),
  `rotation.z = −PI/4` (belt printers print onto a 45° plane). Tip is at
  head-local (0, −0.165) → lean offset (−0.117, −0.117). The head RIDES the
  plane layer by layer: each frame the driver finds the print front — the
  top of the 45° slice where the plane meets the emerging gear (`min(yTop,
  C − leadingEdge)`) — quantises it to `LH = 0.011` layer steps, smooths
  (exp, ~0.3s), and places the tip on the plane at that height
  (`x = C − y`), so the head climbs as the gear builds. The z sweep is a
  constant-speed zigzag (0.9s round trip) whose amplitude is the CHORD
  WIDTH of the slice the plane is currently cutting through the gear
  (`hw = sqrt(R² − dx²)` with the chord x clamped into the plane∩gear
  range) — the nozzle only travels over material it is actually
  extruding. **Stall behaviour (user request):** when NO gear intersects
  the plane, `printing` (smoothed 0..1) eases to 0 and the head glides to
  a home pose — `PARK_Z = −0.32` beside the near leg, tip lifted to
  `yBot + 0.05`, zigzag and y jitter blended out — then eases back into
  the sweep when the next gear arrives. It must NOT pantomime printing
  over an empty belt. NO melt bead — the user vetoed the glowing dot.
- **The print plane — the piece that makes growth read as printing:** a
  45° `THREE.Plane` through the gantry foot (local x + y = PRINT_PX +
  PRINT_PY, normal (−1,−1,0)) assigned as `clippingPlanes` on the part
  materials (`renderer.localClippingEnabled = true`, `clipShadows: true`,
  `side: DoubleSide` so the cut face isn't hollow). Parts are born fully
  behind it (X0 = 0.30) and material genuinely APPEARS at the nozzle line
  as the belt carries them out — no scale-pop. Clipping planes live in
  WORLD space, so it is re-derived from `printerGroup.matrixWorld` every
  frame (the group moves constantly under the dot glue).
- **Parts are small sprockets** (extruded 9-tooth gear Shape with a hub
  hole, 0xB57F27 — echoes the car's sprocket detail and the Rivian accent),
  flat on the belt, fresh yaw each lap.
- **End-of-belt exit:** past `X_EDGE` (−0.945) a part tips forward
  (rotation.z ramps), tumbles off with a slight forward toss (deterministic
  in printT — pause/resume safe), and SINKS through a second clipping
  plane at ground level (`groundPlane`, refreshed per frame like the print
  plane; default clip-union means either plane clips) into a dark
  `CircleGeometry` "hole" in the floor at local (−1.0, ground, −0.09).
  Parts recycle only while hidden below ground (SPAN 1.36 leaves ~2s
  buried) — no pop-out-of-existence.
Group-local geometry measured from the mesh (see `window._printer` debug
handle): belt frame x −0.95..0, rung tops y ≈ −0.15, gantry x 0.1..0.9
(apex y 0.38). The rig is a CHILD of the printer group, so ground glue,
featured glide, scaling, and the fade loop (rig meshes are pushed into
`printerMeshes` — materials must be `transparent: true`) all apply for
free. Driver (RAF): `printOn` eases toward 1 only while `featured ===
'printer'` (scaled by `ease`), and `printT += dt * printOn` — the animation
PAUSES on zoom-out and RESUMES where it was on the next visit (user
requirement). Speed knobs: `BELT_V` 0.045 u/s, `SPAN` 1.15, head sweep
2.4 rad/s.
**Ground slide only, never a world-y lift** — two earlier attempts:
(1) world-y counter-shift: exact damping but lifting off y=0 made the sun
shadow slide out from under its feet (user complaint); (2) analytic
slide-away-from-camera: shadow fixed, but motion toward the vanishing point
added ~5px of horizontal drift for this off-centre object. The raycast form
has neither problem. Gated `ease < 0.4` (faded out past there; zoom camera
makes the plane intersection unstable) and returns to base beyond. It fades
on the plain `fadeOut` curve during any featured zoom.

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

**Steering wheel (July 2026):** `renders/steering.glb` — Project 05, the
smallest prop in the scene (normalised to 1.2 units). Stands upright on its
rim edge at `STEERING_POS = (2.7, −1.7)`: `x−z = 4.4` puts it right of the
car, short of the Rivian's 5.6; `x+z = 1.0` keeps it shallow, so it uses the
SCOOTER's co-rotation mechanic (orbit position + counter-yaw with az — fine
for small `|D|`, tiny footprint; no ground glue needed). Base yaw
`PI/4 − 0.35` — toward the NE camera, a touch off dead-on. Like the printer
and scooter, the GLB ships **no materials** → keep the untouched glTF
default (white base, fully metallic) so it renders the exact same grey as
those props. An earlier `color 0x2B2A27` override made it read darker than
everything else and was removed.

The bottom-right of the four face buttons is split out of the fused mesh and
tinted green — it is the race-game easter egg's entry point (hover + click
channels mirror the sprocket/seat pattern). See
[race-game.md](race-game.md) for the split constants and the game itself.

**Scooter seat/cargo module (July 2026):** the real scooter's rear seat +
cargo rack is removable, so it is the scooter's clickable component (the
sprocket pattern): zoomed on the scooter, the module owns the yellow
glint/hover channel; clicking it detaches it (`seatOpen`), any click
re-seats it, and `_cycleFeatured`/`_zoomOutAll` force it closed. The GLB is
ONE fused mesh (positions only — flat-shaded default material), so the
module is split out at load by triangle-centroid region, two meshes sharing
the original position attribute with different indices (same trick as the
race car's wheel split):

- Region (mesh-local: x = length, handlebars at −x; y = height):
  `(cx > 0.02 && cy > −0.56)` — rack over the rear wheel — OR
  `(cx > −0.12 && cy > CUT_Y && |cz| < 0.13)` — the seat column, a narrow-z
  tube descending to the deck. `CUT_Y = −0.61` cuts at the column base
  (user-tuned twice: first cut at deck-strut height −0.44 left a jagged
  sliver standing on the deck; −0.60 still read as "broken off"). The |cz|
  filter keeps the wide fender/wheel shell out; deck-top centroids are
  ≤ −0.63, safe by 0.02.
- **Clean cut:** the index split leaves ragged part-triangle fringes both
  sides of the plane. Two `BoxGeometry` plates sized to the column's
  cross-section (measured x −0.111..0.009, z ±0.05) hide them — flat end
  plate on the module, slightly LARGER socket collar on the deck (different
  sizes so nested faces never z-fight). Reads as a quick-release joint.
- **Pivot:** the seat mesh is recentred inside a group at the module's own
  centre so the detach tilt turns about the module. The centre is averaged
  from the module's triangles — `computeBoundingBox` is USELESS here, it
  reads the full shared position attribute (both split meshes report the
  whole scooter's bbox; this also means load-time normalise/ground code is
  unaffected by the split).
- **Detach animation** (RAF, `seatOpenT` → smoothstep, 0.7s out / 0.5s
  back): lift leads (`rise = st^0.6 * 0.5`), rearward +x drift arrives late
  (`st² * 0.5`) — an unhook arc — swing `rotation.z = st*0.10 +
  sin(st·π)*0.06` mostly unwinds, scale settles at 1.08. While open the
  frame dims (`1 − seatOpenEase*0.72`, per-mesh via `userData.isSeat`), the
  project panel yields (ppIn multiplies `1 − max(openEase, seatOpenEase)
  *2.5`), and `transitioning` includes the seat ease so hover raycasts pause.
- **Gotcha:** the idle glint sets `outlineYellow.selectedObjects` on its own
  schedule — when the glint gate goes inactive (e.g. the module detaches
  mid-swell) the selection must be cleared in the else-branch or the outline
  sticks at full strength on the floating module.
- Debug handles: `window._feature(name)` zooms a project from rest without
  a click; `window._seatDetach(bool)` drives the detach directly.

---

## Featured-project zoom — all five models clickable

The nav hint promises "Click on a project to view more"; all five models now
deliver it. `featured` (`'car' | 'rivian' | 'scooter' | 'printer' |
'steering'`) is set by the click handler at rest; the SAME zoom animation
(`zoomT`/`ease`, camera dolly to the origin side-view, dot-grid reprojection)
then runs for any of them. Arrow cycling order: `FEATURE_ORDER = ['scooter',
'printer','car','steering','rivian']` — left-to-right by screen position
(modulo uses `FEATURE_ORDER.length` — don't hardcode the count).

- **The clicked model glides to the origin stage** as `ease` advances: its
  rest pose (Rivian and printer: last ground-glue solution, frozen past ease
  0.4; scooter: its az-orbit pose) is lerped to `FEATURE_POSE[name]` —
  position (0,0,0), a stage yaw, and a stage scale.
  The car needs no glide (it already lives at the origin).
- `FEATURE_POSE`: rivian `{scale: 0.75, yaw: -PI/2}` (4.5-unit model scaled to
  ~car size; -PI/2 because its length axis lies along world Z, rear at +z);
  scooter `{scale: 1.10, yaw: PI}` (scale multiplies its base normalisation;
  watch the y-position compensation `scooterBaseY * k` that keeps wheels on
  the ground when scaling about the group origin); printer
  `{scale: 1.45, yaw: PI - 0.4, el: 0.30}` (yaw ~23° off dead-side — user
  wanted a three-quarter view, not a pure profile; its stage position also
  carries the load-time recentring offset `(printerBase − PRINTER_POS) * k`
  so the VISUAL bbox centre — not the arbitrary GLB origin — lands on the
  stage); steering `{scale: 1.80, yaw: -0.55, el: 0.24}` (its face points +z
  at yaw 0 — dead-on to the zoom camera it reads as a flat black cutout, so
  it takes a three-quarter yaw plus a raised camera to show rim depth).
- **Per-project zoom elevation:** `FEATURE_POSE[..].el` overrides the shared
  `ZOOM_EL` (0.10) for that project's dolly — the printer zooms to a higher
  vantage (0.30, looking down onto the bed). Caveat: the dot grid's side-view
  constants (`SV_*`) are derived for ZOOM_EL = 0.10, so a raised el slightly
  mismatches the zoomed dot plane — checked visually at 0.30 and it reads
  fine (the dots are a stylised background band by then), but don't push el
  much higher without re-checking the dots/shadow relationship.
- **Fades:** non-featured side models fade with `1 - ease*2.5`. The car fades
  FASTER (`1 - ease*4`, applied directly — NOT through its smoothed hover-dim
  opacity, whose lag made the car linger while the camera dollied toward its
  own side view, reading as the car transitioning behind the fade). The
  sprocket gear fades on the car's curve (materials need `transparent = true`).
- **Gates:** sprocket click/hover only when `featured === 'car'`; rivian /
  scooter / printer / steering hover-cursor only at rest (`ease < 0.05`);
  car hover outline only when `carOpacity > 0.05` (raycaster hits invisible
  meshes otherwise).
- Zoom-out reverses everything; `featured` persists until the next click
  (harmless — at ease 0 all models are at their rest poses).
- The sheet note shows `Sheet NN/05` — bump the denominator when adding a
  sixth project.

---

## Project info panels + identity caption

Each featured zoom shows a right-side info panel (`#projectPanel`, styled
like the sprocket panel but 300px) with tag / title / bullets / image slot.
Copy lives in the `PROJECT_INFO` map (currently FILLER TEXT — replace with
real project stories). The panel is populated on click (`fillProjectPanel`)
and driven from the RAF loop:

- opacity/slide-in ramps over ease 0.6 → 1.0, and is multiplied by
  `(1 - openEase)` so the car's panel yields to the sprocket detail panel.
- `pointer-events: none` — clicks pass through it and zoom back out.

The identity caption (bottom-left `.identity-meta`) crossfades to the
featured project's `layer`/`desc` copy: its opacity is `|ease - 0.5| * 2`
(dips to 0 at mid-zoom, where the text is swapped) — one formula handles
both directions of the transition.

---

## UI layer — engineering spec-sheet language (July 2026)

The chrome leans into the raw-engineering-model aesthetic. `--mono` (IBM Plex
Mono) is the voice of all small technical labels; Instrument Sans stays for
headlines and body copy.

- **Spec-sheet panel:** corner tick marks (`.tick`, reused on the sprocket
  panel and card frame), mono tag + ghost index numeral header, ruled title,
  numbered rows (CSS counter `pp-row`), figure slot with diagonal hatch
  placeholder ("Fig. 01"; Rivian shows the real press photo), and a
  "REV A · 2026 / YM WORKS" meta footer. Rows stagger in via transition-delays
  when the RAF loop toggles `.in` at ppIn > 0.4.
- **Dimension line** (`#dimLine`): drawing-style dimension with end ticks and
  a mono label under the featured model; geometry per project in
  `PROJECT_INFO[..].dim` = {l, w, y (percent), label}. Draws outward from
  centre with ppIn.
- **Hover callout** (`#hoverTag`): mono tag ("02 · RIVIAN R1S STUDY") with a
  leader line that trails the cursor over any model at rest. Lerped position,
  snaps to the cursor when fully faded to avoid fly-in.
- **Scene hint** swaps to "CLICK ANYWHERE TO RETURN" when zoomed, and the
  **sheet note** (bottom centre, inside `.frame-marks`) reads
  "…· Index" at rest / "…· Sheet 02/03" when featured — both crossfade on the
  same `|ease − 0.5| × 2` dip as the identity caption.
- **Arrow cycling:** while zoomed, the section-nav arrows call
  `window._cycleFeatured(dir)` (module) — zoom out, swap feature at rest,
  zoom back in (`pendingFeature` bounce). Button handlers stopPropagation so
  the card's zoom-out click doesn't also fire; at rest they still cycle the
  page dots.
