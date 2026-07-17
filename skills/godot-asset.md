---
name: godot-asset
description: >
  A router for "I need a 3D object in this Godot game" — picks the cheapest
  source that fits (an asset catalog, then headless parametric generation,
  then live interactive modeling) before reaching for the expensive option,
  then runs the same export/import-verify/place/credit finish regardless of
  which tier produced the asset. Use whenever a Godot game needs a new mesh,
  prop, or model. NOT for 2D sprites, audio, or non-Godot engines. Trigger
  /godot-asset.
user_invocable: true
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. Paths, real game names, and
> internal tooling are genericized; the cost-tiered decision rule and the
> shared "finish" checklist are the real value and are left intact.

# godot-asset — pick the cheapest source for a 3D object, then finish the same way

The trap this avoids: reaching straight for the slowest, most expensive
tool (an interactive 3D modeling session) for something a free catalog
already has, or that a five-second script could generate. One decision,
cheapest option first, then an identical finishing checklist no matter
which tier produced the file.

## Decide the source (top to bottom — stop at the first that fits)

| # | Fits when | Approach | Cost |
|---|-----------|----------|------|
| 1 | A stock model already covers it (crate, corridor, tree, vehicle, generic prop kit) | A free asset catalog you maintain — CC0 bundles, the Godot Asset Library, etc. (see `game-assets` / `gamedev`) | free, instant |
| 2 | A simple low-poly primitive prop (gem, crate, barrel, coin, rock, crystal, pillar, pyramid, shard) | Headless parametric generation — a scripted 3D tool run with no GUI, shape-in / GLB-out (see `blender-asset`) | seconds, no GUI |
| 3 | Bespoke, organic, a very specific shape, or an edit to an existing mesh | Live, interactive 3D modeling — e.g. driving a 3D tool through an automation/scripting bridge so you can build, screenshot, and iterate without manual clicking | slower, needs the modeling tool actually open |

**Rule:** don't skip to tier 3 if tier 1 or 2 covers it — that's the entire
point of routing. Check the catalog first, then the cheap generator, and
reserve interactive modeling for things that genuinely need it.

**Need many at once?** For a batch of tier-2 props, write a manifest (a
list of shape + parameters) and generate the whole batch in one run rather
than one-at-a-time. A dense tier-3 import is usually worth a cleanup pass
(decimate + re-UV) before it goes through the same finish below.

## The finish (identical for every tier)

Whatever tier produced the model file:

1. **Land it** in the project's asset directory, in its own subfolder
   (e.g. `assets/custom/`) — kept separate from vendored/catalog packs so
   it's obvious at a glance which assets are generated vs. sourced.
2. **Verify the engine round-trip.** Import the file headless and confirm
   the engine opens it with zero config errors — don't trust that a file
   that *looks* valid actually imports cleanly. A small verification script
   that runs the engine headless against the file and checks for a clean
   exit is worth keeping around permanently; treat any import failure as a
   hard stop, not a warning to note and move past.
3. **Credit it.** Append one line to the project's `CREDITS.md` /
   `ATTRIBUTION.md` naming the source — catalog pack + license, or "own
   work / generated" for tiers 2 and 3.
4. **Report** the name, rough polygon count, and where it landed. Don't
   commit to version control as a side effect of this — that's a separate,
   explicit step.

## Worked example

"The game needs a fuel barrel, a mineable ore rock, and a custom broken
antenna."

- Barrel → tier 2, a parametric barrel shape.
- Ore rock → tier 2, a parametric rock shape.
- Broken antenna (bespoke, irregular) → tier 3, interactive modeling.
- All three: land in `assets/custom/`, verify the engine import cleanly,
  append credits, report polygon counts.

## Ambiguity rule

If the target project, the object, or where it should go is unclear, stop
and ask — don't guess a project, invent an asset directory, or model
something unspecified. If two tiers could both plausibly work, prefer the
cheaper one unless a bespoke result was explicitly requested.

## Never-do

- Don't reach for tier 3 when a catalog asset (tier 1) or a parametric
  primitive (tier 2) already fits — that defeats the entire router.
- Don't skip the engine-import verification step — an unverified model file
  isn't actually done.
- Don't verify against whatever engine binary happens to be first on PATH;
  package-manager shims (see Gotchas) can silently produce false results.
- Don't overwrite an existing asset without confirming that's intended.

## Stop conditions

- An import verification failure → stop, report exactly which stage failed.
- Tier 3 needs the modeling tool already running with its automation bridge
  connected — if that's not set up, say the exact setup steps and stop;
  don't fake the result.
- No more than two retry attempts per asset before reporting the failure
  and moving on.

## Gotchas

- **Package-manager engine shims lie.** A Windows `winget`-installed Godot
  (or similar shim on other platforms) has shipped as a broken stub before,
  and even when it works it can silently produce different import results
  than the real console-subsystem binary. Pin an explicit path to the real
  engine executable for verification, not whatever resolves first on PATH.
- A headless import can hang on exit even after the import itself succeeds
  — a verification script should bound-kill the process (e.g. after ~2
  minutes) and still check the resulting scene file, rather than waiting
  forever on a process that isn't going to exit cleanly.

## Related skills

- `game-assets` / `gamedev` — tier 1, the catalogs to check first.
- `blender-asset` — tier 2, headless parametric generation.
