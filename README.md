# AI Agent Architecture - The Book

> **From Concept to Production: Framework-Agnostic AI Agent Architecture Patterns**

[![Shannon OSS](https://img.shields.io/badge/Reference%20Implementation-Shannon%20OSS-blue)](https://github.com/Kocoro-lab/Shannon)
[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

**Author**: [Wayland Zhang](https://waylandz.com)

---

## Languages / 语言 / 言語

| Language | Status | Link |
|----------|--------|------|
| **中文** (Chinese) | Complete | [阅读中文版](./zh/README.md) |
| **日本語** (Japanese) | Complete | [日本語版を読む](./jp/README.md) |
| **English** | Coming Soon | - |

---

## About This Book

This book is a practical guide to understanding AI Agent system design patterns, not just another framework tutorial.

**9 Parts, 30 Chapters** covering:

| Part | Topic |
|------|-------|
| Part 1 | Agent Fundamentals (ReAct Loop) |
| Part 2 | Tools & Extensions (MCP, Skills, Hooks) |
| Part 3 | Context & Memory |
| Part 4 | Single Agent Patterns (Planning, Reflection, CoT) |
| Part 5 | Multi-Agent Orchestration (DAG, Supervisor, Handoff) |
| Part 6 | Advanced Reasoning (ToT, Debate, Research) |
| Part 7 | Production Architecture |
| Part 8 | Enterprise Features (Token Budget, OPA, WASI Sandbox) |
| Part 9 | Frontier Practices (Computer Use, Agentic Coding) |

---

## Reference Implementation

This book uses [Shannon](https://github.com/Kocoro-lab/Shannon) as a reference implementation - a three-layer multi-agent system:

```
Orchestrator (Go)    - Orchestration, Budget, Policy
Agent Core (Rust)    - Execution, Sandbox, Rate Limiting
LLM Service (Python) - Inference, Tools, Vectors
```

---

## License

Book content: [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)

Shannon OSS code: [Apache 2.0](https://github.com/Kocoro-lab/Shannon/blob/main/LICENSE)
