---
name: distinction-essay
description: Build the skeleton and outline for a concept-driven essay — an original-argument piece organized around either a sharp distinction (two things people treat as one) or a named concept (an "X debt" / "X tax" style coinage that makes an invisible cost concrete) — from a topic or a cluster of atomic notes. Use this whenever the user wants to write an essay, blog post, or opinion article that makes an argument: phrases like "turn these notes into an outline", "help me structure this piece", "I want to write about X", or when they have a pile of notes and a thesis. Also use when a draft reads like hype or like a war story and needs a backbone. Do NOT use this for tool, feature, or model summaries — that is a different genre (feature-summary style); this skill is only for original argument.
---

# Concept-Driven Essay Skeleton

Turn a distinction or a named concept (and the notes around it) into an outline you can fill. The craft rules below are distilled from studying how strong engineering-essayists actually build these pieces.

## Why this works

Essays that read as "high-level" almost always do one of two things: they split a distinction the reader didn't realize they were conflating, or they give a name to an invisible cost so it suddenly feels real and inevitable. Either way the piece sandwiches abstraction between concrete scenes and actionable moves: all abstraction reads as hype, all scene reads as a war story, and the height lives in the round trip between them. This skill front-loads the one decision that makes or breaks the piece, so you don't write 2,000 words around a thesis that turns out to be mush.

## Step 0 — Gate: write the epigraph and name the spine first

Two cheap filters, both mandatory before outlining:

1. **Write the epigraph.** A 3–5 sentence abstract that contains the *whole* thesis — someone should be able to read only it and get the point. If you can't write it, you don't have the piece yet; keep accumulating atomic notes. This is the single cheapest quality filter.
2. **Name the spine in one sentence.** Either the distinction (*X vs Y, which look identical from the inside*) or the concept (*X debt/tax: the cost of ___, which you pay every ___*). If the user can't, pull candidates from their atomic notes (read the `## Distinction` and `## Harvest > Cost shape` fields) and offer 1–3.

No epigraph or no spine → not a drafting problem, a notes problem. Say so.

## Pick the framework

- **Distinction-driven** — when the core insight is *two things people conflate*. Posture: calibration ("which side of the line am I on?").
- **Named-concept** — when the core insight is *an invisible or compounding cost*. Posture: accounting ("here is the bill, here is when it comes due"). This is the natural home for anything you captured as a compounding **Cost shape** in an atomic note.

When in doubt, a cost that compounds → named-concept; a choice that feels the same but diverges → distinction.

## Framework A — Distinction-driven (nine sections)

Each section = a header with its guiding question and a slot for one bold load-bearing line.

1. **Hook + the distinction** — Draw the line in one or two sentences. *What two things is the reader treating as one?*
2. **Where it shows up** — 3–4 concrete, slightly self-incriminating scenes. *Where have you personally crossed this line?* (Pull from atomic-note Scenes.)
3. **The bigger frame** — Connect the distinction to a larger or prior concept; name the mechanism. *This is the mechanism by which ___ happens.*
4. **Why we're exposed** — Causal structure: ~4 reasons this group is unusually vulnerable. *Why here, why us, why now?*
5. **The calibration question** — The single question the reader carries away. *What should they ask themselves in the moment?*
6. **Personal moves** — Individual heuristics. *What can one person do tomorrow?*
7. **Structural moves** — Team / process / tooling scaffolding. *What can be built so the right behavior is the default?*
8. **Reframe to the positive** — The non-bleak version; what the good posture buys you. *Don't end on the warning.*
9. **Meta-close** — Restate the line; say what you want the piece to do. *What one sentence do you want remembered?*

## Framework B — Named-concept ("X debt / X tax") (eight sections)

1. **Title = the coined term; epigraph = thesis.** Put the name in the title. State in the epigraph what it is and what it costs.
2. **Define it, usually via a taxonomy.** Place it beside its siblings so the boundary is crisp (*technical vs cognitive vs intent debt*). Borrowing an existing model or study to anchor the definition is a strength, not a weakness — you sharpen and popularize, you don't have to invent.
3. **Why this one is different.** What makes it special versus its neighbors? *The one your tools can't fix for you*, *the one nobody sees on the dashboard*, etc.
4. **Why it compounds now.** The AI-era twist that makes an old, tolerable problem newly expensive. *We got away with it because ___; that stopped working because ___.*
5. **Connect to your prior concepts.** Sharpen or extend something you (or others) already named. This is where the concept-web pays off.
6. **What it looks like in the wild.** 2–4 concrete symptoms/scenes the reader will recognize in their own repo.
7. **How to pay it down.** Actionable, bolded heuristics. Diagnosis without prescription is incomplete.
8. **Where the value moved.** Zoom out: what's scarce now, what compounds, what to get good at. Close on a forward-looking, often triadic line.

