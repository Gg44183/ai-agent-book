# Part 4: Single Agent Patterns

> Deep dive into single Agent reasoning capabilities: planning, reflection, chain-of-thought

## Chapter List

| Chapter | Title | Core Question |
|---------|-------|---------------|
| 10 | Planning Pattern | How does an Agent decompose complex tasks? |
| 11 | Reflection Pattern | How does an Agent self-evaluate and improve? |
| 12 | Chain-of-Thought | How to make Agents show their reasoning process? |

## Learning Objectives

After completing this Part, you will be able to:
- Implement automatic task decomposition (Decomposition)
- Design reflection-improvement loops
- Understand CoT principles and best practices
- Evaluate single Agent capability boundaries

## Shannon Code Guide

```
Shannon/
├── go/orchestrator/internal/activities/
│   └── agent_activities.go             # /agent/decompose
├── go/orchestrator/internal/workflows/
│   └── patterns/                       # Reasoning pattern library
└── docs/pattern-usage-guide.md
```

## Pattern Comparison

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| Planning | Multi-step tasks | Medium |
| Reflection | Quality-sensitive tasks | Medium |
| CoT | Logical reasoning tasks | Low |

## Prerequisites

- Part 1-3 completed
- Prompt Engineering basics
