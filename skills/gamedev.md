---
name: gamedev
description: >
  A faceted, meta-tagged index over a large pile of free community game
  assets and tools, queryable by mechanic (fps/driving/flying/weather/
  camera/character-movement…), asset type (3d-model/plugin/shader/tileset/
  icon…), art style (low-poly vs realistic), theme (sci-fi/fantasy/…), genre,
  and license. Use when planning or building any game and you need to know
  "what free thing already does this" before hand-rolling it. Trigger
  /gamedev.
user_invocable: true
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. My actual index currently
> spans a few thousand assets across a couple of catalogs, and it lives only
> on my machine — that index isn't published. What's public here is the
> METHOD: how to build a faceted tag schema over any pile of free assets and
> query it fast enough that checking it becomes a habit instead of a chore.
> Paths, catalog locations, and internal tooling are genericized.

# gamedev — a faceted index over your free asset/tool library

The problem this solves: you accumulate free game assets and Godot addons
across projects, and by the third project you've forgotten what you have.
So you rebuild a camera rig, a weather system, or a save-file format that a
free, MIT-licensed addon already solved — or you grab a CC-BY-NC pack for a
game you plan to sell. A single **faceted index** over everything you've
collected fixes both problems: one query surfaces every candidate for "I
need X", tagged with enough metadata (license, art style, theme) to filter
in seconds.

This skill is the query layer. It fronts two narrower catalogs as
"engines" — a local CC0 art library (see `game-assets`) and a Godot
addon/tool adoption log (see `godot-tools`) — and unifies them under one
schema so a single query can span both.

**When NOT to use this pattern:** downloading brand-new assets from the web
(the index only covers what you've already collected — a miss should be
reported, not silently web-shopped); or as a substitute for actually
evaluating a hit against your game's architecture (see Workflow below).

## The facet schema — the actual reusable part

Ten axes, one JSON `taxonomy.json` defining the legal values per axis. Every
asset/tool gets tagged along whichever axes apply:

- **type** (asset kind): 3d-model · 2d-sprite · tileset · texture-material ·
  shader · plugin-addon · tool-editor · template-starter-kit · animation ·
  vfx-particle · audio-sfx · music · font · ui-kit · icon-set · hdri-skybox
- **subject** (what it depicts): character · creature · prop ·
  environment-building · vehicle · weapon · foliage-nature · terrain-feature ·
  tile · icon · ui-element · fx
- **mechanic** (system it provides): camera · character-movement · fps ·
  vehicles-driving · flying-aircraft · racing · weather · terrain · water ·
  sky · dialogue · save · inventory · netcode · input · ai-pathfinding ·
  physics · procgen · cards · dice · board · tower-defense · combat ·
  building-construction · crafting · progression-skilltree · minimap ·
  fog-of-war · unit-selection · targeting-lockon · stealth-detection · …
- **genre**: fps · rts · racing · flight-sim · card · dice-board ·
  tower-defense · platformer · rpg · space-sim · survival · puzzle · horror ·
  roguelike · metroidvania · deck-builder · …
- **art-style** (the look): low-poly · realistic · pixel · voxel · stylized ·
  cartoon · retro · hand-painted · pbr
- **theme** (the setting): sci-fi · fantasy · city-modern · nature · space ·
  cyberpunk · steampunk · post-apocalyptic · medieval · western · military ·
  underwater · …
- **tech**: rigged · static · low/mid/high-poly · pbr · engine-ready ·
  generic-mesh
- **license**: safe · safe-w-credit · caution · **never**
- **source**: which catalog/scrape it came from (asset-library, a store
  scrape, a bundle purchase, etc.)
- **adoption**: adopted · evaluated · rejected · candidate (fed from the
  adoption log — see `godot-tools`)

A guessed facet value that isn't in the schema should return nothing —
that's a feature, not a bug: it forces you to check the real value list
(`schema`) rather than silently mistagging.

## License rules (the sell-the-game filter)

