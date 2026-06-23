---
layout:     post
title:      "GPT-5.5 vs Gemini 3.5 Flash vs Claude Fable 5 on Solution Design"
subtitle:   "Models can broaden your options, but naming the constraints is still the engineer's job"
date:       2026-06-21
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - LLM
    - Looker
    - AI
lang: en
ref: ai-models-engineering-design
---

> I handed three of the strongest models the same engineering problem, and all three opened with the same "best practice" — one that happened to step right onto a hidden project constraint nobody had mentioned. A model can broaden your options, do the research, and tidy up a proposal, but it can only reason about the constraints you actually name. This Atlas → Looker sync study convinced me that AI is already an excellent co-pilot for solution design, yet still can't make the final architectural call for you. The efficient path isn't to trust one model blindly — it's to use several to broaden the options, use engineering experience to name the key constraints, and let the strongest model converge.

This post is based on a real engineering research task, comparing how GPT-5.5, Gemini 3.5 Flash, and Claude Fable 5 design solutions for a hard technical problem. The goal: sync the metadata maintained in Apache Atlas into Google Cloud Looker so clients see the latest field descriptions.

## 1. Background: syncing field descriptions from Atlas to Looker

We have a metadata management platform, Apache Atlas, which is the source of truth for field descriptions. The goal is a one-way, on-demand sync pipeline:

```
Apache Atlas
   ↓
Sync Pipeline
   ↓
LookML Model Code
   ↓
Deploy to Looker
   ↓
Client sees latest field descriptions
```

The instruction given to all three models was identical:

```
Goal:
Build a one-way, on-demand sync pipeline that propagates Looker field descriptions
from Apache Atlas (the source of truth) into the LookML model code, then deploys so
client can see the latest descriptions.
```

The models and modes compared:

| Model | Mode | Notes |
| --- | --- | --- |
| GPT-5.5 | xhigh | Thorough; high solution completeness |
| Gemini 3.5 Flash | default | Fast output, weaker on complex engineering judgment |
| Claude Fable 5 | extra + deep-research | Strongest structure and proactive research |

## 2. The real question: how should Looker field descriptions be updated?

On the surface this was "sync metadata from Atlas to Looker," but the real technical disagreement was: how should the descriptions be written back into LookML? Broadly, two approaches.

**Approach 1: edit the LookML source files directly.** A script reads the Atlas API, locates the field in the original `.view.lkml` file, and updates `description` in place.

- Pros: the final LookML structure is the most intuitive; no dependency on refinement include order; consistent with how the existing code is presented.
- Cons: you must parse and rewrite hand-written LookML; it easily breaks formatting, comments, and style; many view files produce scattered diffs; rollback and review are costly.

**Approach 2: generate a standalone LookML refinements file.** The script leaves the original views untouched and generates a separate refinement file:

```lookml
view: +order {
  dimension: status {
    description: "Order status from Atlas"
  }
}
```

- Pros: doesn't touch hand-written code; machine output is centrally managed; clean Git diffs; trivial rollback; the more views you have, the more the centralized approach wins.
- Cons: depends on LookML include order; if the project uses wildcard includes heavily, the order in which refinements take effect can be unstable; needs a one-time hookup in the model files.

**The real fork was never "can we sync it" — it was "whose file does the machine-generated description belong in."**

## 3. GPT-5.5: thorough, but leaning toward the safe landing

GPT's overall flow was clear: read metadata via the Atlas API → update the Looker model → trigger deploy manually. For the update step it offered two options (native refinements vs. editing source directly) and, after researching, recommended **editing the source files directly**.

The core reason: the current Looker code uses wildcard includes heavily, and Looker's own docs state that wildcard includes don't guarantee import order — so a generated refinement layer would inject ordering instability exactly where you most need determinism.

This is the key point: whether a refinement takes effect is tightly coupled to include order; under wildcard includes the refinement file may not load in the right place, leaving it uncertain whether the description override succeeds. **GPT's value is that it didn't stop at "best practice" — it weighed the official mechanism against the actual state of your project.** Its weakness: it ultimately leaned conservative and didn't push further on whether the refinement approach could be kept via an architectural tweak.

## 4. Gemini 3.5 Flash: recommended the best practice, but didn't fully account for the project

Gemini also gave two options: an In-Place Edit, and a generated `atlas_refinements.lkml` (using the `view: +view_name` syntax). It strongly recommended the latter — zero parsing risk, zero Git conflicts, cleaner Git history, physical isolation of machine output from human code. All of that holds on its own.

The problem: **Gemini didn't realize the project uses wildcard includes heavily.** It didn't do enough code search and context analysis, and assumed the refinement approach would work stably. When pressed on how to solve the wildcard-include issue, it first said there was no way, then reversed and backed editing the source directly:

> Solution: pivot to precise Direct Source-File Editing. I fully agree with you: for the current project architecture, editing the source files directly is the better approach.

**It can produce a plausible-looking best practice, but once a constraint shifts, its solution stability and self-correction lag behind.**

## 5. Claude Fable 5: strongest structure, and a better answer once pushed

Claude Fable 5 ran the research with the new `/deep-research` dynamic workflow and gave three proposals:

