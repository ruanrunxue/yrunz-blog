---
title: 为什么 Claude Code 要这样设计？
description: 本文将探讨 Claude Code 的设计哲学，带大家理解主导这些组件设计的背后原因。

date: 2026-05-05
categories:
  - AI
tags:
  - AI
  - Agent
author: 元闰子
---

## 前言

Claude Code 已经成为 Coding Agent 的事实标准，现今各类 Agent 都多多少少能看到它的影子。
Claude Code 的设计简洁、优雅：
![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2026-05-05-121230.png)

- Agent Loop 是大脑、核心，驱动着所有组件的运转。
- Permission System 提供权限管理。
- Tools & Execution Environment 是手脚，负责任务执行。
- State & Persistence 提供会话和记忆的持久化，保证对话连贯。

那么，*为什么 Claude Code 要这样设计呢？*

本文将探讨 Claude Code 的设计哲学，带大家理解主导这些组件设计的背后原因。

## 为什么是 Agent Loop？

在 Claude Code 横空出世之前，LangChain 是这领域的王者，它走的是 Workflow 的路线。

用户定义好 Workflow，流程根据不同的状态在各个节点中流转，比如这样：
![](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2026-05-05-122719.png)

好处是，所有流程都在用户框定的范围内；

坏处也是，所有流程都在用户框定的范围内；

人都存在思维局限，而且**现在大模型的水平已经远超普通人**。给模型一个目标，放手让它自由发挥，反而能达到更好的效果。

另外，定义 workflow 需要一定的技术门槛，无法实现 Vibe Coding。

最后，**Workflow 本质上是一个状态机，会导致系统变得复杂，不好维护**。

所以，Claude Code 选择了 Agent Loop。

Agent Loop 遵循 ReAct 模式，本质上是一个 `while` 循环，反复循环”`模型推理->工具执行`“流程，直到任务完成。

**简单、优雅、高效**。

## 为什么要有权限管理？

Agent Loop 突破了 Workflow 的限定，但高自由度会带来安全隐患，*万一模型搞错了怎么办*？

为了避免该问题，**Claude Code 将决策与执行分离**。模型只做决策，决定调用哪些工具；工具执行则交给框架本身，并且在执行前引入 Permission System 做权限管理。

这样设计的好处是，即使模型决策错误，也能在 Permission System 进行防护，判断到底要不要执行工具。

但 Permission System 毕竟也是一段代码程序，无法在所有场景都判断正确，必要时还是需要人类介入。

所以，Permission System 设计了 3 层权限，优先级由高到低分别是：`Deny/Ask/Allow`。

**Deny-Frist 的设计意图是安全第一的同时，避免频繁让用户介入决策**；把握不住时，才 `Ask` 用户；允许用户设置 `Allow` 权限，全程交给 Agent。

## 为什么要有沙箱？

既然有了用户介入，为什么在工具调用时还需要沙箱这种隔离的执行环境？

原因很简单，**人也会犯错**。

另外，Anthropic 统计数据发现，当 Claude Code 询问用户是否执行某项工具调用时，**93% 的回答都是 Allow**！

这意味着用户不会很仔细地阅读 Claude Code 的反馈，进一步扩大了犯错的概率。

所以 Claude Code 需要沙箱、worktree 这类技术来保证，**即使用户犯错了，也不会造成无法挽回的后果**。

## 会话和记忆为什么要明文存储？

会话和记忆能够保证对话的连贯性和实现用户偏好。

LangChain 的做法是在任务完成后提取相关记忆，并存储在向量数据库、图数据库等第三方组件中。且通常不会单独存储会话内容。

Claude Code 则采用明文存储的方案：

- 把用户记忆以明文文本的方式存储到 CLAUDE.md 等本地文档；
- 把会话（用户问题、模型决策、工具调用结果）明文内容原封不动地追加到本地文档。

这个方案可以看做是对 Agent Loop 的设计延续，**让任务执行流程公开，可追溯，且用户能够非常方便地审阅和修改**。

## Claude Code的设计原则

最后，我们能总结出贯穿 Claude Code 架构的 3 个设计哲学：

- **对用户能力的放大**。增加人类单位成本能完成的任务，不是仅仅加速现有工作流，而是充分发挥模型的主观能动性，采取原本想不到、不会尝试的方式。（*Agent Loop*）
- **用户享有最终决策权**。确保用户能够实时看到 Agent 在做什么（*Interface 实时反馈*），参与关键决策（*Permission System*），并在事后能审计、回溯（*会话/记忆明文存储*）
- **安全第一**。Deny-First，且用户犯错时也能保证系统免受伤害（*Permission System*、*Execution Environment*）

> #### 参考
> 
> 1. [Dive into Claude Code: The Design Space of Today’s and Future AI Agent Systems](https://arxiv.org/abs/2604.14228), Jiacheng Liu 等

（完）
