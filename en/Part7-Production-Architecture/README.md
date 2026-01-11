# Part 7: Production Architecture

> Shannon Core Value: Three-Layer Architecture, Temporal Workflows, Observability

## Chapter List

| Chapter | Title | Core Question |
|---------|-------|---------------|
| 20 | Three-Layer Architecture | Why do we need Go/Rust/Python separation? |
| 21 | Temporal Workflows | How to achieve durable execution and time-travel debugging? |
| 22 | Observability | How to monitor and debug production Agent systems? |

## Learning Objectives

After completing this Part, you will be able to:
- Understand the value of control plane and execution plane separation
- Use Temporal to implement durable workflows
- Implement time-travel debugging (deterministic replay)
- Build a complete observability system

## Shannon Code Guide

```
Shannon/
├── go/orchestrator/                    # Go: Orchestration layer
├── rust/agent-core/                    # Rust: Execution layer
├── python/llm-service/                 # Python: LLM layer
└── deploy/                             # Deployment configuration
```

## Core Value

| Feature | Other Frameworks | Shannon |
|---------|------------------|---------|
| Time-Travel Debugging | None | Temporal complete history |
| Deterministic Replay | None | Exportable replay |
| Architecture Separation | Python monolith | Three-layer polyglot |

## Prerequisites

- Parts 1-6 completed
- Distributed systems fundamentals
- Prometheus/Grafana basics