## Minimal viable version

For a first draft or a short post, collapse either framework to three parts: **the spine (distinction or named cost) → one concrete scene → one move.** Ship that, then grow toward the full skeleton as notes accumulate. Early pieces are nodes in a graph; they don't need every section.

## Craft rules (apply under either framework)

- **Epigraph first, always.** Covered in Step 0; it's also the thing readers skim, so it has to carry the argument alone.
- **One load-bearing line per section.** Each section should yield a single bold, quotable sentence — the bit someone screenshots. Build the section around it, don't bury it.
- **Two acts: diagnosis → prescription.** Name the failure mode (grounded in a scene and ideally a number), then give heuristics someone can run tomorrow. Never stop at diagnosis; never prescribe without diagnosing.
- **Ground every abstract claim.** Under each claim put a concrete scene (ideally your own, slightly self-incriminating) or an external study/number. Ungrounded abstraction reads as hype — the thing this genre most needs to avoid.
- **For invisible or compounding costs, commit to a metaphor family.** Accounting (debt, tax, bill, interest, pay down, leverage) or systems (backpressure, bottleneck, throughput, the GIL) make an intangible cost feel concrete and inevitable. Pick one family and stay in it across pieces — the consistency becomes both a brand and an idea-generator (every new invisible cost becomes "another X").
- **Self-link to build a concept-web.** Reference your earlier notes/essays; let the new concept sharpen an old one. A connected corpus compounds — each piece makes the others worth more, and linking is how readers (and your future self) navigate it.
- **State the calibration beat.** Say plainly you're not against the tool/practice; the issue is knowing which side of the line you're on. One or two sentences. It earns trust and pre-empts the lazy alarmist read.
- **Zoom out at the close.** End on where the value moved / what compounds over years / a forward-looking line, often with parallel or triadic structure. The reader should leave holding the macro stake, not just a tactic.
- **Ship rough, on a cadence.** Frequency compounds faster than polish. A shipped B+ essay beats a perfect draft you never publish. Don't let copy-editing block publishing.

## Output format

Write a markdown outline file. Start with an `> ` epigraph block, then the chosen framework's sections as `##` headers. Under each header include the guiding question (as a quote line), a `**Load-bearing line:**` slot, and 2–4 bullets — or, if the user supplied atomic notes, pre-fill by routing each note's parts into the right slot: Scenes → "where it shows up" / "in the wild"; Analogy → the metaphor spine and §3/§2 framing; Cost shape → the "why it compounds" section; Evidence → grounding under claims; Load-bearing line → the section's bold slot. Mark anything pulled from a note with its note id so the user can trace it.

Filename: `<slug>-outline.md`. Save to the current working directory and tell the user where it went.

## Genre check

If the user's real intent is to report on a tool, feature, model, or release, this is the wrong skill — that is feature-summary writing (summary → background → use cases → what changed → personal take → source). This skill is for original argument. When intent is ambiguous, ask which one they want before outlining.

## Worked mini-examples (shape only)

**Distinction-driven.** Spine: *delegating execution while keeping ownership of the reasoning* vs *handing over the reasoning itself without noticing.* Scene (§2): approving a large generated PR on green tests alone. Calibration question (§5): "Am I forming my own view, or adopting the model's wholesale?"

**Named-concept.** Spine: *intent debt — the cost of never writing down WHY the system is the way it is, which you now pay every time a cold agent starts.* Taxonomy (§2): technical debt (in code) vs comprehension debt (in heads) vs intent debt (in artifacts). Why it compounds now (§4): humans absorbed intent over years; agents start every session as strangers, so the un-written why bills you per session. Pay it down (§7): spec the intent not the implementation; treat AGENTS.md as an intent ledger; log decisions where they happen.

These are *shapes*; the writer supplies their own domain's scenes and numbers.
