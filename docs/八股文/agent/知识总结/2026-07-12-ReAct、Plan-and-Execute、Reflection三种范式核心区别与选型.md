---
title: ReAct、Plan-and-Execute、Reflection 三种范式核心区别与选型
date: 2026-07-12
source:
  - https://xiaolinnote.com/ai/agent/6_three_patterns.html
tags:
  - AI
  - Agent
  - 面试
  - 八股文
---

# AI Agent 设计范式

## ReAct、Plan-and-Execute、Reflection 三种范式有什么核心区别？实际项目中该如何选型？

**15秒简答**

ReAct、Plan-and-Execute、Reflection 是 Agent 的三种典型推理控制范式：ReAct 是“边想边做边观察”，Plan-and-Execute 是“先规划再执行”，Reflection 是“执行后自评并修正”。ReAct 强在灵活探索，Plan-and-Execute 强在全局拆解，Reflection 强在质量闭环；工程选型看任务是否明确、链路是否复杂、错误代价是否高。

**3分钟详答**

这三个范式解决的是同一个问题：LLM 不是只生成一段答案，而是要在复杂任务里持续决策、调用工具、处理反馈并修正方向。差别在于它们把“思考、行动、纠错”放在了不同位置。

ReAct 的核心是 Reasoning + Acting。模型每一轮根据当前上下文先判断下一步要做什么，再调用工具，拿到 Observation 后继续下一轮。它适合信息不完整、需要边查边判断的探索型任务，例如检索资料、查询 API、开放式问答。它的问题是缺少全局计划，链路长了容易漂移，token 和延迟也会随轮次增加。

Plan-and-Execute 的核心是先由模型制定完整或半完整计划，再让执行器按步骤推进。它适合目标明确、步骤较多、需要拆解的任务，例如“调研某技术并产出报告”“完成一组文件修改”“多步骤数据处理”。它比 ReAct 更有全局感，能减少走一步看一步的混乱，但计划如果一开始错了，后续执行也会被带偏，所以通常需要中途检查和重规划。

Reflection 的核心是引入自我评估与修正闭环。模型先执行任务，再检查自己的输出、错误、遗漏或不一致之处，然后基于反馈迭代。它适合结果质量要求高、需要反复打磨的任务，例如代码生成、复杂推理、文档审校、长答案修订。它的代价是额外轮次和成本，并且“自我反思”不等于真实正确，关键场景仍需要外部验证器、测试或人工评审。

原文的图表和案例主要想说明：这三种范式不是互斥关系，而是三种控制粒度。ReAct 偏即时决策，Plan-and-Execute 偏任务编排，Reflection 偏质量校正。实际工程里经常组合使用：先 Plan 拆任务，再用 ReAct 执行每一步，最后用 Reflection 做检查与修正。

**追问：ReAct 的核心机制是什么？**

**15秒简答：** ReAct 是 Thought -> Action -> Observation 的循环，模型每轮先推理下一步，再调用工具，最后把工具结果写回上下文继续决策。

**追问：Plan-and-Execute 为什么比 ReAct 更适合复杂任务？**

**15秒简答：** 因为它先做全局任务拆解，再逐步执行，能减少 ReAct 走一步看一步导致的漂移，适合目标清晰、步骤较多的任务。

**追问：Reflection 的价值在哪里？**

**15秒简答：** Reflection 给 Agent 加了一层“自查和修正”机制，让模型不只输出一次结果，而是根据反馈发现遗漏、错误和不一致，再迭代改进。

**追问：这三种范式如何选型？**

**15秒简答：** 任务开放、信息不全选 ReAct；目标明确、步骤复杂选 Plan-and-Execute；结果质量要求高、错误代价高时加 Reflection。

**追问：实际项目中会单独使用某一种范式吗？**

**15秒简答：** 可以，但复杂项目更常组合使用：Plan-and-Execute 负责拆任务，ReAct 负责每步工具执行，Reflection 负责最终检查和修正。

**资料来源**

- 小林 Coding：ReAct、Plan-and-Execute、Reflection 三种范式有什么核心区别？实际项目中该如何选型？  
  https://xiaolinnote.com/ai/agent/6_three_patterns.html
