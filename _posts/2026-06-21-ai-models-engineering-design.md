---
layout:     post
title:      "GPT-5.5、Gemini 3.5 Flash、Claude Fable 5 方案能力对比"
subtitle:   "模型能扩展思路，但命名约束仍是工程师的事"
date:       2026-06-21
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - LLM
    - Looker
    - AI
lang: zh
ref: ai-models-engineering-design
---

> 给三个最强模型同一道工程题，它们一开始都推荐了同一个"最佳实践"——而那个方案恰好踩中项目里一个没人提起的隐藏约束。模型能扩展思路、能调研、能整理方案，但它只能对你说出口的约束做推理。这次 Atlas → Looker 的同步调研让我确信：AI 已经是极好的方案副驾，却还不能替工程师做最终架构判断。真正高效的不是盲信某个模型，而是用多个模型扩展思路、用工程师经验命名关键约束、再让最强模型收敛。

本文基于一次真实工程方案调研，比较 GPT-5.5、Gemini 3.5 Flash 和 Claude Fable 5 在复杂技术问题上的方案设计能力。调研目标是：如何把 Apache Atlas 中维护的元数据，同步到 Google Cloud Looker，让客户在 Looker 中看到最新的字段描述。

## 一、背景：从 Atlas 同步字段描述到 Looker

我们有一个元数据管理平台 Apache Atlas，它是字段描述的 source of truth。目标是设计一条单向、按需触发的同步链路：

```
Apache Atlas
   ↓
Sync Pipeline
   ↓
LookML Model Code
   ↓
Deploy to Looker
   ↓
Client sees latest field descriptions
```

给三个模型的指令完全一致：

```
Goal:
Build a one-way, on-demand sync pipeline that propagates Looker field descriptions
from Apache Atlas (the source of truth) into the LookML model code, then deploys so
client can see the latest descriptions.
```

参与对比的模型与使用模式：

| 模型 | 使用模式 | 备注 |
| --- | --- | --- |
| GPT-5.5 | xhigh | 思考全面，方案完成度较高 |
| Gemini 3.5 Flash | 默认模式 | 输出速度快，但复杂工程判断偏弱 |
| Claude Fable 5 | extra + deep-research | 结构性和主动调研能力最强 |

## 二、核心问题：Looker 字段描述应该怎么更新？

这次调研表面上是"从 Atlas 同步 metadata 到 Looker"，但真正的技术分歧在于：字段描述应该如何写回 LookML？大体有两类方案。

**方案 1：直接修改 LookML 源文件。** 脚本读取 Atlas API，定位原始 `.view.lkml` 文件中的字段，直接更新 `description`。

- 优点：最终 LookML 结构最直观；不依赖 refinement 的 include 顺序；和现有代码呈现形态一致。
- 缺点：要解析并重写开发人员手写的 LookML；容易破坏格式、注释和代码风格；多个 view 文件产生分散 diff；回滚和审查成本高。

**方案 2：生成独立的 LookML Refinements 文件。** 脚本不改原始 view，而是生成独立的 refinement 文件：

```lookml
view: +order {
  dimension: status {
    description: "Order status from Atlas"
  }
}
```

- 优点：不改开发人员手写代码；机器生成内容集中管理；Git diff 清晰；回滚简单；view 越多，集中式管理优势越明显。
- 缺点：依赖 LookML include 顺序；如果项目大量使用 wildcard include，refinement 的生效顺序可能不稳定；需要在 model 文件中做一次性接入。

**真正的分歧从来不是"能不能同步"，而是"机器生成的描述该写进谁的文件"。**

## 三、GPT-5.5：全面，但更偏稳健落地

GPT 给出的整体流程清晰：通过 Atlas API 读取元数据 → 更新 Looker 模型 → 手动触发 deploy。在更新这一步，它给了两个选项（native refinements 与直接改源文件），调研后推荐**直接修改源文件**。

核心理由是：当前 Looker 代码大量使用 wildcard includes，而 Looker 官方文档说明 wildcard include 无法保证导入顺序——如果用 generated refinement layer，会在你最需要确定性的地方引入顺序不稳定问题。

这一点很关键：refinement 是否生效与 include 顺序强相关，通配符 include 下 refinement 文件不一定在正确位置被加载，字段描述能否覆盖成功就有了不确定性。**GPT 的价值在于它没有停在"最佳实践"，而是把官方机制和你项目的现状摆在一起判断。** 不足是它最终偏保守，没有进一步探索"refinement 能否通过架构调整继续保留"。

## 四、Gemini 3.5 Flash：推荐了最佳实践，但没充分结合项目现状

Gemini 同样给了两个方案：In-Place Edit，以及生成独立的 `atlas_refinements.lkml`（利用 `view: +view_name` 语法）。它强烈推荐后者，理由是零解析风险、零 Git 冲突、Git 历史更干净、机器生成与人工代码物理隔离——这些本身都成立。

但问题在于：**Gemini 没有意识到当前项目大量使用 wildcard include**，也就没做足够的代码搜索和上下文分析，便假设 refinement 方案能稳定工作。当继续追问 wildcard include 怎么解决时，它一开始认为没办法，随后改口支持直接修改源文件：

> 解决方案：转向"精准直接修改文件"（Direct Source-File Editing）。我非常赞同您的意见：对于当前的项目架构，直接修改源文件是更好的方案。

**它能给出看起来合理的最佳实践，但约束一变，方案的稳定性和自我纠错就跟不上了。**

