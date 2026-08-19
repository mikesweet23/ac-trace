# adi AC Trace — design notes

One self-contained HTML file served from GitHub Pages. No build step, no
dependencies to install, no server. It opens on its own from a hard disk,
which is deliberate: it has to work on site.

**This document is the handover.** If someone picks this up cold — another
engineer, a developer, or an AI assistant in a fresh conversation — reading
this should be enough to make a safe change. Keep it current.

---

## 1. What it is, and what it is not

| | |
|---|---|
| File | `index.html` |
| URL | the repo root |
| Does | Refrigerant pipework take-off from a scaled layout: place the condensers and indoor units, trace the routes, set a height at every point, name every section, report the lengths. Ductwork, HRV, outlets and controllers off the same sheet |
| Repo | `github.com/mikesweet23/ac-trace` |

**It does not size refrigerant pipe, and it must never look as though it
does.** Line sizes come from the manufacturer's selection software against
the actual model and the actual run. What this tool contributes is the thing
the selection cannot work out for itself — the route measured off the
drawing, the vertical pipe recovered from the heights, and the height
differences. Three places say so out loud: the condenser inspector, the
summary paragraph in the PDF report, and the Summary sheet of the workbook.
If a sizing feature is ever added, all three have to change with it.

It is a sibling of the water-side toolchain (`Alternative-pipe-sizer`:
Pipe Trace → Pipework Sizer → Network Simulator) and borrows that repo's
drawing engine, snapping discipline and 3D view. It is **not** a step in that
chain — nothing hands over to it and it hands over to nothing — so it carries
no step rail. Its save file is `app: 'adi-ac-trace'` and opening a
`adi-pipe-trace` or `adi-pipework-sizer` file explains where that file
belongs rather than failing.

---

## 2. Non-negotiables

### Lengths, and what each figure means

Three different numbers, and mixing them up is the one mistake that makes the
whole take-off wrong:

| Figure | What it is |
|---|---|
| **Plan route** | the polyline measured on the drawing, one direction |
| **Rise** | every vertical: on to the pipe at the start, every change of height along the run, the drop on to the unit at the end, plus twice any `extraRise` over an obstruction |
| **One direction** | plan + rise. This is the installed route, measured once. It is the headline figure and the one a manufacturer's selection asks for |
| **Pipe metres** | one direction × the number of pipes that section carries |

`riseBreakdown()` is the whole of the rise, and it is walked **in the
direction the section is fed**, which is why `buildTree()` has to have run
first.

### Every point carries its own height

This is the difference between this tool and the water-side Pipe Trace, which
holds one height per component. A refrigerant route lifts over a beam, drops
into a ceiling void and drops again at the unit, and each of those is real
pipe and real height difference against a manufacturer's limit. So
`sg.pts[i].h` is the height of point *i*, falling back to `sg.height` and then
`S.settings.pipeHeight`. `vertexH()` is the only place that resolves it.

While tracing, the next point gets `traceHeight` — or `ductHeight`, because
the two networks keep their own: ductwork usually runs above the pipe, and
switching tools must not throw away the height set for the other one.
`activeHeight()` is which of the two the armed tool is drawing at, and the
one box in the hint bar edits it. Change it before a click and that click
starts a riser.

**Both networks get the same point-by-point list in the inspector**, with
*Level the whole run* and *Set all to…*. Ductwork went out without it once,
and the only chance to set a height was while tracing — get it wrong and the
run had to be drawn again.

### The system type is the whole point

Each condenser is one system, and `SYSTEMS[key].shape` decides what is legal:

| shape | Type | Rule |
|---|---|---|
| `direct` | simple split | one indoor unit, nothing between it and the condenser |
| `dedicated` | multi split | every indoor unit on its own set back to the condenser — **no section may carry more than one unit** |
| `tee` | twin, triple | one set leaves the condenser and tees; the legs past the tee should be within 20% of each other |
| `branched` | VRF 2-pipe | branch joints drop off to each unit, the main carries on; no BC boxes |
| `bc` | VRF 2-pipe BC | every indoor unit runs from a BC controller, never off the main |
| `bc3` | VRF 3-pipe | a single BC box immediately before every indoor unit |

