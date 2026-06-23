---
layout:     post
title:      "A New Agent-Harness Practice: Claude Code Dynamic Workflows"
subtitle:   "From prompt-driven to workflow-driven"
date:       2026-06-22
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
lang: en
ref: claude-dynamic-workflows
---

> Anthropic recently shipped Claude Code Dynamic Workflows: instead of a single agent working in one context, Claude Code can write a bespoke harness for the task at hand — a JavaScript workflow that orchestrates multiple subagents. After reading the post and trying it, my main takeaway is this: the things we used to write into prompts ("please check carefully", "don't skip the tests") are turning into structured execution flows. This is a feature walkthrough plus my own thoughts. Source: [A harness for every task: Dynamic Workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code).

## 1. What Dynamic Workflows are

In one line: **Claude Code can write its own harness on the fly, custom-built for the task.** It runs a JavaScript workflow whose special functions spawn and coordinate subagents, rather than cramming everything into a single context window.

Those subagents can have their own context windows and focused, isolated goals; you can decide which model each one uses, and even whether each runs in its own worktree. The workflow is plain JavaScript (JSON, Math, Array, and so on) and supports resuming after an interruption.

The post lists six core orchestration patterns:

- **Classify-and-act**: route tasks to different agents based on a classification.
- **Fan-out-and-synthesize**: split a task into many smaller steps, run an agent on each, then synthesize the results.
- **Adversarial verification**: for each spawned agent, run a separate agent to adversarially verify its output.
- **Generate-and-filter**: generate many candidates on a topic, then filter them by a rubric.
- **Tournament**: have N agents attempt the same task with different approaches and compete; pick the best.
- **Loop until done**: keep spawning agents until a stop condition is met.

## 2. What it's trying to solve

Plain coding tasks are already a strong suit for Claude Code, but as a task gets complex and the chain gets long, a single context starts to strain: everything is crammed into one window and detail overflows or gets lost. Against that backdrop, the post explicitly names three single-context failure modes:

- **Agentic laziness**: on a complex, multi-part task, Claude stops partway and declares the job done after partial progress.
- **Self-preferential bias**: when asked to verify its own output, it tends to prefer its own results — writing and grading its own work has a built-in bias.
- **Goal drift**: across many turns and compactions, fidelity to the original objective, boundary conditions, and special requirements gradually erodes.

Dynamic Workflows are designed to push these down using multiple independent contexts and a structured flow.

## 3. Where it fits

It suits complex, high-value, long-chain tasks rather than simple code edits. The directions the post recommends, plus scenarios of my own:

- Large-scale migrations / refactors
- Deep research and fact-checking (technical docs, multi-perspective review of business/technical proposals)
- Sorting, ranking, and rubric-based selection
- Enforcing one rule across a large body of output (security review, PR risk assessment)
- Root-cause investigation of production issues
- Large-scale triage: tickets, logs, incident records, résumés
- Summarizing common corrections from past sessions
- Lightweight model evaluation

In short: **whenever a task needs decomposition, parallelism, verification, and synthesis, Dynamic Workflows have room to work.**

## 4. What it changes

The biggest shift, I think, is this: **AI coding is moving from prompt-driven to workflow-driven.**

We used to mostly write prompts — "check step by step", "don't skip tests", "please review this." Those are really repeated pleas to the same single context. Dynamic Workflows turn those requests into a more stable execution structure: plan first, then decompose, run multiple agents in parallel, cross-verify with independent agents, synthesize at the end, and loop if the condition isn't met.

That makes the AI feel less like a single assistant and more like a small engineering team you can assemble on demand.

## 5. My own thoughts

After a quick trial, my sense is that the agent's built-in harness ability has gone up another level.

Its most immediate value is relieving context-window pressure on complex tasks: hand different subtasks to different agents with their own isolated contexts, instead of cramming all the information into one window.

More importantly, quality. **Letting the context that did the work also grade it is unreliable; only a separate, independent context escapes the self-certification bias.** One agent implements, another reviews against a checklist, a third synthesizes the verdict — that is exactly the point of the adversarial-verification pattern.

What I think is most worth exploring next is the **Dynamic Workflows + Skills** combination: distill a team's recurring processes into skills — schema-change checks, SQL review, migration checklist, release notes, regression tests, incident analysis, PR-review rubric — then use Dynamic Workflows to chain those skills into a more automated, reusable engineering flow. This may be the key step in an AI assistant evolving from "help me write code" to "help me manage complex engineering tasks."

## 6. Summary

Dynamic Workflows aren't meant to replace the ordinary way of using Claude Code — they give complex tasks stronger organization. For simple tasks, plain Claude Code is enough; for complex ones, especially those that need decomposition, parallelism, verification, and iterative progress, reach for Dynamic Workflows.

I'd recommend trying it against your own real work — the real value isn't just understanding the feature, but finding the usage that fits your team's and your own development flow. Source: <https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code>
