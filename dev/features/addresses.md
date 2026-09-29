# Feature: Addresses, Next Steps and the Wrap-up

**File:** `index.html` — "Wrap-up" and "Addresses" at the end of the main
script; `_goProject` / `_goRest` / `_sceneTarget` / `_cycleFeatured` in the 3D
module; `openFileAt` / `_stepFile` in the digital page script
**Status:** Working (Sept 2026)

---

## Why

The UX review (P1) found the site behaved like one screen with hidden state:
browser Back left the site, nothing could be linked, the arrows wrapped from
05 back to 01 with no ending, and the two sides never pointed at each other.
Recruiters read a portfolio as a sequence with a start, an end and a next
step — and share single projects with hiring engineers.

## The flow

```
Hands-On:  overview → 01 → 02 → 03 → 04 → 05 → wrap-up (Digital · Resume · Contact)
Digital:   register → File 01 → … → File 06 → wrap-up (Hands-On · Resume · Contact)
```

- Every project panel (and phone sheet) ends with its named next step —
  `Next · 02 Baja SAE Steering Wheel`; after the last one, `Next · Wrap-up`.
  The file docs do the same (`Next · File 02 Actiograph`).
- The sequences have ends. Desktop arrows: → at rest starts at 01 (or carries
  on from the last project seen), ← from 01 returns to the overview, → past 05
  opens the wrap-up, ← from the wrap-up goes back to 05. The arrows dim
  (`.section-btn.off`) where there is nothing further. Digital is the same with
  the register in place of the overview. The phone carousel stops wrapping:
  ‹ is disabled at 01, the drag springs back past either end, › at 05 opens
  the wrap-up.
- **Wrap-up card** (`#endCard` over `#endScrim`, z 18 / 17): the other side,
  the resume, contact, and "Back to 05". One card, filled per side
  (`END_COPY`). It is not an opaque page — `_sceneCover()` returns `'end'`, so
  the module stops hover raycasts and fades the scene labels but keeps
  drawing the blurred scene behind it. Scrim click / Escape close it; a page
  flip closes it (each side has its own).
- **Bridges:** the FSAE panel's `Related · Drivetrain Lap Simulation` and the
  lap-sim file's `Related · Formula SAE Drivetrain (Hands-On)`
  (`PROJECT_INFO.car.related`, `DIGITAL_INFO[5].related`); each side's intro
  (the desktop caption at rest, the Digital page's sub-line) links the other.

## Addresses (the router)

| Address | View |
|---------|------|
| (none) | Hands-On overview — on a phone, the carousel at 01 (`#fsae` is the same view there) |
| `#fsae` `#rivian` `#scooter` `#printer` `#steering` | that project (`PROJECT_INFO[..].slug`) |
| `#fsae/sprocket` | the sprocket study |
| `#end` | the Hands-On wrap-up |
| `#digital` · `#digital/<slug>` · `#digital/end` | the register · a file (`DIGITAL_INFO[..].slug`) · the Digital wrap-up |
| `#about` `#skills` `#contact` | that page, over whatever is under it |
| `#race` | the race easter egg |

Two directions:

- **`applyHash(h)`** makes the page match an address: `a[href^="#"]` links
  anywhere (the menu, the top bar's Contact, Next / Related, the wrap-up —
  intercepted in the capture phase so they never reach the 3D click handler),
  Back / Forward (`popstate`), a typed address, and a cold load. Scene
  addresses wait for the module (`window._sceneReady`, called at the end of
  the module script) — the router is in the classic script, which runs first.
- **`reconcile()`**, a 150 ms poll, writes the address for everything else
  that changes the view (a click on a model, the arrows, a swipe, Escape, a
  ✕): `stateHash()` reads where the view is *headed* — `_sceneTarget()` gives
  the project being opened (`pendingFeature` / `zoomDir`, or the carousel's
  `carTarget`), `_fileTarget()` the file (or the one a step is opening) — so a
  bounce between two projects never records "the overview" in between. If
  the new address is the previous entry, it steps Back instead of pushing
  (closing something undoes its entry); otherwise it pushes.

Bookkeeping: `history.state = { i }` and an in-memory `stack` of addresses
(`i` survives a reload; the stack doesn't, so "is the previous entry?" only
answers for entries seen this session). `expectPop` swallows the `popstate`
of Back steps the router took itself. Mid-swipe (`_sceneTarget().moving`) the
poll waits for the carousel to settle.

## Adding a view

A new page, overlay or detail view needs: the card click guard and
`_sceneCover()` (see 3d-scene.md, covered-scene gating), an address in
`stateHash()` and `applyHash()`, and the table above. A view the router
doesn't know about is invisible to Back.

## Gotchas

- Hooks the router calls into the module (`_goProject`, `_goRest`,
  `_sceneTarget`) are absent until three.js has loaded — `stateHash()` returns
  null for scene states until `_sceneReady`, and the poll records nothing.
- `_goProject(key)` on desktop: from rest it zooms in; heading out of that same
  project it re-zooms; from another project it bounces through rest
  (`pendingFeature`). On a phone before the opening zoom lands, it picks the
  station the zoom lands on (a cold load of `#printer` opens on 05).
- A cold load of `#end` on a phone shows the wrap-up before the opening zoom;
  the zoom (onto 05) plays once the card closes — the carousel waits while any
  cover is up.
- Headless: the poll is 150 ms and Back is async — wait for `location.hash`
  or `window._sceneTarget()` rather than sleeping.
