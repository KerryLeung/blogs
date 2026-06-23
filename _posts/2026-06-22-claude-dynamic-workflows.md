---
layout:     post
title:      "Agent Harness 新实践：Claude Code 动态工作流"
subtitle:   "从 prompt 驱动走向 workflow 驱动"
date:       2026-06-22
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - AI
    - LLM
    - agent
lang: zh
ref: claude-dynamic-workflows
---

> Anthropic 最近发布了 Claude Code Dynamic Workflows：Claude Code 不再只是单个 agent 在一个上下文里干活，而是可以为当前任务临时写一套专属 harness——一个 JavaScript workflow，用来调度多个子 agent 协作。我读完原文又试了一下，最大的体会是：过去写进 prompt 里的那些"请仔细检查""别漏测试"，正在变成结构化的执行流程。本文是一篇功能梳理 + 我自己的思考。原文：[A harness for every task: Dynamic Workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)。

## 一、Dynamic Workflows 是什么

核心一句话：**Claude Code 可以为任务即时生成自己的 harness。** 它执行一个 JavaScript workflow，文件里有一组特殊函数，用来 spawn 并协调多个子 agent，而不是把所有事情塞进单一上下文窗口。

这些子 agent 可以有各自独立的上下文窗口和聚焦的目标，还可以指定各自使用的模型，甚至运行在各自独立的 worktree 中；workflow 本身就是普通 JavaScript（可以用 JSON、Math、Array 等），并支持中断后恢复。

原文给出六个典型编排模式：

- **Classify-and-act**：先按分类把任务路由到不同 agent。
- **Fan-out-and-synthesize**：把任务拆成很多小步，每步跑一个 agent，再统一合成结果。
- **Adversarial verification**：每 spawn 一个 agent，就再 spawn 一个独立 agent 去"对抗式"验证它的产出。
- **Generate-and-filter**：先就某个主题生成大量候选，再按 rubric 筛选。
- **Tournament**：让多个 agent 用不同思路同时解同一道题，互相竞争，选出最优。
- **Loop until done**：持续 spawn agent，直到满足某个停止条件。

## 二、它想解决什么问题

传统 Claude Code 处理普通编码任务已经很强，但任务一复杂、链路一长，单上下文就开始吃力：信息全挤在一个窗口里，容易溢出或丢细节。在这个大背景下，原文明确点名了单上下文的三个失败模式：

- **Agentic laziness**：在复杂的多步任务里，Claude 做到一半就宣布"完成了"，留下没做完的部分。
- **Self-preferential bias**：当你让它验证自己的产出时，它倾向于认可自己的结果——自己写、自己检查，天然有偏向。
- **Goal drift**：多轮交互、多次压缩之后，对原始目标、边界条件和特殊要求的"保真度"逐渐流失。

Dynamic Workflows 的设计目标，就是用多个独立上下文和结构化流程把这几类问题压下去。

## 三、适合什么场景

它更适合复杂、高价值、长链路的任务，而不是简单的代码改动。原文推荐的方向，加上我自己想到的场景：

- 大规模迁移 / 重构
- 深度调研与事实核查（技术文档、商业/技术方案多视角评审）
- 排序、排名、按 rubric 选优
- 在大量产出上统一执行某条规则（如安全审查、PR 风险评估）
- 线上问题 root cause 分析
- 大规模 triage：批量处理工单、日志、incident 记录、简历
- 从历史 session 中总结常见纠正点
- 轻量级的模型评测

一句话：**只要一个任务需要拆解、并行、验证、合成，Dynamic Workflows 就有发挥空间。**

## 四、它改变了什么

我觉得最大的变化是：**AI Coding 正在从 prompt 驱动，走向 workflow 驱动。**

过去我们更多是在写 prompt——"请一步步检查""请别漏测试""请帮我 review"。这些其实是对着同一个上下文反复叮嘱。Dynamic Workflows 把这些要求变成更稳定的执行结构：先规划、再拆解、多 agent 并行执行、独立 agent 交叉验证、最后统一合成、不满足条件就继续循环。

这让 AI 更像一个可以临时组建的小型工程团队，而不是单个助手。

## 五、我自己的思考

简单试用后，我的感受是 agent 内置的 harness 能力又上了一个台阶。

它最直接的价值是缓解复杂任务里的上下文窗口压力：不同子任务交给不同 agent、用各自独立的上下文完成，不需要把所有信息都挤在同一个窗口里。

更关键的是质量。**让做这件事的上下文同时去给它打分，是不可靠的；换一个独立上下文来审查，才跳得出自我背书的偏向。** 一个 agent 负责实现，另一个按 checklist 审查，第三个汇总结论——这正是 adversarial verification 模式的意义。

我觉得后续最值得探索的，是 **Dynamic Workflows + Skills 的组合**：把团队的常见流程沉淀成 skill——schema 变更检查、SQL review、migration checklist、release note、regression test、incident analysis、PR review rubric——再用 Dynamic Workflows 把这些 skill 串成一套更自动化、更可复用的工程流程。这可能是 AI 助手从"帮我写代码"进化到"帮我管理复杂工程任务"的关键一步。

## 六、总结

Dynamic Workflows 不是要取代普通的 Claude Code 用法，而是为复杂任务提供更强的组织能力。简单任务，直接用 Claude Code 就够了；复杂任务，尤其是需要拆解、并行、验证、循环推进的任务，再考虑上 Dynamic Workflows。

推荐结合自己的真实工作场景实践一下——真正有价值的，不只是看懂这个功能，而是找到适合自己团队和个人开发流程的用法。原文：<https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code>
