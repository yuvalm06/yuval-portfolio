# Feature: Race Game Easter Egg

**File:** `index.html` — race IIFE at the end of the main `<script>` block
(after the dot grid IIFE), plus `.race-page` markup/CSS and hooks in the
module script
**Status:** Working
**Entry point:** the green button on the steering wheel's face

---

## What it is

A hidden first-person straight-road runner. The steering wheel GLB has four
round buttons on its face plate; the bottom-right one is tinted green and,
when the wheel is featured (zoomed), clicking it swaps the portfolio for the
game. The run accelerates on its own; A/D angle the wheel to steer around
cones, crates, and striped barriers. R restarts after a crash, ESC / ✕ exits
back to the featured steering view.

---

## The green button (module script, steering loader)

The GLB is one fused mesh with no materials. The button is split out with the
scooter-seat pattern — shared `position` attribute, index-only split by
triangle centroid:

```js
const BTN_X = 0.5062, BTN_Y = 0.1559, BTN_R2 = 0.048², BTN_Z = 0.105;
// centroid inside the cylinder (d < 0.048 around the button axis, z > 0.105)
// → button mesh; everything else → body mesh
```

- Constants measured offline from the GLB (numpy over the binary chunk),
  like the car's `WHEEL_AXLES`. Local frame: face points +z, wheel spans
  x ±0.95, y ±0.58.
- The four buttons sit at (±0.66, +0.47) and (±0.50, +0.16); bottom-right
  (viewer's right, since the camera looks at the face from +z) is the one
  at (+0.506, +0.156). Cap + inner ring protrude z 0.10 → 0.149.
- The **tight radius is deliberate**: thumb-grip geometry starts at x ≈ 0.54
  (only ~0.05 from the button edge) and the outer bezel stays grey — the
  green reads as a real button in a dark bezel, and nothing bleeds.
- The default glTF material is fully metallic; the button clone gets
  `color 0x2FA85C, metalness 0, roughness 0.55` (same de-metal treatment as
  the Rivian rear wall's gold).

Interaction wiring follows the sprocket/seat channels exactly:
- `greenBtnMeshes` raycast directly (no proxy, ~1800 tris) — hover only when
  `ease > 0.9 && featured === 'steering'` → pointer cursor + yellow outline.
- Idle glint: the steering branch of `glintMeshes`/`glintHovered` points at
  the button, so it catches the periodic yellow swell like the other
  featured-view secrets.
- Click handler: steering-featured block calls `window._openRace()` and
  returns. Card clicks and ESC are gated on `window._raceOpen` while open.

---

## The game (main script IIFE)

All state in-memory (no storage by design). Canvas 2D on `#raceCanvas`
inside `.race-page` (z 45 — above the pages at 40, under `.nav-overlay` 50).

### Projection
Ground-plane pinhole: `s = focal/z` px per world unit,
`x = W/2 + (wx − playerX)·s`, `y = horizonY + CAM_H·s`, with
`horizonY = 0.40·H`, `focal = 0.62·W`, `CAM_H = 1.35`.

- Road half-width 3.2 world units; spawn horizon `FAR_Z = 55`.
- **Collision plane `HIT_Z = 2.3`** — where the ground plane exits the bottom
  edge (`CAM_H·focal/(H−horizonY)`), so obstacles hit "you" as they reach the
  wheel, not after leaving the screen. Hit window ±0.55 > the largest
  per-frame z step (0.5 at the 30 u/s speed cap) so nothing tunnels.

### Tuning
- Speed: quick ramp to 13 u/s, then +0.5 u/s² to a 30 cap (HUD shows ×4).
- Steering: key target lerped `1 − e^(−7dt)`; lateral speed
  `steer · (2.2 + 0.08·speed)`; playerX clamped to road −0.45.
- Spawn cadence `max(0.42, 1.25 − dist·0.0035)` with ±25% jitter;
  cone 52% / crate 30% / barrier 18%, x kept `half+0.25` inside the edges.

### Drawing (sheet style)
Ground dots march toward the camera (portfolio lattice echo, road kept
clean); cones are gold with a white band, crates use the SVG palette
(#D8D6D0 front / #BCBAB4 side), barriers are gold/ink striped boards; every
obstacle gets a contact-shadow ellipse. The driver's wheel at bottom centre
is a stylised echo of the GLB — dark rim, hub plate, four buttons with the
bottom-right green — rotating `steer · 0.55`. HUD is mono caps
(`DIST 0231 M`), hint text bottom-left because the wheel owns bottom centre.

### Debug handle
`window._raceDebug()` → `{dist, speed, crashed, playerX, obstacles}` or null
when closed; `window._raceDebug(x)` drops a cone at world x, z=8 to force a
collision (used by the Puppeteer checks).

---

## Gotchas

- The ✕ button's click handler needs `stopPropagation()`: `_closeRace()`
  flips `_raceOpen` false, then the same click would bubble to the card and
  the scene raycast handler would zoom out of the steering view.
- The featured pose takes ~4s to fully settle (zoom + glide); automated
  clicks on the button need to wait that long or they hit the grip.
- ESC is intercepted at the top of the global keydown handler while
  `_raceOpen` — otherwise it would also close pages / flip to page 0.
