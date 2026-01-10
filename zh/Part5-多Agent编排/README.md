# Part 5: 多Agent编排

> 从单Agent到多Agent：编排、协调、协作

## 章节列表

| 章节 | 标题 | 核心问题 |
|------|------|----------|
| 13 | 编排基础 | 如何协调多个Agent协同工作？ |
| 14 | DAG工作流 | 如何处理任务依赖关系？ |
| 15 | Supervisor模式 | 如何动态管理Agent团队？ |
| 16 | Handoff机制 | Agent之间如何传递任务和状态？ |

## 学习目标

完成本Part后，你将能够：
- 设计Orchestrator编排架构
- 实现DAG (有向无环图) 工作流
- 使用Supervisor模式管理动态Agent
- 处理Agent间的Handoff和状态传递

## Shannon代码导读

```
Shannon/
├── go/orchestrator/internal/workflows/
│   ├── orchestrator_router.go          # 路由决策
│   ├── dag_workflow.go                 # DAG实现
│   └── supervisor_workflow.go          # Supervisor模式
└── docs/multi-agent-workflow-architecture.md
```

## 核心架构

```
Orchestrator Router
    ├── SimpleTask (复杂度 < 0.3)
    ├── DAG (一般多步任务)
    ├── React (工具密集型)
    ├── Research (信息综合)
    └── Supervisor (> 5个子任务)
```

## 前置知识

- Part 1-4 完成
- 图论基础 (DAG、拓扑排序)
- 并发编程基础
