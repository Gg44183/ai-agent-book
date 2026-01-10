# Part 6: 高级推理

> 复杂决策场景：思维树、多Agent辩论、研究综合

## 章节列表

| 章节 | 标题 | 核心问题 |
|------|------|----------|
| 17 | Tree-of-Thoughts | 如何探索多条推理路径？ |
| 18 | Debate模式 | 如何通过辩论提升决策质量？ |
| 19 | Research Synthesis | 如何综合多源信息生成报告？ |

## 学习目标

完成本Part后，你将能够：
- 实现ToT (思维树) 探索和剪枝
- 设计多Agent辩论机制
- 构建研究综合工作流
- 选择合适的推理模式

## Shannon代码导读

```
Shannon/
├── go/orchestrator/internal/workflows/
│   ├── patterns/tot.go                 # Tree-of-Thoughts
│   ├── patterns/debate.go              # Debate模式
│   └── research_workflow.go            # Research综合
└── docs/pattern-usage-guide.md
```

## 模式选择指南

| 场景 | 推荐模式 | 原因 |
|------|----------|------|
| 开放性问题 | ToT | 需要探索多种可能 |
| 有争议决策 | Debate | 多角度论证 |
| 信息收集 | Research | 多源并行+综合 |

## 前置知识

- Part 1-5 完成
- 决策理论基础
- 信息检索基础
