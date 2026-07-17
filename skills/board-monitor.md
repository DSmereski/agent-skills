---
name: board-monitor
description: >-
  Watch a running autonomous-agent ticket board and keep every ticket moving.
  Verifies each ticket is MOVING or legitimately blocked, catches SILENT
  stalls, applies a fix ladder (operational fixes inline, code fixes
  delegated, decisions surfaced to the owner), and loops on a timer. Trigger:
  /board-monitor, or "monitor the board", "keep the tickets moving", "make
  sure nothing's stuck". Built Karpathy-style: a runnable verifier plus
  hard-won guardrails. NOT for reviewing ticket content/quality, adding new
  tickets (see delegate-to-hive), or general debugging beyond unsticking.
user_invocable: true
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. It comes out of running an
> autonomous multi-agent dev system — a persistent ticket board that a
> dispatcher and local/cloud coding agents build against unattended (see
> github.com/DSmereski/artificer for the public branding this system ships
> under). Machine paths, ports, internal service names, and incident IDs have
> been replaced with generic equivalents. The classification logic, fix
> ladder, and guardrails are unchanged — they're the actual load-bearing
> content, earned the hard way.

# board-monitor — keep an autonomous ticket board moving

## Goal (the decision this drives)

You shouldn't have to babysit a board that's supposed to run itself. Every
non-terminal ticket must be either **MOVING** or have a **named legitimate
blocker** (unmet dependency, owner-gate, attempt-capped, deliberate pause, or
queued behind capacity). Anything **silently STUCK** — could move but isn't —
gets caught and unstuck automatically. An autonomous board can do a lot on
its own, but it historically couldn't tell when it had quietly wedged; this
closes that gap.

## Layer 2 — the verifier (run this, don't eyeball it)

The probe should live **outside** the repo the board builds against, and
should only hit the board's HTTP API — never touch the database or working
tree directly. That separation matters: if the autonomous crew is building
tickets *in that same repo*, its own repo-wide commit/rollback behavior can
capture or wipe a verifier script sitting in `scripts/`. Keep it in its own
directory (a skill folder, a separate tools repo, whatever is outside the
blast radius) and call it by absolute path.

Run it, read the exit code, don't guess:

```
verifier.py            # human summary + exit code
verifier.py --json      # machine-readable
```

It classifies every open ticket and **exits nonzero only if something is
truly STUCK**:

- **MOVING** — actively building with a fresh heartbeat, or sitting in
  review/QA within a normal window.
- **QUEUED** — ready and unblocked, but its assignee is already busy with a
  live build (capacity, not a stall). Not stuck.
- **BLOCKED** — ready, but its declared dependencies aren't all done yet.
  Not stuck.
- **NEEDS_OWNER** — backlog capped at the max-attempt ceiling, or explicitly
  gated for human review. Surface it, don't touch it.
- **PAUSED** — the board was deliberately paused. Respect that.
- **STUCK** — the only class that needs action: ready-and-unblocked with an
  idle lane that isn't claiming it, an in-progress worker whose heartbeat
  has gone stale (a zombie run), or a review that never resolves.

Thresholds live at the top of the verifier (roughly: unclaimed-too-long,
heartbeat-too-stale, review-too-long — tune these per your own board's
cadence). Pull your system's own anomaly/health surface too, if it has one,
as a second signal. Recovery runs through one hardened recovery script, by
absolute path — never an ad hoc restart.

**Done (per tick) means:** the verifier ran, the exit code was read, every
STUCK ticket got exactly one fix-ladder action (or a NEEDS_OWNER surface),
and the next wake is scheduled.

## The loop (each tick)

1. Run the verifier. If the board's API is unreachable, run the one hardened
   recovery script (hard-kill, restart, validate the health endpoint
   returns 200, resume, nudge any dependent UI to refresh).
2. If anything is **STUCK** → walk the fix ladder below. If everything is
   MOVING/QUEUED/BLOCKED/PAUSED/NEEDS_OWNER → healthy, do nothing.
3. Surface NEEDS_OWNER items to the human — never auto-act on decisions that
   belong to them.
4. Reschedule with whatever wake/timer mechanism your harness offers
   (~20 minutes active, longer if the board is deliberately paused and
   idle). Do **not** poll in a tight loop — schedule and return. If no
   scheduling primitive exists, do one pass, report board state, and stop —
   never busy-wait or sleep-poll.

## Fix ladder (STUCK → action)

If a stall doesn't match any rung below, or you can't tell which rung
applies: **stop and surface it** with the verifier output — do not
improvise a fix or guess at an endpoint.

