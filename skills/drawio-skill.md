---
name: drawio-skill
description: Use when the request calls for diagrams, flowcharts, architecture diagrams, ER diagrams, UML/sequence/class diagrams, network topology, ML/DL model figures, mind maps, or any visualization needing custom styling, rich shape vocabulary, swimlanes, or exportable images (PNG/SVG/PDF/JPG). Generates .drawio XML and exports locally via the native draw.io desktop CLI.
---

> **Public / shared version.** This isn't originally mine — it's built on the
> open-source **drawio-skill** by Agents365-ai
> (https://github.com/Agents365-ai/drawio-skill, MIT license). I run it as
> part of my Claude Code setup and adapted it for my own machine; this page
> strips my local binary-path resolution and keeps the workflow, the CLI
> quirks, and the export gotchas, which are the genuinely useful part if
> you're setting this up yourself. Full credit to the original author — go
> get the real thing from the repo above rather than treating this page as
> the source of truth.

# draw.io diagrams from an agent

Generates `.drawio` XML and exports it to PNG/SVG/PDF/JPG locally through the
native draw.io desktop app's CLI — no browser automation needed.

## When to use it / when not to

**Use it for:** polished, precise diagrams (architecture, network, strict
UML, ERD), anything needing solid opaque fills, thousands of stock/branded
shapes, swimlanes, or custom geometry, exported as an editable PNG/SVG/PDF
(PNG/SVG/PDF exports can embed the diagram XML, so opening the export later
in draw.io recovers the editable diagram — use a double extension like
`name.drawio.png` to signal that).

**Don't use it for:** a casual hand-drawn/whiteboard look (use a sketch tool
instead), diagrams-as-code that should live in git and render inline in
Markdown (use Mermaid or PlantUML), or freeform infinite-canvas sketching.

## Prerequisites

The draw.io desktop app must be installed and its CLI reachable:

```bash
# macOS (Homebrew)
brew install --cask drawio
drawio --version

# Windows — installer from the drawio-desktop GitHub releases
"C:\Program Files\draw.io\draw.io.exe" --version

# Linux — .deb/.rpm from GitHub releases (avoid the snap package;
# its sandbox denies secrets/keyring access on servers and crashes)
drawio --version
```

**Resolve the binary name once per machine** — it varies: `drawio` (Homebrew/
Linux packages), `draw.io` (older builds/custom symlinks), the macOS `.app`'s
internal binary, or the Windows `.exe` path. Whichever prints a version first
is the one to substitute for `drawio` in every command below.

Optional: Graphviz (`dot` on PATH) enables auto-layout for large/layout-heavy
diagrams instead of hand-placed coordinates.

## Workflow

1. **Clarify scope** if the request is underspecified — diagram type
   (ERD/UML/sequence/architecture/ML/flowchart/general), output format
   (PNG default, SVG/PDF/JPG), and output location. Skip this if the request
   already answers it.
2. **Check deps** — resolve the CLI binary name/path as above and reuse it
   for the rest of the run.
3. **Plan** — identify shapes, relationships, and layout direction (left-right
   or top-bottom); group by tier/layer.
4. **Generate** the `.drawio` XML. Hand-place coordinates for small/styled
   diagrams; for large or layout-heavy ones (dependency graphs, code
   structure, 15+ nodes), describe the graph as data and run it through a
   Graphviz-backed auto-layout step instead of guessing positions. Run a
   structural lint pass (dangling edges, duplicate ids, broken parents,
   overlaps) before exporting.
5. **Export a draft preview PNG** — **without** the embed-XML flag. The
   embedded-XML variant adds a metadata chunk that breaks most vision-model
   image ingestion (400 "could not process image"). Cap the preview width
   (e.g. `--width 2000`) rather than scaling — an oversized preview can exceed
   a vision API's max-dimensions limit.
6. **Self-check** using vision — read the exported PNG, catch overlapping
   shapes, clipped labels, missing/misrouted connections, off-canvas shapes,
   and edges crossing unrelated shapes; auto-fix up to two rounds before
   showing the user.
