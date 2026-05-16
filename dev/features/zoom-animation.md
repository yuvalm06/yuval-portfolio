# Feature: Click-to-Zoom Side-Profile Animation

**File:** `index.html` — Three.js module script + dot grid IIFE  
**Status:** Working  
**Branch:** `experiment/outline-glow` (active)

---

## What it does

Clicking the 3D car triggers a smooth camera animation that transitions from the
isometric bird's-eye view into a ground-level side profile of the car. The dot grid
ground plane reprojects simultaneously so the entire scene — 3D model AND dots —
moves together as one unified space.

---

## Three.js camera animation

Uses a PerspectiveCamera (FOV=28°). On click, `zoomDir` is set to +1 or -1 and
`zoomT` advances each frame at 1.4×/sec (≈0.7s full transition).

```js
const BASE_AZ   = Math.PI / 4;   // 45° — isometric northeast
const BASE_EL   = Math.PI / 6;   // 30° — exact geometric match to dot grid
const BASE_DIST = 18;

const ZOOM_AZ   = 0;             // side view — camera at +z, looking along -z
const ZOOM_EL   = 0.10;          // nearly horizontal (≈6°)
const ZOOM_DIST = 8.0;           // dollied in close

zoomT = clamp(zoomT + zoomDir * dt * 1.4, 0, 1);
const ease = smoothstep(zoomT);   // zoomT² × (3 - 2×zoomT)

const az   = lerp(BASE_AZ,   ZOOM_AZ,   ease) - c.hSkew * 0.30 * (1-ease);
const el   = lerp(BASE_EL,   ZOOM_EL,   ease) - (1-c.tilt) * 0.11 * (1-ease);
const dist = lerp(BASE_DIST, ZOOM_DIST, ease);
camera.lookAt(0, c.vShift * 0.01 * (1-ease) + 0.5 * ease, 0);
```

The mouse-driven movement (hSkew, tilt, vShift) fades out as ease approaches 1,
so the zoom locks into a clean side profile regardless of cursor position.

---

## Sharing zoom state with the dot grid

Three.js writes to `window._dotCam` each frame:

```js
if (window._dotCam) {
  window._dotCam.zoomEase  = ease;
  window._dotCam.zoomScale = BASE_DIST / dist;  // not used for dots anymore
}
```

The dot grid polls this every frame (zero-latency, no events).

---

## Dot grid dual-projection

The dot grid blends between two complete projections using `ze = zoomEase`.

**Isometric (ze=0):** standard A/D projection driven by mouse.
**Side-view (ze=1):** forward-facing perspective from camera at +z looking along -z.

```js
// Isometric
const sx_i = CX + GS * (A + hSkew * D) / denom_i;
const sy_i = drawCY + GS * 0.5 * tilt * D / denom_i;

// Side-view
const gz = (D - A) * 0.5;   // world z — depth axis from side camera
const gx = (D + A) * 0.5;   // world x — horizontal axis
const SV_GS    = 70.3;
const SV_PERSP = 35.1 / H;  // varies with card height
const SV_TILT  = 0.10;
const SV_CY    = H * 0.6255;

sx_s = CX + SV_GS * gx / (1 - SV_PERSP * gz);
sy_s = SV_CY + SV_GS * SV_TILT * gz / (1 - SV_PERSP * gz);

// Blend
sx = sx_i + (sx_s - sx_i) * ze;
sy = sy_i + (sy_s - sy_i) * ze;
```

**Why gz = (D-A)/2:** The dot grid's diagonal axes are A = gx-gz, D = gx+gz in grid
coords. Decomposing back to (gx, gz) gives the world axes. gz is the actual depth
axis for a camera sitting at +z. gx is horizontal (car's length direction on screen).

---

## Side-view constants — how they were derived

The constants must match the Three.js camera exactly. Derivation:

**K** = world units per grid unit. Match GS in the isometric formula:
```
GS = 0.707 × K × f × H / (2 × BASE_DIST × cos(BASE_EL))
→ K = GS × 2 × BASE_DIST × cos(30°) / (0.707 × f × H)
→ K = GS × 12.69 / H    (with f = 1/tan(14°) = 4.011)
```

**SV_GS** (pixel scale in side view):
```
SV_GS = K × f × H / (2 × ZOOM_DIST × cos(ZOOM_EL))
      = (GS × 12.69/H) × 4.011 × H / (2 × 8 × cos(0.10))
      = 70.3   ← constant, H cancels out
```

**SV_PERSP** (perspective strength):
```
SV_PERSP = K / (ZOOM_DIST × cos(ZOOM_EL))
         = (GS × 12.69/H) / 7.96
         = 35.1 / H   ← scales with card height
```

**SV_CY** (where gz=0 maps on screen):
```
NDC_Y_at_origin = −f × lookAtY / ZOOM_DIST = −4.011 × 0.5 / 8 = −0.2509
SV_CY = (1 + 0.2509) × H/2 = H × 0.6255
```

**SV_TILT** (vertical factor, nearly horizontal camera):
```
SV_TILT ≈ 0.10   (derived by matching gz=5 screen Y to Three.js projection)
```

These values are specific to: FOV=28, ZOOM_DIST=8, ZOOM_EL=0.10, BASE_DIST=18,
BASE_EL=π/6, BASE_AZ=π/4, lookAt.y=0.5 at full zoom.
If any of those change, re-derive using the same method.

---

## pts array — build() filter

The dot grid pre-builds a `pts` array on resize. Originally it filtered dots by
their isometric screen position, which excluded many dots visible in side-view.

**Fix:** Removed the screen-space filter from `build()`. All dots within `reach`
are included. The draw loop handles off-screen culling each frame.

```js
const reach = Math.ceil(Math.max(W, H) / GS) + 16;  // +16 for side-view coverage
// No screen-space filter here — draw loop handles it
pts.push({ A, D, pScale: 1 / denom });
```

---

## Hover outline during zoom

The outline (OutlinePass) is gated off when zoomed:

```js
if (ease < 0.05) {
  // run hover detection and update outlinePass.selectedObjects
}
```

This prevents the outline flickering when the car fills the screen and raycaster
hits it constantly.

---

## Iterating visually

Puppeteer is installed at `/tmp` for headless screenshots during development:

```js
// /tmp/capture.mjs
import puppeteer from 'puppeteer';
const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
const page = await browser.newPage();
await page.setViewport({ width: 1400, height: 820 });
await page.goto('http://localhost:3000', { waitUntil: 'networkidle0' });
await new Promise(r => setTimeout(r, 3000));
await page.screenshot({ path: '/tmp/state_iso.png' });
await page.click('body', { offset: { x: 700, y: 410 } });
await new Promise(r => setTimeout(r, 1500));
await page.screenshot({ path: '/tmp/state_zoomed.png' });
await browser.close();
```

Run `cd /tmp && node capture.mjs`. Read screenshots with the Read tool.
This is the fastest way to iterate on visual changes without manually clicking.
