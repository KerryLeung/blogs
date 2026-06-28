---
layout:     post
title:      "A 149K-Star AI Coding Gem: Skills"
subtitle:   "What matters is not the prompt, but the workflow"
date:       2026-06-28
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
    - open_source
lang: en
ref: ai-coding-skills-workflow
---

> I recently came across an interesting open-source project: [Skills](https://github.com/mattpocock/skills). As of 2026-06-28, the GitHub API showed 149,182 stars. Its author, Matt Pocock, is well known in the frontend and TypeScript world. But what interests me most about this repo is not a specific prompt. It is that the repo turns strong engineering workflows into reusable Agent Skills.

Today I want to focus on one Skill that I think is especially worth studying: [improve-codebase-architecture](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md).

## 1. What this Skill does

In short, it asks an agent to run an architecture diagnosis over a codebase.

It does not simply tell the model, "give me some refactoring ideas." Instead, it defines a concrete process:

- Read `CONTEXT.md` and ADRs to understand domain language and existing decisions
- Use a fixed architecture vocabulary: module, interface, depth, seam, adapter, locality, and leverage
- Look for high comprehension cost, shallow modules, leaking coupling, and hard-to-test areas
- Render candidates into an HTML report
- For every candidate, include files, problem, solution, benefits, before/after diagrams, and recommendation strength
- After the user picks one candidate, enter a grilling loop to clarify the design

The value is that it does not hand "architecture review" to one free-form reasoning pass. It breaks the review into steps that can be repeated.

## 2. Why it is representative

I think this Skill points to an important direction in AI Coding:

**A good Agent Skill is not really Prompt Engineering. It is Workflow Engineering.**

A rough prompt might look like this:

```text
Please analyze this project's architecture and suggest refactors.
```

A more mature workflow defines the sequence:

```text
Collect Context
    ↓
Understand Domain Language
    ↓
Inspect Architecture Friction
    ↓
Generate Candidates
    ↓
Visualize Tradeoffs
    ↓
Grill One Decision
    ↓
Record Context / ADR
```

The first relies on the model improvising. The second encodes the order, checkpoints, and decision constraints of an experienced engineer.

That is why I find it more valuable than "a good architecture-analysis prompt."

## 3. Clear goals reduce agent drift

Many agent failures are not caused by weak model capability. They come from fuzzy goals.

`improve-codebase-architecture` has a very specific target: scan a codebase for deepening opportunities, output a readable visual HTML report, then let the user choose one candidate for deeper discussion.

Two constraints matter:

1. **Only propose candidates first. Do not change code immediately.**
2. **Only after the user chooses one does the design interview begin.**

That keeps the agent from sliding from "discover problems" into "refactor a bunch of files." Architecture work is especially vulnerable to this kind of scope drift: the problem has not been aligned, but the solution is already being implemented.

## 4. Context determines output quality

This Skill starts by reading `CONTEXT.md` and ADRs. That detail matters.

Code tells the agent what the system currently looks like, but many key decisions do not live in code:

- Why are the module boundaries shaped this way?
- Why was this technical approach chosen?
- Why was another approach rejected?
- Which tradeoffs were accepted intentionally?

If that knowledge only lives in people's heads, agents will repeatedly suggest ideas that have already been rejected.

So I increasingly believe this:

**An agent's ceiling is largely determined by the quality of its context.**

More context is not automatically better. The context needs to be complete, accurate, and structured. `CONTEXT.md` captures domain language. ADRs capture historical decisions. Together, they give the agent a more realistic engineering environment.

## 5. Decisions need review

The most interesting part of this Skill is that the architecture report is not the end.

After the report is generated, the user chooses a candidate and the agent enters `/grilling`: it walks through constraints, dependencies, the shape of the module, what should sit behind the seam, and which tests should survive.

That is very close to how we make important design decisions in real engineering work:

```text
Propose a solution
   ↓
Put it under pressure
   ↓
Expose hidden assumptions
   ↓
Adjust boundaries and tradeoffs
   ↓
Only then make a decision
```

Code review matters. Agent decision review matters too. A single reasoning pass can easily confuse "sounds reasonable" with "actually fits this system." A grilling or critique loop forces the agent to surface assumptions and inspect them one by one.

## 6. Results must be verifiable

If a Skill only emits advice, its value is limited.

`improve-codebase-architecture` requires an HTML report, and each candidate must include concrete files, problem, solution, benefits, and before/after diagrams. That makes the output inspectable:

- Does this problem really exist?
- Are the files accurate?
- Does the solution solve the problem, or just rename it?
- Is the benefit large enough?
- Is there a simpler option?

I would pair this kind of architecture analysis with `/grilling` and ADRs to form a complete loop:

```text
Generate report
   ↓
Choose candidate
   ↓
Grill the decision
   ↓
Refine the proposal
   ↓
Record ADR / update domain context
```

The next time an agent enters the repo, it does not have to rediscover the same history.

## 7. Practical suggestions you can use immediately

If you are exploring AI Coding, I would not only run the Skill. I would study it as a workflow design example.

**1. Write an entry goal for complex tasks.**

Do not just write "help me refactor this." A better goal is: "only scan and produce candidate architecture issues; do not edit code; every candidate must include files, problem, solution, risk, and recommendation strength."

**2. Prepare a `CONTEXT.md` for the project.**

Write down the core domain concepts. Make it clear which words are business concepts and which are merely implementation details.

**3. Keep writing ADRs.**

For every important architecture choice, record why you chose it, why you rejected the alternatives, and which tradeoffs you knowingly accepted. ADRs are not only for past readers. They give future humans and agents decision context.

**4. Run a grilling loop for important proposals.**

Ask the agent to do more than produce an answer. Make it accept pressure: what did it miss, which assumptions lack evidence, is the benefit large enough, and is there a simpler solution?

**5. Turn outputs into inspectable artifacts.**

Reports, tables, diffs, ADRs, and issue lists are easier to review and collaborate on than a free-form block of prose.

## 8. A few other Skills worth reading

Besides `improve-codebase-architecture`, I would also look at these related Skills in the repo:

- [`grill-with-docs`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md): grill a proposal while also building domain language and ADRs.
- [`to-prd`](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-prd/SKILL.md): turn the current discussion into a PRD.
- [`to-issues`](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-issues/SKILL.md): break a plan into independently executable issues.
- [`tdd`](https://github.com/mattpocock/skills/blob/main/skills/engineering/tdd/SKILL.md): turn the red-green-refactor loop into an agent workflow.

Together, these Skills show the same direction: make agents less like better chatbots, and more like reliable executors of engineering workflows.

## 9. My take

I increasingly feel that the dividing line in AI Coding is not only model capability. It is whether you can design strong workflows.

We used to keep a lot of engineering judgment in our heads: clarify the requirement first, inspect the architecture with context, review important decisions, verify before refactoring, and write down historical tradeoffs in ADRs.

Now these practices can be encoded as Skills.

In other words:

**The thing worth saving is not the prompt. It is the workflow.**

The future difference between engineers may not be only who writes code faster. It may be who can package good engineering practice into repeatable, composable, reviewable workflows.

## 10. Summary

The valuable part of `improve-codebase-architecture` is not only that it can generate an architecture report. It is that it simulates the full architecture-review process of a senior engineer:

```text
Read context
   ↓
Use shared language to find architecture friction
   ↓
Produce an inspectable report
   ↓
Choose one candidate
   ↓
Question and refine the design
   ↓
Preserve the decision as long-term context
```

If you are exploring AI Coding, I recommend reading the implementation of this Skill, not just running it. The truly valuable part is hidden in the workflow design.

Source repo: [mattpocock/skills](https://github.com/mattpocock/skills)  
Recommended reading: [improve-codebase-architecture/SKILL.md](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md)
