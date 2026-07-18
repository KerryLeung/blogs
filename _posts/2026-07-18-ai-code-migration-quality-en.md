---
layout:     post
title:      "Once AI Makes Code Migration Fast, Quality Assurance Matters More, Not Less"
subtitle:   "Reading Anthropic's large-scale migration playbook: when writing code gets cheap, the bottleneck moves to engineering control"
date:       2026-07-18
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
lang: en
ref: ai-code-migration-quality
---

> The faster AI writes code, the less an engineer can afford to focus on the code itself. The scarce skill is shifting from "getting every line right" to "designing a system that keeps producing correct code": constrain generation with rules, validate the rules with a small experiment, let adversarial agents actively hunt for problems, and let the compiler and tests act as the judge. This is my write-up and reflection after reading Anthropic's "How Anthropic runs large-scale code migrations with Claude Code."

I recently read a piece Anthropic published: "How Anthropic runs large-scale code migrations with Claude Code."

The article describes how they used Claude Code and many AI agents to run several large code migrations, including:

- Migrating a large chunk of Bun from Zig to Rust in under two weeks, producing about 1 million lines of code (across 1,448 files);
- Getting Bun's existing test suite to 100% passing in CI before merge;
- Migrating an internal Python project into about 165,000 lines of TypeScript over a single weekend;
- Using hundreds of agents, 8 phase gates, and 3 rounds of adversarial review, diffing the outputs of the old and new systems line by line.

The article is short, but its density of practices — around rule-writing, task decomposition, quality verification, adversarial review, and cost control — is very high.

## First, my own take

I've done several small-to-medium software service migrations myself, usually in the tens of thousands of lines.

Once you've done a few of these, you realize very clearly: code migration is far more than "translating old code into new code."

The genuinely hard problems usually include:

- How do you finish a migration while the product keeps shipping features?
- How do you avoid implementing the same new feature twice — once in the old system and once in the new?
- How do you split the migration so multiple developers can work in parallel?
- How do you keep the migrated system from developing serious quality problems?
- How do you decide the new system has truly reached feature parity with the old one?

The challenges behind these questions aren't purely about coding ability — they're about project management, architecture, task decomposition, test planning, and risk control.

As AI's coding ability grows stronger, migration speed really can improve by an order of magnitude. But at the same time, the quality-assurance problem can be amplified even further.

When AI can generate tens or hundreds of thousands of lines in a short time, a human can no longer review the whole thing file by file, pull request by pull request.

At that point, the question an engineer must solve shifts from:

> Is every line the AI wrote correct?

to:

> How do I keep the AI from drifting?
> How do I build an objective verification mechanism?
> How do I make errors get discovered, classified, and fixed automatically?
> How do I use AI to run the old-vs-new comparison testing better?

Ordinary engineers rarely get to take part in a million-line language migration. The Bun migration Anthropic disclosed here cost roughly $165,000 at API pricing alone, consuming 5.9 billion uncached input tokens and 690 million output tokens.

So this is not just a product-marketing piece — it can also be read as a very expensive large-scale AI engineering experiment report.

And it genuinely contains many practices you can apply directly to everyday development.

## 1. AI didn't change coding speed — it changed the economics of migration projects

In the past, a large language migration usually meant a multi-year project.

The team had to maintain two codebases for a long time:

- The old system kept carrying production traffic;
- The new system kept filling in features;
- New features might have to be built on both sides at once;
- Even at the end, the two might only reach 90% feature parity.

If the migration failed, the team ended up not with a better system, but with two systems to maintain.

So in the past, only a truly severe problem was enough to justify a large language migration — for example:

- The original language ecosystem was gradually declining;
- There were long-standing memory-safety problems;
- Build speed was seriously hurting release velocity;
- Some architectural bottleneck could no longer be worked around.

AI agents changed this cost model.

Anthropic offers an interesting judgment in the article:

> Now the worst case of a failed migration might just be deleting the branch and doing it again.

This does not mean code migration has become cheap.

The Bun migration still cost about $165,000 at API pricing. But compared with projects that used to run for years and burn millions of dollars of engineering resources, the cost of trial and error has dropped dramatically.

Worth noting: even this *successful* Bun migration saw 19 regressions surface after merge (all since fixed). That's exactly the point of this piece — once migration gets fast, regression and quality verification don't become optional; they become more critical.

So some problems that "weren't worth migrating" before may now be worth recalculating the ROI on.

For example, an internal Python tool at Anthropic used to be built per platform — roughly 8 minutes each — for a total wait of about 30 minutes across the build matrix. After migrating to TypeScript, compile time dropped to about two seconds, startup got 6x faster, and a whole separate deployment pipeline could be retired.