7. **Review loop** — show the image, collect feedback, apply the *minimal*
   targeted XML edit for each request (move/resize/recolor/relabel one shape,
   add/remove a node or edge), re-export the same preview filename (no
   versioned copies), repeat until approved. Full regeneration only for
   layout-wide changes (e.g. swapping left-right for top-bottom).
8. **Final export** — export to all requested formats, **with** the embed-XML
   flag this time (keeps the file editable in draw.io later); save with a
   double extension (`name.drawio.png`) to signal that. Report file paths for
   both the `.drawio` source and the exported image(s).

## XML structure essentials

Every file needs the two root cells (`id="0"` and `id="1"`) — never omit
them; user shapes start at `id="2"` and increment. All shapes need
`parent="1"` (or a container's id). Multi-line label text uses `&#xa;`, never
a literal newline.

**Edges are the easy thing to get wrong:** every edge `mxCell` must contain an
expanded `<mxGeometry relative="1" as="geometry" />` child — a self-closing
edge cell is invalid and silently fails to render. Always include
`rounded=1;orthogonalLoop=1;jettySize=auto` in edge styles for clean routing,
and pin `exitX/exitY/entryX/entryY` on any shape with 2+ connections so lines
don't stack on top of each other.

For containers (a service box holding sub-components), use draw.io's real
parent-child containment (`swimlane;` or `container=1;pointerEvents=0;` on the
parent, children with `parent="<container-id>"` and coordinates relative to
it) — don't just visually stack shapes on top of a bigger shape.

A neutral starting palette (fill/stroke pairs): blue for services/clients,
green for success/databases, yellow for queues/decisions, orange for
gateways/APIs, red for errors/alerts, grey for external/neutral, purple for
security/auth. Snap all coordinates to multiples of 10 so they align on the
default grid.

For vendor/branded icons (cloud provider shapes, network/UML/BPMN symbols) or
any shape you'd otherwise have to guess the style string for, look it up
rather than guessing — a wrong shape name silently renders as a blank box.

## Export commands

Two distinct export modes matter:

```bash
# Preview / self-check — NO embed flag, width-capped for vision ingestion
drawio -x -f png --width 2000 -o diagram.png input.drawio

# Final deliverable — WITH embed flag, double extension
drawio -x -f png -e -s 2 -o diagram.drawio.png input.drawio

# SVG / PDF (final) — embed is safe here, both are text-based formats
drawio -x -f svg -e -o diagram.svg input.drawio
drawio -x -f pdf -e -o diagram.pdf input.drawio

# Headless Linux (CI/server) needs a virtual display
xvfb-run -a --server-args="-screen 0 1280x1024x24" \
  drawio -x -f png -e -s 2 -o diagram.drawio.png input.drawio --disable-gpu
```

**Known CLI bug worth knowing about:** the desktop CLI truncates the IEND
chunk on embed-flag PNG exports (8 bytes missing), which makes the file fail
strict PNG validation and vision-model ingestion. A tiny post-export script
that appends the correct trailing bytes when they're missing fixes it — run
it immediately after every embedded PNG export; it's a safe no-op once
upstream fixes the bug.

**Browser fallback (no CLI needed):** the diagram XML can be
`encodeURIComponent`-encoded, deflate-compressed, and base64'd into a
`diagrams.net` URL fragment, which opens the diagram client-side with nothing
uploaded to a server — useful when the desktop CLI is unavailable or crashes
under a restricted sandbox.

## Fallback chain

| Scenario | Behavior |
|----------|----------|
| CLI missing, scripting runtime available | Generate a browser-fallback URL |
| CLI missing entirely | Deliver the `.drawio` XML only; instruct manual open |
| CLI crashes in a sandboxed environment | Treat as unavailable there; use browser fallback or ask for a non-sandboxed export |
| Vision unavailable | Skip self-check, show the exported PNG directly |
| Export fails headless on Linux | Try a virtual display, then `--no-sandbox` if running as root, then `--disable-gpu` |

## Style presets

The underlying skill supports named, reusable style presets (palette, shapes,
fonts, edges) that fully replace the default color/shape conventions when
active — handy if you want every diagram in a project to share one visual
language instead of restating it each time. See the upstream project for the
full preset-learning and management flow.
