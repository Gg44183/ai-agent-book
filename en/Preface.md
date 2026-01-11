# Preface

## Why I Wrote This Book

In the second half of 2025, I shifted from explaining AI theory to engineering Agent implementations. At first, I thought building an enterprise-ready Agent system using open-source libraries would be straightforward. But once I started building a production-grade Agent system, I ran into a problem: most tutorials either stopped at the "call an API to make a chatbot" demo stage, or were just translations of a framework's documentation.

When I needed answers to these questions, I couldn't find them:

- How do multiple Agents collaborate? DAG or Supervisor?
- How do you allocate token budgets? Per call or across the whole workflow?
- What happens when tool execution fails? How do you persist state?
- How do you do access control and auditing in enterprise environments?

These questions had no ready-made answers -- I could only figure them out by trial and error. Since existing frameworks all had shortcomings that couldn't meet enterprise-grade requirements, I decided to write one from scratch. [Shannon](https://github.com/Kocoro-lab/Shannon) is the product of that exploration -- a three-layer architecture multi-Agent system implemented in Go/Rust/Python.

This book is a systematic organization of those hard-won lessons.

---

## The Writing Philosophy of This Book

**Patterns first, frameworks second.**

Most Agent tutorials on the market are tied to specific frameworks -- how to use LangChain, how to configure CrewAI. Frameworks become outdated, but patterns don't.

Every chapter in this book follows this structure:

1. **Start with the problem** -- What scenario needs this capability?
2. **Then the pattern** -- What's the universal design pattern? (framework-agnostic)
3. **Then the implementation** -- Show one implementation approach using Shannon as an example
4. **Finally, comparison** -- How do other frameworks solve the same problem

If after reading a chapter you can implement the same pattern in any framework -- that chapter has succeeded.

---

## Who This Book Is For

**This book is for you if:**

- You want to build **production-grade** Agent systems, not just demos
- You need to handle **multi-Agent collaboration** in complex scenarios
- You care about **cost control, security, observability** -- enterprise-grade concerns
- You want to understand the **design patterns** behind various Agent frameworks
- You're a backend developer, architect, or technical lead

**This book may not be for you if:**

- You just want to quickly call the ChatGPT API (just read the official docs)
- You need a collection of Prompt Engineering tricks (there are more specialized resources)
- You've never encountered LLM-related concepts (recommend learning basics first)

---

## How to Read This Book

**Quick Start (2-3 days):**
```
All of Part 1 -> Chapter 3 -> Chapter 13 -> Chapter 20
```
Goal: Understand Agent basics, tool calling, multi-Agent orchestration, production architecture

**Systematic Learning (2-3 weeks):**
```
Read Part 1-8 sequentially, with Shannon code alongside
```
Goal: Complete mastery from single Agent to enterprise multi-Agent

**Hot Topics Track (1-2 days):**
```
Chapter 4 (MCP) -> Chapter 27 (Computer Use) -> Chapter 28 (Agentic Coding)
```
Goal: Learn about the hottest Agent topics of 2025-2026

Every chapter ends with a "Chapter Recap" that you can use to verify whether you truly understood the core concepts.

---

## About the Code

This book uses [Shannon](https://github.com/Kocoro-lab/Shannon) as the reference implementation, but it is **not** a Shannon user manual.

Shannon uses a three-layer architecture:

```
Orchestrator (Go)    - Orchestration, budget, policy
Agent Core (Rust)    - Execution, sandbox, rate limiting
LLM Service (Python) - Inference, tools, vectors
```

The code examples in this book demonstrate **design patterns**, not framework APIs. You can absolutely implement the same patterns using LangGraph, CrewAI, or your own framework.

---

## About Timeliness

The AI Agent field is evolving extremely fast. The MCP protocol was only released at the end of 2024, and Computer Use is still rapidly iterating.

This book clearly marks highly volatile content:

> **Timeliness Note** (2026-01): This section's content is based on the MCP specification as of 2026-01.
> Please check the latest documentation to confirm any updates.

Core architectural patterns are relatively stable, but specific APIs and tools may change. When in doubt, defer to official documentation.

---

## Acknowledgments

Thanks to all colleagues and friends who helped during the Agent system building process.

Thanks to the open-source community -- LangChain, LangGraph, CrewAI, AutoGen, and other projects let us stand on the shoulders of giants.

Thanks to Claude Code, OpenAI Codex programming tools and models for pushing the boundaries of rapid code writing, turning this Agent orchestration framework and platform from concept into reality quickly.

---

**[Wayland Zhang](https://waylandz.com)**

January 2026

*"The best way to learn is to build."*

---

## Errata and Feedback

If you find errors in the book, or have any suggestions, please contact us through:

- GitHub Issues: [This book's repository](https://github.com/Kocoro-lab/ai-agent-book)
- Shannon OSS: [github.com/Kocoro-lab/Shannon](https://github.com/Kocoro-lab/Shannon)

Technical books inevitably have omissions -- thank you for your understanding and help.
