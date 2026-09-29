# Feature: Digital Page

**File:** `index.html` — `.digital-page` markup + CSS, wiring in the first
`<script>` block, identity/hint integration in the Three.js module
**Status:** Working

---

## What it does

A second root page for software / docs / file-based work, flipped to
horizontally. The concept: page 1 is the **shop floor** (dot grid + 3D
machines), page 2 is the **drafting table** — same greys, same isometric
axes, but the projects are *files*:

- **Manifest** (left) — a drawing-register style file index in IBM Plex Mono:
  No. / Name / Type / Stack / Year / Status rows with staggered entrance.
  Hovering a row lifts its sheet; clicking opens the file document view.
  (There is deliberately NO hover detail card — it duplicated the register
  and the open document; removed in the 2026-07 cleanup pass.)
- **Sheet stack** (right) — an isometric pile of paper sheets built in SVG at
  runtime, using the exact CLAUDE.md projection (`sx = x·22 − z·22`,
  `sy = x·11 + z·11 − y·16`). Hovering a manifest row slides the matching
  sheet out of the pile (`translate(30px, 15px)` — the ground's +x axis).
- **Graph paper** — `.digital-page::after` draws ruled lines at ±26.57°
  (`repeating-linear-gradient(26.57deg / 153.43deg)`), the same axes the dot
  lattice projects to. Bottom-weighted via `mask-image`.

## Flip mechanics

- `.digital-page` is `z-index: 16`: above the scene and side panels (15),
  below nav / identity / page-dots / section-nav / frame-marks (20), so the
  chrome persists across both pages.
- Slide: `transform: translateX(100%) → 0` at `0.6s cubic-bezier(.76,0,.24,1)`;
  `.digital-inner` starts at `translateX(70px)` for parallax.
- `flipTo(i)` (first script block) owns the state: toggles `.open`,
  `.digital-open` on the card, sets `window._digitalOpen`, syncs the edge
  tabs, and calls `window._zoomOutAll()` so page 1 returns to the isometric
  rest view.
- The site opens straight on the Hands-On page. (A split "front door" —
  Hands-On | Digital halves — used to come first; it was removed in the UX
  review: it asked visitors to choose before they had seen any work.)
- **Edge tabs** (`.edge-tab`, z 21): thin vertical handles on the card sides;
  exactly one is out — the handle to the OTHER world. Hidden while a file
  document is open (`.digital-page.file-open ~ .edge-tab`). These replaced
  the page dots.
- Other triggers: nav overlay **Projects** / **Digital** links, and Escape
  (flips home when no overlay, file or 3D view was open). The section-nav
  arrows never flip pages — on this page they step through the files (below).
- **Keyboard:** edge tabs and register rows are divs with
  `role="button" tabindex="0"`; a delegated keydown in the main script turns
  Enter / Space into a click. Closed pages and hidden
  tabs get `visibility: hidden` (delayed by their slide) so Tab never lands
  on something off-screen — the digital page is still fully laid out while
  hidden, so the stack's build-time measurements are unaffected.

## Module integration (Three.js script)

- `window._digitalOpen` guards the card click handler (first line) and the
  hover raycast block — no zooming, outlines, or cursor changes under the page.
- `window._zoomOutAll()` — zooms out, clears `pendingFeature` + `sprocketOpen`.
- `digitalT` lerps toward `_digitalOpen` (`1 − exp(−6·dt)`), and drives the
  same dip-to-zero crossfade as the featured zoom for the identity caption
  (`IDENTITY_DIGITAL`), scene hint (`HINT_DIGITAL`), and sheet note
  (`Digital · Overview`). `metaFade` multiplies the ease and digital fades.

## Sheet stack constants

```js
SHEET_W = 5.6, SHEET_D = 7.4   // sheet footprint in lattice units
SHEET_T = 0.30                  // slab thickness (visible paper edge)
STACK_STEP = 0.72               // vertical gap between sheets
JITTER = [...]                  // per-sheet ox/oz offset + tiny yaw rotation —
                                // one entry per file (it wraps otherwise, and the
                                // wrapped sheets line up exactly)
```

Row `i` (File 01 at top of the register) maps to sheet `n−1−i` (top of the
pile). Faces: top `#F7F6F4`, +x side `#D8D6D0`, +z side `#C6C4BE` — the
CLAUDE.md face-shading ramp. ViewBox is fitted with `svg.getBBox()` after
building, so geometry changes never need manual viewBox math — by
`fitStack()`, which does nothing until the stack is actually rendered (the
column is `display: none` below 900px, where getBBox and the layout rects
are all 0). It runs at build time, from a ResizeObserver on the column (so a
page loaded narrow — an iPad in portrait — fits the first time it widens),
and again before each sheet stands up.

### Printed sheet logos

Each sheet's corner title block holds the project's logo initials
(`DIGITAL_INFO[i].logo`), rendered as SVG `<text>` **mapped onto the paper
plane** with a matrix built from the projection's basis vectors (after the
sheet's jitter rotation `r`, `co = cos r`, `si = sin r`):

```
u-basis (sheet x): ( 22(co − si), 11(co + si) )
v-basis (sheet z): ( −22(si + co), 11(co − si) )
transform = matrix(ux uy vx vy anchorX anchorY), font-size in local units (0.68)
```

The document text lines with `z < 1.85` stop at `SHEET_W − 2.5` so they don't
strike through the title block. To use real logo images later, swap the
`<text>` for an `<image>` with the same matrix transform.

## File document view

Clicking a register row (or the stack, which opens the active file) does two
things at once:

1. **Doc panel docks left** — `.file-doc` (z6) at `left: 30px`, width
   `min(580px, 55% − 50px)`, over `.file-scrim` (z3, blur + wash). Contents
   mirror an engineering drawing: title-block strip (logo cell + File / Rev /
   Year / Status), title, mono stack line, description, numbered points, two
   hatched figure slots (captions from `DIGITAL_INFO[i].figs`, fill via
   `figImg: [url, url]`), mono footer.
2. **The clicked sheet stands up out of the pile** into a front view on the
   right — the same dual-projection blend as the 3D zoom. Sheet geometry is
   stored in local `[x, z, y]` coords; `renderSheet(sheet, t)` re-projects
   every element each frame between the iso pile pose (`M`, includes jitter)
   and a flat front view `F(x, z)` (`FRONT_S = 58` units per local unit —
   sized against the doc panel). The front view is anchored at the **centre
   of the right column** (0.48 height), measured from live layout rects and
   converted to viewBox units — re-measured by `fitStack()` on every open and
   column resize (it used to be measured once at build time, which read 0×0
   on narrow screens and produced NaN geometry). The standing sheet owns the
   column, overflowing the fitted viewBox (`overflow: visible`). Where the
   column is hidden, the doc opens without the sheet morph.
   Transition polish:
   - **Arc**: `FRONT_ARC * sin(π·t)` vertical offset on all geometry — the
     sheet rises as it's picked up, settles as it lands (both endpoints 0).
   - **Face tracking**: face content is authored in front-view units with
     the sheet's corner at the origin (`FA(x, z) = S·(x, z)`, so it never
     depends on the moving anchor), and a per-frame matrix (blend of two
     affine maps is affine, built from the iso basis stored per sheet in
     `sheet.basis`) maps that space through the current blend — the printed
     contents ride the paper through the whole turn instead of fading in
     detached. Fade starts at t 0.35.
   - **Shadow**: `.sheet.front` gets a `drop-shadow` (transitioned via CSS)
     as it lifts.
   Side faces collapse (F ignores height) and fade; ruled placeholder lines
   fade out; the logo's plane-projection matrix blends to the identity front
   basis; face fonts scale by `FK = FRONT_S / 30`.
   `tweenFront()` runs the rAF tween (620ms open / 480ms close, smoothstep);
   on open the `g` is appended (paint on top) + gets `.front` (kills the
   hover-lift transform, `!important`); on close it's re-inserted at its
   pile slot.

