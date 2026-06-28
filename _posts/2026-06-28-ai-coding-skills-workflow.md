---
layout:     post
title:      "推荐一个 149K Star 的 AI Coding 宝藏仓库：Skills"
subtitle:   "真正值得学习的不是 Prompt，而是 Workflow"
date:       2026-06-28
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
    - open_source
lang: zh
ref: ai-coding-skills-workflow
---

> 最近看到一个很有意思的开源项目：[Skills](https://github.com/mattpocock/skills)。截至 2026-06-28，我通过 GitHub API 查到它已经有 149,182 stars。作者 Matt Pocock 在前端和 TypeScript 圈很有影响力，但这个仓库让我更感兴趣的地方，不是某个具体 prompt，而是它把优秀工程师的工作流程沉淀成了一套套可复用的 Agent Skill。

今天重点推荐其中一个我觉得很值得学习的 Skill：[improve-codebase-architecture](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md)。

## 一、这个 Skill 是做什么的

简单说，它会让 Agent 对一个代码仓库做一次架构诊断。

它不是上来就让模型“给我一些重构建议”，而是先规定了明确流程：

- 读取项目里的 `CONTEXT.md` 和 ADR，理解已有领域语言和历史决策
- 用一套固定架构词汇分析模块、接口、深度、接缝、适配器、局部性和杠杆
- 找出理解成本高、模块过浅、耦合泄漏、测试困难的地方
- 把候选问题渲染成一份 HTML 报告
- 每个候选项都要包含涉及文件、问题、方案、收益、before/after 图和推荐强度
- 用户选中一个候选项后，再进入 grilling loop，把设计树继续问清楚

这个流程的价值在于：它没有把“架构评审”交给一次自由发挥的推理，而是把评审拆成了可以重复执行的步骤。

## 二、为什么它有代表性

我觉得这个 Skill 代表了 AI Coding 里一个很重要的方向：

**优秀的 Agent Skill，本质上不是 Prompt Engineering，而是 Workflow Engineering。**

一个粗糙的 prompt 可能是：

```text
请帮我分析这个项目的架构，并给出重构建议。
```

而一个更成熟的 workflow 会规定：

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

前者依赖模型临场发挥，后者把资深工程师的工作顺序、检查点和决策约束都写进了流程。

这也是为什么我觉得它比“一个好用的架构分析 prompt”更值得学习。

## 三、目标清晰，Agent 才不容易漂移

很多 Agent 失败，并不是模型能力不够，而是目标定义过于模糊。

`improve-codebase-architecture` 的目标很具体：扫描代码库里的 deepening opportunities，输出一份可读的可视化 HTML 报告，然后让用户选择一个候选项继续深入。

注意这里有两个关键约束：

1. **先只提出候选，不直接改代码。**
2. **用户选择后，才进入下一轮设计追问。**

这让 Agent 不会从“发现问题”一路滑到“顺手重构一堆文件”。架构工作最怕这种边界漂移：问题还没对齐，方案已经开始落地。

## 四、上下文决定输出质量

这个 Skill 开始前会读 `CONTEXT.md` 和 ADR。这个细节很关键。

代码只能告诉 Agent 当前系统“长什么样”，但很多关键决策并不在代码里：

- 为什么模块边界这样划分？
- 为什么选择这个技术方案？
- 为什么放弃另一个方案？
- 哪些 tradeoff 是当时有意接受的？

这些信息如果只存在人的脑子里，Agent 很容易反复给出已经被否定过的建议。

所以我越来越认同一点：

**Agent 的能力上限，很大程度上取决于上下文质量。**

上下文不是越多越好，而是要完整、准确、结构化。`CONTEXT.md` 负责领域语言，ADR 负责历史决策，它们共同给 Agent 一个更接近真实工程现场的判断环境。

## 五、决策一定需要评审

这个 Skill 最有意思的地方，是它没有把架构报告当成终点。

报告生成后，它会让用户选择一个候选项，然后进入 `/grilling`：围绕约束、依赖、模块形状、接缝后面应该藏什么、哪些测试应该保留，继续一层层追问。

这其实很接近我们日常做重要设计时的过程：

```text
先提出方案
   ↓
再被严格追问
   ↓
暴露隐藏假设
   ↓
修正边界和取舍
   ↓
最后才形成决策
```

代码评审重要，Agent 的决策评审同样重要。单次推理很容易把“看起来合理”误当成“真的适合当前系统”。通过 grilling 或类似的 critique loop，可以让 Agent 把自己的假设摊开，再逐个检查。

## 六、结果必须可验证

如果一个 Skill 只输出一段建议，它的价值其实有限。

`improve-codebase-architecture` 强制输出 HTML 报告，并要求每个候选项都包含具体文件、问题、方案、收益和 before/after diagram。这让结果至少可以被人检查：

- 这个问题是否真的存在？
- 涉及文件是否准确？
- 方案有没有解决问题，还是只是在换名字？
- 收益是否足够大？
- 是否存在更简单的做法？

我会建议把这类架构分析和 `/grilling`、ADR 一起使用，形成一个完整闭环：

```text
生成报告
   ↓
选择候选项
   ↓
严格追问
   ↓
修正方案
   ↓
记录 ADR / 更新领域上下文
```

这样下一次 Agent 再进入这个仓库时，它不需要重新猜一遍历史背景。

## 七、几个拿来就能用的实践建议

如果你也在探索 AI Coding，我建议不只是运行这个仓库里的 Skill，而是把它当成 workflow 设计样本来学。

**1. 给复杂任务写入口目标。**

不要只写“帮我重构一下”。更好的目标是：“只扫描并产出候选架构问题，不改代码；每个候选项必须包含文件、问题、方案、风险和推荐强度。”

**2. 为项目准备一个 `CONTEXT.md`。**

把领域里的核心概念写进去。哪些词是业务概念，哪些词只是代码实现细节，要让 Agent 分得清。

**3. 坚持写 ADR。**

只要做了重要架构选择，就记录：为什么选它、为什么不选别的、已知 tradeoff 是什么。ADR 不是给过去的人看的，是给未来的人类和 Agent 提供决策上下文。

**4. 重要方案都跑一轮 grilling。**

让 Agent 不只是给答案，还要接受追问：遗漏了什么？哪些假设没有证据？收益是否足够大？有没有更简单方案？

**5. 把输出改成可检查的 artifact。**

报告、表格、diff、ADR、issue 列表，都比一段自由文本更适合复盘和协作。

## 八、另外几个值得看的 Skill

除了 `improve-codebase-architecture`，我也建议顺手看一下这个仓库里的几个相关 Skill：

- [`grill-with-docs`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md)：在追问方案的同时沉淀领域语言和 ADR。
- [`to-prd`](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-prd/SKILL.md)：把当前讨论整理成 PRD。
- [`to-issues`](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-issues/SKILL.md)：把计划拆成可以独立执行的 issues。
- [`tdd`](https://github.com/mattpocock/skills/blob/main/skills/engineering/tdd/SKILL.md)：把红绿重构循环固化成 Agent 工作方式。

这些 Skill 共同体现了同一个方向：不是让 Agent “更会聊天”，而是让 Agent 更稳定地执行工程流程。

## 九、我的思考

最近越来越感觉，AI Coding 的分水岭不只是模型能力，而是你能不能设计出好的工作流。

过去我们把很多工程经验放在脑子里：需求要先问清楚、架构要先看上下文、重要决策要被 review、重构前要有验证、历史取舍要写 ADR。

现在这些经验可以被写成 Skill。

换句话说：

**真正值得沉淀的不是 Prompt，而是 Workflow。**

未来工程师之间的差异，可能不只是“谁写代码更快”，而是谁更会把优秀工程实践封装成可重复、可组合、可审查的工作流。

## 十、总结

`improve-codebase-architecture` 值得学习的地方，不只是它能帮你生成一份架构诊断报告，而是它模拟了一位资深工程师完成架构评审的完整过程：

```text
读上下文
   ↓
用统一语言识别架构摩擦
   ↓
产出可检查报告
   ↓
选择候选方案
   ↓
继续追问和修正
   ↓
沉淀为长期上下文
```

如果你正在探索 AI Coding，我会推荐认真阅读这个 Skill 的实现过程，而不只是直接运行它。因为真正有价值的部分，藏在 workflow 设计里。

原仓库：[mattpocock/skills](https://github.com/mattpocock/skills)  
推荐阅读：[improve-codebase-architecture/SKILL.md](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md)