`systemIssues()` is written against that table and nothing else. Adding a
system type is a row in `SYSTEMS` plus its clause in `systemIssues()` — and
if it is wrong, the take-off will still add up, which is exactly why Check
exists.

> The twin/triple leg comparison measures the legs **past the first branch**,
> not the whole run to each unit. Comparing whole runs hides a real imbalance
> behind a long common main, which is the case it was written to catch.

### Two pipes or three

`pipesOf()`. Everything is 2-pipe (liquid + suction gas) except a 3-pipe VRF,
which is 3 pipes (liquid + high-pressure gas + suction gas) from the
condenser **as far as the BC box** and 2 pipes from the box on to the unit it
serves. `buildTree()` sets `sg.afterBc` walking out from the condenser, and
that flag is the whole switch. A section can override it, and the override is
in the file, so a genuine oddity survives a reload.

### The air is a second network, kept apart from the first

A ducted indoor unit or an HRV is the root of a second tree: `S.ducts`, its
own runs, its own joints (`djoint`, never `branch`), its own schedule tab and
its own sheets. `isAirRoot()` is the only test for a root — a ducted FCU
(`type === 'indoor' && unit === 'ducted'`) or `type === 'hrv'`. `inNet()` is
what keeps the two networks apart — pipework will not snap to a diffuser or
an HRV, and ductwork will not snap to a BC box — and `runsOf(net)` picks
which array a tool is working in. Mixing the two would put a duct in the
refrigerant schedule, which is the one thing that must never happen.

**An HRV is never an indoor unit.** It is `type: 'hrv'`, in `PLANT_TYPES`,
not in `INDOOR_TYPES` and not in `PIPE_NODES`. Check would otherwise demand
a condenser. It is always standalone: the only link to the rest of the job
is a controller ticked against it. Default box 1.00 × 0.80 m, editable the
same way as every other unit.

**Four airs, not two.** `DUCT_KINDS` is `oa` / `ea` / `sa` / `ra`
(outdoor, exhaust, supply, return). Older files stored `supply` / `extract`;
`ductKind()` reads those as SA / EA and keeps writing the four-letter keys.
A ducted FCU still only needs supply and extract. An HRV uses all four:
OA and EA to outside, SA and RA to the rooms. Tick *Open return* when the
unit itself is the return — then Check does not ask for an RA duct.

**The unit figure is the stream total, shared at every tee.** `designFlow()`
is the ticked fan speed stored as l/s off `n.fan` (typed as l/s or m³/h;
1 l/s = 3.6 m³/h). `solveDucts()` walks each kind on its own. Unpinned
outlets (`lps` 0) and section take-offs (`outletLps` 0) share what is left
after anything pinned; then each section rolls up `d.flow = takeOff +
carried(down)`. A supply T that feeds two equal outlets therefore carries
half down each leg, and the same on return. Pin a grille to a different
rate when the rooms are not even; the rest re-share. A ducted FCU's supply
and extract are the same volume twice, not one pot the grilles drink from
together. The setting `S.settings.airVol` (`lps` | `m3h`) is only how the
number is typed and shown. Do **not** overwrite `d.flow` with the unit
total after the roll-up — that is what made every section show 111.

Three ways to take air off, because a real drawing has all three: an outlet
node placed where the diffuser is; a count of outlets on a section with a
total against it; and a section marked `sock`, which spreads its take-off
evenly over its own length and reports l/s per metre. All three are counted
in `totals().outlets` and all three appear on the Outlets sheet, or the air
does not add up on paper. An outlet also carries a grille (`GRILLE_TYPES`:
circular, 600×600, egg crate, open return, external louvre). The face is
what turns its l/s into a face velocity; the plan footprint is only a
starting size.

