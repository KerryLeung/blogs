---
layout:     post
title:      "Coding Agent 的知识管理：不是塞更多上下文，而是构建可复利的工作流"
subtitle:   "Index → Route → Body 的按需加载纪律"
date:       2026-07-02
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
lang: zh
ref: coding-agent-knowledge-routing
---

> Coding Agent 时代，知识管理的核心不是“记住更多”，而是“按需加载”。真正重要的不是 context size，而是 context routing：让正确的知识，在正确的阶段，被正确加载；让每个判断都有证据；让重复经验逐步变成自动化检查。这也是从 prompt engineering 走向 agent system engineering 的关键一步。

最近在思考一个问题：Coding Agent 的开发流程里，知识应该怎么管理？

很多团队在使用 agent 时，第一反应是给它更多上下文：更多文档、更多 memory、更多规则、更多历史案例。

但实际效果不一定更好。因为 agent 不是人。上下文给得越多，它不一定越聪明，反而可能出现几个问题：

- 读到过期信息
- 被历史 case 误导
- 在无关文档里浪费大量 token
- 把“曾经发生过的问题”当成“当前一定存在的问题”
- 最后输出一堆看似合理、但证据不足的 review comment

所以，Coding Agent 的知识管理，不应该只按 topic 分类，比如“metrics 文档”“security 文档”“test 文档”。

更关键的是按 access pattern 管理：

- 什么时候加载这类知识？
- 它支持 agent 做什么决策？
- 如果不加载，会不会影响当前任务判断？

换句话说，核心问题不是“我们知道什么”，而是“当前这个任务，需要加载什么知识来支持判断”。

## 一、核心模式：Index → Route → Body

我比较认同的模式是：

```text
Index → Route → Body
```

也就是：先看索引，再做路由，最后只加载必要正文。

Agent 不应该一上来就把整个知识库读一遍。更合理的做法是：

1. **Index**：默认只加载小型索引、摘要、触发条件。
2. **Route**：根据任务类型、变更文件、影响范围做分类。
3. **Body**：只有当路由命中时，才加载完整文档。

以一个 `team-code-review` 这样的 review loop 为例，它可以先解析 PR，拿到 changed files，然后判断这次改动属于哪类：

- domain model change
- metric change
- locale formatting change
- access control change
- test-only change
- CI/config change

如果 PR 改的是 locale formatting 文件，就加载 locale spec。  
如果 PR 改的是 shared metric，就加载 shared metric 相关规则和历史 lesson。  
如果只是 test config 调整，就不需要加载 locale spec。

这个区别非常重要。

一个低质量 loop 是：“我把所有文档都塞给 agent，希望它自己判断。”

一个可持续进化的 loop 是：“我让 agent 先分类，再只加载当前判断所需的知识。”

**前者只是堆上下文，后者才是在构建一个可复用的工程系统。**

## 二、知识应该分层管理

在 Coding Agent workflow 里，我倾向于把知识分成几类。每一类知识都应该有自己的加载时机，而不是默认进入全局 prompt。

## 三、Contracts / Invariants：什么绝对不能破坏

这是最重要、也应该最早加载的一层。

它回答的问题是：什么东西不能被破坏？

比如：

- 权限控制
- public field name
- API contract
- schema
- module boundary
- data ownership
- 跨系统数据解释规则

如果一个 PR 改了 access-control metadata，review loop 必须在 review 前加载 access-control contract。

因为这类问题一旦出错，影响通常不是局部 bug，而是系统边界被破坏。

但如果当前 PR 完全不涉及这些边界，就没有必要加载。

**不变量应该被早加载，但只在相关边界被触碰时加载。**

## 四、Maps / Indexes：应该去哪里找信息

Map 的作用不是解释所有细节，而是帮助 agent 路由。

好的 map 应该足够小，能被快速扫描。

比如：

