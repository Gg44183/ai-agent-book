# AI Agent Architecture: From Monolith to Enterprise Multi-Agent
## Complete Table of Contents

---

> **2025-2026 Hot Topics Quick Navigation**
>
> - **Human-in-the-Loop**: Chapter 15 (15.12) -- Human-machine collaboration, escalation triggers, interrupt takeover, trust escalation
> - **MCP Protocol**: Chapter 4 -- Model Context Protocol tool standardization
> - **Computer Use**: Chapter 27 -- Browser/desktop automation
> - **Agentic Coding**: Chapter 28 -- Claude Code / Devin mode

---

### Preface
- [Preface - Why I Wrote This Book](./Preface.md)

---

## Part 1: Agent Fundamentals
> *Understanding the essence and core operating mechanisms of Agents*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 1 | [The Essence of Agents](./Part1-Agent-Fundamentals/Chapter-01-The-Essence-of-Agents.md) | Agent definition, differences from traditional software, autonomy boundaries |
| Chapter 2 | [The ReAct Loop](./Part1-Agent-Fundamentals/Chapter-02-The-ReAct-Loop.md) | Reason-Act loop, observe-think-act, loop termination conditions |

---

## Part 2: Tools and Extensions
> *Enabling Agents to interact with the external world*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 3 | [Tool Calling Fundamentals](./Part2-Tools-and-Extensions/Chapter-03-Tool-Calling-Fundamentals.md) | Function Calling, tool definition, parameter validation |
| Chapter 4 | [MCP Protocol Deep Dive](./Part2-Tools-and-Extensions/Chapter-04-MCP-Protocol-Deep-Dive.md) | Model Context Protocol, transport layer, resources and tools |
| Chapter 5 | [Skills System](./Part2-Tools-and-Extensions/Chapter-05-Skills-System.md) | Reusable capability encapsulation, skill composition, dynamic loading |
| Chapter 6 | [Hooks and Event Systems](./Part2-Tools-and-Extensions/Chapter-06-Hooks-and-Events.md) | Lifecycle hooks, event triggers, extension point design |

---

## Part 3: Context and Memory
> *Managing Agent short-term memory and long-term knowledge*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 7 | [Context Window Management](./Part3-Context-and-Memory/Chapter-07-Context-Window-Management.md) | Token limits, context compression, sliding window strategies |
| Chapter 8 | [Memory Architecture](./Part3-Context-and-Memory/Chapter-08-Memory-Architecture.md) | Short-term/long-term memory, vector storage, memory retrieval |
| Chapter 9 | [Multi-Turn Conversation Design](./Part3-Context-and-Memory/Chapter-09-Multi-Turn-Conversation-Design.md) | Session state, context passing, conversation management |

---

## Part 4: Single Agent Patterns
> *Advanced thinking and self-improvement capabilities for individual Agents*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 10 | [Planning Pattern](./Part4-Single-Agent-Patterns/Chapter-10-Planning-Pattern.md) | Task decomposition, plan generation, dynamic adjustment |
| Chapter 11 | [Reflection Pattern](./Part4-Single-Agent-Patterns/Chapter-11-Reflection-Pattern.md) | Self-evaluation, error correction, iterative improvement |
| Chapter 12 | [Chain-of-Thought](./Part4-Single-Agent-Patterns/Chapter-12-Chain-of-Thought.md) | Chain-of-thought reasoning, step-by-step analysis, reasoning explainability |

---

## Part 5: Multi-Agent Orchestration
> *Multiple Agents collaborating to complete complex tasks*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 13 | [Orchestration Fundamentals](./Part5-Multi-Agent-Orchestration/Chapter-13-Orchestration-Fundamentals.md) | Orchestration vs collaboration, communication patterns, task allocation |
| Chapter 14 | [DAG Workflows](./Part5-Multi-Agent-Orchestration/Chapter-14-DAG-Workflows.md) | Directed acyclic graphs, parallel execution, dependency management |
| Chapter 15 | [Supervisor Pattern](./Part5-Multi-Agent-Orchestration/Chapter-15-Supervisor-Pattern.md) | Supervisor architecture, task routing, **Human-in-the-Loop**, escalation triggers, interrupt takeover |
| Chapter 16 | [Handoff Mechanism](./Part5-Multi-Agent-Orchestration/Chapter-16-Handoff-Mechanism.md) | Inter-Agent task handoff, state transfer, context preservation |

---