**HRV ducts are named by stream.** `autoNameDucts()` keeps FCU runs as `D`
mains and `E` branches. Off an HRV the counters are `OA1`, `EA1`, `SA1`,
`RA1` — never `D` — so a schedule cannot mint a second `D1`. The counters
still run across the whole drawing.

**An inline heater is a box on the run**, `type: 'heater'`, not a tick on
the HRV. Drop it on a duct or a `djoint` and it sits in line; deleting it
with two ducts on it heals the run (`mergeAtDuctNode()`). Check expects it
on outdoor air.

**Velocity comes from the size, and only from the size.** `ductArea()` turns
a Ø or a W×H into m², and the velocity is the section's carried flow over
it. `faceAreaM2()` / `faceVelOf()` do the same for a grille. `suggestDia()`
goes the other way for convenience, but it only suggests: sizing ductwork
is a different job and a different tool. The conventions this Check uses
are 2.0–8.0 m/s in the duct (the helper aims at 4.0) and 1.0–3.0 m/s at
the face. An HRV stream more than 10% off the unit volume is called out.

### Controllers are located, not connected

A controller is on neither network. It carries a name, a kind, a model, a
height and `serves`, a list of indoor unit and HRV ids. Its leaders are
drawn **only while it is selected**, because drawing every controller's
links at once buries the pipework under them.

### The three colours — and the four airs

`LINES` — liquid `#b8791b`, suction gas `#2471a3`, high-pressure gas
`#c0392b`. The same three values on the plan, in 3D, in the legend, in the
PDF legend. Nothing else in the tool may use them.

`DUCT_KINDS` owns the four air colours: OA `#1a7a6d`, EA `#8c5bb8`,
SA `#2f8f7a`, RA `#c17a3a`. Same rule — plan, 3D, legend, both reports.
Do not borrow a refrigerant colour for a duct, or the other way round.

### The scale gate

Nothing can be placed or traced before the scale is set: a drawing without
one opens a modal that cannot be dismissed, and every tool in `NEEDS_SCALE`
stays disabled (including Tape). Swapping the sheet behind an existing
take-off asks whether the scale still holds rather than gating, so a
revision at the same scale does not throw the take-off away.

The dashed scale strip (`S.calib`) is a reminder of the two points, not a
tape and not part of the take-off. It can be selected and hidden (Delete
or the inspector) without touching `pxPerM`. Changing the layout drops
the graphic — those pixels belonged to the last sheet — and asks whether
the scale figure still holds. Set scale again if it does not. Do not
clear `pxPerM` just because the mark is hidden.

### Tape measure is not a pipe

**Tape** (`tool: 'tape'`, key `M`) is a plan annotation: click along a
route, change direction at a corner, finish to keep it. Stored in
`S.measures`, not in `S.segs` or `S.ducts`. The figure is the polyline on
the drawing — plan metres only. It does not include rise, it does not go
on the pipe or duct schedule, and it must never look as though it does.
Snapping pulls to a unit, a joint or a run **without cutting a joint**.
Select and Delete (or the inspector) removes one. Undo, save, rotate and
the layout plate all carry them. They are not drawn in 3D.

### Rotating turns the trace with the sheet

`rotateDrawing()` re-renders the bitmap and puts **every node, every
polyline point (pipe, duct and tape) and both calibration points** through
the same rotation about the same centre. The scale is untouched, because
turning a drawing does not change how many pixels there are in a metre.
Rotate anything without rotating all of it and units end up in the wrong
rooms with lengths that still look plausible. Repeated fine angles
re-encode the bitmap and soften it; the dialog says so.

---

## 3. Conventions that are easy to break

