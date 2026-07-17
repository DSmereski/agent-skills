---
name: hive-vault-graph
description: >-
  Traverse the entity knowledge graph built on top of a personal notes
  vault. Use when asking "how is X related to Y", "what's connected to X",
  "shortest path between X and Y", "what are the key entities", or "explain
  X" in a relationship/graph sense — as opposed to keyword or semantic
  search over note text. Read-only structural graph reasoning over a
  confidence-tagged edge set.
user_invocable: true
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. It's a read-only graph
> layer sitting on top of a personal knowledge vault (Obsidian-style
> markdown notes) inside a broader autonomous multi-agent dev system (see
> github.com/DSmereski/artificer for the public branding that system ships
> under). Database paths, tokens, and host/port details have been replaced
> with generic equivalents; the endpoint shapes, confidence model, and
> guardrails are unchanged. Note: this pattern is specifically about
> *querying* an already-built graph — pair it with a separate linking
> discipline (auto-adding backlinks to an index note, avoiding orphan
> files) if you also want write-side graph hygiene enforced.

# Vault Knowledge Graph

If your note-taking system extracts entities and confidence-tagged
relationships out of what you write (rather than just full-text indexing
the notes themselves), you get a second, structural way to query it:
"what connects to X," not just "what mentions X." This pattern assumes a
small entity/edge table sitting alongside your notes, with a handful of
read-only endpoints exposed over your existing backend.

## Endpoints

| Question | Shape | Example |
|---|---|---|
| "How is X related to Y?" | `GET /graph/path?from=<slug>&to=<slug>` | `?from=alpha&to=omega` |
| "What's connected to X?" | `GET /graph/neighbors?slug=<slug>[&depth=1]` | `?slug=alpha&depth=2` |
| "Explain X / what do we know?" | `GET /graph/explain?slug=<slug>` | `?slug=alpha` |
| "Key / most-connected entities" | `GET /graph/god-nodes[?limit=10]` | `?limit=5` |
| Graph-wide insights | `GET /graph/insights` | — |

Every call should return HTTP 200 plus JSON. A missing slug is a lookup
miss to resolve, not an error to retry — find the right slug first (below)
rather than hammering the same bad query.

## Direct DB access (fastest for ad-hoc inspection)

If a CLI SQLite tool isn't installed, any language's built-in SQLite
binding works fine. **Always open the database read-only** — a live
writer process typically owns the write-ahead log, and opening for write
from a second process risks corrupting it.

```bash
# List entities and their edge count (adjust the connection string/path for your setup)
python -c "import sqlite3; c=sqlite3.connect('file:/path/to/vault.db?mode=ro', uri=True); [print(r) for r in c.execute(\"SELECT id, kind, title, json_array_length(relationships) FROM entity_page ORDER BY 4 DESC LIMIT 20\")]"

# Show raw relationships for one entity
python -c "import sqlite3; c=sqlite3.connect('file:/path/to/vault.db?mode=ro', uri=True); [print(r[0]) for r in c.execute(\"SELECT relationships FROM entity_page WHERE id=?\", ('alpha',))]"
```

## Via the running API

```bash
TOKEN="<bearer-token>"
BASE="<your gateway base url>"

# Who is connected to "alpha" within 2 hops?
curl -s "$BASE/graph/neighbors?slug=alpha&depth=2" \
  -H "Authorization: Bearer $TOKEN" | jq .

# How is "alpha" related to "omega"?
curl -s "$BASE/graph/path?from=alpha&to=omega" \
  -H "Authorization: Bearer $TOKEN" | jq .

# Explain the "alpha" entity
curl -s "$BASE/graph/explain?slug=alpha" \
  -H "Authorization: Bearer $TOKEN" | jq .

# Most-connected entities (top 10)
curl -s "$BASE/graph/god-nodes?limit=10" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

## Slug format

Slugs are the slugified titles written whenever a note or entity is
created (e.g. `some-entity-name`). If you don't know the exact slug, run a
keyword/semantic search over the vault first, then pass its resolved `id`
here — don't guess at slugs.

## Confidence levels

| Level | Meaning |
|---|---|
| `EXTRACTED` | Directly stated in source text — high confidence. |
| `INFERRED` | Logically implied, not stated outright — medium confidence. |
| `AMBIGUOUS` | Uncertain or contradictory signals — low confidence. |

Surface the confidence level alongside any relationship you report — an
`AMBIGUOUS` edge is a different kind of answer than an `EXTRACTED` one, and
collapsing that distinction misrepresents how sure the system actually is.

## Reindex note

If you add per-chunk embeddings to a system that previously only embedded
whole notes, existing notes indexed before the change will be missing
chunk-level vectors. Either run your reindex path explicitly or let it
backfill lazily on next write, and check the pending count before assuming
it's caught up:

```bash
python -c "import sqlite3; c=sqlite3.connect('file:/path/to/vault.db?mode=ro', uri=True); print(c.execute('SELECT COUNT(*) FROM notes WHERE id NOT IN (SELECT DISTINCT note_id FROM note_chunks)').fetchone()[0])"
```

## When NOT to use this pattern

- For keyword or semantic search over note *bodies* — that's a separate
  full-text/vector search skill, not this one.
- For writing new entities or edges — that should go through your system's
  dedicated write path, never ad hoc from a read-oriented skill like this.
- For graph *writes* generally — that's the responsibility of whatever
  coordinator/pipeline stage populates the graph in the first place, not a
  query-time skill.

## Guardrails

- **Read-only, always.** Never INSERT/UPDATE/DELETE against the graph
  database directly, and never open it without explicit read-only mode.
  If your tool layer already blocks direct writes to the vault at the
  hook/permission level, treat that as a backstop, not a substitute for
  discipline here.
- A 401 is a token problem; a connection failure means the backing service
  is down. Report either — don't try to restart infrastructure from a
  read-only query skill.
- Unknown slug → search for it first; still ambiguous after that → stop and
  ask rather than guessing. Cap slug-lookup attempts at two before
  reporting back instead of retrying indefinitely.