```text
如果 shared_metrics/ 目录下的文件发生变化，
加载 shared metric rule，
并执行 metric validation path。
```

这比下面这种写法好很多：

```text
这里是我们关于 metrics 的全部知识。
```

前者是可执行的路由规则。后者只是信息堆积。

**Map 不是知识正文，Map 是让 agent 少读错文档的路由表。**

## 五、Specs：什么才是正确行为

Spec 是 correctness oracle。

当一个 PR 声称修改了某个行为，agent 不应该凭记忆判断这个行为是否正确，而应该加载对应 spec。

比如 PR 改了 date formatting 行为，review loop 应该加载 formatting spec，然后检查：

- 新格式是否符合 spec
- 所有受影响 surface 是否都覆盖
- 测试是否证明输出变化符合预期

如果 spec 里没有定义预期行为，agent 不应该自己编一个标准。

更合理的 finding 是：

```text
当前 expected behavior 没有被定义，建议补充 spec。
```

这比“凭经验判断对错”更工程化。

## 六、Procedures / Skills：这个任务应该怎么做

Procedure 回答的是：在这个项目里，这类任务应该怎么执行？

比如 code review 主命令不应该包含所有 review checklist。它更适合负责 orchestration。

具体 checklist 可以拆到不同 skill：

- domain field review skill
- formatting review skill
- security/access review skill
- test adequacy review skill
- PR comment posting skill

这样主流程保持轻量，具体领域的 gotchas 也可以放在对应 skill 里。

**主流程管编排，skill 管过程细节。**

这其实也是 agent workflow 可维护性的关键。

## 七、ADRs：为什么这里是这样设计的

ADR 的价值在于避免 agent “修复”一个 intentional design。

很多时候，agent 看到复杂代码，会本能地想简化。这有时是对的，但有时这个复杂度是历史兼容、跨系统契约、性能约束或迁移策略下的有意选择。

所以当 PR 触碰这些内容时，应该触发 ADR：

- established module boundary
- long-lived naming convention
- cross-system contract
- known tradeoff
- compatibility decision

ADR 不应该每次都加载。它应该通过索引命中，在相关区域发生变化时再加载。

否则 agent 很容易在无关场景里被历史设计讨论干扰。

## 八、Plans：临时计划，不应该永久化

Plan 很有用，但它是临时脚手架。

它适合用在：

- stacked PR
- phased migration
- active refactoring
- 多阶段上线

但 plan 不应该长期留在默认上下文里。

一旦工作完成，里面的 durable knowledge 应该沉淀到不同地方：

- 稳定行为 → spec
- 设计原因 → ADR
- 反复踩坑 → lesson / gotcha
- 可自动验证的问题 → test / CI check

一个简单原则是：

```text
Plans expire.
Specs and ADRs survive.
```

**Plan 是脚手架，不是建筑本身。**

## 九、Lessons / Rules：只有可触发，才有价值

Lesson 最常见的问题是太泛。

不好的 lesson：

```text
Be careful with metrics.
```

好的 lesson：

```text
When a PR hides a shared metric,
check whether that metric is reused by multiple surfaces before approving.
```

好的 lesson 一定有 trigger。

在 review loop 里，agent 应该先扫描 lesson header，而不是加载所有 lesson body。如果 changed files 命中了 trigger，再加载完整内容。如果没命中，就跳过。

这样既保留了团队记忆，又不会污染上下文窗口。

## 十、Gotchas：局部陷阱放在局部流程里

Gotcha 是某个 procedure 内部的小陷阱。

比如：

- legacy field comparison 的特殊规则
- PR comment posting API 的 workaround
- 某个 formatter 的历史兼容行为
- 某类测试 case 必须覆盖的边界条件

如果一个 gotcha 只在某个 skill 中使用，就应该放在这个 skill 里，而不是放到全局 prompt。

原则是：

```text
If only one procedure needs it,
keep it inside that procedure.
```