- **Section naming.** `M1, M2` mains from a condenser; `B1, B2` branches off a
  main; `B1a, B1b` sub-branches off a branch, and `B1a1` deeper again; `X1`
  anything not connected. A run cut into sections by the joints along it is
  numbered from the condenser — `M1.1, M1.2`. A run in one piece keeps its
  plain number. **The counters run across the whole drawing, not per
  condenser**, so no two sections on one sheet can share a name — a schedule
  with two `M1`s on it is unusable. At each junction the leg carrying the most
  duty carries on as the same run and the rest branch off it, which is why
  `rollUpDuty()` has to run before the walk. Ductwork off a ducted FCU is
  `D` / `E`. Off an HRV it is `OA` / `EA` / `SA` / `RA` — never reuse `D`.
- **A run can only be joined to a point that already exists** — a unit, a
  joint, or a corner that has been traced. The cursor is pulled to the nearest
  one and the connection is ringed and named *before* the click. Hold Alt, or
  turn the rule off in Job & defaults, to cut a joint exactly where the cursor
  is. **Joint / T-piece (J)** cuts into whichever network is nearest
  (`hitJointAt()`) — pipe or duct — so a nearby pipe does not steal a duct
  click. The duct palette has the same T-piece. A joint cut into the middle
  of a run takes **the height the pipe is actually at there**
  (`heightAlong()`), not the project default. Air on a duct T-piece
  readjusts on the next `solveDucts()`.
- **Alt is the override key, and it means one thing: ignore the constraint in
  the way.** Over open paper it flips the corner lock, so a square job takes
  one free angle and a free-form job one square corner without changing a
  setting. Over a run already traced it cuts the joint where the cursor is.
  The hint bar turns accent-coloured while it is held. `setOverride()` is
  called from **keydown as well as pointermove** — holding the key and seeing
  nothing change until you jog the mouse is what made it look as though it
  did nothing.
- **The corner lock is stated in three places, because it silently changes
  what a click does.** The status strip carries it at all times (`square` /
  `free`, with `alt` beside it while the key is held); the hint bar has the
  tick box and the key; and Job & defaults has the full explanation. It
  started life only in the dialog, and the first thing anyone asked was how to
  draw a free angle. The live angle badge in the hint bar is the other half of
  that: it reads the leg you are drawing and goes accent-coloured the moment
  the angle is not a multiple of 45°, so a free angle is something you agree
  with rather than something that just happened.
- **The hint bar must stay one or two rows.** It is a reminder of what the
  armed tool does; *How to use* is the manual. It is absolutely positioned
  with **both `left` and `right` set** so shrink-to-fit measures against the
  whole stage — with only `left: 50%` the available width is half the stage,
  and a sentence wraps into a block 200 px tall that sits over the drawing and
  swallows the clicks meant for it. That is a bug that looks like "clicking
  does nothing", so keep the lines short and keep both offsets.
- **Units are drawn at their real footprint once you are zoomed in far
  enough**, and at a legible minimum when you are not (`unitBox()`). That is
  what puts a 600 cassette on its ceiling tile. Both sides of the box scale
  together, so the aspect is never distorted to make a unit visible.
- **The type's size and angle are a starting figure, not a fact.** `n.w` and
  `n.d` are the footprint in real metres and `n.rot` is any angle in degrees —
  set by dragging the corner handles and the arm on the plan, or typed into
  the inspector. `unitFoot()` is the only place that resolves them, falling
  back to the type. Changing the unit type does **not** move a size that has
  been set by hand; `sized` is what tells them apart, and the inspector says
  which it is and offers *Back to type size*.
- **A section remembers where on its end unit it connects.** A refrigerant
  connection is a stub on the side or the end of the casing, and on a large
  unit the difference from the centre is real metres. `sg.aOff` / `sg.bOff`
  hold `{u, v}`, a fraction of that unit's half-extents in the unit's own
  frame — `null` means the centre, which is what every older file has and what
  the centre magnet still gives you. Kept as a fraction so it stays on the
  same corner of the casing when the unit is resized, and turns with it.
  `connPoint()` resolves it and `syncSegEnds()` — first thing in `solve()` —
  puts every section's ends back where its units say they are, so moving,
  resizing or turning a unit carries its pipework with it and nothing else has
  to remember to. With square corners on, `squareIntoUnit()` then keeps the
  last hop onto a casing (indoor, condenser, BC — not a joint or an outlet)
  horizontal or vertical, sliding or inserting a stub so the rest of the run
  is not pulled off-square. That is refrigerant only; ductwork is left as
  traced.
