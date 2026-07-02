---
layout:     post
title:      "Knowledge Management for Coding Agents: Stop Stuffing Context, Build a Compounding Workflow"
subtitle:   "The Index → Route → Body discipline for on-demand loading"
date:       2026-07-02
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
lang: en
ref: coding-agent-knowledge-routing
---

> In the Coding Agent era, the core of knowledge management is not "remember more." It is "load on demand." What matters is not context size, but context routing: load the right knowledge at the right stage, make every judgment evidence-backed, and gradually turn repeated experience into automated checks. This is a key step from prompt engineering to agent system engineering.

I have been thinking about a question: how should knowledge be managed in a Coding Agent development workflow?

When many teams start using agents, their first instinct is to give the agent more context: more docs, more memory, more rules, more historical cases.

But the result is not always better. An agent is not a human. More context does not automatically make it smarter. It can create new problems:

- It reads stale information
- It gets misled by old cases
- It wastes tokens on irrelevant documents
- It treats "something happened before" as "this must be happening now"
- It emits review comments that sound plausible but lack evidence

So knowledge management for Coding Agents should not only be organized by topic, such as "metrics docs," "security docs," or "test docs."

The more important dimension is access pattern:

- When should this knowledge be loaded?
- What decision does it support?
- If we do not load it, will the current task suffer?

In other words, the core question is not "what do we know?" It is "what does this task need loaded to support judgment?"

## 1. The core pattern: Index → Route → Body

The pattern I like is:

```text
Index → Route → Body
```

That means: read the index first, route second, and only then load the necessary body.

An agent should not read the whole knowledge base at startup. A better approach is:

1. **Index**: by default, load only small indexes, summaries, and trigger conditions.
2. **Route**: classify by task type, changed files, and impact area.
3. **Body**: load the full document only when routing hits.

Take a `team-code-review` review loop as an example. It can parse the PR, get changed files, then classify the change:

- domain model change
- metric change
- locale formatting change
- access control change
- test-only change
- CI/config change

If the PR touches locale formatting files, load the locale spec.  
If the PR touches a shared metric, load shared metric rules and relevant lessons.  
If it only changes test config, do not load the locale spec.

This distinction matters.

A low-quality loop says: "I give the agent every document and hope it can figure it out."

A sustainable loop says: "I ask the agent to classify first, then only load the knowledge needed for the current judgment."

**The first merely piles up context. The second builds a reusable engineering system.**

## 2. Knowledge should be layered

In a Coding Agent workflow, I tend to separate knowledge into several layers. Each layer should have its own loading moment instead of entering the global prompt by default.

## 3. Contracts / Invariants: what must never break

This is the most important layer, and often the earliest one to load.

It answers the question: what must not be broken?

For example:

- access control
- public field name
- API contract
- schema
- module boundary
- data ownership
- cross-system interpretation rules

If a PR changes access-control metadata, the review loop must load the access-control contract before reviewing.

If this kind of issue is wrong, the impact is usually not a local bug. It is a broken system boundary.

But if the PR does not touch these boundaries at all, there is no need to load them.

**Invariants should be loaded early, but only when the relevant boundary is touched.**

## 4. Maps / Indexes: where to find information

A map is not supposed to explain every detail. Its job is to help the agent route.

A good map should be small enough to scan quickly.

For example:

```text
If files under shared_metrics/ change,
load the shared metric rule,
and run the metric validation path.
```

That is much better than:

```text
Here is everything we know about metrics.
```

The first is an executable routing rule. The second is an information pile.

**A map is not the knowledge body. A map is the routing table that keeps the agent from reading the wrong documents.**

## 5. Specs: what correct behavior means

A spec is the correctness oracle.

When a PR claims to change a behavior, the agent should not judge correctness from memory. It should load the corresponding spec.

If a PR changes date formatting behavior, the review loop should load the formatting spec and check:

- whether the new format matches the spec
- whether every affected surface is covered
- whether tests prove the output change is expected

If the spec does not define the expected behavior, the agent should not invent a standard.

A better finding is:

```text
The expected behavior is not defined. Add a spec before treating this as correct.
```

That is more engineering-oriented than judging from experience.

## 6. Procedures / Skills: how this task should be done

A procedure answers: how should this kind of task be performed in this project?

For example, the main code review command should not contain every review checklist. It is better suited for orchestration.

Specific checklists can be split into skills:

- domain field review skill
- formatting review skill
- security/access review skill
- test adequacy review skill
- PR comment posting skill

This keeps the main workflow light, while domain-specific gotchas live inside the corresponding skill.

**The main workflow owns orchestration. Skills own procedural detail.**

That is a key part of making agent workflows maintainable.

## 7. ADRs: why this design exists

The value of ADRs is that they stop agents from "fixing" an intentional design.

Many times, when an agent sees complex code, it instinctively wants to simplify it. Sometimes that is right. But sometimes the complexity is intentional: compatibility, cross-system contracts, performance constraints, or migration strategy.

