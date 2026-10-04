---
layout:     post
title:      "A Cursor Plugin Worth Reading: pstack Turns a Coding Agent from \"Can Write Code\" into an Engineering Team"
subtitle:   "The bottleneck for coding agents is moving from generation to trust"
date:       2026-10-04
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
    - open_source
lang: en
ref: pstack-coding-agent-engineering-team
---

> I recently read a project in Cursor's official plugins repo: [pstack](https://github.com/cursor/plugins/tree/main/pstack). It is no longer about "how do I write a better prompt so the AI generates more code." It is about a question that matters more every month: **once a coding agent writes code fast enough, how do you make sure what it wrote is reliable?**

That matches something I have been feeling more and more.

The bottleneck in AI coding used to be whether the model could write the code at all. That problem is disappearing quickly. An agent can work for hours, touch dozens of files, run its own tests, start several subagents in parallel, and finish in one night what used to take days.

The new problem is: **do you dare to trust it?**

If three agents are changing your code at the same time:

- Will they write to the same shared state?
- Did they actually understand the existing architecture?
- Did the bug fix reach the root cause, or add a workaround that looks useful?
- The unit tests pass. Does the real product work?
- When the agent says "Done," what is the evidence?
- When several agents produce different designs, which one do you pick?
- After an agent has worked for hours, how do you know which wrong decisions it made along the way?

These are the problems pstack sets out to solve.

## What pstack is