- **The pipes of one section are a true parallel offset.** `offsetPoly()`
  (and `offsetPoly3D()`) shifts each leg by a constant and mitres at
  `lineHit()`. Averaging the incoming and outgoing headings was what made a
  set flare at a corner and pinch back in on the next straight. Strokes use
  `stroke-linejoin: miter`. Do not go back to a heading average.
- **The connection is measured against the TRUE footprint**, `unitBoxTrue()`,
  never the legibility minimum `unitBox()` applies when zoomed out. Measure it
  against the drawn box and a connection slides as you zoom, and every length
  in the schedule moves with it.
- **Resizing and turning a unit now moves its connections, so they can change
  a length.** That is correct — the pipe lands where the stub is — but it is a
  real change from the original rule that the two were drawing truth only.
  A unit whose pipes connect at the centre is still unaffected by either.
- **The box turns; the writing does not.** The rect, its edge band and its
  glyph go inside one rotated `<g>`; the name, the duty and the height are
  drawn outside it, placed off `unitHalf()` — the axis-aligned extent of the
  turned box — so they clear it at any angle. A name and a duty read left to
  right on the sheet however the unit sits, which is the whole reason the
  labels were kept off the box in the first place.
- **The plan is the drawing of record and 3D follows it.** `cuboid3D()` takes
  the same `rot`, and the box uses the real footprint with the type's nominal
  casing depth `t`, grown only when it would otherwise be too small to see
  against the extent of the job. If the two views ever disagreed about where a
  unit is or which way it faces, one of them would be lying.
- **Deleting a joint heals the pipework.** A joint with two sections on it
  merges them back into one and keeps every metre. Three or more cannot be
  healed, so it says so and offers the destructive option rather than doing it
  quietly. `mergeAtBranch()`.
- **Undo is snapshots of the take-off only** — nodes and sections. Not the
  drawing, the scale or the settings: a snapshot carrying the image would be
  megabytes. `pushUndo(label)` goes *before* the change; `asOneUndo()` and the
  `undoSuspended` flag group several edits that are really one action.
  Rotating pushes an undo entry for the geometry, but the bitmap does not come
  back — undoing a rotation puts the trace back on a turned sheet. Turn it
  back instead.
- **Nothing `solve()` works out is written to the save file.**
  `projectBundle()` strips every derived field off nodes and sections. It all
  comes back from the geometry, and a stale derived figure in a file is a bug
  that survives reloads.
- **The take-off is also kept in the browser.** `autosave()` runs off the back
  of `render()`, and is offered back on the next visit rather than restored
  silently — opening the tool to find someone else's job in it is worse than
  one extra click. The drawing is stored under its own key because it is by
  far the biggest part; if it will not fit, the take-off is still kept.
- **Number inputs must not respond to the scroll wheel.** Wheel over a focused
  number field blurs it. Arrow keys blocked, spinners hidden. This is a
  deliberate safety decision — scrolling a page was silently changing design
  figures in the water-side tools.
- **"adi" is always lowercase**, never ADI. That includes anything a
  `text-transform: uppercase` would catch.
- **British English** throughout.
- **The brand bar is shared with the water-side tools.** The block marked
  `adi SHARED BRAND BAR … end shared brand bar` is the same in `trace.html`,
  `index.html` and `simulator.html` of `Alternative-pipe-sizer` — same logo at
  30 px, same `#10151a` bar, same 3 px adi-blue rule, same title and strapline
  scale, same tokens (`--adi-bar`, `--adi-bar-ink`, `--adi-bar-muted`,
  `--adi-bar-line`, `--adi-blue`). **Change one copy and change all of them.**
  AC Trace carries no step rail because it is not in that chain; if it is ever
  put in one, the `.adi-chain` block drops in unchanged.
