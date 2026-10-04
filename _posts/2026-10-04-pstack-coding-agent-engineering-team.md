---
layout:     post
title:      "推荐一个 Cursor 插件 pstack：把 Coding Agent 从“会写代码”升级成“工程团队”"
subtitle:   "Coding Agent 的瓶颈，正在从生成能力转向信任"
date:       2026-10-04
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
    - open_source
lang: zh
ref: pstack-coding-agent-engineering-team
---

> 最近读了 Cursor 官方 plugins 仓库里的一个项目：[pstack](https://github.com/cursor/plugins/tree/main/pstack)。它讨论的已经不是“怎么写一个更好的 Prompt，让 AI 帮我生成更多代码”，而是另一个越来越重要的问题：**当 Coding Agent 写代码已经足够快之后，怎么保证它写出来的东西是可靠的？**

这也是我最近越来越明显的一个感受。

过去使用 AI Coding，最大的瓶颈是模型能不能把代码写出来。现在这个问题正在快速消失：Agent 可以连续工作很长时间，可以修改几十个文件，可以自己跑测试，可以并行启动多个 Subagent，甚至可以一晚上完成过去几天的开发工作。

新的问题反而变成了：**你敢不敢相信它？**

如果三个 Agent 同时帮你改代码：

- 它们会不会同时修改同一个共享状态？
- 有没有真正理解原来的架构？
- 修 Bug 是找到 Root Cause，还是加了一个看起来有用的 workaround？
- 单元测试通过了，真实产品真的工作吗？
- Agent 说“Done”以后，到底有什么证据？
- 多个 Agent 产生不同方案，到底选哪个？
- 一个 Agent 工作几个小时以后，你怎么知道中间做过哪些错误决策？

pstack 想解决的就是这些问题。

## pstack 是什么

pstack 是 [cursor/plugins](https://github.com/cursor/plugins) 仓库中的一个 Developer Tools 插件，MIT 协议。本文基于 2026-10-04 的 `main` 分支（插件版本 0.15.9）。

作者 Lauren Tan，也就是 poteto，在 README 里介绍自己在 Meta、Netflix 和 Cursor 处理过数百万行规模的代码库，同时是 React Core Team 成员，参与构建和维护 React Compiler。

README 开头有一句我觉得很能概括它的理念：

> throughput without quality is not a goal i aspire to. if you want to go fast, go deep first.

没有质量的吞吐量不是目标。想快，先往深处走。

所以 pstack 的目标不是让 Agent 写更多代码。README 的原话是 "the goal is not to maximize loc, in fact it's the opposite"：**写更少的代码，但写出质量更高的代码。**

安装很简单：

```bash
/add-plugin pstack
```

第一次使用先运行 `/setup-pstack`，选择 reasoning budget 和每个角色使用的模型。之后对于需要严谨处理的任务，基本只需要：

```text
/poteto-mode <你的任务>
```

真正有意思的地方，就是 `poteto-mode` 后面的设计。

## 一张图理解 pstack

我把整个项目理解成下面几层：

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

这其实已经非常接近一个轻量级的 **AI Software Engineering Operating System**。

## 第一层：poteto-mode 不是 Prompt，而是 Router

很多 Coding Agent Skill 的问题是：Skill 越写越多以后，用户反而不知道什么时候应该调用哪个。

pstack 的解决办法很直接：**让用户只描述目标，由 `poteto-mode` 自己选择工作流。**

比如：

```text
/poteto-mode this pr has a subtle bug where the scroll drifts every 750ms even when idle.
repro first, then fix and verify.
```

它会匹配到 Bug Fix Playbook。如果你说：

```text
/poteto-mode build a small feature behind a feature flag. verify it really works.
```

则会进入 Feature Playbook。

匹配之后，它会先开一个 todo list，把 Playbook 的步骤原样抄进去作为前几项；决定跳过的步骤也要留在列表里，并写一行 `skip: <reason>`。也就是说，流程不是“建议”，而是摆在明面上的清单。

目前 README 里列出了 **23 个 Playbook**，包括 Investigation、Bug Fix、Perf、Hillclimb、Runtime Forensics、Trace Forensics、Feature、Refactoring、Prototype、Visual Parity、Eval、Babysit、Shipping、Autonomous Run、Orchestrate、Session Pickup、Multi-phase Plan、Opening a PR 等等。

所以用户面对的是 Goal，Agent 内部面对的是 Structured Workflow。这是我认为 pstack 最重要的第一个设计：

**不要要求用户学习 Agent Workflow，让 Router 根据任务自动选择 Workflow。**

## 第二层：把软件工程经验编码成 Playbook

再往里面看，会发现 pstack 的核心实现其实并不复杂。它没有实现一个庞大的 Agent Orchestration Service，大量逻辑被编码在 Markdown 里：

```text
skills/*/SKILL.md
skills/poteto-mode/playbooks/*.md
agents/*.md
```

再加上少量辅助脚本（比如盯 PR 状态的 watcher）。

但是这些 Markdown 并不是简单 Prompt，它更像是 **Executable Engineering Process**。比如 Bug Fix Playbook 一共六步：

```text
1. 自己在对应的运行界面上复现 Bug
2. Binary-search 原因：列出候选假设，用 how / why 做种子，每一轮拿运行时证据排除
3. 规划修复：跨函数边界先走 architect，实现委托给 Subagent
4. 在同一个界面上验证：原始 repro 现在通过
5. 安排 commit 顺序：失败的 repro 先于 fix 进入 git 历史
6. 走 Opening a PR
```

里面还有一个我很喜欢的要求：

> Every shipped line traces to runtime evidence.

**每一行随 Bug Fix 交付的代码，都应该能追溯到某个运行时证据。** “这个保护应该有用”在 Playbook 里被明确称为 hypothesis，不是 fix，不能 ship；证据推翻了某个假设，因它而写的代码就要 revert。

它要的不是“这个保护应该有用”，而是：

```text
我们观察到了 X
→ 验证假设 Y
→ 修改 Z
→ 原始 repro 消失
```

这是非常典型的工程师 Debug 思维。

## 第三层：流程（Playbook）和原则（Principle）是分开的

pstack 还有一个比较有意思的设计：它把“怎么做事情”和“做事情时遵循什么原则”拆开了。

Playbook 负责步骤。原则则是 **24 个很短的 Skill，每个只讲一条规则**，分成 core、architecture、verification、delegation、meta 五组。`poteto-mode` 在任务开始时读取原则索引，其他 Skill 按名字引用它们。例如：

- **Laziness Protocol**：偏向删除，以及能解决问题的最小改动，而不是增加抽象。
- **Model the Domain**：不要用散落各处的条件判断表达业务状态，而是用 state machine、typed model、table / registry、reducer 这类结构来表达 Domain。
- **Fix Root Causes**：不要修症状。先复现，一直追问 Why，直到找到 Root Cause。
- **Test Behavior, Not Implementation**：按用户调用代码的方式去测，断言用户观察到的结果，而不是内部实现细节。
- **Separate Before Serializing Shared State**：多个并发执行者可能写同一个文件、分支或状态时，第一选择不是“加锁”，而是**先把共享拆掉**。原文甚至说，把“we need a lock”当成一个需要检查的 design smell。

最后这一条特别适合 Multi-Agent Coding。传统代码设计里很多不起眼的共享状态，在并行 Agent 环境下会迅速变成工程问题。

## 第四层：针对不同问题的并行模式

pstack 不是简单地 spawn 5 个 Agent 然后等结果。它针对不同问题定义了不同的并行模式，这个设计我觉得非常值得研究。

### Arena：多个 Agent 做同一道题

当一次尝试可能把产物锁定在错误的形状上时，用 `/arena`：

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

重点不是“投票”，而是：

1. 多个模型拿到同一份 prompt，各自独立产出方案和 rationale
2. 一个只读的 cross-judge（尽量来自和主 Agent 不同的模型家族）按 rubric 逐项打分，推荐一个 base
3. 主 Agent 自己把所有方案从头到尾读完，按 rubric 打分，再和 judge 对照
4. 选择一个 Base
5. 把其他方案里值得保留的部分手工 graft 进去
6. 验证最终合成的方案

Skill 里还有两个判断我很喜欢：N 个方案收敛到同一个形状，是强一致信号，直接 ship；N 个方案严重发散，说明问题框定得不够清楚，应该重新 frame 再跑，而不是把分歧平均掉。

这比简单 Majority Voting 更像真正的软件设计 Review。

### Swarm：用并行换 Coverage

`Arena` 是大家做同一道题，而 `Swarm` 主要是大家负责不同的 Slice（它也支持让多个 worker 赛跑同一份 brief）。比如 README 里的例子：

```text
/swarm check every package under packages/ against its check.sh. one worker per package.
one report.
```

可以变成：

```text
package A → Worker A
package B → Worker B
package C → Worker C
package D → Worker D
```

每个 worker 用 `PASS`、`ISSUES` 或 `BLOCKED` 加证据汇报，主 Agent 汇总成一份报告。没交回结果的 slice 记为 gap，**gap 不算 pass**。

我觉得这种模式特别适合 Migration、大规模代码检查、Package Validation、Cross-service Analysis 这类任务。也就是：**用并行换 Coverage。**

### Interrogate：让不同模型攻击你的方案

还有一个我很喜欢的 Skill 是 `/interrogate`。它不是让多个 Agent 继续写代码，而是**让多个不同模型同时尝试证明你的实现有问题**。

每个 Reviewer 收到相同的 Intent、相同的 Diff、相同的 Rubric，然后独立 Review。Skill 里有一句话点出了设计意图：对抗信号来自模型多样性，而不是靠分配人设。

最后 Lead Agent 把结果分成四类：

```text
Act On
Consider
Noted
Dismissed
```

这里有一个细节很好：**pstack 不会自动把 Reviewer 提出的所有问题都修掉。** 因为 Agent Review 和 Human Review 一样，也会产生很多 Nitpick 和 False Positive，最终仍然需要一个 Lead Judgment。而 Dismissed 这一栏也不是凑数，它把被驳回的意见和理由摆出来，方便你在不同意的时候推翻 Lead 的判断。

## 第五层：不同模型承担不同角色

这是 pstack 另一个很值得参考的地方。很多 Multi-Agent 系统实际上是“同一个模型 × N”，pstack 则明显更倾向**根据模型特点分配角色**。

截至 2026-10-04（v0.15.9），`/setup-pstack` 里的默认映射大致是：

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

也就是写代码的委托默认给 Grok，最难的改动、文字和判断给 Claude Opus，而 Arena、Architect、Interrogate 这类 Panel 默认混合三个模型家族。默认值会随版本变化，用户也可以通过 `/setup-pstack` 自己配置 Role → Model Mapping。

我认为这个方向会越来越重要。未来 Multi-Agent 的核心可能不是“哪个模型排名第一”，而是：

> **哪个模型最适合这个 Role？**

Coding、Planning、Review、Research、Writing、Debugging，本身可能就应该由不同模型承担。

## 第六层：Verification 是整个系统最重要的一环

如果让我从 pstack 里面只挑一个最值得学习的思想，我会选 **Prove It Works**。

pstack 的 Guide 里直接写：

> "It compiles" is not evidence.

编译通过不是证据。Bug Fix Playbook 里也说，单元测试展示的是分支行为，不是 Bug 不存在。

它要求验证方式和改动的类型匹配：

```text
CLI change         → 真正运行那条命令
UI change          → 在运行中的 App 里走一遍被改动的流程
Parser / Migration → 回放保存下来的真实输入
Performance change → 对比 before / after profile
Storage change     → 把写进去的值真正读回来
```

如果验证做不了，不能说“应该没问题”，而应该明确说 inconclusive。Guide 的原话是：一个没有证据却很自信的回复，应该被当成 red flag。

这看似只是一个小原则，但我觉得它其实是 Agent Coding 能不能真正进入生产环境的关键。

## 第七层：Agent 可以工作一晚上，但必须留下审计记录

pstack 专门设计了 Autonomous Run Playbook。Guide 里给的“过夜合同”是这样的：

```text
/poteto-mode im going to bed. migrate every caller to the new parser in a fresh worktree off <base>.
done means zero old callers, all parser fixtures pass, old api deleted.
keep a decision log. don't ask me before committing.
/loop until done. if you're truly stuck after a few hours, stop and write up why.
```

（`/loop` 是 Cursor 内置的命令，不是 pstack 的 Skill。）

Agent 首先把 Exit Condition 写成一个可检查的 predicate，然后循环：

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

这里最关键的是：**没有改善结果的修改要被丢弃**，而不是“反正已经写了，先留着”。另外两条约束也很实在：plateau 不是停下来的理由，而是换思路的信号；永远不能通过放松 predicate 来宣布胜利。

同时 `/show-me-your-work` 会记录一份 TSV Decision Log，每个决策一行：

```text
ts
phase
decision
why
evidence
result
```

其中 evidence 必须是一个指针（commit SHA、PR 号、`file:line`、截图路径），而不是一段话；日志只追加，错误的决定用新的一行覆盖，不改历史。

你第二天不需要重新读一遍 Agent 的全部 Transcript，只需要 Review **它做过哪些关键决策，以及这些决策有什么证据**。

这其实已经开始接近 **Agent Observability**。

## 第八层：把 Agent 犯过的错误变成系统约束

还有两个 Skill 我非常喜欢：`/correct` 和 `/reflect`。`/reflect` 是在一个长任务落地之后，把这次学到的做法沉淀成对已有 Skill 的修改；`/correct` 的理念尤其值得关注。

假设 Agent 总犯同一种错误。常见做法往往是在 `CLAUDE.md` 或 rules 文件里再写一句“以后不要这样做”。pstack 把这排在最后，理由只有一句：Agent 跳过文档的时候，什么都不会失败。

它的优先级是：

```text
Architecture
    ↓
Types（以及 Lint / CI）
    ↓
Tests
    ↓
Documentation
```

也就是说，如果 Agent 经常用错某个旧写法，不要告诉它“请不要使用这个 API”，而是**删掉旧写法，把内部实现藏起来，让错误的 import 直接失败**。如果某种状态不合法，不要写文档告诉 Agent，而是**让 Type System 无法表达这个状态**。而且每一个新加的检查，都要证明它在一个真实发生过的错误上会失败。

这个思想其实不只适用于 Agent，它本身就是优秀的软件工程原则。只不过 Agent 把这个问题放大了。

## 我觉得 pstack 最值得学习的是什么

看完整个项目后，我觉得它真正值得借鉴的不是某一个 Skill，而是下面三个设计思想。

### 1. Prompt 正在变成 Software Engineering Process

早期 Coding AI 更关注 Prompt Engineering。现在模型越来越强以后，Prompt 本身的重要性反而在下降，真正重要的是：

```text
Context
Workflow
Verification
Parallelism
Observability
Feedback Loop
```

也就是整个 Harness。pstack 本质上就是在设计 **Agent Harness**。

### 2. Multi-Agent 的核心不是数量，而是职责边界

启动 20 个 Agent 很容易。难的是：

```text
谁负责设计？
谁负责实现？
谁负责 Review？
谁负责验证？
谁负责最终决策？
谁可以修改哪些文件？
谁不能和谁共享状态？
```

pstack 里的 Arena、Swarm、Interrogate、Architect、Orchestrate，其实都在回答同一个问题：

> **怎么设计 Agent Team 的组织结构？**

这个问题未来可能和传统软件架构同样重要。

### 3. Coding Agent 的真正瓶颈正在从生成能力转向信任

今天的模型已经越来越少出现“这个代码我完全不会写”。更常见的问题是：**它写得太快了，我没有时间 Review。**

尤其当 Agent 一次产生 20 个文件、3000 行 Diff、10 个 Commit 的时候，人工逐行 Review 已经越来越不现实。所以未来真正需要加强的可能不是 Generation，而是：

```text
Verification
Independent Review
Decision Trace
Reproducibility
Constraint Enforcement
```

这也是为什么我觉得 pstack 很值得阅读。

## pstack 也不是所有任务都适合

它是一个非常 Opinionated 的工程体系。如果只是改一个变量名、修一个简单 CSS、写几十行 Demo，完整跑一遍 architect、arena、interrogate、verification 显然没有必要。

另外 Multi-model + Multi-agent 本身也意味着**更高的 Token 和推理成本**。

所以我觉得 pstack 更适合：

- 中大型 Codebase
- 复杂 Bug
- Refactoring 和 Migration
- Performance Optimization
- 跨模块 Feature
- 长时间 Autonomous Coding
- 对生产质量要求比较高的项目

对于简单任务，则应该保持轻量。pstack 自己其实也有类似的原则：Laziness Protocol，能少做就少做。`poteto-mode` 的设定也是只在匹配到 Playbook 或任务需要严谨时才介入，平时不碍事。

## 最后

如果你正在大量使用 Cursor、Claude Code、Codex 这类 Coding Agent，我建议不一定马上把 pstack 整套搬进自己的工作流，但是非常建议把这个 Repo 阅读一遍。特别是：

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

这些 Skill 里面有很多非常成熟的软件工程经验。

我越来越觉得，下一阶段 Coding Agent 的竞争重点，可能不再只是“谁的模型会写代码”，而是：

**谁能把模型组织成一套稳定、可验证、可以长期运行的软件工程系统。**

pstack 是我最近看到在这个方向上比较有代表性的实践之一。

## 原项目

- GitHub：[cursor/plugins/pstack](https://github.com/cursor/plugins/tree/main/pstack)
- 入门：[pstack guide](https://github.com/cursor/plugins/tree/main/pstack/docs/guide)
- 安装：`/add-plugin pstack`
- 初始化：`/setup-pstack`
- 日常复杂任务：`/poteto-mode <your task>`

README 里提到，`/deslop`、`control-cli`、`control-ui` 这几个被 `poteto-mode` 引用的 Skill 不在 pstack 里，而是在 `cursor-team-kit` 插件中，想要完整的一套可以一起安装。

如果你正在研究 Coding Agent、Agent Harness、Multi-Agent Software Engineering，这个项目值得花时间读一下。
