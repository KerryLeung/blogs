---
name: atomic-note
description: Turn one raw work moment — a bug, a design call, an AI-coding decision, or any felt friction — into a structured atomic note built around a named distinction, and harvest the seeds (cost, analogy, evidence, one-liner) a future essay will need. Use this whenever the user wants to capture a work or coding experience: phrases like "note this", "capture this", "make an atomic note", "I just noticed X at work", "this felt off and I want to remember why", or when they describe a single moment worth keeping. Also use when the user is growing a knowledge base / lessons-learned corpus and wants a new entry. Prefer this over a freeform note whenever the goal is a reusable note that could later seed an essay, because the fixed structure forces the abstraction (the distinction) and the harvest that a quick note skips.
---

# Atomic Note

Capture one moment as a note that is still useful six months later — and that an essay can later be built from without reconstructing anything.

## Why this structure

A note that only records what happened is a war story: vivid, but you can't reuse it. A note that names the *distinction* underneath — the two things that looked the same but weren't — becomes raw material you can think and write with later. On top of that, the strongest notes harvest the three things an essay always needs and that are painful to reconstruct months on: the **cost** the moment reveals, the **analogy** it rhymes with, and the one **load-bearing line**. Capture them while the moment is fresh.

One note = one moment = one distinction. If a moment contains several distinctions, write several notes and link them.

## The six core fields

Fill every field. The order matters: concrete first, abstract last.

1. **Scene** — What actually happened, concretely: when, which system, what you did, what the agent/code/teammate did. Specific enough to reconstruct the moment. This is the blood of the note; vagueness here makes everything below hollow.
2. **Friction** — What felt off, surprising, or wrong in the moment. This is the ore you're mining. If nothing felt off, this may not be a note yet — just an observation.
3. **Naming attempt** — Give the pattern a short handle. Bad names are fine; volume produces the occasional good one. Don't fall in love with the label — the value is in field 4, not here.
4. **Distinction** — The core move. What two things that "look the same from the inside" are being conflated? Write the dividing line in one sentence. The strongest distinctions are between two things that *feel identical in the moment but diverge later* — e.g., a fix that removes a symptom vs one that repairs your mental model: both make the test green today, only one helps you in six months. If you can't state the line, the note isn't ready — keep it as a raw observation and come back.
5. **Links** — Which existing notes or concepts does this connect to? This is where the corpus becomes a graph instead of a pile, and where future essays get their depth. Reference note ids or concept names.
6. **Counterexample** — When does this *not* hold? The case where the distinction blurs or reverses. This keeps the note honest and stops it from becoming a slogan.

## Harvest (optional but high-value) — seeds for a future essay

Fill any that apply. These are exactly the parts that are cheap now and expensive to rebuild later.

- **Cost shape** — If the friction is a hidden cost, name it and say *when the bill comes due* and *whether it compounds*. This is the single most important harvest. A one-time cost is just a note; a *compounding* cost is the seed of an entire concept — this is how "X debt" and "X tax" essays are born (a momentary annoyance reframed as interest you pay every week).
- **Analogy** — What does this rhyme with in a domain you already know well? (backpressure, the GIL, Amdahl's law, schema drift, savepoints/checkpoints, idempotency, entropy.) A transferred analogy is the engine of the strongest essays; one good one can power a whole piece. Capture it while it's vivid.
- **Evidence** — Any number, benchmark, study, or external concept that would ground this if you wrote it up? A claim with a number behind it stops reading as hype.
- **Load-bearing line** — The one sentence you'd bold if this became an essay. The quotable take. Often it falls out of the Distinction.

## Process

- Pull whatever you can from the conversation first; only ask the user for fields that are genuinely missing.
- Push hardest on **Distinction**. Most weak notes fail here. If the user can't state it, offer 1–2 candidate dividing lines drawn from the Scene and let them pick or reject.
- If the Friction is a *cost*, always ask the compounding question: does this bill once, or every time? That single question turns a complaint into a concept.
- Keep the user's own words for Scene and Friction — don't sand off the specifics.
- Never invent a counterexample, analogy, or number that isn't real; a forced one is worse than a flagged gap ("counterexample: not yet found").

## Output format

Write a markdown file. ALWAYS use this exact template (omit empty Harvest lines rather than leaving placeholders):

```markdown
---
id: <YYYYMMDD-slug>
date: <YYYY-MM-DD>
tags: [<domain>, <pattern>]
links: [<other-note-ids>]
status: <seedling | growing | ready-to-cite>
---

# <short title naming the pattern>

## Scene
<concrete account>

## Friction
<what felt off>

## Naming attempt
<handle(s), throwaway-ok>

## Distinction
<one-sentence dividing line: X vs Y, often "feels identical now, diverges later">

## Links
<connections to other notes/concepts>

## Counterexample
<when it doesn't hold>

## Harvest
- Cost shape: <what it costs, when the bill comes due, does it compound?>
- Analogy: <what it rhymes with in another domain>
- Evidence: <number / study / external concept, if any>
- Load-bearing line: <the one sentence you'd bold>
```

Filename: `<YYYY-MM-DD>-<slug>.md`. Save into `.claude/knowledge/notes/` if that directory exists; otherwise save to the current working directory and tell the user where it went.

## What good looks like

**Example**

- Scene: Accepted a 40-line agent patch to a Flink `keyBy` that made an alert stop firing; two weeks later a related lag alert fired and I couldn't explain the original fix.
- Friction: I had "fixed" something I never understood.
- Naming attempt: silent-fix / symptom-removal.
- Distinction: *Removing a bug's visible symptom* vs *repairing your mental model of why it occurred* — both make the alert go green; they feel identical the day you ship, and only one leaves you able to debug the next occurrence.
- Links: comprehension-debt, cognitive-surrender.
- Counterexample: For a truly trivial typo-level fix there is no mental model to repair — symptom removal is the whole job.
- Harvest:
  - Cost shape: Bills later, and compounds — every un-understood fix makes the next change to that code another un-understood fix. Interest, not a one-time fee.
  - Analogy: Like silencing a Flink backpressure alarm by widening a buffer instead of fixing the slow operator — the lag is still upstream.
  - Evidence: (look for the Anthropic skill-formation comprehension-gap numbers if writing this up.)
  - Load-bearing line: "A green test proves the symptom is gone, not that you understand why it was there."

## Anti-patterns

- A note with a rich Scene but an empty or generic Distinction. That's a diary entry, not an atomic note.
- A clever name with no dividing line under it. The label is decoration; the distinction is the asset.
- Recording a cost as one-time when it actually compounds. The compounding reframe is the gold — don't leave it on the floor.
- Skipping the counterexample. Without it the note drifts toward a slogan you'll over-apply.