- Below the bar it keeps a **deliberately different visual identity** so it is
  obvious which tool you are in: Manrope, indigo `#6c7ae0`. Sizer is blue and
  Barlow Condensed, Simulator is IBM Plex and a dark viewport, Pipe Trace is
  Archivo and drawing paper. The 3D view keeps the paper background rather
  than borrowing the simulator's dark one.

---

## 4. The two reports

Both carry the same figures and the same two pictures, drawn by the same code
that draws them on screen, so a report can never show something the tool
does not.

**PDF** — `buildReport()` writes a complete A3-landscape document into a new
window and the browser's own *Save as PDF* is the PDF writer. That is why it
needs nothing installed and comes out with selectable text rather than a
picture of a table. If pop-ups are blocked the same file is downloaded
instead, and the modal says to open and print it.

**Excel** — a real `.xlsx`, written here. A workbook is a zip of XML parts;
`zipStore()` stores rather than deflates, which is an ordinary zip and saves
carrying a compressor for a file this size. The sheets that always appear:
Summary, Pipe sections, Materials, Units, Systems, Check, Layout plan, 3D
plan. Air sheets — Ductwork, Outlets, Ventilation, Controllers — are
dropped when there is nothing on them. Layout plan and 3D plan are picture
sheets — `drawingXml()` plus `xl/media/imageN.png`, anchored below their
caption rows. The Summary sheet still says this file does not size
refrigerant pipe; add the same sentence for ductwork if a sizing feature
is ever added.

**The PDF has to read as a document, not as a web page printed out.** The
rules that keep it that way, all in the report's own `<style>` block:

- **The `@page` margin is the trim and nothing may touch it.** The body
  carries its own gutter on top of it. Text running into the edge of the
  sheet is the single thing that made it look amateur.
- **Prose is held to a measure** (`.lede`, `.warn`, `.endnote`, captions).
  A line of body text 400 mm wide is unreadable; tables and plates are the
  only things that get the full width.
- **Tables cap at 330 mm and numeric columns are `width: 1%`**, so the
  figures stay tight under their headings and the slack falls into the text
  columns rather than being shared out into a scatter across the sheet.
- **A column marked `opt` is dropped when every cell in it is blank.**
  Liquid, gas, HP gas, note, heating kW and room are empty until someone
  types them in, and four empty columns across an A3 sheet is a third of the
  table saying nothing. `repTable()` does this — build a table as columns and
  rows of strings and it comes for free.
- **The running footer is `position: fixed`**, which Chrome repeats on every
  printed page. It sits at the foot of the content box, inside both margins,
  and the body's bottom padding keeps the content off it. It is hidden on
  screen, because a fixed bar over a scrolling preview reads as a mistake.
- **No plate may be taller than the page.** `figure img` is capped to 218 mm
  so `break-inside: avoid` can actually keep it whole; without the cap a tall
  plan is simply sliced across two sheets.

Two things about the pictures:

- `planCrop()` cuts the layout plate down to what was actually traced, plus a
  margin big enough for the labels that hang off it, so the plate is the
  take-off rather than a sheet of empty paper with the job in one corner. A
  trace already covering most of the sheet is left alone. `planImage()`
  resolves `{ url, w, h }` because the crop changes the size and the workbook
  needs the real figures to anchor its picture.

- `planImage()` re-renders the overlay **at a fixed zoom**, not whatever the
  screen happens to be at, so labels come out the same size every time; it
  restores the zoom and selection afterwards. A serialised SVG has no
  document to read `:root` from, so `EXPORT_VARS` are resolved and written
  into the clone. Add a colour variable to the drawing code and add it there
  too, or it exports black.
