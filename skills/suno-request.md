---
name: suno-request
description: >
  Request/generate any audio (music, loops, menu themes, stingers, ambient beds,
  best-effort SFX) from an AI music-generation API and drop the file(s) wherever
  your project needs them. Use when you (or another skill/agent/build step) needs
  audio assets — "make a background track for X", "generate a menu loop",
  "I need a victory stinger", "create ambient for this scene", or /suno-request.
user_invocable: true
---

> **Public / shared version** — trimmed from a private Claude Code setup and posted
> at smereski.com as a reusable pattern. The skill drives a local audio-generation
> CLI you configure yourself. You supply your own generation backend and credentials;
> no personal data, keys, or internal paths are included here.

# suno-request — generate audio assets for any project

Generate AI music/audio and save it where your project needs it
(e.g. `assets/audio/`, `public/sounds/`, or any target directory). Instrumental
by default, which is what most games and apps want.

## What you need

An audio-generation backend (CLI) that:

1. Accepts a text description + style tags as arguments.
2. Handles auth to your chosen music-generation provider (e.g. via a config file
   that holds your session credentials).
3. Outputs generated audio file(s) to a specified directory.
4. Prints a JSON result on stdout: `{"files": ["<path>", ...], "count": N}`, or
   `{"error": "<message>"}` on stderr when something goes wrong.

The private implementation behind this skill uses a small Python CLI wrapper
around Suno's generation API. You can adapt the pattern to any provider
(Suno, Udio, ElevenLabs Music, etc.) — the skill's workflow is provider-agnostic.

**Configuring your backend:** Store credentials in a local config file (e.g.
`~/.config/audio-gen/config.json` or a project-level `.env`). Never commit
credentials to source control.

## Arguments (passed to the skill)

| Argument | Description | Default |
|---|---|---|
| First text (quoted or unquoted) | Description of the audio | *(required)* |
| `kind:music\|loop\|stinger\|ambient\|sfx` | Shapes prompt + tags | `music` |
| `tags:"..."` | Explicit style tags, overrides kind defaults | *(kind's defaults)* |
| `out:<path>` | Output directory or `.mp3`/`.wav` file path | `~/Music/audio-gen` |
| `format:mp3\|wav` | Output format | `mp3` |
| `vocals` | Allow lyrics/singing | *(instrumental)* |
| `title:"..."` | Optional track title hint | *(none)* |
| `limit:1\|2` | How many generated clips to keep | `2` |

## Kind → prompt and tag guidance

- **music** — full track. Tags like `"<genre>, <mood>"`.
- **loop** — seamless background bed. Add `loop` to tags; describe "seamless
  looping" in the prompt.
- **stinger** — short cue (victory/defeat/level-up). Keep description short;
  tags `"short, cinematic, stinger"`.
- **ambient** — atmospheric bed. Tags `"ambient, atmospheric, <setting>"`.
- **sfx** — best-effort one-shot. AI music models are optimized for music, not
  SFX; results are hit-or-miss. Force instrumental, keep description very short.
  **Tell the user SFX may be a poor fit and a dedicated SFX tool would be better
  for one-shots.**

## Execution workflow

1. Parse the request into: `description`, `kind`, `tags`, `out`, `format`,
   `vocals`, `title`, `limit`.
2. If `tags:` was not provided, build tags from `kind` using the guidance above.
3. Run your backend CLI. Generic shape:

```bash
<your-audio-cli> "<description>" \
  --tags "<tags>" \
  --out "<output-path>" \
  --format <mp3|wav> \
  [--instrumental] \
  --limit <N>
```

   Replace `<your-audio-cli>` with the path or command for your configured
   backend. Add `--vocals` / omit `--instrumental` if the `vocals` flag was given.

4. Parse the stdout JSON. On success, report the saved file path(s) to the user.
   On error, relay the message plainly (common causes: expired/stale session
   token, rate limit, content policy rejection).

## Notes on generation and auth

- Generation typically takes 30–120 s per request (two clips may render in
  parallel if your backend supports it).
- **Session token expiry** — if your provider uses browser-session cookies
  (common with Suno's unofficial API access), these expire. When you see an auth
  error referencing token validation (`422`, `401`, "refresh the page", etc.):
  - This is an auth issue, NOT a payload or model bug.
  - Do NOT retry with different tags or model versions.
  - Refresh the session token in your backend config (usually requires an
    interactive browser step), then re-run.
- Files are real audio — a full mp3 track is typically 3–8 MB. If your game or
  app needs short loops, trim/convert downstream (e.g. with `ffmpeg`).

## Adapting to your stack

The core pattern is:

```
request (description + kind + tags)
  → CLI call with credentials from local config
    → poll / wait for generation
      → download audio file(s) to target path
        → return file paths to caller
```

Any provider with a generation + download flow fits this shape. Key things to
implement in your CLI wrapper:

- Read credentials from a local config file (never from env vars checked into CI).
- Accept `--out <dir>` and write downloaded files there.
- Print `{"files": [...]}` on success, `{"error": "..."}` on failure.
- Handle polling internally so the caller just waits for the process to exit.

## Self-improvement

Each use that hits friction (a kind that needs better tags, an API change, an
output-path edge case) → update this file to capture the new guidance.
