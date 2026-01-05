# Part 4: 单Agent模式

> 深入单Agent的推理能力：计划、反思、链式思考

## 章节列表

| 章节 | 标题 | 核心问题 |
|------|------|----------|
| 10 | Planning模式 | Agent如何分解复杂任务？ |
| 11 | Reflection模式 | Agent如何自我评估和改进？ |
| 12 | Chain-of-Thought | 如何让Agent展示推理过程？ |

## 学习目标

完成本Part后，你将能够：
- 实现任务自动分解 (Decomposition)
- 设计反思-改进循环
- 理解CoT的原理和最佳实践
- 评估单Agent的能力边界

## Shannon代码导读

```
Shannon/
├── go/orchestrator/internal/activities/
│   └── agent_activities.go             # /agent/decompose
├── go/orchestrator/internal/workflows/
│   └── patterns/                       # 推理模式库
└── docs/pattern-usage-guide.md
```

## 模式对比

| 模式 | 适用场景 | 复杂度 |
|------|----------|--------|
| Planning | 多步骤任务 | 中 |
| Reflection | 质量敏感任务 | 中 |
| CoT | 逻辑推理任务 | 低 |

## 前置知识

- Part 1-3 完成
- Prompt Engineering基础