- **API down / connection refused** → run the one hardened recovery script.
  Cap it at two attempts per session; still down → hand to the human.
- **board paused with ready work** → was the pause deliberate? A pause flag
  usually has no actor attached, so if a human paused it this session,
  **respect it — surface, don't auto-resume.** Only resume if a
  wedge-recovery itself left it paused mid-repair. When unsure: surface,
  don't resume.
- **local-model request failures on a context-size mismatch** — a common
  gotcha with self-hosted inference: if the context window configured on a
  request doesn't match what the model was actually loaded with, requests
  fail unpredictably. Durable fix: pin a safe context size in every request
  builder; if a model is already loaded wrong, unload and reload it with the
  correct size.
- **zombie in-progress ticket** (a stale heartbeat holding a single-flight
  lane) → a background reaper should clear this on its own. If it's known to
  miss cases, don't restart the whole system to clear one ticket — re-drive
  just that ticket on a fresh worker instead (an "unstick" action that
  bypasses the dead run). Never blanket-kill agent processes by name (see
  guardrails). If the reaper keeps missing, that's a bug in the board
  itself — fix it with a coding agent, not a workaround here.
- **ready ticket, idle lane** → call the board's unstick action for that
  ticket once.
- **stuck review** → unstick once, or let a review-timeout auto-promote it.
- **unstick-resistant ticket** (comes back STUCK again after you already
  unstuck it) → do not loop the unstick — that's re-driving a run that's
  already failed once. Surface it as NEEDS_OWNER; it needs a human to
  triage (unsatisfiable criteria, a genuinely broken task, or a reviewer
  hanging on this specific content). Track which slugs you've already
  unstuck this session so you don't re-hit them.
- **capped / owner-gated** → NEEDS_OWNER: surface, don't auto-build.
- **a bug in the board's own code** → diagnose, then delegate the fix to a
  coding agent; verify with tests and the probe before committing.

## Guardrails (every one earned by a real failure — don't relearn them)

- **QUEUED ≠ STUCK.** A ready ticket whose assignee is mid-build is queued
  behind capacity, not stalled. Don't unstick a healthy queue.
- **Dependency-blocked ≠ stuck.** A ready ticket with unmet dependencies is
  correctly waiting.
- **Respect a deliberate pause.** Surface "paused with work," never
  auto-resume a human's pause.
- **Never blanket-kill processes by name.** Your own monitoring session
  likely runs under the same process name as the workers — killing all of
  them kills you too. Reap by specific PID only.
- **Two-writer collision:** before you (or a subagent) edit a repo the
  autonomous crew *also* builds against, pause the board first — an
  in-flight autonomous build's own commit/rollback can capture or wipe your
  uncommitted work.
- **Never run a diagnostic hard-reset (checkout/stash) in the live working
  tree** — it can revert or wipe in-flight work. Recover via a checkpoint
  commit or history if you must undo something.
- **Restart through the supervised recovery script, not a bare background
  start.** A bare start can leave the *old* process alive holding the port —
  new code gets committed but never actually served, so a feature looks
  shipped while a stale process keeps answering requests.
- **Verify a restart actually loaded new code by hitting a route that
  exercises it** — a bare import check only proves the file parses, not
  that the running process serves it. After shipping an endpoint, call it
  and assert the new behavior, not just that a module imports cleanly.
- **Never re-drive a completed subagent** off its own echoed "done"
  notifications — verify its work yourself instead of looping on the echo.

## Layer 3 — environment

- Verifier: lives outside the repo it observes, read-only, HTTP-API-only,
  never mutates the board directly.
- A health/anomaly endpoint, if your system exposes one — treat it as a
  second, server-side signal that backs up the client-side classification.
- One hardened recovery script — hard-kill, restart, validate, resume.
- A loop/wake mechanism, self-paced; fall back to a scheduled-interval skill
  if your harness has one, or a single manual pass if neither exists.

## Worked example

Verifier exits 1: `stuck: build-142 (ready 42m, lane idle)`. Correct action:
call the unstick action for `build-142`, note the slug as "already unstuck
once," and re-run the verifier on the *next* tick — not immediately in a
tight loop. If `build-142` comes back STUCK again next tick, escalate to
NEEDS_OWNER. Wrong action: killing agent processes, restarting the whole
system for one ticket, or unsticking the same ticket repeatedly.

## Self-improvement

Every run that hits a new stall pattern, or a false positive/negative, is a
reason to tune the verifier's thresholds or the fix ladder above — then
persist the update through however you keep your own skills in sync. Treat
this file the same way you'd treat any other piece of code the system
exercises: fix it where it broke, not around it.
