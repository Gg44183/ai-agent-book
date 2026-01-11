# Part 2: Tools and Extensions

> Giving Agents real execution capability: Tool Calling, MCP Protocol, Skills System

## Chapter List

| Chapter | Title | Core Question |
|---------|-------|---------------|
| 03 | Tool Calling Fundamentals | How do you make LLMs call external functions? |
| 04 | MCP Protocol Deep Dive | How do you standardize Agent-to-external-system connections? |
| 05 | Skills System | How do you build reusable Agent capabilities? |
| 06 | Hooks and Events | How do you extend Agent lifecycle and package for distribution? |

## Learning Objectives

After completing this Part, you will be able to:
- Implement Function Calling tool definitions
- Understand MCP (Model Context Protocol) architecture
- Design reusable Skills systems
- Use Hooks to extend Agent behavior

## Shannon Code Guide

```
Shannon/
├── python/llm-service/tools/           # Tool implementations
├── python/llm-service/roles/presets.py # Skills presets
└── docs/pattern-usage-guide.md         # Pattern guide
```

## Hot Topic Connections

- **MCP**: Protocol for standardizing tool connections between Clients and Servers (support varies by product/version)
- **Hooks**: Event-driven extension mechanism (varies by framework/client)
- **Plugins**: Capability packaging and community sharing

## Prerequisites

- Part 1 completed
- JSON Schema basics
- HTTP/gRPC basics
