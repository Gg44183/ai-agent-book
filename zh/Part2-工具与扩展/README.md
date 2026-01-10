# Part 2: 工具与扩展

> 让Agent具备真正的执行能力：工具调用、MCP协议、Skills系统

## 章节列表

| 章节 | 标题 | 核心问题 |
|------|------|----------|
| 03 | 工具调用基础 | 如何让LLM调用外部函数？ |
| 04 | MCP协议详解 | 如何标准化Agent与外部系统的连接？ |
| 05 | Skills系统 | 如何构建可复用的Agent能力？ |
| 06 | Hooks与Plugins | 如何扩展Agent的生命周期和打包分发？ |

## 学习目标

完成本Part后，你将能够：
- 实现Function Calling工具定义
- 理解MCP (Model Context Protocol) 协议架构
- 设计可复用的Skills系统
- 使用Hooks扩展Agent行为

## Shannon代码导读

```
Shannon/
├── python/llm-service/tools/           # 工具实现
├── python/llm-service/roles/presets.py # Skills预设
└── docs/pattern-usage-guide.md         # 模式指南
```

## 热门话题关联

- **MCP**: Claude Code、Cursor等工具的标准协议
- **Hooks**: Claude Code事件驱动扩展机制
- **Plugins**: 能力打包与社区分享

## 前置知识

- Part 1 完成
- JSON Schema基础
- HTTP/gRPC基础
