# Part 7: 生产架构

> Shannon核心价值：三层架构、Temporal工作流、可观测性

## 章节列表

| 章节 | 标题 | 核心问题 |
|------|------|----------|
| 20 | 三层架构设计 | 为什么需要Go/Rust/Python分离？ |
| 21 | Temporal工作流 | 如何实现持久化执行和时间旅行调试？ |
| 22 | 可观测性 | 如何监控和调试生产Agent系统？ |

## 学习目标

完成本Part后，你将能够：
- 理解控制面与执行面分离的价值
- 使用Temporal实现持久化工作流
- 实现时间旅行调试 (确定性重放)
- 搭建完整的可观测性体系

## Shannon代码导读

```
Shannon/
├── go/orchestrator/                    # Go: 编排层
├── rust/agent-core/                    # Rust: 执行层
├── python/llm-service/                 # Python: LLM层
└── deploy/                             # 部署配置
```

## 核心价值

| 特性 | 其他框架 | Shannon |
|------|----------|---------|
| 时间旅行调试 | 无 | Temporal完整历史 |
| 确定性重放 | 无 | 可导出重放 |
| 架构分离 | Python单体 | 三层polyglot |

## 前置知识

- Part 1-6 完成
- 分布式系统基础
- Prometheus/Grafana基础
