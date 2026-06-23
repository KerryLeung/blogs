---
layout:     post
title:      "让 Coding Agent 不再跑偏的 Goal 写法"
subtitle:   "从一份「写给 AI 读」的 spec 里偷来的消歧纪律"
date:       2026-06-22
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
lang: zh
ref: goal-writing-for-coding-agents
---

> 给 Claude Code / Cursor 派活，效果不好往往不是 agent 笨，而是 Goal 描述只在我们自己脑子里才不歧义。读自己写的字时，大脑会自动用已有的上下文把空缺补满，于是你永远看不见那些空缺；agent 没有那份上下文，遇到同一个空缺只会补成别的东西。OpenAI 开源的 [Symphony Service Specification](https://github.com/openai/symphony/blob/main/SPEC.md) 给了一套现成解药——它明知读者是 AI，于是把「消歧」做成了第一性设计。这篇把它的六个装置拆出来，变成你下次写 Goal 就能直接用的纪律。

## 一个被误判的因果

我们复盘 agent 写崩的活，第一反应常是「这模型不行」。但更多时候，真正的问题在上游：**Agent 跑偏，多半不是它笨，是你的 Goal 只在你自己脑子里才不歧义。**

这是两件「从内部看一模一样」的事——*一份对作者清晰的 Goal*，和*一份对缺少作者上下文的读者也无歧义的 Goal*。在纸面上它们长得一样，因为你读自己的字时会自动补全自己的空缺；只有当一个不共享你上下文的读者（agent）把同一个空缺补成了别的东西，二者才分叉。问题不出在 agent 的智力，出在那份你看不见的歧义。

最近读到的 Symphony spec，恰好是一份系统性解决这个问题的范本。它是 OpenAI 开源的、调度 coding agent 干活的编排服务规范。它最打眼的地方不是描述了什么系统，而是**作者从头到尾都清楚：读者是 AI，不是人类工程师。** 于是它把消歧的「补全规则」从作者脑子里，搬到了纸面上。下面六个装置，每一个都能反过来变成你写 Goal 的一条纪律。

## 一、规范语言纪律严明

spec 全文严格使用 RFC 2119 的 `MUST` / `SHOULD` / `MAY` 分级，再用 `Implementation-defined` 标记给「必须做选择、但允许多种答案」的灰区留出口。Agent 拿到这种规范，不会把建议当强制，也不会忽略必须明示的策略。

**这对你写 Goal 意味着：别用「最好」「尽量」「记得」这种人类靠语气分轻重的词——把每条要求标成「必须 / 应该 / 可选」，再把「你来定」的地方显式写成「实现自定」。** 含糊的强弱，是 agent 第一个会猜错的东西。

## 二、六层抽象切分清晰

Policy → Configuration → Coordination → Execution → Integration → Observability，六层逐层可实现、可测试。Agent 不会迷失在全局依赖里。

**这对你写 Goal 意味着：把一个大目标切成可以逐层交付、逐层验证的子目标，而不是甩一团互相纠缠的需求。** 能被分层的目标，才能被分段验证；不能分段验证的目标，agent 只能一口气赌完。

## 三、状态语义双轨制

spec 显式拆开两套状态：业务/tracker 状态（`Todo` / `In Progress` / `Done`）与服务内部的调度状态（`Unclaimed` / `Claimed` / `Running` / `RetryQueued` / `Released`），并明说「这跟 tracker 状态不是一回事」。混淆这两类状态是经典反模式，作者一上来就把它们分开。

**这对你写 Goal 意味着：当你的需求里同一个词承载了两种含义（「完成」是指代码写完，还是指上线？），先把它拆成两个名字。** 一词多义是歧义的温床，命名是最便宜的消歧。

## 四、Reference Algorithms 给伪代码

关键流程 spec 直接给伪代码（Section 16 一口气给了六个算法），happy path 和 error path 都写出来。Agent 不需要从散文里反向推导逻辑，实现歧义大幅降低。

**这对你写 Goal 意味着：流程类需求，与其用三段散文描述，不如写五行伪代码或带编号的步骤，并且明确写出错误分支。** 你不写 error path，agent 就会替你即兴发挥一个。

## 五、安全模型独立成章

3 条 Safety Invariants 单独成章（Section 9.5）：agent 只能在「每个 issue 自己的 workspace 路径」里运行；workspace 路径必须留在 workspace root 之内；workspace key 必须被 sanitize。这不是事后补丁，而是设计的第一性原则。

**这对你写 Goal 意味着：把「绝对不能违反的约束」从一堆需求里拎出来、单独列、并写成不变量的形式（「X 必须始终成立」），而不是埋在第七条 bullet 里。** 安全约束一旦和普通需求并列，就会被当成可权衡项谈掉。

## 六、边界声明克制

spec 明确说自己是 scheduler / runner、只读 tracker，*不*负责写 ticket（那是 workflow prompt 和 agent 工具的事）。Non-Goals 章节写得和 Goals 一样认真。

**这对你写 Goal 意味着：认真写 Non-Goals。明说「这次不要动 X、不要顺手重构 Y」，和说清楚要做什么同样重要。** Agent 的功能膨胀，几乎都发生在你没划下的那条边界上。

## 把六个装置收成一张 Goal 清单

下次写 `/goal` 之前，过一遍这六问：

1. 每条要求标了 **必须 / 应该 / 可选** 吗？「你来定」的地方写明了吗？
2. 目标能 **逐层交付、逐段验证** 吗，还是一团缠在一起？
3. 有没有 **一词两义**？把它拆成两个名字了吗？
4. 流程写成 **带错误分支的步骤 / 伪代码** 了吗？
5. **不可违反的约束** 单独列成不变量了吗？
6. **Non-Goals** 写了吗——哪些明确不要做？

## 收尾：把 agent 当编译器，不是当同事

这六条背后是同一个动作：**把意图从你的脑子里，搬到纸面上的结构里。** 同事会揣摩你的意图，编译器只执行你写下的字面——而 agent 更像后者。RFC 2119 的 `MUST` 之于散文，就像类型签名之于注释：它让读者的义务变得可被机械校验。

歧义的 Goal 不是付一次账。它让 agent 猜、你纠、它再猜，每个欠规范的目标都变成一次次往返，按任务计息。而把消歧前置，是一次性的写作成本，之后反复摊薄——还能沉淀成一份你自己的 Goal 模板。

下次 agent 又跑偏，先别急着换模型。回头看一眼那句 Goal：它是真的无歧义，还是只在你自己脑子里无歧义？

> 原始 spec：<https://github.com/openai/symphony/blob/main/SPEC.md>
