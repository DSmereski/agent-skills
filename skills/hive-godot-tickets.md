---
name: hive-godot-tickets
description: >-
  Author Godot game-dev tickets that a small local coding model can actually
  land unattended — surgical single-file scope, an explicit API contract in
  the ticket body, owner-owned integration, headless probes as the
  pass/fail gate, and a respec-not-rewrite loop when a ticket keeps failing.
  Use when filing tickets for an autonomous or semi-autonomous build crew
  against a Godot project. Trigger /hive-godot-tickets.
user_invocable: true
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. It's distilled from running
> Godot game builds through an autonomous multi-agent dev system, where a
> small local coding model (roughly 30B-class, run through a local inference
> proxy) does the actual code-writing against a ticket board (see
> github.com/DSmereski/artificer for the public branding this system ships
> under). Project names, incident/ticket identifiers, and dates have been
> stripped; the envelope rules and failure-mode lessons — the actual
> hard-won content — are unchanged.

# hive-godot-tickets — author Godot tickets a local model can land

A small local coder model landed well-specified, **single-file creation**
tickets on the first attempt, and failed nearly every attempt at
cross-file integration work — scene wiring, project-config edits,
"pick whichever file makes sense" — even when the ticket was fully
specified and steered afterward. Author tickets to the model's actual
envelope. Don't fight it.

## The envelope (hard rules)

1. **One ticket = one new-or-rewritten script file.** The files-of-interest
   list names exactly that one file. Never a scene file, never the
   project's engine config, never "and related files."
2. **The ticket body *is* the spec**: a full API contract — function
   signatures with types and return shapes, constants, a short example
   call — plus an explicit line like *"do not touch any other file, or any
   file with 'Manager' or 'Test' in the name."*
3. **Integration is owner-owned.** Pick one pre-existing entry point (a main
   scene's root script) and wire it by hand, once. The final integration
   ticket, if you file one at all, rewrites that entry point from an exact
   skeleton pasted directly into the ticket body — never "wire things up"
   as an open-ended instruction.
4. **Probe first, ticket second.** Before filing a ticket, write the
   headless verification probe yourself: a small scene-tree script that
   exercises the new API, prints a pass marker and exits 0 on success, or
   exits nonzero on any failed assertion. Only arm it in your verification
   config once the ticket that creates the target file actually exists —
   an armed probe for a script that doesn't exist yet fails every earlier
   ticket's verification run too.
5. **Gates carry the loop, not self-reports.** Chain checks: does it parse,
   do the declared files exist, do the probes pass, does a frame-entropy
   check catch a blank/static render (tune the "is this frame actually
   different from a blank screen" threshold per project — dark palettes and
   genuinely empty scenes measure very differently). Never accept "should
   work" from the model that wrote the code.
6. **On a failed attempt cap:** steer once (append guidance, don't replace
   the ticket). Still failing → respec tighter, not looser — paste exact
   function bodies into the ticket if that's what it takes — then reset the
   attempt count and requeue. The model writes the code; you sharpen the
   spec. Loop until it's green, don't hand-write the fix yourself as a
   default.
7. **Repo hygiene for automated commits:** no throwaway backup directories
   (they become drift magnets); commit with an explicit inline identity
   (`user.name`/`user.email` flags) since an automated service account
   often has no configured git identity; and never let an automated worker
   run a repo-wide `git add -A` on a project mid-build without pausing the
   board first — it can capture or wipe someone else's uncommitted work.

## Ticket template

```
title: <Script>.gd — <one capability, one sentence>
files_of_interest: ["scripts/<Script>.gd"]
body:
  Create/replace scripts/<Script>.gd EXACTLY to this contract. No other file.
  <constants, if any>
  <function signatures: name, arg types, return type, one line of behavior each>
  Example call: <2-3 line usage snippet>
  VERIFY: run the headless probe for this script — must exit 0.
  FINISH: commit with an explicit git identity, no other files touched.
  Do NOT touch any other file.
```

## Additional lessons from repeated runs

8. **Red-team every probe before arming it.** Run it against (a) a
   deliberately broken fixture — it must fail fast — and (b) a known-good
   reference implementation in an isolated copy of the project — it must
   pass. Engine-specific traps that faked a passing result: a load call
   that returns a non-null-but-broken object (guard with an
   "is this actually instantiable" check, or the probe hangs to a fake
   timeout instead of failing); a check that passes purely by spawn-order
   luck (make placement/ordering deterministic in the probe); and a probe
   that gets armed before its own target ticket exists, which silently
   fails every *earlier* ticket's verification run.
9. **State probes and entropy floors are blind to render-order bugs.** A
   node whose draw call renders underneath its own children can look
   perfect in a state probe and be static on screen — the simulation state
   is correct but nothing visible changed. The gate that actually catches
   this class of bug is a synthetic-input probe: simulate the input event,
   capture a frame, and have a human visually inspect it. Automated state
   assertions alone will miss it every time.
10. **Pre-validate any reference snippet before pasting it into a ticket.**
    If you're about to paste a "known good" code block verbatim into a
    ticket body, run it through the full probe suite in an isolated copy
    first. The automated worker transcribes what you give it — it should
    never be the first thing to actually execute untested code.
11. **There's a transcription ceiling.** Short scripts (tens of lines) land
    on the first attempt reliably; long ones (a hundred-plus lines) fail
    repeatedly even with a correct spec. Split anything bigger into two
    tickets, or plan to hand-finish it yourself.
12. **Assume the automated lane is context-blind.** A background worker
    driven through a ticket board typically does not get the same
    contextual injection (project conventions, prior decisions, style
    notes) that an interactive assistant session gets. Until your pipeline
    explicitly wires that in, treat the ticket body as the *only* channel
    information reaches the worker through — write every ticket
    self-contained, assuming zero outside context.

## Self-improvement

Each use that surfaces a new failure mode: fold it into this file as a
numbered lesson, then persist the update through however you keep your own
skills in sync. Treat "the model failed this ticket" as a spec bug to fix
here, not a one-off to shrug at.