- The elevation stretches the vertical ×`ELEV_EX`, **and the caption says
  so**. On a floor 70 m across, three metres of height is a few pixels true to
  scale and the picture says nothing. An elevation that lies about its scale
  without saying so is worse than no elevation.

---

## 5. Checking a change did not break it

Five minutes, and it exercises everything:

1. Open `index.html`. Load any PDF or image. Confirm the **scale gate**
   appears and that the condenser, indoor, BC, trace, joint and tape tools
   are all disabled behind it.
2. Set a scale. Confirm the dashed scale mark can be **selected and
   hidden** (Delete or Hide the mark) and that *Route one way* is
   unchanged. Arm **Tape** (M). Click three corners of a room and
   finish. Confirm the label is the plan length, that it is not a pipe
   section, and that Select then Delete removes it. **Change layout**
   to a different sheet, keep the same scale: the old scale mark is
   gone (those points belonged to the last sheet); a leftover tape can
   still be selected and deleted. **Rotate 90°** and confirm the
   drawing turns, the tape turns with it, and *Route one way* is
   unchanged.
3. Place a condenser set to **VRF 3-pipe**, two indoor units of different
   types, and a single BC box in front of each. Trace them up. Confirm the
   sections from the condenser show **3** in the pipe badge and the sections
   after each box show **2**.
4. **Select an indoor unit and drag a corner handle**, then the arm above it.
   Confirm the footprint readout under the box follows, and that the name and
   the duty stay square to the sheet while the box turns. Change the unit type
   afterwards and confirm the hand-set size survives it.
5. **Zoom right in and trace on to the corner of a unit**, not its middle.
   Confirm a ring is drawn where the pipe lands, that the last hop onto the
   casing stays horizontal or vertical (`squareIntoUnit()`), that the two or
   three pipes stay a true parallel offset through every corner
   (`offsetPoly()` mitres; they must not flare and pinch), that the length
   is measured to that corner rather than the centre, and that moving and
   turning the unit afterwards carries the connection round with it.
6. Trace a run that **changes height mid-way** — press `]` between two clicks.
   Confirm a riser marker with the height change appears on the plan, that the
   run inspector lists a height per point, and that *Rise* on the schedule is
   not zero.
7. **Check.** It should come back clean. Now change the condenser to *Multi
   split* and run Check again: it must name the sections carrying more than
   one unit.
8. **3D view.** Confirm every unit sits at the height it was given, that the
   height staff reads off, that a 3-pipe run shows three lines, and that the
   unit you resized and turned stands the same way there as on the plan.
9. **Ductwork.** Put low/medium/high on a ducted unit as l/s, then toggle
   to m³/h and confirm 111 l/s reads as 400 m³/h. Tick a speed, then trace
   a supply duct that T-pieces to two outlets. Confirm each outlet and each
   leg past the tee shows **half** the unit figure. Pin one outlet to a
   different volume and confirm the other re-shares what is left, and that
   the main rolls up the sum. Cut a T-piece (J) into a run already traced
   and branch a third outlet; the shares must readjust. A size gives a
   velocity. Ticking *Ventilated sock* reports l/s per metre. Place a
   controller and tick a unit against it.
10. **HRV.** Place an HRV. Type low/medium/high as l/s or m³/h, tick a
    speed. Trace outdoor and exhaust to louvres, supply and return to rooms.
    Confirm the sections name as `OA` / `EA` / `SA` / `RA`, not `D` / `E`.
    Drop an inline heater on the OA run — it sits in the line, not as a
    tick. Confirm a supply T halves the air left and right (and the same
    on return), that a duct size gives a velocity, and that a grille face
    gives a face velocity.
    Tick *Open return* and confirm Check stops asking for an RA duct.
    Change the box size by dragging a corner. Tick the HRV on a controller.
    Run Check on an HRV-only sheet: it must come back clean without asking
    for a condenser.