AI doesn't just make migration faster — it lowers the bar for a company to re-evaluate its technical debt.

## 2. The single most important sentence in the article

Anthropic states a core idea in the article:

> You don't fix the code. You fix the process (loop) that produced the code.

This one line can almost summarize the entire AI migration methodology.

In traditional development, when an engineer finds a bug, they usually edit the corresponding file.

But in a large-scale agent migration, if dozens of files exhibit the same class of error, editing them one by one does not solve the real problem.

What actually needs to change might be:

- An incomplete migration rule;
- A missing constraint in the prompt;
- An unreasonable task split;
- A review agent that doesn't cover a certain class of risk;
- A test judge that can't recognize a certain class of error;
- A work queue that doesn't correctly record failure state.

The right move is not to have an engineer hand-edit dozens of files, but to:

1. Find the shared rule that produced this class of error;
2. Fix the rule or the workflow;
3. Regenerate the affected code;
4. Verify again through tests and review.

This is one of the biggest differences between agent engineering and traditional coding.

The engineer's focus starts to shift from "producing code directly" to "designing a system that can keep producing correct code."

## 3. Anthropic's six-step method for large-scale code migration

### Prerequisite: first build a reliable judge

Before the real migration, you first need a "judge" that can evaluate both the old and the new system.

That judge might be:

- An automated test suite;
- A compiler;
- An old-vs-new output comparison tool;
- API contract tests;
- Replay of real business scenarios;
- Performance and resource-usage benchmarks.

A good judge must satisfy two conditions:

First, it passes normally on the old system.

Second, when the system is deliberately broken, it must be able to detect the problem.

The article puts it bluntly:

> A judge that can't detect failure isn't a judge.

In reality, many old projects have tests that rely on internal functions or language features and can't be reused directly in the new language.

So you first have to classify the tests:

- Which can be run through external interfaces;
- Which depend on internal implementation;
- Which need to be reworked into cross-language tests;
- Whether the rewritten tests weakened the original assertions.

Lacking a complete test suite doesn't mean you can't migrate.

The Python-to-TypeScript project built only seven real business scenarios, then ran them on both the old and new systems and diffed the output of every command.

The point here isn't how many tests you have — it's that you must establish an objective standard both sides accept.

The old system itself is the new system's most important ground truth.

### Step 1: build the rulebook, dependency graph, and difference list

This is the most important — and most engineer-time-consuming — phase of the whole migration.

**1. The rulebook**

The rulebook tells all migration agents:

- How types should convert;
- How error handling should be implemented;
- How memory and resources should be managed;
- What patterns to use for logging, config, and dependency injection;
- Which old-language idioms must not be copied directly;
- Which architecture must stay consistent;
- Which parts are allowed to be redesigned.

If the goal is to preserve the existing structure, the rulebook may mostly be a set of language and type mapping rules.

If the migration also involves architectural refactoring, the rulebook is closer to a full design document.

It's not an ordinary prompt — it's an engineering spec every agent must obey.

**2. The dependency graph**

Large-scale migration needs heavy parallelism.

But parallelism requires knowing:

- Which files can be migrated independently;
- Which modules must be handled together;
- Which low-level dependencies should be done first;
- Which circular dependencies must be broken up in advance.

So Anthropic first uses a deterministic script to analyze code dependencies, then has an agent review and fix the dependency graph.

There's an important principle here:

> Don't leave to the model's guessing what a deterministic program can do.

**3. The difference list**

There are inevitable semantic differences between the old and new languages. For example:

- One core difference from Zig to Rust is memory ownership;
- One core difference from Python to TypeScript is interfaces and type contracts;
- Implicit behavior in a dynamic language must be declared explicitly in a static one;
- Circular dependencies the old system allowed may not compile in the new one.

These problems can't be solved by simple syntax substitution and must go into the difference list ahead of time.

The rulebook defines the default approach; the difference list records the cases the default rules don't cover.

### Step 2: first run a small, throwaway experiment

I think this is one of the most worth-learning practices in the whole article.

Anthropic doesn't migrate all the code the moment the rules are done. Instead, it first picks a small set of representative files and runs a small-scale migration experiment.

In the Bun migration, they set up:

- One agent that migrated three files strictly by the rulebook;
- One agent that migrated the same files the way a "senior Rust engineer" would;
- A third agent that compared the two results and generated new migration rules.

From this experiment alone, they caught two serious problems ahead of time.

If those problems hadn't been exposed at the small scale but had instead spread across all 1,448 files, the cost of fixing them later would have been enormous.

Even more counterintuitive:

> The code the experiment produces should be thrown away.

