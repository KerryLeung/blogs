---
layout:     post
title:      "Writing Goals That Keep Coding Agents on Track"
subtitle:   "Disambiguation discipline stolen from a spec written to be read by AI"
date:       2026-06-22
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
lang: en
ref: goal-writing-for-coding-agents
---

> When Claude Code or Cursor produces bad work, it's usually not that the agent is dumb — it's that the Goal was only unambiguous inside your own head. Reading your own words, your brain auto-fills the gaps from context you already hold, so you never see those gaps; the agent doesn't have that context, and fills the same gap with something else. OpenAI's open-source [Symphony Service Specification](https://github.com/openai/symphony/blob/main/SPEC.md) is a ready-made antidote — it knows its reader is an AI, so it makes disambiguation a first-class design principle. This post pulls out its six devices and turns each into a rule you can use the next time you write a Goal.

## A misdiagnosed cause

When we post-mortem a task an agent botched, the first instinct is "this model isn't good enough." But more often the real problem is upstream: **an agent going off the rails usually isn't dumb — your Goal was only unambiguous inside your own head.**

These are two things that "look identical from the inside": *a Goal that's clear to its author*, and *a Goal that's unambiguous to a reader who lacks the author's context*. On the page they look the same, because when you read your own words you auto-complete your own gaps; they only diverge the moment a reader who doesn't share your context (the agent) fills the same gap with something else. The problem isn't the agent's intelligence — it's the ambiguity you can't see.

The Symphony spec I read recently is a systematic model for fixing exactly this. It's OpenAI's open-source specification for an orchestration service that schedules coding agents. Its most striking quality isn't the system it describes — it's that **the author is aware, start to finish, that the reader is an AI, not a human engineer.** So it moves the disambiguation "fill-in rules" out of the author's head and onto the page. Each of the six devices below flips into a rule for writing your own Goals.

## 1. Strict normative language

The spec uses RFC 2119 `MUST` / `SHOULD` / `MAY` throughout, plus an `Implementation-defined` marker for the gray zones where "you must choose, but multiple answers are valid." An agent reading this won't mistake a suggestion for a mandate, nor ignore a policy that must be stated.

**For your Goals this means: drop words like "ideally," "try to," "remember to" — the ones humans rank by tone — and tag every requirement MUST / SHOULD / MAY, then mark the "your call" spots explicitly as implementation-defined.** Vague emphasis is the first thing an agent guesses wrong.

## 2. Clean abstraction layers

Policy → Configuration → Coordination → Execution → Integration → Observability — six layers, each implementable and testable on its own. The agent doesn't get lost in global dependencies.

**For your Goals this means: cut one big objective into sub-goals that can be delivered and verified layer by layer, instead of dumping one tangled ball of requirements.** A goal that can be layered can be verified in stages; a goal that can't be verified in stages forces the agent to bet the whole thing in one shot.

## 3. Two-track state semantics

The spec explicitly separates two state machines: business/tracker states (`Todo` / `In Progress` / `Done`) and the service's internal scheduling states (`Unclaimed` / `Claimed` / `Running` / `RetryQueued` / `Released`), and says outright "this is not the same as tracker states." Conflating the two is a classic anti-pattern, and the author splits them from the start.

**For your Goals this means: when one word in your requirement carries two meanings ("done" = code written, or shipped?), split it into two names first.** One word with two meanings is a breeding ground for ambiguity, and naming is the cheapest disambiguation there is.

## 4. Reference algorithms as pseudocode

The spec gives pseudocode for the key flows (Section 16 lays out six algorithms in a row), spelling out both the happy path and the error path. The agent doesn't have to reverse-engineer logic from prose, so implementation ambiguity drops sharply.

**For your Goals this means: for procedural requirements, instead of three paragraphs of prose, write five lines of pseudocode or numbered steps — and spell out the error branches.** If you don't write the error path, the agent will improvise one for you.

## 5. The safety model in its own chapter

Three Safety Invariants get their own section (Section 9.5): the agent runs only inside the per-issue workspace path; the workspace path MUST stay inside the workspace root; the workspace key is sanitized. These aren't an afterthought patch — they're a first-principles part of the design.

**For your Goals this means: pull the "must never be violated" constraints out of the requirement pile, list them separately, and write them as invariants ("X must always hold") — not buried in the seventh bullet.** The moment a safety constraint sits next to ordinary requirements, it gets negotiated away as a tradeoff.

## 6. Restrained boundary declarations

The spec states plainly that it is a scheduler / runner and tracker reader, and does *not* write tickets (that logic lives in the workflow prompt and agent tooling). The Non-Goals section is written as seriously as the Goals.

**For your Goals this means: write Non-Goals in earnest. Saying "don't touch X this time, don't opportunistically refactor Y" matters as much as saying what to do.** An agent's scope creep almost always happens along the boundary you didn't draw.

## The six devices as a Goal checklist

Before your next `/goal`, run these six questions:

1. Is each requirement tagged **MUST / SHOULD / MAY**? Are the "your call" spots stated?
2. Can the goal be **delivered and verified layer by layer**, or is it one tangled ball?
3. Any **word with two meanings**? Did you split it into two names?
4. Are procedures written as **steps / pseudocode with error branches**?
5. Are the **inviolable constraints** pulled out as invariants?
6. Did you write the **Non-Goals** — what explicitly not to do?

## Closing: treat the agent like a compiler, not a colleague

Behind all six is the same move: **get the intent out of your head and into structure on the page.** A colleague infers your intent; a compiler executes the letter of what you wrote — and an agent is closer to the latter. RFC 2119's `MUST` is to prose what a type signature is to a comment: it makes the reader's obligations mechanically checkable.

An ambiguous Goal doesn't bill once. It makes the agent guess, you correct, it guesses again — every under-specified goal becomes a round trip, charged per task. Front-loading the disambiguation is a one-time writing cost that amortizes over and over — and it distills into a Goal template of your own.

Next time an agent goes off the rails, don't reach for a different model first. Look back at that one line of Goal: is it actually unambiguous, or only unambiguous inside your own head?

> Source spec: <https://github.com/openai/symphony/blob/main/SPEC.md>