## Part 6: Advanced Reasoning
> *Multi-path exploration and synthesis for complex problems*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 17 | [Tree-of-Thoughts](./Part6-Advanced-Reasoning/Chapter-17-Tree-of-Thoughts.md) | Thought tree search, branch exploration, path evaluation |
| Chapter 18 | [Debate Pattern](./Part6-Advanced-Reasoning/Chapter-18-Debate-Pattern.md) | Multi-Agent debate, viewpoint opposition, consensus building |
| Chapter 19 | [Research-Synthesis](./Part6-Advanced-Reasoning/Chapter-19-Research-Synthesis.md) | Multi-source research, information synthesis, report generation |

---

## Part 7: Production Architecture
> *Architecture evolution from demo to production environment*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 20 | [Three-Layer Architecture](./Part7-Production-Architecture/Chapter-20-Three-Layer-Architecture.md) | Orchestrator/Agent/LLM layering, responsibility separation |
| Chapter 21 | [Temporal Workflows](./Part7-Production-Architecture/Chapter-21-Temporal-Workflows.md) | Persistent execution, failure recovery, long-running tasks |
| Chapter 22 | [Observability](./Part7-Production-Architecture/Chapter-22-Observability.md) | Distributed tracing, metrics monitoring, log aggregation |

---

## Part 8: Enterprise Features
> *Governance and security for large-scale deployment*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 23 | [Token Budget Control](./Part8-Enterprise-Features/Chapter-23-Token-Budget-Control.md) | Cost management, quota allocation, usage monitoring |
| Chapter 24 | [Policy Governance](./Part8-Enterprise-Features/Chapter-24-Policy-Governance.md) | OPA policy engine, access control, audit logging |
| Chapter 25 | [Secure Execution](./Part8-Enterprise-Features/Chapter-25-Secure-Execution.md) | WASI sandbox, code isolation, resource limits |
| Chapter 26 | [Multi-Tenant Design](./Part8-Enterprise-Features/Chapter-26-Multi-Tenant-Design.md) | Tenant isolation, resource quotas, data separation |

---

## Part 9: Frontier Practices
> *Latest Agent application directions for 2025-2026*

| Chapter | Title | Core Content |
|---------|-------|--------------|
| Chapter 27 | [Computer Use](./Part9-Frontier-Practices/Chapter-27-Computer-Use.md) | Browser automation, desktop operations, GUI interaction |
| Chapter 28 | [Agentic Coding](./Part9-Frontier-Practices/Chapter-28-Agentic-Coding.md) | Code generation, auto-repair, development assistance |
| Chapter 29 | [Background Agents](./Part9-Frontier-Practices/Chapter-29-Background-Agents.md) | Background execution, async tasks, long-running processes |
| Chapter 30 | [Tiered Model Strategy](./Part9-Frontier-Practices/Chapter-30-Tiered-Model-Strategy.md) | Model routing, cost optimization, capability matching |

---

## Appendix

| Appendix | Title | Core Content |
|----------|-------|--------------|
| Appendix A | [Glossary](./Appendix/Appendix-A-Glossary.md) | Core concept definitions, bilingual terms |
| Appendix B | [Pattern Selection Guide](./Appendix/Appendix-B-Pattern-Selection-Guide.md) | Decision trees, scenario matching, configuration examples |
| Appendix C | [FAQ](./Appendix/Appendix-C-FAQ.md) | 27 frequently asked questions answered |

---

## Statistics

| Metric | Value |
|--------|-------|
| Total Chapters | 30 chapters |
| Appendices | 3 |
| Parts | 9 topics |
| Reference Implementation | [Shannon OSS](https://github.com/Kocoro-lab/Shannon) |

---

## Learning Path Recommendations

### Quick Start (2-3 days)
```
All of Part 1 -> Chapter 3 -> Chapter 13 -> Chapter 20
```
Goal: Agent basics -> Tool calling -> Multi-Agent orchestration -> Production architecture

### Deep Understanding (1-2 weeks)
```
Read Part 1-5 sequentially
```
Goal: Master complete knowledge from single Agent to multi-Agent orchestration

### Complete Learning (3-4 weeks)
```
Read entire book sequentially, practice with Shannon code
```
Goal: Gain ability to design and implement production-grade Agent systems

### Production Deployment (as needed)
```
Part 7 + Part 8
```
Goal: Architecture design, governance strategies, secure execution

### Hot Topics Track (1-2 days)
```
Chapter 4 (MCP) -> Chapter 15 Section 15.12 (HITL) -> Chapter 27 (Computer Use) -> Chapter 28 (Agentic Coding)
```
Goal: Learn about the hottest Agent topics of 2025-2026 (MCP, Human-in-the-Loop, Computer Use, Agentic Coding)
