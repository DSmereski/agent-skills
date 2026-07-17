# agent-skills

Public, trimmed versions of the Claude Code agent skills I run in my own
setup — shared as reusable patterns. Each file is a `SKILL.md` you can drop
into your own agent environment and adapt.

These are the same files served on [smereski.com/skills](https://smereski.com/skills),
where each one has a short card explaining what it does and why.

## Install

```bash
mkdir -p ~/.claude/skills/<name>
cp skills/<name>.md ~/.claude/skills/<name>/SKILL.md
```

Some skills describe small companion scripts (a generator, a verifier, a
query CLI). Those stay local to my machine — the skill file documents the
contract each script has to meet so you can write your own in a few dozen
lines.

## The skills

### Method

| Skill | What it is |
|---|---|
| [karpathy](skills/karpathy.md) | Spec → verifier → environment: a three-layer method for keeping an AI coding agent honest on non-trivial work. |
| [writing-great-skills](skills/writing-great-skills.md) | The vocabulary for making skills predictable: context load, completion criteria, progressive disclosure, failure modes. |

### Gamedev pipeline

| Skill | What it is |
|---|---|
| [godot-new-game](skills/godot-new-game.md) | Scaffold a Godot 4.6 project that boots green on the first headless probe run. |
| [gamedev](skills/gamedev.md) | A faceted index over your free asset/tool library — query by mechanic, art style, theme, license. |
| [godot-tools](skills/godot-tools.md) | Catalog + adoption log for free, commercially-safe Godot addons — including why things were rejected. |
| [game-assets](skills/game-assets.md) | A local, indexed CC0 art library with search-and-copy plus automatic attribution. |
| [godot-asset](skills/godot-asset.md) | Router for getting a 3D object into a Godot game: catalog → parametric generation → live modeling. |
| [blender-asset](skills/blender-asset.md) | Headless Blender: shape parameters in, verified Godot-ready low-poly GLB out. |
| [game-balance](skills/game-balance.md) | Tune game numbers against a measured target — one knob at a time through a headless sim. |
| [playtest-capture](skills/playtest-capture.md) | Golden-frame screenshots of a running build, inspected before a human playtest. |
| [store-listing](skills/store-listing.md) | Store copy + asset checklist, gated by a privacy scrub and a trademark-safety check. |
| [play-policy-review](skills/play-policy-review.md) | A repeatable Google Play compliance audit run before every submission. |

### Hive orchestration

Patterns from running [Artificer](https://github.com/DSmereski/artificer),
my self-hosted autonomous AI dev system.

| Skill | What it is |
|---|---|
| [board-monitor](skills/board-monitor.md) | Keep an autonomous ticket board moving — verifier classification, fix ladder, hard-won guardrails. |
| [delegate-to-hive](skills/delegate-to-hive.md) | The decision gate: hand work to the autonomous crew, or do it interactively? |
| [hive-godot-tickets](skills/hive-godot-tickets.md) | Author Godot tickets a small local coding model can actually land. |
| [dashboard-panel](skills/dashboard-panel.md) | Scaffold a self-registering, relevance-scored dashboard panel plugin from an endpoint. |
| [hive-vault-graph](skills/hive-vault-graph.md) | Query a markdown knowledge vault as a confidence-tagged entity graph. |

### Agent utilities

| Skill | What it is |
|---|---|
| [suno-request](skills/suno-request.md) | Generate music/audio assets for any app or game and land the files where the project needs them. |
| [youtube-transcript](skills/youtube-transcript.md) | Pull a video's transcript from its caption track — no download, no API key. |
| [android-emulation](skills/android-emulation.md) | Boot an emulator, install an APK, and drive the UI entirely over adb. |
| [deploy-android-apps](skills/deploy-android-apps.md) | Build every Flutter app in a workspace and install them to a connected phone in one shot. |
| [competitive-feature-analysis](skills/competitive-feature-analysis.md) | Map your product, research the field, and rank the feature gaps into a build plan. |
| [prompt-injection-defense](skills/prompt-injection-defense.md) | Fence untrusted external content as data, never instructions. |
| [smereski-publish](skills/smereski-publish.md) | Ship a public content site through a mandatory privacy/secret scrub (genericized as "ship-with-privacy"). |

## Notes

- Everything here is my own work. Every file is scrubbed: no machine paths,
  no internal hosts, no personal data. Where the private version pins a local
  script or catalog, the public version documents the pattern and the
  script's contract instead.
- License: MIT.