Because the point of this phase isn't to make migration progress — it's to validate the migration rules.

This is a lot like a spike, PoC, or pilot in everyday development, except AI makes such experiments much cheaper.

Exposing rule problems with a little code first, then running massively in parallel, is more reliable than chasing migration progress from the start.

### Step 3: massively parallel migration

Only after the rules are validated does the real full-code migration begin.

Anthropic's basic loop is:

> Implement → Review → Fix

Multiple implementation agents work on different files or modules in parallel.

To control cost, high-throughput implementation work can go to smaller, cheaper models; complex rule design, architectural judgment, and code review go to stronger models.

The migration queue is managed by a script, not by the agent's own memory. For example:

- Check whether the target file already exists;
- Automatically compute the files not yet migrated;
- Split tasks by dependency order;
- Assign tasks to new agents;
- Resume from on-disk state after a failure.

Designed this way, the migration naturally supports pause and resume.

Wherever an agent can't decide how to migrate, it marks it uniformly:

```text
// TODO(port): <reason>
```

After that, compiler errors, smoke-test crashes, and test failures automatically become the next batch of fix tasks.

The queue doesn't need long-term manual maintenance, because the failures themselves keep generating new tasks.

### Step 4: compile, and fix by error pattern

Once the initial migration is done, the compilation phase begins.

Here you have to decide where to place the compiler based on the project's characteristics.

TypeScript compiles fast, so it can sit inside each agent's loop.

A full build of a large Rust project can take minutes, so the Bun migration didn't have each agent run compilation independently. Instead, a unified orchestration script ran the full build, then handed the error list out to multiple fix agents.

This design also reflects cost control. If every agent repeatedly ran the full build, it would waste huge amounts of CPU, time, tokens, context, and CI resources.

So they set up a single build daemon.

Only this process can rebuild the program. Other agents only submit patches, and the build daemon batches the patches, does a unified build, and re-runs the relevant tests.

Essentially, it centralizes the most expensive operation instead of letting a swarm of agents repeat it.

### Step 5: run smoke tests, cluster problems by root cause

Passing compilation doesn't mean the program runs. Next you run smoke tests to find:

- Startup failures;
- Runtime crashes;
- Config-loading errors;
- Resource-initialization issues;
- Module-boundary errors;
- Abnormal external-dependency calls.

Here, too, you can't look at a single error at a time.

If dozens of crashes come from the same root cause, you should fix the upstream rule or architecture, not have dozens of agents submit ad-hoc patches separately.

Anthropic's approach is to cluster problems by root cause first, then have adversarial review agents check the classification and the fix plan.

### Step 6: compare old-vs-new system behavior

The final step is confirming the old and new systems truly behave the same.

By now, the new code has been translated, compiled, smoke-tested, and had its basic problems fixed. But to prove the migration succeeded, you still need to compare real behavior.

Anthropic splits the test tasks up. Each failing test is handled by an independent fix agent that looks at, all at once:

- The old code;
- The new code;
- The test input;
- The actual output;
- The expected behavior.

After the fix, an adversarial review agent checks the change.

The Python-to-TypeScript migration also used seven real usage scenarios to call both systems, then diffed the output item by item.

After that, Claude autonomously designed an end-to-end test suite and ran, fixed, and re-ran it for four nights in a row, surfacing edge cases that a hand-written scenario list would struggle to anticipate.

This is exactly where I think AI is well suited to shine:

not just having AI generate business code, but having AI:

- Analyze the old system's behavior;
- Generate comparison tests;
- Look for edge scenarios;
- Classify failing results;
- Propose fixes;
- Then call deterministic tools to verify.

## 4. The four engineering practices most worth learning

### 4.1 Human time should be invested up front

AI can generate code fast, but migration rules, architectural boundaries, and verification standards still require heavy engineer time.

Anthropic's conclusion:

> Human effort should be front-loaded as much as possible.

The parts that cost engineers the most time are:

- Writing the rules;
- Identifying language and architecture differences;
- Building the test judge;
- Designing the task queue;
- Running the small-scale stress test.

Once these are done, the rest is mostly letting the task queue converge.

This is exactly the opposite of how many people use coding agents.

Many people first have AI generate a lot of code, then slowly patch in rules after problems appear. But for large tasks, the more sensible order is:

> Spend time defining the system first, then let AI execute fast.

### 4.2 Review must be adversarial

The article repeatedly mentions adversarial review.

An implementation agent's goal is to finish the task.

A review agent's goal should not be to help the implementation agent prove itself right — it should actively hunt for:

