# Lesson: GLB Asset Workflow

## Getting models

**Free AI generation (fastest):**
- Meshy.ai — text-to-3D, exports GLB directly, free tier sufficient
- Tripo3D — similar quality, alternative if Meshy is slow
- Quality is fine for stylised/geometric objects; don't expect photorealism

**Blender (most control):**
- Export: File → Export → glTF 2.0, format: GLB
- For this aesthetic: Orthographic camera, X=60° Z=45° rotation, roughness 1.0 material

---

## File placement

Put GLB files in `renders/`. Path used in loader: `'renders/filename.glb'`.

---

## Draco compression — REQUIRED before committing

Raw Meshy exports are huge (3-46 MB each; the original seven totalled
~131 MB and made the deployed site take forever to show any model).
Compress every GLB before it ships:

```
npx --package=@gltf-transform/cli gltf-transform draco in.glb out.glb
```

~13x smaller (the full set is ~10 MB). The module script uses ONE shared
`gltfLoader` with a `DRACOLoader` attached (decoder wasm from the same
three CDN) — new loads must use `gltfLoader.load(...)`, not
`new GLTFLoader().load(...)`.

Why plain `draco` and not `optimize`: Draco decodes back to float32
positions in the ORIGINAL local space, so the offline-measured mesh-split
constants (car wheel axles, scooter seat region, steering green button)
keep working. `optimize`/`gltfpack` quantize with a baked node transform,
which would break every one of those measurements. No simplification
either — the car's wheel-split seam relies on dense tessellation.

---

## Local server requirement

**GLTFLoader uses fetch(), which is blocked on file:// protocol.**
Always serve via HTTP:
```
npx serve        # from project root, auto-finds available port
```
Check which port it actually started on (it logs it). If 3000 is taken it'll pick another.

---

## Material replacement

Meshy models come with their own textures/materials. For the SVG palette look,
replace everything:

```js
model.traverse(child => {
  if (!child.isMesh) return;
  child.castShadow    = true;
  child.receiveShadow = true;
  child.material = new THREE.MeshStandardMaterial({
    color: 0xD4D2CC,
    roughness: 1.0,
    metalness: 0.0,
  });
});
```

---

## Facing direction

Camera sits northeast of origin (BASE_AZ = 45°). Most models default to facing +z
(which is northwest from the camera — facing away).
**Always add `model.rotation.y = Math.PI` after loading.**

---

## Sizing

Normalise to 3 world units on the longest axis. At ORTHO_SIZE=4.5 this fills
roughly 1/3 of the card height — visible but not overwhelming.
Adjust ORTHO_SIZE to control apparent size (larger = smaller model on screen).
