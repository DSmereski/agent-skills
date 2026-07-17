---
name: game-assets
description: >
  A local, indexed CC0 asset library — e.g. a bought Kenney bundle — searchable
  by theme, with a CLI that copies chosen packs (or single items) into any
  project and auto-logs attribution. Use when prototyping a game and you need
  models fast, instead of hunting the web each time. Trigger /game-assets.
user_invocable: true
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. My actual catalog and its
> exact location are local — what's public here is the METHOD: index a CC0
> art bundle once, then pull from it in seconds instead of re-searching the
> web (or re-downloading the same pack) every project.

# game-assets — a local, searchable CC0 art library

If you've bought or collected a large CC0 asset bundle (Kenney's packs are
the obvious example — commercially clean, no attribution required, huge
variety), the bottleneck stops being "is there free art for this" and
becomes "which of the thousands of files do I actually want, and how do I
get it into this project without a manual file-copy-and-hope." A small
indexed catalog plus a copy command fixes that.

In my case the library is Kenney's "All-in-1" bundle — roughly 47
pack-level 3D kits, several thousand `.glb` models total, tagged by theme.
The pattern works the same at any scale.

**When to use:** you're building or prototyping a game and need models, or
want to know what art is available for a given theme. Reach for a local
catalog like this *before* suggesting a web download.

**When NOT to use:** Godot plugins/tools/shaders (a separate addon catalog
handles that — see `godot-tools`, or the faceted cross-catalog view in
`gamedev`); 2D sprites or audio, if your catalog doesn't cover them yet —
say so rather than forcing a 3D pack to substitute.

## Building the index

1. **Catalog at pack level.** For each pack in the bundle, record: pack
   name, file count, a short theme tag list (space, fantasy, city, vehicles,
   tower-defense, characters, nature, …), and its path.
2. **Assign stable item codes.** Give every individual asset a short,
   case-insensitive, stable code — `<PACKCODE>-<NNN>` (e.g. `SK-017`,
   `TDK-042`). Stable means the same code always resolves to the same file
   across rebuilds, so a teammate (or an agent) can point at one exact asset
   unambiguously without spelling out a full filename.
3. **Write a small CLI** over that index — this is the whole interface:

```bash
# List every theme you can search by
python catalog/assets_cli.py themes

# Find packs for a theme, ranked by model count
python catalog/assets_cli.py search <theme>
#   e.g. space, fantasy, city, vehicles, tower-defense, characters, nature

# Copy a pack's files into a target project (+ auto-logs CC0 credit)
python catalog/assets_cli.py copy "<Pack Name>" <dest_project_dir>
#   -> copies into <dest>/assets/<source>/<Pack>/ and appends a
#      source/license line to <dest>/CREDITS.md

# Resolve + copy a single item by its stable code
python catalog/assets_cli.py get <CODE>                 # -> filename, type, pack, path
python catalog/assets_cli.py copy-item <CODE> <project>  # copy just that one file
```

**Verify after copy (done means):** the destination directory exists and
contains the expected files, *and* `CREDITS.md` gained a new attribution
line. If either is missing, the copy silently failed somewhere — report it,
don't claim success.

## Workflow

1. **Map the game to themes.** An RTS in space → `space`; a tower defense →
   `tower-defense`; a kart racer → `vehicles`/`racing`. Run `search <theme>`
   to see the candidate packs.
2. **Pick pack(s)** by theme fit and file count. If it's ambiguous, show the
   options rather than guessing.
3. **Copy** the chosen pack(s) in. Files land in a source-tagged subfolder
   under `assets/`, and CC0 attribution gets logged automatically — never
   skip this even though CC0 doesn't legally require it; it keeps the
   project's provenance trail clean.
4. In most engines, model files like `.glb` import as usable scenes
   directly — instance them as unit or prop meshes with no conversion step.

## Rules

- CC0 assets are free for commercial use with no attribution required, but
  logging credit anyway is good practice and costs nothing.
- Cataloging at pack level (not per-file) keeps the index small; browse the
  handful of files inside a pack by name once you've picked it — good asset
  packs use descriptive filenames (`craft_racer.glb`, `turret_double.glb`).

## Stop conditions

- `search <theme>` returns nothing useful → report the miss (see
  Self-improvement below); don't invent a pack name or guess a path.
- A named pack or item code isn't found → stop and show the nearest matches
  from `search`/`themes`; never fabricate a code.
- Destination project directory doesn't exist or is ambiguous → stop and
  ask which project; never create a new directory just to receive assets.
- A command errors → retry once, then report the exact error.

## Worked example

"I need space ships for an RTS prototype":

```bash
python catalog/assets_cli.py search space
# -> ranked by count: "Space Station Kit  177 models", "Space Kit  153 models", ...
python catalog/assets_cli.py copy "Space Kit" <path-to-project>
# verify: the pack's .glb files exist under <project>/assets/<source>/Space Kit/
#         and CREDITS.md gained a new line
```

## Browse visually

If your catalog is big enough to be worth eyeballing, a small local web UI
or a static thumbnail gallery pays for itself quickly — much faster than
opening dozens of `.glb` files one at a time to see what's in a pack.

## Self-improvement

When a search misses (a game theme with no good pack match, or a mis-tagged
pack), fix the theme mapping, rebuild the index, and note the fix.

## Related skills

- `godot-tools` — the addon/tool catalog (not art) for Godot projects.
- `gamedev` — the faceted cross-catalog query layer fronting both this and
  `godot-tools`.
- `godot-asset` — the decision router for getting one specific 3D object
  into a Godot game, which checks a catalog like this one first.
