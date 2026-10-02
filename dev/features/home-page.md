# Feature: Home Page (the front door)

**File:** `index.html` — `.home-page` markup (inside `.card`, before the
identity caption), its CSS after the wrap-up card's, "Home" in the main
script (just before "Wrap-up"), the router's `''` address, and the warm-up
draw at the end of the 3D module's `loop()`
**Status:** Working (Oct 2026)

---

## Why

The UX review's P0 removed a split "front door" (Hands-On | Digital halves)
that asked visitors to choose a side before they had learned anything. That
left the site opening straight on the 3D scene — memorable, but "who is
this?" only came from a small role line in the top bar, and the current roles
were two clicks away on About. The fix (option B of two mocked side by side)
is a home page that answers who, what now, and where to look — with previews
of both sides, so the work is still on the first screen.

## What it shows

```
HI, I'M
Yuval Munz.                                   [photo — desktop]
MECHANICAL ENGINEERING · QUEEN'S UNIVERSITY · CLASS OF 2028
I work out how systems run, find where … (the resume summary)
NOW   Queen's Formula SAE          Gear reduction & sprockets lead   → #fsae
      Workato                      Product intern                    → #digital/genie-agent
      Pursr                        Co-founder & CTO                  → #digital/pursr
      Actiograph                   Founding engineer & CTO           → #digital/actiograph
      Queen's Additive Mfg         Mechanical operations lead        → #printer
[ Hands-On →  5 engineering projects ]  [ Digital →  6 software projects ]
● OPEN TO 2027 INTERNSHIPS & CO-OPS   RESUME   ABOUT
```

- Each current role links to its work (the router opens it). Titles follow
  the About page / resume (FSAE is "Gear Reduction & Sprockets Lead", not
  "powertrain lead").
- The door counts come from `PROJECT_INFO` / `window._fileSlugs`.
- Door previews: `renders/lineup.webp` (the five real models at rest, UI
  hidden, rendered headless at 2× and cropped — `background-size: contain`
  on the page colour so the end models aren't cropped) and
  `renders/pursr-home.webp`.
- The photo is `assets/yuval.jpeg` (About's); on narrow screens it becomes
  a round avatar beside the name (`.home-avatar`, a background).

## How it sits on the site

- **Layer:** `.home-page` is z 30 — over the scene, the Digital page (16),
  the wrap-up (17/18) and the side chrome (20/21), under the top bar (35),
  the pages (40), the race (45) and the menu (50). Its background is the
  page colour at 92%, so the (frozen) dot grid shows through faintly.
- **State:** `window._homeOpen`, `openHome()` / `closeHome(instant)` in the
  main script. The markup starts open (`.home-page.open`, `.card.home-open`);
  an address that names a view closes it before first paint (`instant`: no
  fade on arrival). `openHome()` puts everything back to rest — the Hands-On
  page, no file, no wrap-up, the scene at rest (`_goRest`).
- **`.card.home-open`** hides, with `visibility` (no faint copies, no Tab
  stops): the top bar's name / role line / note (the page says them), the
  WebGL canvas, and the scene chrome — identity, hint, labels (each label
  too: the module sets their visibility inline every frame), arrows, edge
  tabs, the phone switch / pager / stage / sheet, panels, wrap-up, Digital.
- **Covered scene:** `_sceneCover()` returns `'page'` while home is open, so
  the module skips hover raycasts, the phone carousel's opening zoom waits
  (it plays when you walk through the Hands-On door), and after 0.6 s the
  scene and the dot grid stop drawing. **Warm-up:** once, when all five
  models are in, the loop draws one frame behind the home page anyway
  (`warmed`) — that first draw compiles the shaders and uploads the
  geometry, so the Hands-On door doesn't stall on it.
- **Guards:** the card click handler skips `.home-page`; the phone swipe and
  the ← → keys do nothing while it is open; Escape does nothing on it (it is
  the top) and, from a side's overview (the scene at rest, the Digital
  register, the carousel), steps out to it.

## Addresses

The plain address is home; the Hands-On overview is `#hands-on` (on a phone
that is the carousel at 01, the same view as `#fsae` — `norm()` maps one to
the other). The top bar's name links home (`href="#"`), the menu lists Home
first, and every "Hands-On" link (menu, the Digital caption, the Digital
wrap-up) points at `#hands-on`. A cold load of `#about` / `#skills` /
`#contact` opens the page over home; closing it returns home. See
addresses.md.

## Layouts

- **Desktop:** two columns (copy | photo, `clamp(190px, 21vw, 300px)`),
  vertically centred. Under 760px tall (laptop windows under their browser
  bars) it tightens so the doors stay on screen — at 1280×610 everything fits.
- **≤ 900px wide** (narrow windows, phones, portrait tablets): one column,
  avatar beside the name, each role's title under its org.
- **Small phones (≤ 640 × 700):** tighter again — the doors reach the first
  screen at 360×640.
- **Landscape phones (≤ 900 wide, ≤ 500 tall):** the doors move up into a
  right-hand column beside the intro.

## Gotchas

- `stateHash()` checks `_homeOpen` after the pages (they open over it) and
  before everything else.
- Headless tests that expect the scene must load `#hands-on`; the plain
  address now shows home (and hides the WebGL canvas).
- Pointer pokes in tests: on a phone most of the home page is links (the
  roles) — click empty space near the bottom when checking that clicks don't
  leak.