**Stacking-context gotcha:** `.digital-inner` has a transform (flip parallax),
which traps children's z-index. The scrim + doc therefore live INSIDE
`.digital-inner`, so `.file-open .digital-right { z-index: 5 }` can raise the
stack column above the scrim (crisp) while the register blurs beneath it.
The right column gets `pointer-events: none` while open (scrim catches
clicks) except `#sheetStack` (click closes). Siblings in the pile fade to
0.4 via `.file-open #sheetStack .sheet:not(.front)`.

**Arrows:** on this page the section-nav arrows (and ←/→) step through the
files in register order via `window._stepFile(dir)`, with ends: from the
register they start at File 01 (or carry on from the last file opened); ←
from File 01 returns to the register; → past the last file opens the wrap-up
card (addresses.md), and ← from it reopens the last file. `openFileAt(i)`
does the move: straight away from the register; from an open file, that file
closes first and the next opens 520 ms later, once its sheet has settled into
the pile (`tweenFront` runs one sheet at a time). Escape during that gap
cancels the step (`window._fileStepping()`); `closeFile()` clears the timer.

**Links and addresses:** each file doc ends with its next step
(`Next · File 02 Actiograph`, after the last `Next · Wrap-up`) and, where there
is one, the matching Hands-On project (`Related · Formula SAE Drivetrain`,
`DIGITAL_INFO[..].related`). Every file has an address (`#digital/pursr` —
`DIGITAL_INFO[..].slug`); `window._fileTarget()` tells the router which file is
open or on its way. The title block is File / Year / Status: the decorative
"Rev" letter is gone (the sheet faces print `FILE 01 · 2026 · ACTIVE` and
`YUVAL MUNZ`), and the page label reads "Software & ventures".

Close paths: ✕ button, scrim click, stack click, Escape (closes the doc
first, flips home on the next press), and `flipTo(0)` (calls
`window._closeFileDoc()`). `window._fileDocOpen` is the shared state flag;
`openFile` is a no-op while it is set (a focused row under the scrim can
still receive Enter). The nav strip above the page is click-through
(`pointer-events: none`, children opt back in), so a tall doc's ✕ is never
blocked by the empty width of the nav.
While open, the sheet note reads `Digital · File NN of 06`.

## Gotchas

- The scene hint / identity / sheet-note opacities are set **inline every
  frame** by the module loop — CSS rules can't override them; integrate with
  `digitalT` instead.
- The manifest row stagger uses a `--d` custom property per
  `nth-child` so the hover `background` transition keeps `0s` delay.
- Content lives in `DIGITAL_INFO` (first script) + the manifest rows in the
  markup — keep the two in sync (the resume, `assets/resume.pdf`, has fuller
  Pursr / Actiograph stories than the register does).
- Phones (≤640px) show only No. / Name / Status in the register; type,
  stack, and year are in the opened file.
