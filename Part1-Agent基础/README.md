# Part 1: Agent基础

> 理解AI Agent的本质，从LLM到自主智能体的演进

## 章节列表

| 章节 | 标题 | 核心问题 |
|------|------|----------|
| 01 | Agent的本质 | 什么是Agent？与普通Chatbot有何不同？ |
| 02 | ReAct循环 | Agent如何思考和行动？ |

## 学习目标

完成本Part后，你将能够：
- 理解Agent的定义和自主性谱系
- 掌握ReAct (Reason-Act-Observe) 基础循环
- 区分Agent与传统Chatbot的本质差异
- 了解Shannon架构的整体设计理念

## Shannon代码导读

```
Shannon/
├── docs/multi-agent-workflow-architecture.md  # 架构总览
├── go/orchestrator/internal/workflows/strategies/react.go   # ReactWorkflow（工作流层）
└── go/orchestrator/internal/workflows/patterns/react.go     # ReactLoop（模式层）
```

## 前置知识

- LLM基础概念 (Prompt、Token、Temperature)
- 基本编程能力 (Go/Python任一)
