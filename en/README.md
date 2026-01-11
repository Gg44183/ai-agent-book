# AI Agent Architecture: From Monolith to Enterprise Multi-Agent

> **From Concept to Production: Framework-Agnostic AI Agent Architecture Patterns and Practices**

[![Shannon OSS](https://img.shields.io/badge/Reference%20Implementation-Shannon%20OSS-blue)](https://github.com/Kocoro-lab/Shannon)
[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

**Author**: [Wayland Zhang](https://waylandz.com)

---

## What This Book Covers

![From Single Agent to Enterprise Multi-Agent Systems](assets/book-overview-hero.svg)

**Not framework documentation—this is a practical guide to Agent system design patterns.**

Most agent tutorials stop at:
- Calling APIs to make a chatbot demo
- Configuration instructions and API translations for some framework
- Collections of Prompt tricks

These only let you "use" Agents, not "build" production-grade Agent systems.

Real production systems need to answer:
- **How do multiple Agents collaborate?** DAG, Supervisor, or Handoff?
- **How do you control token budgets?** Per call or across the whole workflow?
- **What happens when tool execution fails?** Retry, degrade, or human intervention?
- **How do you ensure security?** Sandbox isolation, access control, audit logs

This book answers these questions, using the open-source project [Shannon](https://github.com/Kocoro-lab/Shannon) as a complete reference implementation.

---

## Target Audience

| Reader Type | What You'll Get |
|-------------|-----------------|
| **Backend Developers** | Complete path to building Agent systems from scratch |
| **Architects** | Design patterns for multi-Agent orchestration, enterprise governance |
| **Technical Leads** | Decision framework for evaluating and implementing Agent solutions |

### Prerequisites

- **Required**: Basic programming ability (any of Go/Python/Rust)
- **Required**: LLM basic concepts (Token, Prompt, Temperature)
- **Helpful**: Familiarity with REST/gRPC APIs
- **Not needed**: No need to know any specific Agent framework

### This book may not be for you if:

- You just want to quickly call the ChatGPT API (just read the official docs)
- You need a collection of Prompt Engineering tricks (there are more specialized resources)
- You've never encountered LLM-related concepts (recommend learning basics first)

---

## Content Structure

The book is divided into **9 parts, 30 chapters**:

| Part | Topic | Core Content |
|------|-------|--------------|
| **Part 1** | Agent Fundamentals | Agent essence, ReAct loop |
| **Part 2** | Tools and Extensions | Function Calling, MCP, Skills, Hooks |
| **Part 3** | Context and Memory | Context management, Memory architecture, session design |
| **Part 4** | Single Agent Patterns | Planning, Reflection, Chain-of-Thought |
| **Part 5** | Multi-Agent Orchestration | DAG, Supervisor, Handoff |
| **Part 6** | Advanced Reasoning | Tree-of-Thoughts, Debate, Research |
| **Part 7** | Production Architecture | Three-layer architecture, Temporal, Observability |
| **Part 8** | Enterprise Features | Token budget, OPA policy, WASI sandbox, multi-tenancy |
| **Part 9** | Frontier Practices | Computer Use, Agentic Coding, Background Agents |

```
Part1-Agent-Fundamentals/     Agent concepts, ReAct loop
Part2-Tools-and-Extensions/   Tools, MCP, Skills, Hooks
Part3-Context-and-Memory/     Context management, Memory architecture
Part4-Single-Agent-Patterns/  Planning, Reflection, CoT
Part5-Multi-Agent-Orchestration/ DAG, Supervisor, Handoff
Part6-Advanced-Reasoning/     ToT, Debate, Research
Part7-Production-Architecture/ Three-layer architecture, Temporal, Observability
Part8-Enterprise-Features/    Budget control, OPA, WASI sandbox
Part9-Frontier-Practices/     Computer Use, Agentic Coding
Appendix/                     Cases, templates, framework comparison
```

---

## Reference Implementation: Shannon

This book uses [Shannon](https://github.com/Kocoro-lab/Shannon) as the reference implementation. Shannon is a three-layer architecture multi-Agent system:

```
Orchestrator (Go)    - Orchestration, budget, policy
Agent Core (Rust)    - Execution, sandbox, rate limiting
LLM Service (Python) - Inference, tools, vectors
```

Shannon isn't the only choice. LangGraph, CrewAI, AutoGen can all do similar things. This book's goal is to teach you **design patterns**, not how to use Shannon.

---

## Writing Philosophy

**Patterns first, frameworks second.**

Frameworks become outdated, but patterns don't. Every chapter follows this structure:

1. **Start with the problem** -- What scenario needs this capability?
2. **Then the pattern** -- What's the universal design pattern? (framework-agnostic)
3. **Then the implementation** -- Show one implementation approach using Shannon as an example
4. **Finally, comparison** -- How do other frameworks solve the same problem

If after reading a chapter you can implement the same pattern in any framework -- that chapter has succeeded.

---

## Reading Paths

### Quick Start (2-3 days)
```
All of Part 1 -> Chapter 3 -> Chapter 13 -> Chapter 20
```
Goal: Agent basics -> Tool calling -> Multi-Agent orchestration -> Production architecture

### Systematic Learning (2-3 weeks)
```
Read Part 1-8 sequentially, practice with Shannon code
```
Goal: Complete mastery from single Agent to enterprise multi-Agent

### Hot Topics Track (1-2 days)
```
Chapter 4 (MCP) -> Chapter 27 (Computer Use) -> Chapter 28 (Agentic Coding)
```
Goal: Learn about the hottest Agent topics of 2025-2026

---

## Running Project Throughout the Book

> **Project: Intelligent Research Assistant (Research Agent)**

From Part 1 to Part 9, we progressively build a complete research assistant Agent:

| Part | Project Evolution | Capability Enhancement |
|------|-------------------|----------------------|
| Part 1 | Simple Q&A Agent | Basic ReAct loop |
| Part 2 | + Tool calling | Search, file reading |
| Part 3 | + Memory system | Multi-turn conversations, history recall |
| Part 4 | + Autonomous planning | Task decomposition, reflective improvement |
| Part 5 | + Multi-Agent collaboration | Search + Analysis + Writing Agents |
| Part 6 | + Advanced reasoning | Multi-source comparison, debate synthesis |
| Part 7 | + Production architecture | Temporal persistence, observability |
| Part 8 | + Enterprise governance | Token budget, access control |
| Part 9 | + Frontier capabilities | Browser operation, code generation |

---

## About Timeliness

The AI Agent field is evolving extremely fast. This book clearly marks highly volatile content:

> **Timeliness Note** (2026-01): This section's content is based on the MCP specification as of 2026-01.

Core architectural patterns are relatively stable, but specific APIs and tools may change. When in doubt, defer to official documentation.

---

## Quick Navigation

| Link | Description |
|------|-------------|
| [Preface](./Preface.md) | Why I wrote this book, writing philosophy, who should read |
| [Complete Table of Contents](./Table-of-Contents.md) | Full 30-chapter table of contents and learning paths |
| [Chapter 1: The Essence of Agents](./Part1-Agent-Fundamentals/Chapter-01-The-Essence-of-Agents.md) | Start reading |
| [Shannon OSS](https://github.com/Kocoro-lab/Shannon) | Reference implementation code |

---

## License

Book content is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).

Companion code Shannon OSS is licensed under [Apache 2.0](https://github.com/Kocoro-lab/Shannon/blob/main/LICENSE).

---

> *"The best way to learn is to build."*