- **safe:** MIT, Apache-2.0, CC0, BSD-2/3, MPL-2.0, ISC, Zlib, Unlicense →
  ship freely.
- **safe-w-credit:** CC-BY-3.0/4.0 → attribution line in `CREDITS.md`.
- **never:** GPL/AGPL (viral copyleft), CC-BY-NC (non-commercial),
  CC-BY-SA (share-alike), proprietary → not in a closed commercial project.
- **caution:** LGPL or unknown license → verify from the source repo's
  actual LICENSE file before using; don't assume.

Tag `license` on every entry so a query can pre-filter to `license=safe`
before you even look at the candidates.

## Building the index

1. **Collect metadata** from each catalog you have (a bundle's manifest, a
   scraped store listing, a folder of models) into one JSON array: name,
   description, tags, license, source, path.
2. **Tag against the schema.** A small script maps keywords in each item's
   name/description/existing tags to facet values — e.g. "camera" or
   "orbit" in the description → `mechanic=camera`. Keep this mapping as an
   editable lexicon (a dict of keyword → facet value), not inline logic, so
   fixing a miss is a one-line edit.
3. **Write a golden-query test set** — a handful of `(query, expected hits)`
   pairs covering your most common lookups. Run it after every retag; treat
   under ~90% pass rate as a real regression, not noise.
4. If you also generate a browsable graph (Obsidian-style markdown, or any
   static site with cross-linked category pages), add a link-integrity check
   too — zero orphaned pages, zero broken links.

## Querying (the fast path)

A small Python CLI over the JSON index, run from anywhere:

```bash
python gamedev/game_toolkit.py schema     # list every facet axis + legal values
python gamedev/game_toolkit.py query "<free text>" --facet <axis>=<value> [--facet ...] [--limit N]
```

Compose facets freely — hybrid games need cross-facet queries:

```bash
game_toolkit query --facet mechanic=camera
game_toolkit query --facet type=3d-model --facet theme=sci-fi --facet art-style=low-poly
game_toolkit query "weather" --facet mechanic=weather
game_toolkit query --facet license=never          # what to actively avoid
```

A query that returns nothing: check `schema` for the exact facet value, fix
and retry once, then report the miss honestly rather than inventing a
plausible-sounding asset name.

## Workflow (planning a game)

1. **List the systems the game needs** from its design notes — camera,
   movement, terrain, save, weather, netcode, plus the art it needs.
2. **Query per system and per art need.** `--facet mechanic=<system>` for
   tools; `--facet type=3d-model --facet theme=<t> --facet art-style=<look>`
   for art.
3. **Shortlist** by license (safe first) → rating/recognition → prior
   adoption status. Check the adoption log for past REJECTIONS of the same
   category first — someone already spent the research time.
4. **Evaluate honestly against the architecture.** A flat-heightmap terrain
   tool does not fit a spherical/procedural planet; a ten-line hand-roll
   can beat a heavy addon. Don't install something just because it exists.
5. **Adopt and log it** — copy art with credit, install tools with their
   license vendored, and record the decision (adopted *and* rejected —
   rejections save the next project's research time).

## Known gaps / roadmap

Coverage is only as good as what's scraped. Natural next sources to fold
into the same schema, one at a time (scraper → tag → rebuild):
Quaternius, Poly Haven, ambientCG, Mixamo (rigged animations),
OpenGameArt, Freesound (audio), Google Fonts.

## Self-improvement

Every miss (a real need returns nothing useful, or a mistag) is a signal:
add or fix a keyword in the tagging lexicon, or add a missing value to the
taxonomy, add the case to the golden-query set, rebuild, confirm the tests
and link check both pass, and note the fix.

## Related skills

- `game-assets` — the local CC0 art library this fronts.
- `godot-tools` — the addon adoption/rejection log this fronts.
- `godot-asset` — the decision router for getting one specific 3D object
  into a Godot game (catalog → parametric generation → live modeling).
