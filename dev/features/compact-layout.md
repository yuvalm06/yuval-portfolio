# Feature: Compact Layout (phones + portrait tablets)

**File:** `index.html` — mode script in `<head>`; the COMPACT LAYOUT section at
the end of the stylesheet; `.m-stage` / `.m-pager` / `.mode-switch` markup;
"Compact carousel controls" in the main script; `frame()` / `side()` in the dot
grid; "Compact carousel" + `compactStep()` in the 3D module
**Status:** Working (Sept 2026)

---

## Why

The desktop scene doesn't fit a phone as-is. The camera has a fixed vertical
FOV, so a portrait screen crops it sideways: at rest only the car is in frame,
the featured views crop the model, and the info panel covers it.

A first pass swapped the scene on phones for a scroll of spec sheets with
pre-rendered model images. It read cleanly, but it dropped the part of the
site that makes it: the projects standing in the 3D space. The current layout
keeps the scene and makes it phone-shaped instead — **one project at a time,
swiped sideways**, with its sheet docked underneath.

## The mode switch — decided once, in `<head>`

```js
matchMedia('(max-width: 640px), (pointer: coarse) and (orientation: portrait), ' +
           '(pointer: coarse) and (max-height: 500px)')
```

- Phones in either orientation (landscape phones are < 500px tall), portrait
  tablets, and desktop windows loaded narrower than 640px → `html.compact`.
- Landscape tablets and desktops → the desktop scene, unchanged.
- Decided ONCE before anything loads; rotating or resizing later keeps the
  mode (switching live would mean re-staging the whole scene mid-session).
- Preloads: desktop preloads all seven GLBs; compact only the car and its
  sprocket (see Loading).

## The carousel (3D module)

- **Opening zoom.** The first time the Hands-On page is actually on screen
  (intro dismissed, not under Digital or a page) the desktop zoom plays into
  the car — once its GLB is in, or after 3 s regardless. Until then the scene
  sits at rest behind the intro, exactly like desktop.
- **Stations.** Once the zoom lands (`zoomT === 1` → `carousel`), every model
  holds its `FEATURE_POSE` at its own station along world x:
  `stageT(name)` is `ease` for all of them (desktop: only `featured`) and
  `stageX(name) = stationDelta(name) × spacing`. Stations wrap (each model
  sits at its nearest copy), so the line has no ends.
