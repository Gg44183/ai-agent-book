# Part 1: Agent Fundamentals

> Understanding the essence of AI Agents, from LLMs to autonomous intelligent entities

## Chapter List

| Chapter | Title | Core Question |
|---------|-------|---------------|
| 01 | The Essence of Agents | What is an Agent? How does it differ from a regular Chatbot? |
| 02 | The ReAct Loop | How does an Agent think and act? |

## Learning Objectives

After completing this Part, you will be able to:
- Understand the definition and autonomy spectrum of Agents
- Master the ReAct (Reason-Act-Observe) basic loop
- Distinguish the fundamental difference between Agents and traditional Chatbots
- Understand the overall design philosophy of Shannon architecture

## Shannon Code Guide

```
Shannon/
├── docs/multi-agent-workflow-architecture.md  # Architecture overview
├── go/orchestrator/internal/workflows/strategies/react.go   # ReactWorkflow (workflow layer)
└── go/orchestrator/internal/workflows/patterns/react.go     # ReactLoop (pattern layer)
```

## Prerequisites

- LLM fundamentals (Prompt, Token, Temperature)
- Basic programming ability (Go/Python either one)