这样可以避免全局 prompt 越来越臃肿。

## 十一、Case Records：历史案例是证据，不是默认上下文

Case record 记录的是一次具体历史运行：

- state
- validation logs
- subagent outputs
- review payload
- final decision
- postmortem notes

它的价值在于审计、争议复盘、问题追踪。

但它不应该成为每次 review 默认加载的上下文。

比如上一次 PR 的 review report，不应该默认影响下一次 PR。除非当前任务明确是在 resume 同一个 PR，或者当前问题直接引用了那个历史 case。

**历史案例是 evidence，不是 standing knowledge。**

## 十二、Automated Checks：最好的知识最终应该变成代码

知识管理的终点，不是让 prompt 越写越长。

真正好的知识，最后应该沉淀成可执行检查：

- shell check
- lint rule
- unit test
- integration test
- CI gate
- review script
- validation job

Lesson 只是提醒：

```text
Remember to check X.
```

Test 才是保障：

```text
X cannot regress silently.
```

如果 agent 每次都在人工检查同一个问题，那说明这个知识还没有沉淀完成。它应该被升级成自动化检查。

## 十三、Lead Agent 的角色：不是汇总，而是裁判

在 multi-agent review loop 里，subagents 可以扩大覆盖面。

比如一个 subagent 使用外部 review 工具，另一个 subagent 使用项目内部 review skill，第三个 subagent 专门看测试覆盖。

但最终 lead agent 不能只是合并结果。

它必须负责：

- 验证每一个 reported issue
- 要求可运行的 reproduction
- 拒绝没有证据的 finding
- 区分 project-wide baseline failure 和 PR-specific regression
- 判断 severity
- 在发 comment 前请求确认
- review 结束后提出知识更新建议

Subagent 生成的是 candidates。

Lead agent 产出的应该是 evidence-backed findings。

这点非常关键。否则 multi-agent review 很容易变成“多个 agent 一起制造噪音”。

## 十四、一个可持续进化的知识闭环

我认为 Coding Agent workflow 真正有价值的地方，不是一次性帮你 review 一个 PR，而是它可以把知识持续沉淀下来。

比较理想的闭环是：

```text
incident
  → distilled lesson
  → routed skill gotcha or review rule
  → automated check
  → less prose in the prompt
```

也就是：

1. 线上问题或 review 问题暴露出来
2. 人把它提炼成 lesson
3. lesson 被放到正确的 skill 或 rule 里
4. 如果反复出现，就升级成自动化检查
5. 最后 prompt 反而变短，因为判断被系统化了

这才是 agent loop compound 的关键。

不是让 agent 记住所有东西，而是让它：

- 在正确时间加载正确知识
- 基于证据做判断
- 把重复的人类经验转化为可执行检查

## 十五、我的思考

现在很多 AI Coding 实践还停留在“给 agent 更多上下文”的阶段。

但从工程系统角度看，真正重要的不是 context size，而是 context routing。

Agent 不应该像一个被塞满文档的新人。它更应该像一个有流程、有索引、有验证机制的工程系统。

我越来越觉得，未来高质量的 coding agent workflow，核心竞争力不只是模型能力，而是三件事：

- 知识分层
- 按需加载
- 自动验证

模型负责生成候选方案。Workflow 负责约束路径。验证机制负责证明结果。

如果没有这些结构，agent 越强，可能只是更快地产生更多不确定输出。

如果有这些结构，agent 才能从一次性工具，变成可以持续复利的工程系统。

## 十六、总结

Coding Agent 的知识管理，目标不是让它记住一切。

目标是：

- 让正确的知识，在正确的阶段，被正确加载
- 让每个判断都有证据
- 让重复经验逐步变成自动化检查

这也是从 prompt engineering 走向 agent system engineering 的关键一步。

**真正成熟的 agent workflow，不是一个很长的 prompt，而是一个会路由、会验证、会沉淀的系统。**
