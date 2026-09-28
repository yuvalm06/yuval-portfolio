# Feature: Compact Layout (phones + portrait tablets)

**File:** `index.html` — mode script in `<head>`, `.m-projects` / `.mode-switch`
markup, the COMPACT LAYOUT section at the end of the stylesheet, the compact
builder near the top of the main script
**Status:** Working (Sept 2026)

---

## Why

The desktop scene doesn't survive a phone. The camera has a fixed vertical
FOV, so a portrait screen crops the scene horizontally: at rest only the car
(and a sliver of the Rivian) is on screen, zoomed views crop the featured
model, and the project panel covers it. The scene is also ~10 MB of GLBs and
7.5M triangles per frame — heavy for a phone.

Fixing that inside the 3D scene means re-deriving the camera ⇄ dot-grid match
for portrait aspect (the constants CLAUDE.md warns about). Instead, phones get
their own presentation of the same content, and the desktop scene is untouched.

## The mode switch — decided once, in `<head>`

```js
matchMedia('(max-width: 640px), (pointer: coarse) and (orientation: portrait), ' +
           '(pointer: coarse) and (max-height: 500px)')
```

- Phones in either orientation (landscape phones are < 500px tall), portrait
  tablets, and desktop windows loaded narrower than 640px → `html.compact`.
- Landscape tablets and desktops → the 3D scene, exactly as before.
- Decided ONCE before anything loads; rotating or resizing later keeps the
  mode. That is deliberate — switching live would mean tearing down or
  booting the WebGL scene mid-session.
- The same script adds the desktop preloads (GLBs, three.js, addons, Draco
  wasm) only when NOT compact, so a phone never downloads a model. The import
  map moved into `<head>` ahead of it so the addon modulepreloads can resolve
  `three`.

## The 3D module never starts on compact

The module's first statement parks it forever when compact
(`await new Promise(() => {})`), and three.js + addons are dynamic imports
after that line — static imports would be fetched before any code could run.
Consequences:

- Every `window._…` hook the module defines (`_zoomOutAll`, `_cycleFeatured`,
  `_stepBack`, `_feature`, `_openRace`…) is **absent** on compact. The main
  script already guards each call (`window._x && window._x()`) — keep doing
  that for any new hook.
- `_dotCam.hidden` is never set, so the dot grid times its own cover pause on
  compact (`coverT`, 0.7 s under the digital page or an opaque page).
- The dot grid ignores `mousemove` on compact: a tap's position would stick
  and leave the grid skewed toward the last tap.

## Hands-On page: project sheets

`#mProjects` (z 12 — over the dot grid, under the digital page) is a scroll
container that starts at `--nav-h`, so sheets never scroll behind the name.
It is built from `PROJECT_INFO` — the SAME object the desktop panels read —
by the compact builder at the top of the main script:

- header (Build 1.0 / Hands-On.), then a **sticky index** of chips
  (01 FSAE … 05 Steering): tap to jump, the chip for the sheet under the index
  highlights (`aria-current`, rAF-throttled scroll listener).
- one `.m-card` spec sheet per project, reusing the desktop panel's classes
  (ticks, `.pp-tag`, `.pp-index`, `.pp-title`, numbered `.pp-points`,
  `.pp-meta`) — plus a **hero render** of the model, the figure (whole
  image, caption below), the WIP stamp for the printer, and the car's
  `detail` (Detail A: sprocket study with the three FEA images).
- Desktop-only interaction lines live in `PROJECT_INFO[..].hint` and are NOT
  rendered on compact ("Click the sprocket…" means nothing there).

Gotchas:
- **Never `scrollIntoView` inside the list.** It aligns the target in every
  scrollable ancestor — including the full-screen `.card` (overflow: hidden
  is still programmatically scrollable) — which slid the nav off the top.
  Index jumps use `list.scrollTo(offsetTop − index height − 12)`, and a
  scroll listener pins `.card` at 0 as a backstop (focus moves can do it too).
- Every `<img>` in the sheets carries width/height (the `SIZE` map; heroes
  are 760×490) so lazy images reserve their space — otherwise a jump lands
  on the wrong sheet when images above it load mid-scroll.
- The desktop panel classes start hidden for the staggered reveal
  (`.pp-head`, `.pp-points li`, …: opacity 0); the compact CSS forces them
  visible inside `.m-card`.

### Hero renders (`renders/mobile/*.jpg`)

Each is the project's featured desktop view (real models), chrome hidden,
cropped to the card-relative box (290, 100, 760, 490) of a 1400×820 viewport.
Re-render after changing a model or its `FEATURE_POSE`: headless Chromium +
`window._feature(name)`, wait for `_dotCam.zoomEase === 1`, screenshot the
clip. Pin `performance.now` to a phase where the idle glint is off
(`t − t % 2400 + 2000`), or the yellow outline may be baked into the image.

## Navigation

- **Hands-On | Digital switch** (`.mode-switch`, bottom centre, z 21) replaces
  the edge tabs; `updateTabs()` drives both (shown once the intro is
  dismissed, `aria-pressed` on the current page). Hidden while a file doc is
  open (`.digital-page.file-open ~ .mode-switch`).
- Hidden on compact (meaningless without the 3D scene): identity, scene hint,
  frame marks, section-nav arrows, edge tabs, dimension line, hover tag,
  detail callout, both 3D panels.
- The digital page's register column scrolls; About is one column on every
  compact screen; the card fills the screen (`100dvh` follows mobile
  browser toolbars).
- An open file doc anchors just below the nav line (`top: --nav-h`) instead
  of centring: on a landscape phone the centred doc put its ✕ under the nav.
  Separately, the `.nav` strip is `pointer-events: none` (its children opt
  back in) on every layout, so its empty width never swallows taps meant for
  whatever sits under it.
- Skills: `.skills-layout > * { flex-shrink: 0 }` — the grid's
  `overflow: hidden` otherwise lets flexbox squash it and clip the last tags
  on a tall phone layout; a short last grid row spans to the edge.
