---
name: blender-asset
trigger: /blender-asset
description: >
  Generate a custom low-poly 3D asset with headless Blender (parametric shape
  -> GLB), verify it imports cleanly into Godot, and drop it into a target
  project with attribution logged. Use when you need a BESPOKE prop that is
  NOT in your existing asset catalogs — "make a low-poly gem/crate/crystal/
  coin/rock", "generate a custom 3D model for this game", "I need a barrel
  mesh." Runs fully headless (no Blender GUI, no clicks). NOT for pulling
  existing catalog art (check your asset library first — cheaper), and NOT
  for organic/arbitrary sculpting or edits to an existing model (use an
  interactive Blender-MCP-style workflow for that).
user_invocable: true
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. Absolute paths to my machine's
> Blender/Godot installs are genericized; the pipeline, the verifier contract,
> and the gotchas are the real ones.

# blender-asset — headless low-poly asset factory

Headless Blender builds a parametric low-poly mesh, exports GLB, and a
stdlib verifier confirms the GLB is valid AND imports cleanly into Godot.
Proven pipeline: Blender (headless) → GLB → Godot `--import` → `.scn`.

**Prefer catalogs first.** Custom modeling is a last resort. Before reaching
for this, check whatever free/CC0 asset library you already maintain — if
something already fits, use it. Only generate bespoke geometry when nothing
in the catalog matches.

## What you need

- Blender 4.x/5.x with Python scripting (`bpy`) — any recent headless-capable
  install.
- Godot 4.x, the **real downloaded executable**, not a package-manager shim.
  On Windows specifically: a `godot.exe` picked up from `winget` is a known
  broken stub for headless import — always point at the actual console binary
  you downloaded from godotengine.org, and verify with `--version` first if
  you're not sure which one is on PATH.
- A generator script and a verifier script (the pattern below assumes you've
  written or adapted equivalents — see "Adapting this" at the end).

## Shapes

`gem crystal crate barrel coin rock pillar pyramid shard` — all single-mesh,
flat-shaded, deterministic. Params: `--color R,G,B` (0..1), `--scale S`,
`--name`.

## Workflow

### 1. Generate (headless)
```bash
"<your-blender-exe>" -b \
  -P "~/.claude/skills/blender-asset/scripts/make_asset.py" \
  -- --out "<project>/assets/models/gem.glb" --shape gem --color 0.2,0.7,0.9 --scale 1.0
```
**Verifier:** stdout contains `EXPORTED_OK` and the `.glb` exists. Else stop.

### 2. Verify (structural + Godot round-trip)
```bash
python ~/.claude/skills/blender-asset/scripts/verify_asset.py \
  "<project>/assets/models/gem.glb" \
  --godot "<your-godot-console-exe>"
```
**Verifier:** stdout ends with `VERIFY_OK`. Any `VERIFY_FAIL ...` = stop,
report the reason.

### 3. Place into the target project
Copy the `.glb` into the project's asset dir (e.g. `<project>/assets/models/`)
and append a line to that project's attribution log
(`<project>/CREDITS.md` or `assets/ATTRIBUTION.md` if one exists):
`- <name>.glb — generated (parametric low-poly <shape>), CC0/own work.`

If no attribution file exists, create `assets/ATTRIBUTION.md` with that line.

**Done means:** `EXPORTED_OK` seen, `VERIFY_OK` seen, `.glb` copied into the
target project, and one attribution line written. State the tri count.

## Batch example (worked)

Situation: "I need low-poly pickups — a coin and a gem — for the Godot game
at `<project>`."

Correct actions:
1. Generate coin.glb (`--shape coin --color 0.95,0.8,0.2`)
2. Generate gem.glb (`--shape gem --color 0.2,0.8,0.95`)
3. Verify both → `VERIFY_OK` each
4. Append both to `<project>/assets/ATTRIBUTION.md`
5. Report: "coin (12 tris) + gem (20 tris) generated, verified, placed."

## Cleanup a dense import — `cleanup_asset.py`

Any GLB from an AI 3D-generation tool (image-to-3D, text-to-3D, or a
Sketchfab download) lands dense and un-UV'd. Make it game-ready — decimate to
a tri budget + smart-UV unwrap — before it goes into a low-poly game:
```bash
"<your-blender-exe>" -b \
  -P "~/.claude/skills/blender-asset/scripts/cleanup_asset.py" \
  -- --in "<path>/raw.glb" --out "<path>/clean.glb" --target-tris 2000
```
**Verifier:** stdout `CLEANUP_OK ... tris <big>-><small>`; then run
`verify_asset.py` on the output. `--ratio 0..1` overrides the tri target.

## Batch from a manifest — `build_manifest.py`

A game declares the objects it needs; all get generated + verified in one
call:
```bash
python ~/.claude/skills/blender-asset/scripts/build_manifest.py \
  "<manifest.json>" --godot "<your-godot-console-exe>"
```
`manifest.json` = `{"out_dir": "...", "assets": [{"name","shape","color","scale"}, ...]}`.
**Verifier:** stdout ends `MANIFEST_DONE <n>/<n> generated, <n>/<n> verified`.
Any FAIL/mismatch → stop, report which asset. Then append CREDITS for the set.

## Ambiguity rule

If the shape, output path, or target project is unclear, or the requested
shape isn't in the supported list, stop and ask — don't guess a shape, don't
invent a target directory, don't create files outside the stated output path.

## Never-do

- Don't run the generator script with bare `python` — it imports `bpy`; it
  only runs *inside* Blender (`blender -b -P ...`).
- Don't verify against a package-manager shim of Godot — false results.
- Don't overwrite an existing `.glb` in a project without confirming — pick a
  new name or ask first.
- Don't hand-edit generated files as "art"; regenerate with new params
  instead — keeps the pipeline deterministic and reproducible.

## Stop conditions

- Generation fails twice with the same error → stop, paste the Blender error,
  ask.
- `VERIFY_FAIL` → stop, report which stage (GLB export vs. Godot import) and
  the reason.
- No polling, no retry loops beyond two attempts.

## Gotchas

- Recent Blender exports GLB fine via `bpy.ops.export_scene.gltf`.
- Godot's `--import` can hang on exit even after a successful import — the
  verifier should kill the process on a timeout and still check for the
  resulting `.scn`/import artifacts, since import usually finishes before the
  hang.
- Cones with `radius2=0` make clean triangle-fan tips — handy for crystal /
  pyramid / shard shapes.
- For a genuinely organic/arbitrary model this parametric generator can't
  make, switch to an interactive Blender workflow instead of forcing it here.

## Adapting this

The core contract is: a headless-Blender script that takes shape params and
emits a GLB + an `EXPORTED_OK` sentinel, and a separate verifier that checks
GLB validity plus a real import round-trip into your target engine, emitting
`VERIFY_OK`/`VERIFY_FAIL`. Any engine with a scriptable headless import step
(not just Godot) fits this same two-stage shape.