pstack is a Developer Tools plugin in the [cursor/plugins](https://github.com/cursor/plugins) repo, MIT licensed. This post is based on the `main` branch as of 2026-10-04 (plugin version 0.15.9).

The author is Lauren Tan, also known as poteto. The README introduces the author as someone who has worked with millions of lines of code at Meta, Netflix, and Cursor, and who is on the React core team, helping build and maintain React Compiler.

One line near the top of the README sums up the philosophy:

> throughput without quality is not a goal i aspire to. if you want to go fast, go deep first.

So the goal of pstack is not to make the agent write more code. In the README's words, "the goal is not to maximize loc, in fact it's the opposite": **write less code, but higher quality code.**

Installing it is simple:

```bash
/add-plugin pstack
```

On first use, run `/setup-pstack` to pick a reasoning budget and the model for each role. After that, for any task that needs rigor, you mostly type:

```text
/poteto-mode <your task>
```

The interesting part is the design behind `poteto-mode`.

## pstack in one diagram

I read the whole project as these layers:

```text
User Goal
   ↓
poteto-mode
   ↓
Playbook Router
   ↓
Skills / Principles
   ↓
Subagents / Multi-model
   ↓
Implementation
   ↓
Verification
   ↓
PR → Review → Shipping
```

That is already very close to a lightweight **AI Software Engineering Operating System**.

## Layer 1: poteto-mode is a router, not a prompt

A common problem with coding agent skills: the more skills you write, the less the user knows which one to call when.

pstack's answer is direct: **the user only describes the goal, and `poteto-mode` picks the workflow.**

For example:

```text
/poteto-mode this pr has a subtle bug where the scroll drifts every 750ms even when idle.
repro first, then fix and verify.
```

This matches the Bug Fix playbook. If you say:

```text
/poteto-mode build a small feature behind a feature flag. verify it really works.
```

it enters the Feature playbook.

After matching, it opens a todo list whose first items are the playbook's steps, copied in verbatim. A step it decides to skip stays in the list with a one-line `skip: <reason>`. The process is not a suggestion. It is a checklist in plain sight.

The README currently lists **23 playbooks**, including Investigation, Bug Fix, Perf, Hillclimb, Runtime Forensics, Trace Forensics, Feature, Refactoring, Prototype, Visual Parity, Eval, Babysit, Shipping, Autonomous Run, Orchestrate, Session Pickup, Multi-phase Plan, and Opening a PR.

So the user faces a goal, and the agent internally faces a structured workflow. I think this is the first important design choice in pstack:

**Do not ask the user to learn agent workflows. Let a router pick the workflow from the task.**

## Layer 2: engineering experience encoded as playbooks

Look further in and the core implementation turns out to be simple. pstack does not implement a large agent orchestration service. Most of the logic lives in Markdown:

```text
skills/*/SKILL.md
skills/poteto-mode/playbooks/*.md
agents/*.md
```

plus a small number of helper scripts, such as a watcher that tracks PR status.

But this Markdown is not a simple prompt. It reads more like an **executable engineering process**. The Bug Fix playbook has six steps:

```text
1. Reproduce the bug yourself on the matching surface
2. Binary-search the cause: form candidate hypotheses, seed them with how / why, rule them out with runtime evidence each pass
3. Plan the fix: architect first if it crosses a function boundary, delegate implementation to a subagent
4. Verify on the same surface: the original repro now passes
5. Stage the commits so the failing repro lands before the fix in git history
6. Run Opening a PR
```

It also carries a requirement I like a lot:

> Every shipped line traces to runtime evidence.

**Every line shipped with a bug fix should trace back to runtime evidence.** The playbook explicitly calls "this guard might help" a hypothesis, not a fix, and it does not ship. When evidence refutes a hypothesis, the code it motivated gets reverted.

What it wants is not "this guard should help," but:

```text
We observed X
→ tested hypothesis Y
→ changed Z
→ the original repro is gone
```

This is how an engineer debugs.

## Layer 3: process (playbooks) and principles are separate

pstack makes another interesting choice: it separates "how to do the work" from "which principles to follow while doing it."

Playbooks own the steps. Principles are **24 short skills, one rule each**, grouped into core, architecture, verification, delegation, and meta. `poteto-mode` reads the principle index at task start, and other skills reference principles by name. For example:

- **Laziness Protocol**: bias toward deletion and the smallest change that solves the problem, not more abstraction.
- **Model the Domain**: do not express business state through scattered conditionals. Encode the domain in a structure such as a state machine, a typed model, a table or registry, or a reducer.
- **Fix Root Causes**: do not fix symptoms. Reproduce first, then keep asking why until you reach the root cause.
- **Test Behavior, Not Implementation**: call the code the way its users do and assert the result they observe, not internal details.
- **Separate Before Serializing Shared State**: when concurrent actors might write the same file, branch, or state, the first move is not "add a lock." It is to **eliminate the sharing first**. The source even says to treat "we need a lock" as a design smell to check.

That last one fits multi-agent coding especially well. Shared state that looks harmless in traditional code design quickly becomes an engineering problem once agents run in parallel.

## Layer 4: different parallel patterns for different problems

pstack does not just spawn five agents and wait. It defines different parallel patterns for different problems, and I think this design deserves study.

### Arena: several agents on the same problem

When a single attempt would lock the artifact into the wrong shape, use `/arena`:

```text
                    Candidate A
                   /
One Task → Panel → Candidate B
                   \
                    Candidate C
                         ↓
                    Cross Judge
                         ↓
                    Pick Base
                         ↓
                 Graft Best Ideas
                         ↓
                      Verify
```

The point is not voting. It is:

1. Several models get the same prompt and each produces a candidate and a rationale independently
2. One read-only cross-judge, preferably from a different model family than the lead agent, scores each rubric criterion and recommends a base
3. The lead agent reads every candidate end to end, scores them against the rubric, and compares with the judge
4. It picks a base
5. It grafts the parts worth keeping from the other candidates in by hand
6. It verifies the synthesized result

The skill also contains two judgment calls I like. When N candidates converge on the same shape, that is a strong agreement signal, so ship it. When N candidates diverge wildly, the task was under-specified, so reframe and re-run instead of averaging the divergence.

This looks much more like a real software design review than simple majority voting.

### Swarm: trade parallelism for coverage

`Arena` is everyone on the same problem. `Swarm` is mainly everyone on a different slice (it also supports racing several workers on the same brief). The README example:

```text
/swarm check every package under packages/ against its check.sh. one worker per package.
one report.
```

becomes:

```text
package A → Worker A
package B → Worker B
package C → Worker C
package D → Worker D
```

Each worker reports `PASS`, `ISSUES`, or `BLOCKED` with evidence, and the lead agent aggregates one report. A slice with no result is recorded as a gap, and **a gap does not count as a pass**.

I think this pattern fits migrations, large-scale code checks, package validation, and cross-service analysis. In other words: **trade parallelism for coverage.**

### Interrogate: let different models attack your work

Another skill I like is `/interrogate`. It does not ask more agents to keep writing code. It **asks several different models to try, at the same time, to prove your implementation is wrong**.

Every reviewer gets the same intent, the same diff, and the same rubric, then reviews independently. One sentence in the skill states the design intent: the adversarial signal comes from model diversity, not assigned personas.

The lead agent then sorts the findings into four buckets:

```text
Act On
Consider
Noted
Dismissed
```

There is a good detail here: **pstack does not automatically fix everything the reviewers raise.** Agent review, like human review, produces plenty of nitpicks and false positives, so a lead judgment is still needed. The Dismissed bucket is not filler either. It shows what was rejected and why, so you can override the lead where you disagree.

## Layer 5: different models take different roles

This is another part of pstack worth borrowing. Many multi-agent systems are really "the same model × N." pstack clearly prefers **assigning roles by model strength**.

As of 2026-10-04 (v0.15.9), the default mapping in `/setup-pstack` is roughly:

```text
feature / refactoring, bug-fix, perf-issue, hillclimb
        → grok-4.7-xhigh-fast

judgment and prose, hardest tasks
        → claude-opus-5-5-max

reflect tooling
        → gpt-5.6-sol-max

arena runners / architect runners / interrogate reviewers
        → claude-opus-5-5-max, gpt-5.6-sol-max, grok-4.7-xhigh-fast
```

So code delegates go to Grok by default, the hardest changes, prose, and judgment go to Claude Opus, and panels such as Arena, Architect, and Interrogate mix three model families. The defaults change between versions, and you can set your own role → model mapping with `/setup-pstack`.

I think this direction will matter more and more. The core question of multi-agent work may not be "which model ranks first," but:

> **Which model fits this role best?**

Coding, planning, review, research, writing, and debugging may each belong to a different model.

## Layer 6: verification is the most important part of the system

If I could take only one idea from pstack, it would be **Prove It Works**.

The pstack guide says it directly:

> "It compiles" is not evidence.

Compiling is not evidence. The Bug Fix playbook adds that unit tests show branch behavior, not bug absence.

It asks for a check that matches the kind of change:

```text
CLI change         → run the real command
UI change          → walk the changed flow in the running app
Parser / Migration → replay a saved real input
Performance change → compare before / after profiles
Storage change     → read back the value that was written
```

If a check cannot run, the answer is not "it should be fine." The answer is "inconclusive." In the guide's words, a confident reply without evidence should be treated as a red flag.

This looks like a small principle, but I think it decides whether agent coding can really enter production.

## Layer 7: the agent can work all night, but it must leave an audit trail

pstack has a dedicated Autonomous Run playbook. The guide's "overnight contract" looks like this:

```text
/poteto-mode im going to bed. migrate every caller to the new parser in a fresh worktree off <base>.
done means zero old callers, all parser fixtures pass, old api deleted.
keep a decision log. don't ask me before committing.
/loop until done. if you're truly stuck after a few hours, stop and write up why.
```

(`/loop` is a Cursor built-in command, not a pstack skill.)

The agent first states the exit condition as a checkable predicate, then loops:

```text
Check Predicate
      ↓
Smallest Change
      ↓
Verify
      ↓
Progress?
  ↙       ↘
Yes       No
 ↓         ↓
Commit   Discard
  \       /
 Decision Log
      ↓
Next Iteration
```

The key rule: **changes that did not help get discarded**, not left in because "it's already written." Two more constraints are just as practical. A plateau is not a reason to stop, it is a signal to pivot. And the predicate never gets relaxed to declare victory.

Meanwhile `/show-me-your-work` keeps a TSV decision log, one row per decision:

```text
ts
phase
decision
why
evidence
result
```

The evidence cell must be a pointer (a commit SHA, a PR number, `file:line`, a screenshot path), never a paragraph. The log is append-only: a wrong call gets a new row that supersedes it, and history is never edited.

The next morning you do not re-read the agent's whole transcript. You review **which key decisions it made and what evidence backs each one**.

This is already close to **agent observability**.

## Layer 8: turn the agent's past mistakes into system constraints

Two more skills I like a lot: `/correct` and `/reflect`. `/reflect` runs after a long task lands and captures what was learned as an edit to an existing skill. The idea behind `/correct` deserves particular attention.

Suppose the agent keeps making the same mistake. The usual move is to add one more line to `CLAUDE.md` or a rules file: "don't do this again." pstack puts that last, for one reason: nothing fails when an agent skips the docs.

Its order of preference is:

```text
Architecture
    ↓
Types (plus Lint / CI)
    ↓
Tests
    ↓
Documentation
```

So if the agent keeps reaching for an old way of doing something, do not tell it "please don't use this API." **Delete the old way, hide the internals, and make the wrong import fail.** If a state is illegal, do not document that for the agent. **Make the type system unable to express it.** And every new check has to be proven to fail on a real past mistake.

This idea is not specific to agents. It is simply good software engineering. Agents just amplify the problem.

## What I think is most worth learning from pstack

After reading the whole project, what I would borrow is not any single skill, but three design ideas.

### 1. Prompts are becoming software engineering process

Early AI coding focused on prompt engineering. As models get stronger, the prompt itself matters less. What matters is:

```text
Context
Workflow
Verification
Parallelism
Observability
Feedback Loop
```

That is the whole harness. pstack is, at its core, a design for an **agent harness**.

### 2. Multi-agent is about responsibility boundaries, not headcount

Starting 20 agents is easy. The hard part is:

```text
Who designs?
Who implements?
Who reviews?
Who verifies?
Who makes the final call?
Who may change which files?
Who must not share state with whom?
```

Arena, Swarm, Interrogate, Architect, and Orchestrate in pstack all answer the same question:

> **How do you design the org structure of an agent team?**

That question may become as important as traditional software architecture.

### 3. The real bottleneck is moving from generation to trust

Models today rarely hit "I cannot write this code at all." The more common problem is: **it writes too fast, and I do not have time to review.**

When one agent run produces 20 files, a 3000-line diff, and 10 commits, line-by-line human review is less and less realistic. So what needs strengthening may not be generation, but:

```text
Verification
Independent Review
Decision Trace
Reproducibility
Constraint Enforcement
```

That is why I think pstack is worth reading.

## pstack is not for every task

It is a very opinionated engineering system. If you are renaming a variable, fixing a simple CSS rule, or writing a few dozen lines of demo code, running the full architect, arena, interrogate, and verification chain is clearly unnecessary.

Multi-model plus multi-agent also means **higher token and inference cost**.

So I think pstack fits best with:

- Medium and large codebases
- Complex bugs
- Refactoring and migration
- Performance optimization
- Cross-module features
- Long autonomous coding runs
- Projects with a high bar for production quality

Simple tasks should stay light. pstack has a principle for that too: the Laziness Protocol, do less when less is enough. `poteto-mode` itself is designed to apply only when a playbook matches or the task needs rigor, and to stay out of the way otherwise.

## Closing

If you use coding agents such as Cursor, Claude Code, or Codex heavily, I would not necessarily move the whole of pstack into your workflow right away. But I strongly recommend reading the repo. In particular:

```text
poteto-mode
bug-fix playbook
architect
arena
swarm
interrogate
show-me-your-work
correct
```

These skills contain a lot of mature software engineering experience.

More and more, I think the next stage of competition among coding agents will not only be about "whose model can write code," but about:

**Who can organize models into a software engineering system that is stable, verifiable, and able to run for a long time.**

pstack is one of the more representative attempts I have seen in that direction.

## Source project

- GitHub: [cursor/plugins/pstack](https://github.com/cursor/plugins/tree/main/pstack)
- Getting started: [pstack guide](https://github.com/cursor/plugins/tree/main/pstack/docs/guide)
- Install: `/add-plugin pstack`
- Setup: `/setup-pstack`
- Everyday complex tasks: `/poteto-mode <your task>`

The README notes that `/deslop`, `control-cli`, and `control-ui`, which `poteto-mode` references, are not shipped in pstack. They live in the `cursor-team-kit` plugin, which you can install alongside it for the full set.

If you are studying coding agents, agent harnesses, or multi-agent software engineering, this project is worth your time.