- Rule violations;
- Hidden assumptions;
- Uncovered edge conditions;
- Behavior changes;
- Type and interface inconsistencies;
- Weakened tests;
- Temporary hacks treated as final implementations.

Anthropic has two review agents check the same result in mutually independent contexts. If they disagree, a third agent decides.

This kind of mechanism suits not just code migration but also everyday work: architecture design reviews, data migrations, API refactors, database upgrades, framework upgrades, large code cleanups, security reviews, and test-plan reviews.

Not every agent should play "collaborator." Some agents should be explicitly designed as skeptics, opponents, and breakers.

### 4.3 Review the process outcomes, not every line of code

Facing a million lines of AI-generated code, no human can review it line by line. What engineers should really watch is:

- Which errors keep recurring;
- Which rule produces the most problems;
- Which class of module has the highest failure rate;
- Which phase consumes the most tokens;
- Which tests never converge;
- Which agents keep violating the same constraint;
- Which tasks should go to a stronger model;
- Which judgments could be turned into deterministic scripts.

In other words: you no longer just review the code — you review the loop that generates the code.

This doesn't mean code review is unimportant; it means human attention should focus on high-risk code and systemic problems. Ordinary, repetitive, bulk code relies more on compilers, static checks, tests, diffs, contracts, adversarial agents, and automated gates.

### 4.4 Cost management is architecture design

The article discusses cost control in many places. For example:

- Use smaller models for bulk implementation;
- Use stronger models for rule-writing and final review;
- Avoid having each agent repeatedly run an expensive build;
- Use a single build daemon to batch patches;
- Place the compiler in the right spot;
- Make tasks pausable and resumable;
- Only regenerate the files affected by a rule change;
- Hand mechanical judgments to scripts, not the language model.

This shows that in a large-scale agent system, model choice, task queue, build pipeline, and verification mechanism can no longer be considered separately.

Together they determine the project's completion speed, token cost, infrastructure cost, error-convergence speed, and final quality.

Cost optimization for AI agents isn't just "switch to a cheaper model" — it's redesigning the entire execution pipeline.

## 5. How to apply these practices to everyday projects

Most engineers won't migrate a million lines, but this method still applies to ordinary development.

For a Java version upgrade, a Spring Boot upgrade, a database migration, an API refactor, or a front-end framework migration, you can use a similar flow:

**First, build a rules file.** Make explicit: which patterns must be replaced; which interfaces must not change; which directories may not be modified; which compatibility behaviors must be preserved; which exception-handling styles must be unified; and what conditions require stopping and reporting.

**Second, build an old-vs-new comparison tool.** For the same input, call both systems and compare: return values, error codes, logs, database changes, message output, performance, and memory and resource usage.

**Third, run a throwaway experiment first.** Pick a few representative modules to migrate. Don't rush to keep the code — use the experiment to find out whether the rules are complete, whether the agent understands the task, whether the tests can catch errors, whether the module split is reasonable, and whether the prompt is ambiguous.

**Fourth, have different agents play different roles.** At minimum distinguish: implementation agent, rule-review agent, code-review agent, test-design agent, behavior-comparison agent, and risk-review agent.

**Fifth, fix the rule that produces the errors.** When multiple modules show the same problem, don't fix them one by one. First decide whether to change the rule, change the prompt, change the task split, add tests, add static checks, or change the model division of labor.

This may be the most important mindset shift when using coding agents at scale.

## Closing

AI agents are making some once-high-risk, long-cycle engineering projects feasible again.

But the biggest takeaway for me isn't "Claude can generate a million lines in two weeks."

What's really worth noting is how Anthropic designed a complete engineering control system around AI's uncertainty:

- Constrain generation with rules;
- Validate the rules with a small experiment;
- Execute in parallel with many agents;
- Actively hunt for problems with adversarial agents;
- Use the compiler, tests, and diffs as the judge;
- Guarantee recoverable flow with a unified task queue;
- Control cost with models of different tiers;
- Fix the process — not just the code — when systemic problems appear.

The faster AI writes code, the less an engineer can afford to focus on the code alone.

The more important skill for future engineers may be:

> Designing constraints, building judges, organizing agents, and constructing an engineering loop that keeps finding and fixing errors.

For any engineer using Claude Code, Codex, Cursor, or another coding agent, I strongly recommend reading the original. Even without an immediate large-migration need, it's worth seeing how Anthropic designs prompts, rules, reviews, tests, and agent workflows.

These methods apply not only to language migration, but also to everyday refactoring, architecture upgrades, and complex feature development.

**Original:** [How Anthropic runs large-scale code migrations with Claude Code](https://claude.com/blog/ai-code-migration) (authored by / copyright Anthropic; this post is my write-up and personal reflection).
