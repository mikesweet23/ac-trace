# AC Trace

Refrigerant pipework take-off for air conditioning systems — adi Climate Systems.

One self-contained HTML file. No build step, no dependencies, no server: it
opens on its own from a hard disk, because it has to work on site.

**Live:** open `index.html`, or the GitHub Pages URL for this repo.

## What it does

Load a PDF or image of the layout, square it up, set the scale from a known
dimension, then place the plant and trace the routes between it.

- **Condensers**, each one a system: simple split, multi split, twin, triple,
  VRF 2-pipe with branch joints, VRF 2-pipe with a BC controller, or VRF
  3-pipe with a single BC box before each indoor unit.
- **Indoor units**: wall mounted, 900 mm ceiling cassette, 600 mm ceiling
  cassette, ducted, and 1-way blow underslung — each drawn at its real
  footprint once the drawing is zoomed in far enough to read it.
- **Heights everywhere.** Every unit and every point along every run carries
  its own height, so the risers and drops are measured with the plan run
  rather than guessed afterwards.
- **Every section named** — `M1, M2` mains, `B1, B2` branches, `B1a` sub-
  branches, sections numbered from the condenser — and every name, unit type,
  model and duty editable.
- **3D view** to check the risers and the height differences.
- **Check** reads every system against its own type and says what will not
  work: a multi split sharing a section, a 3-pipe VRF missing a BC box, a twin
  split tee'd unequally.
- **PDF report** — summary, system figures, layout plan, 3D plan and
  elevation, and the schedule by system.
- **Excel report** — the same figures as a real `.xlsx`, seven sheets,
  including the layout plan and the 3D plan as pictures.

## What it does not do

It does not size refrigerant pipe. Line sizes come from the manufacturer's
selection software against the actual model and the actual run. This tool
measures the route, keeps the heights honest and names every section; the
sizes are typed in from that selection and carried through to both reports.

## Notes for whoever picks this up next

`CLAUDE.md` is the handover — the rules that keep the take-off honest, how
the system types are enforced, and a five-minute check that a change has not
broken anything.