- **Motion.** `carPos` (continuous, in stations) is the view; `carTarget` the
  integer station it settles on through an exact critically damped spring
  (ω 10 — about 0.4 s; a flick's velocity carries straight in).
  `featured` is the nearest station: it swaps at the halfway point, where the
  sheet copy has faded out (`fillProjectPanel` + a `featured` event on the
  card, which the pager follows).
- **The truck.** The dot ground scrolls under the models:
  `window._dotCam.truckX = (carPos − truckBase) × spacing` (world units). The
  dot grid converts it to lattice units, slides every dot by the fraction
  (`trkF`) and feeds the whole dots passed (`trkI2`) to the wave, so each dot
  keeps its phase. The models pass in true perspective (the Rivian, pulled
  forward, shows its flank as it slides in) — it reads as the camera
  travelling along the ground, not cards sliding.
- A model renders only while |stationDelta| < 1 — at most two at a time.
- The zoom elevation blends between the stations either side (printer 0.30,
  steering 0.24, the rest 0.10); on compact the dot grid's `SV_TILT` follows
  it (`_dotCam.zoomEl`) so the ground stays under the raised views.
- Once, 1.4 s after it first lands, the carousel nudges toward the next
  project and springs back — "this moves sideways". Skipped if the user has
  already moved it.

### Entry points (main script → module; all guarded, the module loads async)

| Hook | Used by | Does |
|------|---------|------|
| `_swipe(±1)` | pager arrows, arrow keys (via `_cycleFeatured`) | one station; presses stack on the target |
| `_swipeTo(key)` | pager dots | that project, the short way round |
| `_swipeDrag('move' \| 'end', dx, v)` | the swipe gesture | the model tracks the finger 1:1 (px per station = `spacing × ppu`); on release a flick (> 0.35 px/ms) goes on in its direction, otherwise the nearer station — a third of the way is enough to leave |
| `_carouselState()` | tests | `{ carousel, carPos, carTarget, featured, dragging }` |
| `_feature(name)` | tests | at rest: pre-selects the station the opening zoom lands on; after: `_swipeTo` |

Desktop hooks on compact: `_cycleFeatured` → `_swipe`; `_zoomOutAll` (flip
to Digital) only closes a detail view — the carousel stays put; `_stepBack`
never zooms out. The card click handler never zooms on compact, but the
sprocket, seat-module and green-button taps work as on desktop.

## Framing (dot grid `frame()`)

- `.m-stage` is the CSS box the model is framed into: under the nav, above
  the pager and the collapsed sheet in portrait; left of the side sheet in
  landscape. The dot grid measures it on every resize.
- Scale: `ppu = min(0.84 × stageW / 3.0, 0.82 × stageH / 1.75, desktop's)` px
  per world unit — the widest featured model (the car, 3.0) and the tallest
  (scooter / Rivian pulled forward, ~1.75) fit, and it never comes closer than
  desktop's ZOOM_DIST 8. `ZD = H / (2·tan 14° · ppu)`.
- View offset: the scene centre moves to the stage centre, lifted `0.3·ppu`
  so the models' visual centre (y ≈ 0.8; the camera looks at 0.5) sits there.
  The camera takes it as `setViewOffset(W, H, −offX, −offY, W, H)` (checked
  every frame); the dots through `CX`, `CY` and `SV_CY`.
- The side-view constants scale with `8 / ZD` (derivation in
  [dot-grid-reprojection.md](../lessons/dot-grid-reprojection.md)); at ZD 8
  they are exactly the desktop constants.
- `spacing = 1.1 stage widths` in world units: the next model waits just off
  the stage and slides in as this one leaves.
- Featured-pose extents, measured (world units, w × h): car 3.0 × 1.3,
  Rivian 1.6 × 1.5 (at z +1.2), scooter 2.2 × 1.7, printer 3.0 × 1.0,
  steering 2.0 × 1.3. Re-measure if a `FEATURE_POSE` scale changes.

Typical results: 390×844 phone ZD ≈ 14.8; 768×1024 tablet ≈ 9.1; landscape
phones stay at 8 (height-limited).

## Loading

- `<head>` preloads only the car and sprocket on compact, and the module's
  `loadGLB` queue holds the other GLBs until both are in (`glbDone`). Split
  seven ways, a phone's bandwidth delivered the first project nearly last.
- While the featured model's GLB is missing the card carries
  `model-loading` → a "Loading model" note in the stage (0.3 s delay, so a
  cached model doesn't flash it). The sprocket stays hidden until its car is
  in. Swiping to a model still on its way shows the same note.
- The Rivian's ground glue only solves at rest; one that loads after the zoom
  starts from its load pose (`rivianGlue` default) — the featured pose
  ignores the glue anyway.

## The sheet (`#projectPanel` on compact)

- **Portrait:** docked above the switch at a fixed collapsed height
  (`--sheet-h`, 128px — the stage is framed around it): tag, title and a
  two-line summary (the project's `desc`, `<br>` dropped). Tapping it — or
  Details — opens it up the screen (`.expanded`: rows, figure with its
  `imgCap` caption, meta; it scrolls). Close, a tap on the scene above, or
  rotating the phone folds it back. The pager steps aside while it's open
  (`sheet-open` on the card).
- **Landscape:** beside the stage, always open.
- Between stations the sheet frame stays; its copy wrapper `.pp-body`
  (`display: contents` on desktop — no box, no layout change) fades out a
  third of the way across and drifts with the swipe, and `.in` re-runs the
  row stagger on arrival. The loop also owns the sheet's `pointer-events`
  (on only while it shows) — CSS must not set them.
- The hint line reads "Tap…" instead of "Click…" (`fillProjectPanel`).
- **Sprocket study:** same dock (bottom sheet ≤ 46% in portrait, side sheet
  in landscape) with a ✕; as on desktop a tap anywhere closes it. The gear
  flies to the middle of the free space above the sheet at the size that
  fits there — `gearOpenPose()`, measured on the tap.
- **Seat module:** the scooter slides 0.45 right while the module lifts, so
  the module stays on a narrow stage.

## Touch

- The card is `touch-action: none` on compact — a sideways drag is the
  carousel's and nothing pans or zooms the stage. Scrollers inside it (the
  open sheet, the register, About, an open file) still pan: a scroll
  container resets the effective touch-action for its subtree. The sheets are
  `pan-y pinch-zoom`, so a sideways drag on them still swipes.
- The swipe is Pointer Events on the card (main script): 8px slop, direction
  lock (a vertical start is not ours), pointer capture, smoothed velocity; a
  finger that stops before lifting is not a flick. A mouse drag works too
  (narrow desktop window); the click it ends in is swallowed by a capture
  listener so it never reaches the scene.
- `overscroll-behavior: none` on the root — no pull-to-refresh.
- **Race game:** hold the left or right half to steer (tracked per pointer
  id, slide across to switch), tap to restart after a crash; the HUD wording
  follows `(hover: none)`. This also makes the game playable on landscape
  tablets (desktop layout). A desktop mouse click on the road still does
  nothing.

## Navigation and pages

- **Pager** `‹ • • • • • ›` between the stage and the sheet (under the stage
  in landscape): arrows step, dots jump. Shown once the carousel is up
  (`carousel-on`); hidden under Digital, the open sheet and the sprocket
  study (`detail-open`).
- **Hands-On | Digital switch** (`.mode-switch`, bottom centre; under the
  sheet column in landscape) replaces the edge tabs; `updateTabs()` drives
  both. Hidden while a file doc is open.
- Hidden on compact: identity, scene hint, frame marks, section-nav arrows,
  edge tabs, dimension line, hover tag.
- The digital page's register column scrolls; About is one column on every
  compact screen; the card fills the screen (`100dvh` follows mobile browser
  toolbars) and is pinned against programmatic scroll (focus moves).
- An open file doc anchors just below the nav line (`top: --nav-h`) instead
  of centring: on a landscape phone the centred doc put its ✕ under the nav.
  Separately, the `.nav` strip is `pointer-events: none` (its children opt
  back in) on every layout, so its empty width never swallows taps meant for
  whatever sits under it.
- Skills: `.skills-layout > * { flex-shrink: 0 }` — the grid's
  `overflow: hidden` otherwise lets flexbox squash it and clip the last tags
  on a tall phone layout; a short last grid row spans to the edge.

## Gotchas

- The phone rules from before compact existed (`@media (max-width: 640px)`:
  panels `top: 84px`, `.project-panel .pp-img-slot { display: none }`) still
  apply to a desktop-mode window narrowed below 640px. Compact overrides them
  by specificity — which is why the expanded sheet sets `display: block` on
  the figure explicitly.
- Rotating recomputes the framing and spacing; the dot scroll jumps once
  (truck = `(carPos − truckBase) × spacing`).
- The printer's local `stageX` / `stageZ` were renamed `posX` / `posZ` — the
  loop now has a `stageX(name)` helper in scope.
- Headless testing: `window._carouselState()` to wait on settling; drive
  touch drags with CDP `Input.dispatchTouchEvent` (Playwright's touchscreen
  only taps).