So ADRs should be triggered when a PR touches:

- established module boundary
- long-lived naming convention
- cross-system contract
- known tradeoff
- compatibility decision

ADRs should not be loaded every time. They should be hit through an index and loaded when related areas change.

Otherwise, the agent can get distracted by historical design discussions in irrelevant situations.

## 8. Plans: temporary scaffolding, not permanent context

Plans are useful, but they are temporary scaffolding.

They fit:

- stacked PRs
- phased migrations
- active refactors
- multi-stage rollouts

But a plan should not stay in the default context forever.

Once the work is complete, durable knowledge should move elsewhere:

- stable behavior → spec
- design rationale → ADR
- repeated failure → lesson / gotcha
- automatically verifiable issue → test / CI check

A simple rule:

```text
Plans expire.
Specs and ADRs survive.
```

**A plan is scaffolding, not the building.**

## 9. Lessons / Rules: only triggerable lessons are valuable

The most common problem with lessons is that they are too generic.

A bad lesson:

```text
Be careful with metrics.
```

A good lesson:

```text
When a PR hides a shared metric,
check whether that metric is reused by multiple surfaces before approving.
```

A good lesson has a trigger.

In a review loop, the agent should scan lesson headers first, not load every lesson body. If changed files hit a trigger, then load the full content. If not, skip it.

That preserves team memory without polluting the context window.

## 10. Gotchas: local traps belong in local procedures

A gotcha is a small trap inside a specific procedure.

For example:

- special rules for legacy field comparison
- workaround for a PR comment posting API
- historical compatibility behavior of a formatter
- boundary cases a certain test category must cover

If only one skill uses a gotcha, put it inside that skill instead of the global prompt.

The rule is:

```text
If only one procedure needs it,
keep it inside that procedure.
```

That keeps the global prompt from becoming bloated.

## 11. Case Records: historical cases are evidence, not default context

A case record captures one specific historical run:

- state
- validation logs
- subagent outputs
- review payload
- final decision
- postmortem notes

Its value is audit, dispute review, and issue tracing.

But it should not become default context for every review.

For example, the last PR's review report should not influence the next PR by default, unless the current task is explicitly resuming the same PR or directly references that historical case.

**Historical cases are evidence, not standing knowledge.**

## 12. Automated Checks: the best knowledge eventually becomes code

The end state of knowledge management is not a longer prompt.

The best knowledge should eventually become executable checks:

- shell check
- lint rule
- unit test
- integration test
- CI gate
- review script
- validation job

A lesson is only a reminder:

```text
Remember to check X.
```

A test is a guarantee:

```text
X cannot regress silently.
```

If the agent manually checks the same issue every time, that knowledge has not finished maturing. It should be upgraded into an automated check.

## 13. The Lead Agent is not a summarizer, but a judge

In a multi-agent review loop, subagents can expand coverage.

For example, one subagent can use an external review tool, another can use internal project review skills, and a third can focus on test coverage.

But the lead agent must not merely merge the results.

It must:

- verify every reported issue
- require a runnable reproduction
- reject findings without evidence
- distinguish project-wide baseline failures from PR-specific regressions
- judge severity
- request confirmation before posting comments
- suggest knowledge updates after the review

Subagents produce candidates.

The lead agent should produce evidence-backed findings.

This is critical. Otherwise, multi-agent review easily becomes "several agents producing noise together."

## 14. A sustainable knowledge loop

The real value of a Coding Agent workflow is not that it reviews one PR for you. It is that it can keep compounding knowledge.

The ideal loop looks like this:

```text
incident
  → distilled lesson
  → routed skill gotcha or review rule
  → automated check
  → less prose in the prompt
```

In practice:

1. An incident or review issue appears
2. A human distills it into a lesson
3. The lesson is placed into the right skill or rule
4. If it repeats, it becomes an automated check
5. The prompt becomes shorter because the judgment has been systematized

That is the key to agent loop compounding.

Do not make the agent remember everything. Make it:

- load the right knowledge at the right time
- judge from evidence
- convert repeated human experience into executable checks

## 15. My take

Many AI Coding practices are still stuck at "give the agent more context."

But from an engineering-system point of view, the real issue is not context size. It is context routing.

An agent should not feel like a junior engineer buried under documents. It should behave like an engineering system with process, indexes, and verification.

I increasingly think that high-quality coding agent workflows will compete on three things, not only model capability:

- knowledge layering
- on-demand loading
- automated verification

The model generates candidates. The workflow constrains the path. The verification mechanism proves the result.

Without these structures, a stronger agent may simply produce more uncertain output faster.

With these structures, an agent can move from a one-off tool to a compounding engineering system.

## 16. Summary

The goal of knowledge management for Coding Agents is not to make them remember everything.

The goal is:

- load the right knowledge at the right stage
- make every judgment evidence-backed
- gradually turn repeated experience into automated checks

This is a key step from prompt engineering to agent system engineering.

**A mature agent workflow is not a very long prompt. It is a system that routes, verifies, and learns.**
