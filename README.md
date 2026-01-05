# 《AI Agent 架构：从单体到企业级多智能体》

> **从概念到生产：框架无关的 AI Agent 架构模式与实战**

[![Shannon OSS](https://img.shields.io/badge/Reference%20Implementation-Shannon%20OSS-blue)](https://github.com/Kocoro-lab/Shannon)
[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

**作者**: [Wayland Zhang](https://waylandz.com)

---

## 这本书讲什么？

**不是框架文档，而是让你理解 Agent 系统设计模式的实战指南。**

市面上的 Agent 教程大多停留在：
- 调用 API 实现一个 chatbot Demo
- 某个框架的配置说明和 API 翻译
- Prompt 技巧的堆砌

这些只能让你"用"Agent，不能让你"构建"生产级 Agent 系统。

真正的生产系统需要回答：
- **多 Agent 如何协作？** DAG、Supervisor、还是 Handoff？
- **Token 预算如何控制？** 单次调用还是整个 workflow？
- **工具执行失败怎么办？** 重试、降级、还是人工介入？
- **如何保证安全？** 沙箱隔离、权限控制、审计日志

这本书回答这些问题，配合开源项目 [Shannon](https://github.com/Kocoro-lab/Shannon) 提供完整的代码参考。

---

## 目标读者

| 读者类型 | 你会获得什么 |
|----------|-------------|
| **后端开发者** | 从零构建 Agent 系统的完整路径 |
| **架构师** | 多 Agent 编排、企业级治理的设计模式 |
| **技术负责人** | 评估和落地 Agent 方案的决策框架 |

### 前置要求

- **必需**：基本编程能力 (Go/Python/Rust 任一)
- **必需**：LLM 基础概念 (Token, Prompt, Temperature)
- **有帮助**：了解 REST/gRPC API
- **不需要**：不需要了解任何特定 Agent 框架

### 可能不适合你，如果：

- 你只想快速调用 ChatGPT API（直接看官方文档）
- 你需要 Prompt Engineering 技巧集锦（有更专门的资料）
- 你从未接触过 LLM 相关概念（建议先了解基础）

---

## 内容结构

全书分为 **9 个部分、30 章**：

| 部分 | 主题 | 核心内容 |
|------|------|----------|
| **Part 1** | Agent 基础 | Agent 本质、ReAct 循环 |
| **Part 2** | 工具与扩展 | Function Calling、MCP、Skills、Hooks |
| **Part 3** | 上下文与记忆 | Context 管理、Memory 架构、会话设计 |
| **Part 4** | 单 Agent 模式 | Planning、Reflection、Chain-of-Thought |
| **Part 5** | 多 Agent 编排 | DAG、Supervisor、Handoff |
| **Part 6** | 高级推理 | Tree-of-Thoughts、Debate、Research |
| **Part 7** | 生产架构 | 三层设计、Temporal、可观测性 |
| **Part 8** | 企业级特性 | Token 预算、OPA 策略、WASI 沙箱、多租户 |
| **Part 9** | 前沿实践 | Computer Use、Agentic Coding、Background Agents |

```
Part1-Agent基础/        Agent概念、ReAct循环
Part2-工具与扩展/       Tools、MCP、Skills、Hooks
Part3-上下文与记忆/     Context管理、Memory架构
Part4-单Agent模式/      Planning、Reflection、CoT
Part5-多Agent编排/      DAG、Supervisor、Handoff
Part6-高级推理/         ToT、Debate、Research
Part7-生产架构/         三层架构、Temporal、可观测性
Part8-企业级特性/       预算控制、OPA、WASI沙箱
Part9-前沿实践/         Computer Use、Agentic Coding
附录/                   案例、模板、框架对比
```

---

## 参考实现：Shannon

本书以 [Shannon](https://github.com/Kocoro-lab/Shannon) 作为参考实现。Shannon 是一个三层架构的多 Agent 系统：

```
Orchestrator (Go)    - 编排、预算、策略
Agent Core (Rust)    - 执行、沙箱、限流
LLM Service (Python) - 推理、工具、向量
```

Shannon 不是唯一选择。LangGraph、CrewAI、AutoGen 都能做类似的事。本书的目标是教你**设计模式**，不是教你用 Shannon。

---

## 写作理念

**模式优先，框架其次。**

框架会过时，但模式不会。每一章都遵循这样的结构：

1. **先讲问题** — 什么场景需要这个能力？
2. **再讲模式** — 通用的设计模式是什么？（框架无关）
3. **然后看实现** — 以 Shannon 为例展示一种实现方式
4. **最后对比** — 其他框架如何解决同样的问题

如果你读完一章，能在任何框架中实现同样的模式——那这一章就成功了。

---

## 阅读路径

### 快速入门（2-3 天）
```
Part1 全部 → 第03章 → 第13章 → 第20章
```
目标：Agent 基础 → 工具调用 → 多 Agent 编排 → 生产架构

### 系统学习（2-3 周）
```
Part1-8 顺序阅读，配合 Shannon 代码实践
```
目标：完整掌握从单 Agent 到企业级多 Agent 的全部内容

### 前沿热点（1-2 天）
```
第04章(MCP) → 第27章(Computer Use) → 第28章(Agentic Coding)
```
目标：了解 2024-2025 年最热门的 Agent 话题

---

## 贯穿全书的实战项目

> **项目：智能研究助手 (Research Agent)**

从 Part1 到 Part9，我们将逐步构建一个完整的研究助手 Agent：

| Part | 项目演进 | 能力提升 |
|------|----------|----------|
| Part 1 | 简单问答 Agent | 基础 ReAct 循环 |
| Part 2 | + 工具调用 | 搜索、文件读取 |
| Part 3 | + 记忆系统 | 多轮对话、历史召回 |
| Part 4 | + 自主规划 | 任务分解、反思改进 |
| Part 5 | + 多 Agent 协作 | 搜索 + 分析 + 写作 Agent |
| Part 6 | + 高级推理 | 多源对比、辩论综合 |
| Part 7 | + 生产架构 | Temporal 持久化、可观测性 |
| Part 8 | + 企业治理 | Token 预算、权限控制 |
| Part 9 | + 前沿能力 | 浏览器操作、代码生成 |

---

## 关于时效性

AI Agent 领域发展极快。本书对高变动性内容会明确标注：

> **时效性提示** (2025-01): 本节内容基于 MCP 规范 v1.0。

核心的架构模式相对稳定，但具体的 API 和工具可能会变化。遇到疑问时，请以官方文档为准。

---

## 快速导航

| 链接 | 说明 |
|------|------|
| [前言](./前言.md) | 为什么写这本书、写作理念、适合谁读 |
| [完整目录](./TABLE_OF_CONTENTS.md) | 30 章完整目录与学习路径 |
| [第 1 章：Agent 的本质](./Part1-Agent基础/第01章：Agent的本质.md) | 开始阅读 |
| [Shannon OSS](https://github.com/Kocoro-lab/Shannon) | 参考实现代码 |

---

## License

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 协议。

配套代码 Shannon OSS 采用 [Apache 2.0](https://github.com/Kocoro-lab/Shannon/blob/main/LICENSE) 协议。

---

> *"The best way to learn is to build."*
