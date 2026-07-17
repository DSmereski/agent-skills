---
name: writing-great-skills
description: >
  Reference for writing and editing Claude Code skills well — the
  vocabulary and principles that make a skill predictable.
---

> **Public / shared version** — trimmed from a private Claude Code setup and
> posted at smereski.com as a reusable pattern. This one is pure reference
> material (no project-specific content to scrub) — vocabulary for thinking
> clearly about how skills are structured, independent of any particular
> codebase.

# Writing great skills

A skill exists to wrangle determinism out of a stochastic system.
**Predictability** — the agent taking the same *process* every run, not
producing the same output — is the root virtue; every lever below serves
it.

## Invocation

Two choices, trading different costs:

- A **model-invoked** skill keeps a description, so the agent can fire it
  autonomously *and* other skills can reach it (you can still invoke it by
  name too). It contributes to **context load** — the description sits in
  the context window every turn. Mechanics: leave model-invocation enabled,
  and write a model-facing description with rich trigger phrasing ("Use
  when the user wants…, mentions…").
- A **user-invoked** skill strips the description from the agent's
  autonomous reach: only a human typing its name can invoke it, and no
  other skill can reach it automatically. Zero context load, but it spends
  **cognitive load** instead — *you* become the index that has to remember
  it exists. Mechanics: disable model-invocation; the description becomes
  human-facing, a one-line summary with trigger lists stripped out.

Pick model-invocation only when the agent genuinely needs to reach the
skill on its own, or another skill needs to reach it. If it only ever fires
by hand, make it user-invoked and pay no context load for it.

When user-invoked skills multiply past what a person can remember, that
piled-up cognitive load is cured by a **router skill**: one user-invoked
skill that names all the others and says when to reach for each.

## Writing the description

A model-invoked description does two jobs — state what the skill is, and
list the **branches** that should trigger it. Every word in it increases
context load, so a description earns even harder pruning than the body:

- **Front-load the leading word.** The description is where the skill does
  its invocation work — put the concept that should fire it first.
- **One trigger per branch.** Synonyms that just rename a single branch are
  duplication — "build features using TDD … asks for test-first
  development" is one branch written twice. Collapse them; keep only
  genuinely distinct branches.
- **Cut identity that's already in the body.** Keep the description to
  triggers, plus any "when another skill needs…" reach clause.

## Information hierarchy

A skill is built from two content types — **steps** and **reference** —
that mix freely: a skill can be all steps, all reference, or both. The core
decision is which to use and where each sits on a hierarchy ranked by how
immediately the agent needs the material:

1. **In-skill step** — an ordered action in the skill's main file: what the
   agent does, in order. Each step ends on a **completion criterion**, the
   condition that tells the agent the work is done. Make it *checkable*
   (can the agent tell done from not-done?) and, where it matters,
   *exhaustive* ("every modified model accounted for," not "produce a
   change list") — a vague criterion invites **premature completion**.
2. **In-skill reference** — a definition, rule, or fact in the main file,
   consulted on demand. Often a legitimately flat peer-set (every rule of a
   review sitting on one rung) — that's a fine arrangement, not a smell.
3. **External reference** — reference pushed out of the main file into a
   separate linked file, reached by a **context pointer**, loaded only when
   the pointer fires. This spans a *disclosed* reference file that still
   ships with the skill through to reference that lives fully outside the
   skill system, reachable by any skill.

A demanding completion criterion drives thorough legwork — the digging the
agent does within the work — whether the skill has steps or not, since
"every rule applied" binds flat reference just as tightly as "every step
done" binds a sequence.

Push too little down and the top of the file bloats; push too much down and
you hide material the agent actually needed inline. That tension is the
whole decision.

**Progressive disclosure** is the move down this ladder — out of the main
file into a linked file — so the top stays legible. Mechanics: a linked
file in the skill's folder, named for what it holds. Some skills are used
in more than one way, and each distinct way is a **branch** — a different
run taking a different path through the skill. Branching is the cleanest
disclosure test: inline what every branch needs, push behind a pointer what
only some branches reach. A context pointer's *wording*, not its target,
decides when and how reliably the agent actually follows it.

Where the ladder decides *how far down* a piece of content sits,
**co-location** decides *what sits beside it* once it's there: keep a
concept's definition, rules, and caveats under one heading rather than
scattered, so reading one part brings its neighbors with it.

## When to split

**Granularity** is how finely you divide skills, and each cut spends one of
the two loads above, so split only when the cut earns it. Two kinds of cut:

- **By invocation** — split off a new model-invoked skill when you have a
  distinct leading concept that should trigger it on its own, or another
  skill needs to reach it independently. You pay context load for the new
  always-loaded description, so that independent reach has to be worth it.
- **By sequence** — split a run of steps when the steps still ahead tempt
  the agent to rush the one in front of it (premature completion). Keeping
  later steps out of view encourages more legwork on the current one.

## Pruning

Keep each meaning in a single source of truth: one authoritative place, so
changing the behavior is a one-place edit.

Check every line for relevance: does it still bear on what the skill does?

Then hunt no-ops sentence by sentence, not just line by line: run the no-op
test on each sentence in isolation, and when one fails, delete the whole
sentence rather than trim words from it. Be aggressive — most prose that
fails this test should go, not get rewritten.

## Leading words

A **leading word** is a compact concept already living in the model's
pretraining that the agent thinks with while running the skill (things like
*lesson*, *fog of war*, *tracer bullets*). Repeated through the text (though
not always necessary — a strong one might only need saying once), it
accumulates a distributed definition and anchors a whole region of behavior
in the fewest tokens, by recruiting priors the model already holds.

It serves predictability twice. In the body it anchors *execution*: the
agent reaches for the same behavior every time the word appears. In the
description it anchors *invocation*: when the same word shows up in your
prompts, docs, and code, the agent links that shared language back to the
skill and fires it more reliably.

Hunt for opportunities to refactor a skill toward leading words. A quality
spelled out at three separate sites (duplication), a description spending a
whole sentence gesturing at one idea — each is a passage begging to
collapse into a single token. Examples:

- "fast, deterministic, low-overhead" → *tight* — one quality restated
  across a phrase, collapsed into a single pretrained word (a *tight*
  loop).
- "a loop you believe in" → *red* — converts a fuzzy gate into a binary
  observable state (the loop goes *red* on the bug, or it doesn't).

You win twice: fewer tokens, and a sharper hook for the agent to hang its
thinking on. Assume every skill you write is carrying restatements that
leading words would retire — go find them.

## Failure modes

Use these to diagnose problems a skill might be having in practice:

- **Premature completion** — ending a step before it's genuinely done,
  attention slipping toward *being done*. Fix in order: sharpen the
  completion criterion first (cheap, local); only if it's irreducibly fuzzy
  *and* you actually observe the rush, hide the post-completion steps by
  splitting the sequence.
- **Duplication** — the same meaning stated in more than one place. Costs
  maintenance and tokens, and inflates that meaning's apparent importance
  past its real rank.
- **Sediment** — stale layers that settle because adding feels safe and
  removing feels risky. The default fate of any skill with no pruning
  discipline.
- **Sprawl** — a skill simply too long, even when every line is live and
  unique. Hurts readability and wastes tokens. The cure is the hierarchy:
  disclose reference behind pointers, split by branch or sequence so each
  path only carries what it needs.
- **No-op** — a line the model already obeys by default, so you pay load to
  say nothing. Test: does it change behavior versus the default? A weak
  leading word ("be thorough" when the agent is already thorough-ish) is a
  no-op; the fix is a stronger word ("relentless"), not a different
  technique.
- **Negation** — steering by prohibition tends to backfire: "don't think of
  an elephant" names the elephant and makes it *more* available, not less.
  Prompt the positive — state the target behavior so the banned one never
  needs saying. Keep a prohibition only as a hard guardrail you genuinely
  can't phrase positively, and even then pair it with what to do instead.
