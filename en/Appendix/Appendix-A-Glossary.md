# Appendix A: Glossary

> **This appendix helps you quickly look up key terms, understand their definitions and technical concepts. Organized alphabetically with definitions, related chapters, and practical examples.**

---

## A - Agent Core Concepts

### Agent

**Definition**: A software entity that can perceive its environment, make autonomous decisions, and take actions to achieve goals. In the LLM era, Agents use large language models for reasoning and tools for environment interaction.

**Related Chapters**: Chapter 1, Chapter 2, Chapter 14

**Example**: A customer service Agent receives user questions, queries a knowledge base, calls an order system API, and generates a response

**Related Terms**: [ReAct](#react-reason-act-loop), [Tool Use](#tool-use), [Multi-Agent](#multi-agent-system)

### Agentic Coding

**Definition**: A programming paradigm where AI Agents autonomously complete code generation, debugging, testing, and deployment. The Agent understands requirements, plans implementation, writes code, and iteratively optimizes.

**Related Chapters**: Chapter 28

**Example**: A developer describes "add user authentication," and the Agent automatically generates code, writes tests, and submits a PR

**Related Terms**: [Computer Use](#computer-use), [Reflection](#reflection)

### API Gateway

**Definition**: A unified entry point that handles routing, authentication, rate limiting, and protocol conversion for all external requests. In Agent systems, it exposes Agent capabilities as standard APIs.

**Related Chapters**: Chapter 21, Chapter 22

**Example**: Kong gateway handles all HTTP requests, routes `/v1/chat` to the Orchestrator, and applies Token quotas

**Related Terms**: [Orchestrator](#orchestrator), [Guardrails](#guardrails)

### Asynchronous Workflow

**Definition**: A workflow pattern where task execution doesn't block the caller, with results obtained via callbacks, message queues, or polling. Suitable for long-running Agent tasks.

**Related Chapters**: Chapter 21, Chapter 22

**Example**: User submits a research task, receives a task_id, Agent executes in background, client polls for progress

**Related Terms**: [Temporal](#temporal-workflow-engine), [Background Agent](#background-agent)

---

## B - Backpressure and Budget

### Background Agent

**Definition**: An Agent that runs continuously in the background, executing scheduled tasks, monitoring events, or processing async requests. Doesn't require real-time user interaction.

**Related Chapters**: Chapter 29

**Example**: Automatically scrapes competitor information hourly, analyzes trends, generates weekly reports, and sends emails

**Related Terms**: [Asynchronous Workflow](#asynchronous-workflow), [Temporal](#temporal-workflow-engine)

### Backpressure

**Definition**: A flow control mechanism where downstream systems signal upstream to "slow down." Prevents producer speed from exceeding consumer processing capacity, avoiding system crashes.

**Related Chapters**: Chapter 22, Chapter 23

**Example**: When LLM Service is overwhelmed, it returns 429 errors; Orchestrator pauses sending new requests

**Related Terms**: [Circuit Breaker](#circuit-breaker), [Rate Limiting](#rate-limiting)

### Batch Processing

**Definition**: Aggregating multiple requests for unified processing, improving throughput and resource utilization. Commonly used for embedding generation, batch inference, etc.

**Related Chapters**: Chapter 23

**Example**: Accumulate 10 documents before calling Embedding API once, rather than calling for each document

**Related Terms**: [Token Budget](#token-budget), [Cost Optimization](#cost-optimization)

---

## C - Chain-of-Thought and Cost

### Caching

**Definition**: Storing computation results to avoid redundant calculations, speeding up responses and reducing costs. In Agent systems, can cache LLM responses, tool call results, or vector retrievals.

**Related Chapters**: Chapter 11, Chapter 23

**Example**: Cache the answer to "what's the company address?" for 24 hours, avoiding redundant LLM calls

**Related Terms**: [Memory](#memory), [Session](#session)

### Chain-of-Thought (CoT)

**Definition**: A prompting technique that has LLMs generate intermediate reasoning steps, solving complex problems step by step. Improves reasoning accuracy and explainability.

**Related Chapters**: Chapter 12, Chapter 13, Chapter 17

**Example**: Computing "23 x 47," LLM outputs "20x47=940, 3x47=141, 940+141=1081"

**Related Terms**: [Tree-of-Thoughts](#tree-of-thoughts-tot), [Reflection](#reflection)

### Circuit Breaker

**Definition**: When downstream service failure rate exceeds a threshold, automatically cuts off requests to prevent cascading failures. Attempts recovery after a period.

**Related Chapters**: Chapter 22, Chapter 23

**Example**: After 10 consecutive vector database timeouts, circuit breaker opens, all queries fail fast, half-open recovery attempted after 30 seconds

**Related Terms**: [Backpressure](#backpressure), [Guardrails](#guardrails)

### Computer Use

**Definition**: An Agent's ability to interact with computer environments by operating browsers, desktop applications, or command-line tools. Enables RPA automation and complex task execution.

**Related Chapters**: Chapter 27

**Example**: Agent opens browser, logs into system, fills forms, uploads files, takes screenshots to confirm results

**Related Terms**: [Tool Use](#tool-use), [Agentic Coding](#agentic-coding)

### Context Window

**Definition**: The maximum number of tokens an LLM can process at once. Includes input prompts, conversation history, and output content.

**Related Chapters**: Chapter 7, Chapter 9

**Example**: A 128K-context model (e.g., GPT-4o) can fit on the order of ~100K English words (very rough)

**Related Terms**: [Token](#token), [Summarization](#summarization)

### Cost Optimization

**Definition**: Strategies to reduce LLM usage costs through model selection, caching, batch processing, prompt compression, etc.

**Related Chapters**: Chapter 23

**Example**: Use a low-cost model (e.g., Claude Haiku) for simple Q&A, and a top-tier model (e.g., Claude Opus) for complex reasoning

**Related Terms**: [Token Budget](#token-budget), [Caching](#caching)

---

## D - DAG and Debate

### DAG (Directed Acyclic Graph)

**Definition**: A graph structure where nodes represent tasks and edges represent dependencies. Used to define execution order of multi-Agent workflows.

**Related Chapters**: Chapter 14, Chapter 21

**Example**: Research task: search (Node A) → analyze (Nodes B, C in parallel) → synthesize (Node D depends on B, C)

**Related Terms**: [Orchestrator](#orchestrator), [Temporal](#temporal-workflow-engine)

### Debate Pattern

**Definition**: A reasoning pattern where multiple Agents with different viewpoints engage in multi-round debate, improving decision quality through adversarial discussion.

**Related Chapters**: Chapter 18

**Example**: Legal Agents playing plaintiff, defendant, and judge roles debate contract clause reasonability, reaching consensus

**Related Terms**: [Multi-Agent](#multi-agent-system), [Reflection](#reflection)

### Deterministic Replay

**Definition**: A mechanism where workflow engines can re-execute from checkpoints after failure, guaranteeing identical input produces identical results. Core feature of Temporal.

**Related Chapters**: Chapter 21

**Example**: Agent task crashes at step 5, Temporal replays from checkpoint, skipping completed steps 1-4

**Related Terms**: [Temporal](#temporal-workflow-engine), [Asynchronous Workflow](#asynchronous-workflow)

---

## E - Embedding and Error Handling

### Embedding

**Definition**: Converting text into high-dimensional vector numerical representations that capture semantic information. Used for similarity search, clustering, and RAG.

**Related Chapters**: Chapter 11, Chapter 19

**Example**: Text "machine learning" converted to 1536-dimensional vector [0.23, -0.45, ...]

**Related Terms**: [RAG](#rag-retrieval-augmented-generation), [Vector Database](#vector-database)

### Error Handling

**Definition**: Mechanisms for identifying, catching, and recovering from system exceptions. In Agent systems, includes retries, degradation, circuit breaking, and user-friendly error messages.

**Related Chapters**: Chapter 22, Chapter 23

**Example**: After LLM call timeout, auto-retry 3 times; if still failing, degrade to "Service busy, please try later"

**Related Terms**: [Circuit Breaker](#circuit-breaker), [Guardrails](#guardrails)

---

## F - Function Calling and Failure Recovery

### Fallback Strategy

**Definition**: A fault tolerance mechanism that automatically switches to alternatives when the primary service is unavailable. Ensures basic functionality during partial failures.

**Related Chapters**: Chapter 22, Chapter 23

**Example**: When vector search fails, degrade to keyword search; when LLM unavailable, return predefined templates

**Related Terms**: [Circuit Breaker](#circuit-breaker), [Error Handling](#error-handling)

### Function Calling

**Definition**: LLM generates structured function call requests based on conversation, which the system executes and returns results. The standardized implementation of Tool Use.

**Related Chapters**: Chapter 3, Chapter 4

**Example**: User asks "Beijing weather," LLM returns `get_weather(city="Beijing")`, system calls weather API

**Related Terms**: [Tool Use](#tool-use), [ReAct](#react-reason-act-loop)

---

## G - Guardrails and Governance

### Guardrails

**Definition**: Security mechanisms that constrain and validate Agent behavior. Includes input validation, output filtering, permission checking, and policy enforcement.

**Related Chapters**: Chapter 24, Chapter 25

**Example**: Intercept outputs containing sensitive words, block Agent access to unauthorized databases, limit Token usage

**Related Terms**: [OPA](#opa-open-policy-agent), [WASI](#wasi-wasm-sandbox)

---

## H - Handoff and Hooks

### Handoff

**Definition**: A collaboration pattern where one Agent transfers a task to another more specialized Agent. Common in multi-Agent system responsibility division.

**Related Chapters**: Chapter 15, Chapter 16

**Example**: General Agent identifies user needs a refund, hands off conversation to Refund Specialist Agent

**Related Terms**: [Multi-Agent](#multi-agent-system), [Orchestrator](#orchestrator)

### Hooks

**Definition**: Event callback mechanisms triggered at specific points in Agent execution flow. Used for logging, monitoring, auditing, or custom logic injection.

**Related Chapters**: Chapter 6, Chapter 29

**Example**: `before_tool_call` hook logs tool name and parameters, `after_llm_response` hook filters sensitive information

**Related Terms**: [Plugins](#plugins), [Observability](#observability)

---

## I - Integration and Idempotency

### Idempotency

**Definition**: The property where the same operation produces the same result when executed multiple times. Used for safe retries and deduplication in distributed systems.

**Related Chapters**: Chapter 21, Chapter 22

**Example**: Using request_id ensures payment request retries don't result in duplicate charges

**Related Terms**: [Deterministic Replay](#deterministic-replay), [Temporal](#temporal-workflow-engine)

---

## L - LLM and Logging

### LLM Service

**Definition**: A service layer that encapsulates multiple LLM provider calls, tool selection, and execution. Provides unified API, handles authentication, retries, caching, and other common logic.

**Related Chapters**: Chapter 3, Chapter 21

**Example**: Shannon's Python service layer supporting OpenAI, Anthropic, Azure with unified response format

**Related Terms**: [Agent](#agent), [Function Calling](#function-calling)

### Logging

**Definition**: Mechanisms for recording system runtime events, errors, and state changes. In Agent systems, used for debugging, auditing, and performance analysis.

**Related Chapters**: Chapter 22

**Example**: Structured logging records each LLM call's model, tokens, latency, and cost

**Related Terms**: [Observability](#observability), [Tracing](#tracing-distributed-tracing)

---

## M - MCP and Memory

### MCP (Model Context Protocol)

**Definition**: An open protocol that standardizes LLM connections to external data sources and tools. Defines discovery and invocation specifications for resources, tools, and prompts.

**Related Chapters**: Chapter 4, Chapter 5

**Example**: Expose Google Drive files via MCP Server, allowing Agent to list, read, and search documents

**Related Terms**: [Tool Use](#tool-use), [Skills](#skills)

> **Timeliness Note** (2026-01): MCP specification is still rapidly evolving.
> Check the [latest documentation](https://spec.modelcontextprotocol.io/) for transport layer and capability updates.

### Memory

**Definition**: Mechanisms for Agents to store and retrieve historical information. Includes working memory (current session), short-term memory (persistent storage), and semantic memory (knowledge graphs).

**Related Chapters**: Chapter 7, Chapter 8, Chapter 9

**Example**: User says "the project I mentioned last time," Agent retrieves context from Session to understand the reference

**Related Terms**: [Session](#session), [RAG](#rag-retrieval-augmented-generation)

### Metrics

**Definition**: Mechanisms for collecting, aggregating, and visualizing system performance data. In Agent systems, monitors key metrics like Token usage, latency, and success rates.

**Related Chapters**: Chapter 22

**Example**: Prometheus collects average response time, Token consumption, and tool call counts per Agent

**Related Terms**: [Observability](#observability), [Logging](#logging)

### Multi-Agent System

**Definition**: A system architecture where multiple Agents collaborate to complete complex tasks. Agents can interact in parallel, sequential, or dynamic patterns.

**Related Chapters**: Chapter 13, Chapter 14, Chapter 16

**Example**: In e-commerce, Search Agent, Recommendation Agent, and Customer Service Agent collaboratively handle user shopping flow

**Related Terms**: [Orchestrator](#orchestrator), [DAG](#dag-directed-acyclic-graph)

---

## O - Orchestration and Observability

### Observability

**Definition**: The ability to understand system internal state through logs, metrics, and traces. In Agent systems, used for debugging complex reasoning chains and tool calls.

**Related Chapters**: Chapter 22

**Example**: Trace ID tracks a request's complete path through Gateway(20ms) → Orchestrator(50ms) → Agent(300ms) → LLM(2s)

**Related Terms**: [Logging](#logging), [Tracing](#tracing-distributed-tracing), [Metrics](#metrics)

### OPA (Open Policy Agent)

**Definition**: A general-purpose policy engine that uses Rego language to define and execute authorization, validation, and compliance rules.

**Related Chapters**: Chapter 24

**Example**: Define policy "financial data can only be accessed by finance department," intercept unauthorized Agent queries

**Related Terms**: [Guardrails](#guardrails), [WASI](#wasi-wasm-sandbox)

### Orchestrator

**Definition**: A control layer that coordinates multiple Agents executing complex workflows. Responsible for routing, scheduling, result aggregation, and error handling.

**Related Chapters**: Chapter 13, Chapter 21

**Example**: Shannon's Go Orchestrator executes 3 Agents in parallel based on DAG definition, aggregates results

**Related Terms**: [DAG](#dag-directed-acyclic-graph), [Multi-Agent](#multi-agent-system)

---

## P - Planning and Plugins

### P2P (Peer-to-Peer)

**Definition**: A collaboration pattern where Agents communicate directly without a centralized orchestrator. Suitable for dynamic, self-organizing multi-Agent scenarios.

**Related Chapters**: Chapter 16

**Example**: Research Agent autonomously discovers and requests Analysis Agent to provide data, without Orchestrator involvement

**Related Terms**: [Multi-Agent](#multi-agent-system), [Handoff](#handoff)

### Planning

**Definition**: A reasoning pattern where Agents formulate step-by-step plans before execution. Decomposes complex goals into executable subtask sequences.

**Related Chapters**: Chapter 10

**Example**: User requests "organize team building," Agent generates plan: 1. Determine date 2. Book venue 3. Send invitations

**Related Terms**: [ReAct](#react-reason-act-loop), [Tree-of-Thoughts](#tree-of-thoughts-tot)

### Plugins

**Definition**: Pluggable modules encapsulating specific capabilities, extending Agent functionality through standard interfaces. Supports hot loading and version management.

**Related Chapters**: Chapter 29

**Example**: After installing Slack plugin, Agent can send messages, create channels, get history

**Related Terms**: [Skills](#skills), [Hooks](#hooks), [MCP](#mcp-model-context-protocol)

### Prompt

**Definition**: Input text sent to LLMs, including instructions, examples, context, and questions. The core interface for controlling Agent behavior.

**Related Chapters**: Chapter 2, Chapter 12

**Example**: `"You are a professional customer service Agent. User question: {question}. Please refer to knowledge base: {context}"`

**Related Terms**: [Chain-of-Thought](#chain-of-thought-cot), [ReAct](#react-reason-act-loop)

---

## R - RAG and Reflection

### RAG (Retrieval-Augmented Generation)

**Definition**: A technique that retrieves relevant documents before generating responses, providing retrieval results as context to LLMs.

**Related Chapters**: Chapter 8, Chapter 19

**Example**: User asks "return policy," vector search finds relevant documents, LLM generates answer based on documents

**Related Terms**: [Embedding](#embedding), [Vector Database](#vector-database)

### Rate Limiting

**Definition**: A flow control mechanism limiting request count per time unit. Prevents abuse, protects downstream services, and controls costs.

**Related Chapters**: Chapter 23, Chapter 24

**Example**: Each user limited to 10 LLM calls per minute, exceeding returns 429 Too Many Requests

**Related Terms**: [Token Budget](#token-budget), [Backpressure](#backpressure)

### ReAct (Reason-Act Loop)

**Definition**: A decision pattern where Agents cyclically execute "reason → act → observe." LLM generates reasoning process and next action, observes results after execution.

**Related Chapters**: Chapter 2, Chapter 3

**Example**: Think: need to check weather → Act: call weather_api → Observe: Beijing 15C → Think: answer user

**Related Terms**: [Agent](#agent), [Tool Use](#tool-use)

### Reflection

**Definition**: A self-optimization pattern where Agents evaluate their own output quality, identify errors, and improve. Increases result accuracy through multiple iterations.

**Related Chapters**: Chapter 11

**Example**: Agent generates code, then self-checks "any syntax errors? What's test coverage?", fixes issues

**Related Terms**: [Chain-of-Thought](#chain-of-thought-cot), [Debate](#debate-pattern)

---

## S - Session and Skills

### Sandbox

**Definition**: An isolated execution environment that limits code access to system resources. In Agent systems, used to safely run untrusted tool code.

**Related Chapters**: Chapter 25

**Example**: Run user-defined Python tool in WASI sandbox, unable to access filesystem or network

**Related Terms**: [WASI](#wasi-wasm-sandbox), [Guardrails](#guardrails)

### Session

**Definition**: A complete interaction process between user and Agent, containing multiple conversation turns and related context. Sessions can be persisted for cross-session dialogue.

**Related Chapters**: Chapter 7, Chapter 9

**Example**: User's conversation history, preferences, and unfinished tasks after login are all bound to the same session_id

**Related Terms**: [Memory](#memory), [Context Window](#context-window)

### Skills

**Definition**: Reusable Agent capability modules encapsulating domain-specific prompts, tools, and workflows. Can be shared and composed across Agents.

**Related Chapters**: Chapter 5

**Example**: `code_review` skill includes code analysis prompts, static checking tools, and formatting tools

**Related Terms**: [Plugins](#plugins), [MCP](#mcp-model-context-protocol)

### Streaming

**Definition**: A pattern where LLM generates content incrementally and clients receive partial results in real-time. Improves user experience by reducing time-to-first-token.

**Related Chapters**: Chapter 3, Chapter 22

**Example**: ChatGPT-style character-by-character output, users see progress without waiting for complete response

**Related Terms**: [Asynchronous Workflow](#asynchronous-workflow)

### Summarization

**Definition**: A technique for compressing long text into brief summaries. In Agent systems, used to handle context window limits and reduce Token costs.

**Related Chapters**: Chapter 7

**Example**: Compress 1000 turns of conversation history into summary "User inquired about refund process, resolved"

**Related Terms**: [Context Window](#context-window), [Memory](#memory)

### Supervisor Pattern

**Definition**: A multi-Agent architecture pattern where one Agent acts as coordinator, assigning tasks to specialist Agents and aggregating results.

**Related Chapters**: Chapter 15

**Example**: Supervisor Agent breaks down "write article" into research, writing, and proofreading tasks, assigns to three specialists

**Related Terms**: [Orchestrator](#orchestrator), [Handoff](#handoff)

---

## T - Token and Tools

### Temporal (Workflow Engine)

**Definition**: A distributed, durable workflow engine supporting long-running tasks, deterministic replay, and automatic retries. Suitable for complex Agent orchestration.

**Related Chapters**: Chapter 21

**Example**: Define "daily report" workflow that can resume from breakpoint even after service restart

**Related Terms**: [DAG](#dag-directed-acyclic-graph), [Orchestrator](#orchestrator)

### Token

**Definition**: The unit of text that LLMs process and bill against. Tokenization depends on the tokenizer and language (a common rough rule is ~4 English characters per token).

**Related Chapters**: Chapter 7, Chapter 23

**Example**: Text "Hello World" equals approximately 2 Tokens

**Related Terms**: [Context Window](#context-window), [Token Budget](#token-budget)

### Token Budget

**Definition**: Mechanisms for managing and allocating Token usage limits per request, session, or user. Controls costs and prevents abuse.

**Related Chapters**: Chapter 23, Chapter 24

**Example (hypothetical)**: Free users get 10K Tokens/day, paid users get 1M Tokens/day; exceeding triggers degradation or denial

**Related Terms**: [Rate Limiting](#rate-limiting), [Cost Optimization](#cost-optimization)

### Tool Use

**Definition**: Mechanisms for Agents to extend capabilities by calling external tools (APIs, databases, calculators, etc.). The core way Agents interact with their environment.

**Related Chapters**: Chapter 3, Chapter 4

**Example**: Call `search_api("Claude 3.5")` for latest info, call `calculator(23*47)` for computation

**Related Terms**: [Function Calling](#function-calling), [MCP](#mcp-model-context-protocol)

### Tracing (Distributed Tracing)

**Definition**: Tracking a request's complete path through distributed systems, recording each service's latency and status. In Agent systems, used for debugging complex call chains.

**Related Chapters**: Chapter 22

**Example**: Jaeger shows request path: Gateway(20ms) → Orchestrator(50ms) → Agent(300ms) → LLM(2s)

**Related Terms**: [Observability](#observability), [Logging](#logging)

### Tree-of-Thoughts (ToT)

**Definition**: An advanced reasoning pattern that models the problem solution space as a tree, explores multiple reasoning paths, and finds optimal solutions through backtracking and pruning.

**Related Chapters**: Chapter 17

**Example**: When solving a math problem, try 3 methods, expand 2 steps for each, evaluate and select the best path to continue

**Related Terms**: [Chain-of-Thought](#chain-of-thought-cot), [Planning](#planning)

---

## V - Vector Database

### Vector Database

**Definition**: A database specialized for storing and retrieving high-dimensional vectors. Supports similarity search, commonly used for RAG and semantic retrieval.

**Related Chapters**: Chapter 8, Chapter 19

**Example**: Pinecone, Qdrant store document embeddings, return most similar Top-K results in milliseconds

**Related Terms**: [Embedding](#embedding), [RAG](#rag-retrieval-augmented-generation)

---

## W - WASI and Workflow

### WASI (WASM Sandbox)

**Definition**: WebAssembly System Interface, a standard interface allowing WASM modules to safely access system resources like files and network. Used for isolated execution of untrusted code.

**Related Chapters**: Chapter 25

**Example**: Shannon uses WASI to run user-defined tools, limited to only accessing specified directories and APIs

**Related Terms**: [Sandbox](#sandbox), [Guardrails](#guardrails)

### Workflow

**Definition**: Process orchestration that defines task execution order, dependencies, and error handling. In Agent systems, used to organize multi-step tasks.

**Related Chapters**: Chapter 14, Chapter 21

**Example**: E-commerce order workflow: verify inventory → charge → ship → send notification

**Related Terms**: [DAG](#dag-directed-acyclic-graph), [Temporal](#temporal-workflow-engine)

---

## How to Use This Glossary

### Usage Guide

1. **Quick lookup**: Organized alphabetically for easy term location
2. **Understand concepts**: Each term includes concise definition and practical example
3. **Deep learning**: Jump to detailed content via "Related Chapters" links
4. **Extended reading**: Understand relationships between terms via "Related Terms" links

### Update Notes

The AI Agent field is evolving rapidly; some term definitions and best practices may change over time. This glossary is written based on early 2026 industry consensus. We recommend using it alongside "Timeliness Notes" in chapters and external references.

### Feedback and Contributions

If you find inaccurate definitions or need to add new terms, please provide feedback through GitHub Issues.

---

## Index Reference

**By Topic**:
- **Agent Basics**: Agent, ReAct, Tool Use, Function Calling, Prompt
- **Reasoning Patterns**: CoT, ToT, Planning, Reflection, Debate
- **Multi-Agent**: Multi-Agent, Orchestrator, DAG, Supervisor, Handoff, P2P
- **Extension Mechanisms**: MCP, Skills, Hooks, Plugins
- **Context Memory**: Context Window, Memory, Session, Summarization, RAG
- **Production Architecture**: Temporal, Observability, Logging, Tracing, Metrics
- **Security Compliance**: OPA, WASI, Guardrails, Sandbox
- **Cost Optimization**: Token Budget, Rate Limiting, Caching, Batch Processing
- **Fault Tolerance**: Circuit Breaker, Backpressure, Fallback Strategy, Error Handling
- **Frontier Practices**: Computer Use, Agentic Coding, Background Agent

**By Technology Stack**:
- **LLM Related**: Token, Context Window, Prompt, Streaming, Function Calling
- **Data Storage**: Vector Database, Embedding, Caching
- **Workflow Engines**: Temporal, DAG, Workflow, Deterministic Replay
- **Observability**: Logging, Tracing, Metrics, Observability
- **Security Isolation**: WASI, Sandbox, OPA, Guardrails

---

**This glossary covers core concepts from all 30 chapters, with 60+ key terms. We recommend using it alongside chapter content and code examples for deeper understanding.**