## 五、Claude Fable 5：结构性最强，追问后给出更优解

Claude Fable 5 用 `/deep-research` dynamic workflow 做调研，给出三个方案：

- Proposal A — Generated refinement layer（推荐）
- Proposal B — 用 lkml parser 直接改源文件
- Proposal C — 精准的 in-place 文本补丁

一开始它也推荐 Proposal A，同样没有主动指出 wildcard include 的问题。真正有意思的是：追问之后，它没有否定方案 A，而是给出一个更好的结构性解法——

```lookml
# 机器生成的描述层，独立目录，并放在所有 view include 之后
include: "/views/**/*.view.lkml"
include: "/generated/atlas_refinements/*.lkml"
```

也就是把所有机器生成的 refinement 放到独立目录，并在每个 model 文件中显式排在所有 view include 之后。Claude 进一步说明，这个发现反而强化了方案 A：当前项目有几十个 dimension view，方案 B/C 要去几十个源文件精确定位并改 description，diff 散落、全量解析回写可能扰乱格式注释，文件越多误伤越大；而方案 A 把机器管理的描述集中在一个独立目录，原始 view 一个字都不动，diff 集中、删除 generated 目录即可回滚，view 越多优势越明显。唯一代价是一次性接入——每个 model 文件加一行独立 include。

**追问之后它没有推翻方案 A，而是把方案 A 做得更稳——这才是结构性推理的价值。**

## 六、三者对比总结

| 维度 | GPT-5.5 | Gemini 3.5 Flash | Claude Fable 5 |
| --- | --- | --- | --- |
| 方案完整性 | 高 | 中 | 高 |
| 工程风险识别 | 强 | 较弱 | 强 |
| 对项目现状的结合 | 较好 | 不足 | 较好 |
| 主动探索复杂问题 | 较强 | 一般 | 最强 |
| 结构化表达 | 强 | 中 | 最强 |
| 遇到反例后的修正能力 | 稳健 | 偏被动 | 很强 |
| 最终方案质量 | 高 | 中 | 最高 |

```
                 Same Engineering Goal
                /          |            \
           GPT-5.5     Gemini 3.5     Claude Fable 5
              |            |               |
        Direct source  Generated      Generated layer +
        file editing   refinement     explicit include
              |        (missed         after all views
        risk-aware     wildcard)            |
                      fast but         best long-term
                      less context     maintainability
```

我的主观感受是：Claude Fable 5 综合实力最强，尤其在复杂方案设计、结构化拆解和长期维护成本判断上；GPT 表现也很好，角度全面、对风险敏感；Gemini 3.5 Flash 差一档——它能快速给出符合"通用最佳实践"的方案，但对具体代码结构、项目约束和隐藏风险的识别不够充分。

## 七、这次调研给我的三个启示

**1. Claude Fable 5 的综合方案能力更强。** 尤其在追问 wildcard include 之后，它没有简单推翻原方案，而是找到更好的折中：保留 generated refinement layer 的长期维护优势，同时用显式 include 顺序解决 Looker refinement 的不确定性。它不只回答"能不能做"，而是在找"更适合当前系统长期演进的方案"。

**2. 模型仍然无法替代工程师的主动判断。** 即使表现最好的 Claude，一开始也没主动指出 wildcard include 的风险。如果工程师自己没意识到这个关键点，模型很可能给出一个看似合理、实则埋雷的方案。AI 能提升调研效率、扩展思路、整理方案，但关键约束、系统现状和长期维护成本，仍然需要工程师主动判断。**模型只能对你说出口的约束做推理；你忘掉的那个，正是会沉掉整个设计的那个。**

**3. 多模型交叉思考，再由最强模型汇总，效果更好。**

```
先让不同模型独立分析
   ↓
找出它们的分歧点
   ↓
人工补充关键项目约束
   ↓
再交给最强模型做汇总和方案收敛
```

GPT 更全面、风险意识强；Claude 更擅长结构化方案和复杂推理；Gemini 速度快，适合做快速 brainstorming。只用一个模型容易被单一方案带偏，多个模型分别思考再对比差异，方案完备性会明显提高。

## 八、我更认可的方案

如果基于这次调研做最终方案，我更倾向 Claude 修正后的 Proposal A：

```
Atlas API
   ↓
Generate atlas_refinements.lkml
   ↓
Place generated files under an independent directory
   ↓
Explicitly include generated refinements after all view includes
   ↓
Commit / PR review
   ↓
Deploy Looker
```

核心设计原则：Atlas 仍是字段描述的 source of truth；原始 LookML view 不被脚本修改；机器生成的 description layer 独立管理；model 文件显式控制 include 顺序；通过 PR diff 审查生成内容；出问题删除 generated 目录即可快速回滚。

**一句话：不要让脚本直接改人写的 LookML；让脚本只管理机器生成的描述层。**

## 九、结论

这次对比最大的感受是：AI 模型已经非常适合辅助复杂工程方案调研，但还不能替代工程师做最终架构判断。Claude Fable 5 的结构化推理和 deep-research 确实更强，GPT 的全面性和风险意识也很好，Gemini 3.5 Flash 在快速生成方案上有价值，但复杂项目里需要更谨慎地验证它的结论。

真正高效的方式不是盲信某一个模型，而是：用多个模型扩展思路，用工程师经验识别关键约束，再用最强模型做方案收敛。模型负责把思路铺开，工程师负责把约束顶到台面上——这两件事谁也替不了谁，而这，可能会成为以后技术方案设计的一种新工作流。