- Proposal A — Generated refinement layer (recommended)
- Proposal B — Direct source edit via an lkml parser
- Proposal C — Surgical in-place text patch

It too recommended Proposal A at first, and it too didn't proactively flag the wildcard-include issue. The interesting part: once pushed on it, Claude didn't reject Proposal A — it produced a better structural fix —

```lookml
# machine-generated description layer, own directory, after all view includes
include: "/views/**/*.view.lkml"
include: "/generated/atlas_refinements/*.lkml"
```

That is: put all machine-generated refinements in an independent directory and explicitly include them *after* every view include in each model file. Claude explained this finding actually strengthens Proposal A: the project has dozens of dimension views, so Proposals B/C would require pinpointing and editing `description` across dozens of source files, with diffs scattered everywhere and full parse-and-rewrite risking formatting and comment damage — the more files, the more collateral noise. Proposal A keeps every machine-managed description in one isolated directory, touches the original views not at all, concentrates the diff, and rolls back by deleting the generated directory; the more views, the bigger the win. The only cost is a one-time hookup — one extra include line per model file.

**Pushed on the constraint, it didn't overturn Proposal A — it made Proposal A sturdier. That's what structural reasoning buys you.**

## 6. Side-by-side summary

| Dimension | GPT-5.5 | Gemini 3.5 Flash | Claude Fable 5 |
| --- | --- | --- | --- |
| Solution completeness | High | Medium | High |
| Engineering-risk awareness | Strong | Weaker | Strong |
| Fit to project reality | Good | Insufficient | Good |
| Proactive exploration | Strong | Average | Strongest |
| Structured expression | Strong | Medium | Strongest |
| Self-correction after a counterexample | Steady | Passive | Very strong |
| Final solution quality | High | Medium | Highest |

```
                 Same Engineering Goal
                /          |            \
           GPT-5.5     Gemini 3.5     Claude Fable 5
              |            |               |
        Direct source  Generated      Generated layer +
        file editing   refinement     explicit include
              |        (missed         after all views
        risk-aware     wildcard)            |
                      fast but         best long-term
                      less context     maintainability
```

My subjective take: Claude Fable 5 is the strongest overall, especially on complex solution design, structural decomposition, and judging long-term maintenance cost; GPT is also very good — broad in perspective and sensitive to risk; Gemini 3.5 Flash is a tier below — it quickly produces a "generic best practice" answer but doesn't sufficiently recognize specific code structure, project constraints, and hidden risk.

## 7. Three takeaways from this study

**1. Claude Fable 5 has the stronger overall solution-design ability.** Especially after being pushed on wildcard includes, it didn't just overturn its original plan — it found a better compromise: keep the long-term maintenance advantage of a generated refinement layer while resolving Looker's refinement ordering uncertainty with explicit include order. It doesn't only answer "can it be done" — it looks for "the solution best suited to how this system will evolve."

**2. Models still can't replace the engineer's proactive judgment.** Even the best performer, Claude, didn't flag the wildcard-include risk on its own. If the engineer doesn't notice this key point, the model will likely hand you a plausible solution with a hidden landmine. AI can speed up research, broaden thinking, and organize proposals — but the key constraints, the system's actual state, and long-term maintenance cost still need the engineer's active judgment. **A model can only reason about the constraints you name; the one you forget is the one that sinks the whole design.**

**3. Cross-thinking with multiple models, then converging with the strongest, works better.**

```
Let different models analyze independently
   ↓
Find where they disagree
   ↓
Add the key project constraints by hand
   ↓
Hand it to the strongest model to converge
```

GPT is broader and risk-aware; Claude is better at structured solutions and complex reasoning; Gemini is fast, good for quick brainstorming. One model alone is easily pulled toward a single answer; several thinking separately, then comparing the differences, noticeably improves completeness.

## 8. The solution I'd back

If I were to finalize based on this study, I'd lean toward Claude's corrected Proposal A:

```
Atlas API
   ↓
Generate atlas_refinements.lkml
   ↓
Place generated files under an independent directory
   ↓
Explicitly include generated refinements after all view includes
   ↓
Commit / PR review
   ↓
Deploy Looker
```

Core design principles: Atlas stays the source of truth for descriptions; the script never modifies the original LookML views; the machine-generated description layer is managed independently; the model files control include order explicitly; generated content is reviewed via PR diff; and on any problem you roll back fast by deleting the generated directory.

**In one line: don't let the script edit hand-written LookML; let it manage only the machine-generated description layer.**

## 9. Conclusion

The biggest takeaway from this comparison: AI models are already great at assisting complex engineering research, but they can't yet make the final architectural call. Claude Fable 5's structured reasoning and deep-research are genuinely stronger, GPT's breadth and risk awareness are excellent, and Gemini 3.5 Flash has value for fast drafts — but in complex projects you need to verify its conclusions more carefully.

The efficient path isn't blind trust in one model: use several to broaden the options, use engineering experience to identify the key constraints, and let the strongest model converge. The model spreads the options out; the engineer pushes the constraints onto the table — neither can stand in for the other. And that may well become a new workflow for technical solution design.