11. **Line sizes.** Open the schedule, tick every section, set a liquid, a gas
    and an HP gas size and apply them. Confirm the HP gas is skipped on the
    2-pipe sections and named in the toast, and that the inspector shows the
    same sizes as the drawer for whichever section is selected.
12. **Save**, reload the page, **Open** the file. Every section name, length,
    point height, footprint, angle, connection point, line size, duct size,
    air volume, HRV stream, heater, grille, controller and tape measure
    must come back identical.
13. **PDF report** — the layout plan and both 3D pictures must be there, in
    colour, with the section labels legible. An HRV job must show the
    ventilation heading and the room in/out line. **Excel** — open it and
    confirm *Pipe sections* totals match the schedule drawer, that a job
    with an HRV has a *Ventilation* sheet, and that the two picture sheets
    carry their pictures.

Reconcile one figure by hand at least once: for any indoor unit, the sum of
*One way m* down its Route in the report must equal its *Pipe run m*.

---

## 6. Deploying

1. Tag the current state: **Releases → Draft a new release** → new tag →
   publish.
2. Open `index.html` → pencil → select all → paste the new version.
3. Commit with a message that says what changed.
4. Wait for the green tick on **github-pages** in Deployments.
5. Hard-refresh the live URL (`Ctrl+Shift+R`). On iPhone, force-close the
   home-screen app and reopen — iOS caches hard.
6. Tag again once you have confirmed it works.

To revert: open the last good release → **Browse files** → `index.html` →
**Raw** → copy → paste back → commit.

---

## 7. Standing assumptions

Replace when better figures arrive; none of them changes a length.

| Assumption | Currently |
|---|---|
| Indoor unit footprints | Wall 1.05 × 0.24, 900 cassette 0.95 × 0.95, 600 cassette 0.62 × 0.62, ducted 1.40 × 0.70, 1-way underslung 1.20 × 0.50 m. Representative, not any one manufacturer's — and only a starting size, because every unit can be dragged or typed to the size it really is |
| HRV / heater footprints | HRV 1.00 × 0.80, inline heater 0.40 × 0.25 m. Starting size only — both can be dragged or typed |
| Casing depth, 3D only (`t`) | Condenser 1.30, ducted and HRV 0.35, wall and 900 cassette 0.30, BC controller 0.30, 600 cassette 0.26, 1-way underslung 0.25, single BC box and heater 0.22 m. Nominal: nothing here is told how deep a real unit is, and no length depends on it |
| Default heights | condenser 0.30, pipe run 3.20, ductwork 3.40, indoor 2.70, BC box 2.90, HRV and heater 2.90 m above finished floor |
| Connected-ratio flag | over 130% is called out. Real limits are per range and per refrigerant |
| Twin/triple leg tolerance | 20% between the legs past the tee |
| Pipe separation on the plan and in 3D | drawn wider than true so three pipes read as three pipes. True separation is one line on screen |
| Duct velocity | helper aims at 4.0 m/s; Check warns below 2.0 and above 8.0. Face velocity warns below 1.0 and above 3.0. All conventions, not limits — the job decides |
| HRV stream balance | a stream more than 10% off the unit volume, or supply and exhaust more than 10% apart, is called out |
| Round duct sizes offered | 100 to 800 mm in the standard steps. Rectangular is whatever W×H is typed |
| Fan speeds | Low, medium and high, each l/s and Pa, all typed in from the unit's data. Nothing here works an airflow out |
| Insulation | 1/2" wall by default, 3/8" the thinner option, per section. Insulated length is the pipe metres, because every line is insulated over its whole length |
| Line size list | 1/4 to 1 5/8 inch in the nine steps refrigerant pipe is actually bought in. Sizes are picked from that list, never typed, because a typed figure is one nobody can order |

The one figure this tool will never carry is a refrigerant line size worked
out by itself. Sizes are typed in from the manufacturer's selection, per
section, and travel through to both reports as typed.
