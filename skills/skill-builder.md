---
name: skill-builder
description: Create new Claude Code Skills with proper YAML frontmatter, progressive disclosure structure, and complete directory organization. Use when you need to build custom skills for specific workflows, generate skill templates, or understand the Claude Skills specification. NOT for editing an existing skill's behavior (edit that skill's own SKILL.md directly) or for one-off scripts that don't need to persist as a skill.
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. The base spec content below
> follows Anthropic's published Agent Skills format; my own contribution is
> the "house rules" section, which I've genericized here (the private version
> references my own tool-level guardrails and internal sync step — those are
> replaced with the general pattern they follow).

# Skill Builder

Creates production-ready Claude Code Skills with proper YAML frontmatter,
progressive disclosure architecture, and complete file/folder structure —
skills Claude can autonomously discover and use across Claude.ai, Claude
Code, the SDK, and the API.

## Quick start

```bash
# 1. Create skill directory (MUST be top-level, no nesting/namespacing)
mkdir -p ~/.claude/skills/my-first-skill

# 2. Create SKILL.md with proper frontmatter
cat > ~/.claude/skills/my-first-skill/SKILL.md << 'EOF'
---
name: "My First Skill"
description: "Brief description of what this skill does and when Claude should use it. Maximum 1024 characters."
---

# My First Skill

## What This Skill Does
[Your instructions here]

## Quick Start
[Basic usage]
EOF

# 3. Restart Claude Code / refresh Claude.ai to pick it up
```

## House rules I apply to every skill I build

- **Stop on ambiguity.** If the skill's purpose, trigger, or target directory
  is unclear, stop and ask — don't scaffold a guessed skill, don't create
  files outside the skill directory.
- **Never overwrite an existing skill dir** without reading its SKILL.md
  first; on a name collision, stop and ask.
- Skills live **only** under `~/.claude/skills/<name>/` or
  `<project>/.claude/skills/<name>/` — nowhere else.
- `name:` should track the directory name (lowercase-hyphenated) — the
  harness lists skills by directory name, so keep frontmatter and dirname
  matched.
- **Harden every SKILL.md for a less-capable model reading it cold:**
  1. a description saying when to use it AND when not to;
  2. copy-pasteable commands with real paths (never "run the appropriate
     script");
  3. a verifier per action + an explicit "done means X" line;
  4. an ambiguity line ("if unsure / target missing, stop and report — don't
     guess or create files");
  5. a never-do list naming the destructive edges, and — if your setup has a
     protected directory (a knowledge base, a vault, a shared store) — a
     tool-level guardrail (a pre-write hook) rather than a prompt-only
     promise not to touch it;
  6. stop conditions (retry caps, no-polling rules, hand-back-to-owner
     triggers);
  7. at least one worked example (situation → correct action);
  8. known gotchas.
- Keep SKILL.md under ~500 lines — push detail into a `references/` subdir
  loaded on demand.
- After creating or editing a skill, if you keep a synced copy elsewhere
  (a shared drive, a second machine, a team repo), resync it.
- **Runnable frontmatter check** (done means it prints `OK`):
  ```bash
  python -c "import io,re,sys; t=io.open(sys.argv[1],encoding='utf-8').read(); m=re.match(r'^---\s*\n(.*?)\n---\s*\n',t,re.S) or sys.exit('FAIL: no frontmatter'); fm=m.group(1); (('name:' in fm) and ('description:' in fm)) or sys.exit('FAIL: missing name/description'); print('OK')" "<path-to-skill>/SKILL.md"
  ```

## YAML frontmatter (required)

Every SKILL.md must start with frontmatter containing exactly two required
fields:

```yaml
---
name: "Skill Name"                    # REQUIRED: max 64 chars
description: "What this skill does    # REQUIRED: max 1024 chars
and when Claude should use it."       # include BOTH what & when
---
```

**`name`** — human-friendly display name, Title Case, concise and
descriptive. `"skill-1"` is not descriptive; a 70-character name is too long.

**`description`** — must state both *what* the skill does and *when* to
invoke it, front-loading trigger words. `"A comprehensive guide to API
documentation"` fails (no "when" clause); `"Documentation tool"` fails (too
vague). Good: `"Generate OpenAPI 3.0 docs from Express.js routes. Use when
creating API docs, documenting endpoints, or building API specifications."`

Only `name` and `description` are read by Claude. Extra frontmatter fields
(`version`, `author`, `tags`, …) are ignored by the matcher but harmless to
include for your own bookkeeping.

## Directory structure

Minimal:
```
~/.claude/skills/my-skill/
└── SKILL.md
```

Full-featured:
```
~/.claude/skills/my-skill/
├── SKILL.md
├── scripts/          # executable scripts Claude can run
├── references/       # deep reference material, loaded on demand
└── resources/        # templates, examples, schemas
```

Skills must sit **directly** under `~/.claude/skills/<name>/` (personal, all
projects) or `<project>/.claude/skills/<name>/` (project-scoped, checked into
git for a team). No nested subdirectories or namespaces.

## Progressive disclosure — why this structure matters

A 3-level system lets you install 100+ skills without a context penalty:

1. **Metadata (name + description)** — loaded at startup for every skill,
   ~200 chars each. This is what makes autonomous skill matching cheap.
2. **SKILL.md body** — loaded only when a skill is actually triggered/matched.
3. **Referenced files** (`references/`, `resources/`) — loaded on-demand as
   Claude navigates into them mid-task.

Keep SKILL.md itself lean (roughly 2–10KB) and push lengthy detail —
troubleshooting tables, full API references, copy-paste templates — into
separate files that the body links to. Claude loads those only if it needs
them.

## Validation checklist

Before calling a new skill done:

- [ ] Frontmatter starts/ends with `---`, has `name` + `description`, no YAML
      syntax errors.
- [ ] Description states both what and when.
- [ ] SKILL.md sits directly in the skill directory, no nesting.
- [ ] Core instructions fit in SKILL.md; advanced/lengthy content lives in
      `references/`.
- [ ] At least one concrete, runnable example.
- [ ] A troubleshooting/gotchas section for known failure modes.
- [ ] Scripts (if any) actually execute; examples work as documented.

## Further reading

The frontmatter/structure spec above follows Anthropic's official Agent
Skills documentation (docs.claude.com → Agents & Tools → Agent Skills) and
the public `anthropics/skills` example repo on GitHub — check those for the
authoritative, up-to-date spec rather than treating this page as canonical.
