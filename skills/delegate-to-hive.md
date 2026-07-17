---
name: delegate-to-hive
description: >-
  Use BEFORE taking on a substantive implementation task on any project that
  has an autonomous build board — pause and ask whether the work should
  become a ticket for the autonomous crew to build instead of the assistant
  doing it directly. Triggers when asked to build/add/fix/implement a feature
  on a project the board covers, or when the work is well-specified,
  parallelizable, and not urgent. Not for conversational answers, quick
  edits, or interactive/exploratory work.
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. It's the decision gate that
> sits in front of an autonomous multi-agent dev system — a persistent
> ticket board that local/cloud coding agents build against unattended (see
> github.com/DSmereski/artificer for the public branding this system ships
> under). Project names, endpoints, and internal identifiers have been
> replaced with generic equivalents; the decision heuristic and the
> ticket-creation checklist are unchanged.

# Delegate to the board (board item vs. do it myself)

If you run a background autonomous build pipeline, plenty of tasks a human
hands an interactive assistant could instead be **dropped on the board** for
the crew to build — freeing the assistant, running in parallel/background,
and dogfooding the pipeline. Before grabbing a substantive project task
directly, **decide, and ask.**

## When this applies

A task is a **delegation candidate** when ALL of:

- It's real implementation work (a feature/fix/refactor) on a project that
  has a board project behind it (check the board's project list first).
- It's reasonably **well-specified**, or specable into clear, measurable
  acceptance criteria (a tight spec — see a spec-first planning method like
  the `karpathy` skill).
- It's **not urgent and not interactive** — nobody needs it this minute and
  doesn't need to watch it happen.

Skip (just do it yourself, no ask) when:

- Conversational / a question / research / planning.
- A quick one-or-two-file edit, a config tweak, or something faster to do
  than to spec.
- Urgent, interactive, or exploratory ("let's figure this out together").
- It touches the board/pipeline infrastructure itself, or something the
  autonomous crew can't safely build unattended.

## The ask

When a task is a delegation candidate, before starting, ask plainly:

> "Want me to add this to the board for the crew to build, or handle it
> myself now? Board = autonomous/parallel/background; me = now/interactive."

Give a one-line recommendation:

- **Lean board** when: well-specced, parallelizable, background-friendly, a
  good dogfood case, or you're already busy with other work.
- **Lean direct** when: urgent, needs back-and-forth, small, or the
  autonomous crew has repeatedly struggled with this class of task.

If the human already said which way they want it, don't re-ask — respect it.

## How to add a board item (if delegating)

The board exposes a small task/state API over HTTP, separate from any other
application API on the same service. Auth is typically a short-lived session
token fetched from a token endpoint, or a longer-lived device token.

1. Confirm the right project identifier exists and is **enabled** — fetch
   the board's state/project list first. Only autonomous-build-eligible,
   enabled projects with a valid repo path actually get worked. Pick the
   *real* project (e.g. the shipping app might be `myapp-v2`, not a disabled
   legacy `myapp`). **If no matching enabled project exists, stop and ask —
   never invent an identifier or enable a project yourself.**
2. Create the task with a title, a body that *is* the spec, and measurable
   acceptance criteria — apply a spec-first method (goal, not just task) so
   the crew has something it can actually verify against, not vibes.
3. To make the crew actually work it: assign it to the autonomous worker,
   then move it to the ready/queued state. The dispatcher picks up
   ready-and-assigned tasks (typically one in flight per assignee at a
   time). Leave it unassigned/backlog if it's just queued for later triage.
4. For a big or fuzzy goal, prefer a decompose/plan action if the board
   offers one — a planning step that breaks a goal into a
   dependency-chained set of tickets, rather than filing one giant vague
   ticket yourself.
5. **Verify (done means):** re-fetch board state and confirm the new ticket
   is present with the intended status/assignee. If creation fails, retry
   once; a second failure → report the error verbatim, don't keep hammering.
6. Don't poll after handing it off — it runs asynchronously; results land
   back on the board through its own review/QA/done pipeline. Never sit in
   a status-check loop waiting on it.

## Worked example

Request: "add a dark-mode toggle to the mobile app" — not urgent,
well-specced. Correct: ask the board-vs-direct question, lean board; on
"board," confirm the app's project identifier is enabled in the board's
state, create the ticket with three measurable acceptance criteria, assign
it to the autonomous worker, move it to ready, confirm the ticket appears,
stop. Wrong: creating the ticket under a guessed project identifier, polling
the board every minute afterward, or silently doing the work directly
without asking first.

## Self-improvement

When the board-vs-direct call turns out wrong — something got delegated
that really needed direct interactive work, or something got done directly
that the autonomous crew should have owned — refine the heuristic above,
then persist the update through however you keep your own skills in sync.
