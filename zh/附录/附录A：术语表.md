# 附录 A：术语表 (Terminology Glossary)

> **本附录帮助你快速查找关键术语，理解中英对照和技术概念。按字母顺序组织，包含定义、相关章节和实际示例。**

---

## A - Agent 核心概念

### Agent（智能体）
**定义**：能够感知环境、自主决策并采取行动以实现目标的软件实体。在 LLM 时代，Agent 通过大语言模型进行推理，使用工具与环境交互。
**英文**：Agent
**相关章节**：第 1 章、第 2 章、第 14 章
**示例**：客服 Agent 接收用户问题，查询知识库，调用订单系统 API，最终生成回复
**相关术语**：[ReAct](#react推理-行动循环)、[Tool Use](#tool-use工具调用)、[Multi-Agent](#multi-agent多智能体系统)

### Agentic Coding（Agent 编程）
**定义**：由 AI Agent 自主完成代码生成、调试、测试和部署的编程范式。Agent 理解需求、规划实现、编写代码并迭代优化。
**英文**：Agentic Coding
**相关章节**：第 28 章
**示例**：开发者描述需求"添加用户认证"，Agent 自动生成代码、编写测试、提交 PR
**相关术语**：[Computer Use](#computer-use计算机使用)、[Reflection](#reflection反思)

### API Gateway（API 网关）
**定义**：统一的入口点，处理所有外部请求的路由、认证、限流和协议转换。在 Agent 系统中负责暴露 Agent 能力为标准 API。
**英文**：API Gateway
**相关章节**：第 21 章、第 22 章
**示例**：Kong 网关处理所有 HTTP 请求，将 `/v1/chat` 路由到 Orchestrator，应用 Token 限额
**相关术语**：[Orchestrator](#orchestrator编排器)、[Guardrails](#guardrails护栏)

### Asynchronous Workflow（异步工作流）
**定义**：任务执行不阻塞调用方，通过回调、消息队列或轮询获取结果的工作流模式。适合长时间运行的 Agent 任务。
**英文**：Asynchronous Workflow
**相关章节**：第 21 章、第 22 章
**示例**：用户提交研究任务，返回 task_id，Agent 在后台执行，客户端定期查询进度
**相关术语**：[Temporal](#temporal工作流引擎)、[Background Agent](#background-agent后台-agent)

---

## B - 背压与预算

### Background Agent（后台 Agent）
**定义**：在后台持续运行的 Agent，执行定时任务、监控事件或处理异步请求。不需要实时用户交互。
**英文**：Background Agent
**相关章节**：第 29 章
**示例**：每小时自动抓取竞品信息，分析趋势，生成周报并发送邮件
**相关术语**：[Asynchronous Workflow](#asynchronous-workflow异步工作流)、[Temporal](#temporal工作流引擎)

### Backpressure（背压）
**定义**：下游系统向上游发送"慢下来"信号的流控机制。防止生产者速度超过消费者处理能力导致系统崩溃。
**英文**：Backpressure
**相关章节**：第 22 章、第 23 章
**示例**：LLM Service 处理不过来时，返回 429 错误，Orchestrator 暂停发送新请求
**相关术语**：[Circuit Breaker](#circuit-breaker熔断器)、[Rate Limiting](#rate-limiting速率限制)

### Batch Processing（批处理）
**定义**：将多个请求聚合后统一处理，提高吞吐量和资源利用率。常用于 Embedding 生成、批量推理等场景。
**英文**：Batch Processing
**相关章节**：第 23 章
**示例**：积累 10 个文档后一次性调用 Embedding API，而非每个文档单独调用
**相关术语**：[Token Budget](#token-budget-token-预算)、[Cost Optimization](#cost-optimization成本优化)

---

## C - 思维链与成本

### Caching（缓存）
**定义**：存储计算结果以避免重复计算，加速响应并降低成本。在 Agent 系统中可缓存 LLM 响应、工具调用结果或向量检索。
**英文**：Caching
**相关章节**：第 11 章、第 23 章
**示例**：相同问题"公司地址是什么"缓存答案 24 小时，避免重复调用 LLM
**相关术语**：[Memory](#memory记忆系统)、[Session](#session会话)

### Chain-of-Thought（思维链，CoT）
**定义**：让 LLM 生成中间推理步骤，逐步解决复杂问题的 Prompt 技术。提高推理准确性和可解释性。
**英文**：Chain-of-Thought (CoT)
**相关章节**：第 12 章、第 13 章、第 17 章
**示例**：计算"23 × 47"时，LLM 输出 "20×47=940, 3×47=141, 940+141=1081"
**相关术语**：[Tree-of-Thoughts](#tree-of-thoughtstot思维树)、[Reflection](#reflection反思)

### Circuit Breaker（熔断器）
**定义**：当下游服务故障率超过阈值时，自动切断请求，避免雪崩效应。一段时间后尝试恢复。
**英文**：Circuit Breaker
**相关章节**：第 22 章、第 23 章
**示例**：向量数据库连续 10 次超时后，熔断器打开，所有查询快速失败，30 秒后半开尝试恢复
**相关术语**：[Backpressure](#backpressure背压)、[Guardrails](#guardrails护栏)

### Computer Use（计算机使用）
**定义**：Agent 通过操作浏览器、桌面应用或命令行工具与计算机环境交互的能力。实现自动化 RPA 和复杂任务执行。
**英文**：Computer Use
**相关章节**：第 27 章
**示例**：Agent 打开浏览器，登录系统，填写表单，上传文件，截图确认结果
**相关术语**：[Tool Use](#tool-use工具调用)、[Agentic Coding](#agentic-coding-agent-编程)

### Context Window（上下文窗口）
**定义**：LLM 一次能够处理的最大 Token 数量。包括输入提示词、历史对话和输出内容。
**英文**：Context Window
**相关章节**：第 7 章、第 9 章
**示例**：GPT-4 的上下文窗口为 128K Token，约 96,000 个英文单词
**相关术语**：[Token](#tokentoken令牌)、[Summarization](#summarization摘要压缩)

### Cost Optimization（成本优化）
**定义**：通过模型选择、缓存、批处理、Prompt 压缩等手段降低 LLM 使用成本的策略。
**英文**：Cost Optimization
**相关章节**：第 23 章
**示例**：简单问答用 Claude Haiku（$0.25/M tokens），复杂推理用 Opus（$15/M tokens）
**相关术语**：[Token Budget](#token-budget-token-预算)、[Caching](#caching缓存)

---

## D - DAG 与辩论

### DAG（有向无环图）
**定义**：Directed Acyclic Graph，节点代表任务，边代表依赖关系的图结构。用于定义多 Agent 工作流的执行顺序。
**英文**：DAG (Directed Acyclic Graph)
**相关章节**：第 14 章、第 21 章
**示例**：研究任务：搜索（节点 A）→ 分析（节点 B、C 并行）→ 综合（节点 D 依赖 B、C）
**相关术语**：[Orchestrator](#orchestrator编排器)、[Temporal](#temporal工作流引擎)

### Debate（辩论模式）
**定义**：多个 Agent 持不同观点进行多轮辩论，通过对抗性讨论提高决策质量的推理模式。
**英文**：Debate Pattern
**相关章节**：第 18 章
**示例**：法律 Agent 分别担任原告、被告和法官，辩论合同条款合理性，最终达成共识
**相关术语**：[Multi-Agent](#multi-agent多智能体系统)、[Reflection](#reflection反思)

### Deterministic Replay（确定性重放）
**定义**：工作流引擎在失败后能够从检查点重新执行，保证相同输入产生相同结果的机制。Temporal 的核心特性。
**英文**：Deterministic Replay
**相关章节**：第 21 章
**示例**：Agent 任务执行到第 5 步崩溃，Temporal 从检查点重放，跳过已完成的 1-4 步
**相关术语**：[Temporal](#temporal工作流引擎)、[Asynchronous Workflow](#asynchronous-workflow异步工作流)

---

## E - Embedding 与错误处理

### Embedding（向量嵌入）
**定义**：将文本转换为高维向量的数值表示，捕获语义信息。用于相似度搜索、聚类和 RAG。
**英文**：Embedding
**相关章节**：第 11 章、第 19 章
**示例**：文本"机器学习"转换为 1536 维向量 [0.23, -0.45, ...]
**相关术语**：[RAG](#ragretrieval-augmented-generation检索增强生成)、[Vector Database](#vector-database向量数据库)

### Error Handling（错误处理）
**定义**：识别、捕获和恢复系统异常的机制。在 Agent 系统中包括重试、降级、熔断和用户友好的错误提示。
**英文**：Error Handling
**相关章节**：第 22 章、第 23 章
**示例**：LLM 调用超时后自动重试 3 次，仍失败则降级返回"当前服务繁忙，请稍后重试"
**相关术语**：[Circuit Breaker](#circuit-breaker熔断器)、[Guardrails](#guardrails护栏)

---

## F - 函数调用与失败恢复

### Fallback Strategy（降级策略）
**定义**：主服务不可用时，自动切换到备选方案的容错机制。保证系统在部分故障时仍能提供基础功能。
**英文**：Fallback Strategy
**相关章节**：第 22 章、第 23 章
**示例**：向量搜索失败时，降级为关键词搜索；LLM 不可用时，返回预定义模板
**相关术语**：[Circuit Breaker](#circuit-breaker熔断器)、[Error Handling](#error-handling错误处理)

### Function Calling（函数调用）
**定义**：LLM 根据对话生成结构化的函数调用请求，由系统执行并返回结果。是 Tool Use 的标准化实现方式。
**英文**：Function Calling
**相关章节**：第 3 章、第 4 章
**示例**：用户问"北京天气"，LLM 返回 `get_weather(city="北京")`，系统调用天气 API
**相关术语**：[Tool Use](#tool-use工具调用)、[ReAct](#react推理-行动循环)

---

## G - 护栏与治理

### Guardrails（护栏）
**定义**：对 Agent 行为进行约束和验证的安全机制。包括输入验证、输出过滤、权限检查和策略执行。
**英文**：Guardrails
**相关章节**：第 24 章、第 25 章
**示例**：拦截包含敏感词的输出，阻止 Agent 访问未授权的数据库，限制 Token 使用
**相关术语**：[OPA](#opaopen-policy-agent策略引擎)、[WASI](#wasiwasm-sandbox-wasm-沙箱)

---

## H - Handoff 与钩子

### Handoff（任务交接）
**定义**：一个 Agent 将任务转交给另一个更专业的 Agent 的协作模式。常见于多 Agent 系统的责任分工。
**英文**：Handoff
**相关章节**：第 15 章、第 16 章
**示例**：通用 Agent 识别用户需要退款，将对话交接给退款专员 Agent
**相关术语**：[Multi-Agent](#multi-agent多智能体系统)、[Orchestrator](#orchestrator编排器)

### Hooks（钩子）
**定义**：在 Agent 执行流程的特定时刻触发的事件回调机制。用于日志记录、监控、审计或自定义逻辑注入。
**英文**：Hooks
**相关章节**：第 6 章、第 29 章
**示例**：`before_tool_call` 钩子记录工具名称和参数，`after_llm_response` 钩子过滤敏感信息
**相关术语**：[Plugins](#plugins插件)、[Observability](#observability可观测性)

---

## I - 集成与幂等性

### Idempotency（幂等性）
**定义**：相同的操作执行多次产生相同结果的性质。在分布式系统中用于安全重试和去重。
**英文**：Idempotency
**相关章节**：第 21 章、第 22 章
**示例**：使用 request_id 确保支付请求重试时不会重复扣款
**相关术语**：[Deterministic Replay](#deterministic-replay确定性重放)、[Temporal](#temporal工作流引擎)

---

## L - LLM 与日志

### LLM Service（LLM 服务层）
**定义**：封装多个 LLM Provider 调用、工具选择和执行的服务层。统一 API，处理认证、重试、缓存等通用逻辑。
**英文**：LLM Service
**相关章节**：第 3 章、第 21 章
**示例**：Shannon 的 Python 服务层，支持 OpenAI、Anthropic、Azure，统一返回格式
**相关术语**：[Agent](#agent智能体)、[Function Calling](#function-calling函数调用)

### Logging（日志）
**定义**：记录系统运行时事件、错误和状态变化的机制。在 Agent 系统中用于调试、审计和性能分析。
**英文**：Logging
**相关章节**：第 22 章
**示例**：结构化日志记录每次 LLM 调用的 model、tokens、latency 和 cost
**相关术语**：[Observability](#observability可观测性)、[Tracing](#tracing分布式追踪)

---

## M - MCP 与记忆

### MCP（模型上下文协议）
**定义**：Model Context Protocol，标准化 LLM 与外部数据源、工具连接的开放协议。定义资源、工具和提示词的发现与调用规范。
**英文**：MCP (Model Context Protocol)
**相关章节**：第 4 章、第 5 章
**示例**：通过 MCP Server 暴露 Google Drive 文件，Agent 可以列出、读取和搜索文档
**相关术语**：[Tool Use](#tool-use工具调用)、[Skills](#skills技能)

> ⚠️ **时效性提示** (2026-01): MCP 规范仍在快速演进中。
> 请查阅 [最新文档](https://spec.modelcontextprotocol.io/) 确认传输层和能力更新。

### Memory（记忆系统）
**定义**：Agent 存储和检索历史信息的机制。包括短期记忆（当前会话）、长期记忆（持久化存储）和语义记忆（知识图谱）。
**英文**：Memory
**相关章节**：第 7 章、第 8 章、第 9 章
**示例**：用户说"我上次提到的项目"，Agent 从 Session 中检索上下文，理解指代
**相关术语**：[Session](#session会话)、[RAG](#ragretrieval-augmented-generation检索增强生成)

### Metrics（指标监控）
**定义**：收集、聚合和可视化系统性能数据的机制。在 Agent 系统中监控 Token 使用、延迟、成功率等关键指标。
**英文**：Metrics
**相关章节**：第 22 章
**示例**：Prometheus 收集每个 Agent 的平均响应时间、Token 消耗和工具调用次数
**相关术语**：[Observability](#observability可观测性)、[Logging](#logging日志)

### Multi-Agent（多智能体系统）
**定义**：多个 Agent 协作完成复杂任务的系统架构。Agent 之间可以并行、串行或动态交互。
**英文**：Multi-Agent System
**相关章节**：第 13 章、第 14 章、第 16 章
**示例**：电商系统中，搜索 Agent、推荐 Agent 和客服 Agent 协同处理用户购物流程
**相关术语**：[Orchestrator](#orchestrator编排器)、[DAG](#dag有向无环图)

---

## O - 编排与可观测性

### Observability（可观测性）
**定义**：通过日志、指标、追踪理解系统内部状态的能力。在 Agent 系统中用于调试复杂的推理链和工具调用。
**英文**：Observability
**相关章节**：第 22 章
**示例**：通过 Trace ID 追踪一个请求经过 Orchestrator → Agent → LLM → Tool 的完整路径
**相关术语**：[Logging](#logging日志)、[Tracing](#tracing分布式追踪)、[Metrics](#metrics指标监控)

### OPA（策略引擎）
**定义**：Open Policy Agent，通用的策略引擎，使用 Rego 语言定义和执行授权、验证和合规规则。
**英文**：OPA (Open Policy Agent)
**相关章节**：第 24 章
**示例**：定义策略"财务数据只能由财务部门访问"，拦截未授权的 Agent 查询
**相关术语**：[Guardrails](#guardrails护栏)、[WASI](#wasiwasm-sandbox-wasm-沙箱)

### Orchestrator（编排器）
**定义**：协调多个 Agent 执行复杂工作流的控制层。负责路由、调度、结果聚合和错误处理。
**英文**：Orchestrator
**相关章节**：第 13 章、第 21 章
**示例**：Shannon 的 Go Orchestrator 根据 DAG 定义，并行调用 3 个 Agent，汇总结果
**相关术语**：[DAG](#dag有向无环图)、[Multi-Agent](#multi-agent多智能体系统)

---

## P - 规划与插件

### P2P（点对点通信）
**定义**：Peer-to-Peer，Agent 之间直接通信，无需中心化编排器的协作模式。适合动态、自组织的多 Agent 场景。
**英文**：P2P (Peer-to-Peer)
**相关章节**：第 16 章
**示例**：研究 Agent 自主发现并请求分析 Agent 提供数据，无需 Orchestrator 介入
**相关术语**：[Multi-Agent](#multi-agent多智能体系统)、[Handoff](#handoff任务交接)

### Planning（规划）
**定义**：Agent 在执行前制定分步计划的推理模式。将复杂目标分解为可执行的子任务序列。
**英文**：Planning
**相关章节**：第 10 章
**示例**：用户要求"组织团建"，Agent 生成计划：1. 确定日期 2. 预订场地 3. 发送邀请
**相关术语**：[ReAct](#react推理-行动循环)、[Tree-of-Thoughts](#tree-of-thoughtstot思维树)

### Plugins（插件）
**定义**：封装特定能力的可插拔模块，通过标准接口扩展 Agent 功能。支持热加载和版本管理。
**英文**：Plugins
**相关章节**：第 29 章
**示例**：安装 Slack 插件后，Agent 可以发送消息、创建频道、获取历史记录
**相关术语**：[Skills](#skills技能)、[Hooks](#hooks钩子)、[MCP](#mcp模型上下文协议)

### Prompt（提示词）
**定义**：发送给 LLM 的输入文本，包括指令、示例、上下文和问题。是控制 Agent 行为的核心界面。
**英文**：Prompt
**相关章节**：第 2 章、第 12 章
**示例**：`"你是一个专业的客服 Agent。用户问题：{question}。请参考知识库：{context}"`
**相关术语**：[Chain-of-Thought](#chain-of-thoughtcot思维链)、[ReAct](#react推理-行动循环)

---

## R - RAG 与反思

### RAG（检索增强生成）
**定义**：Retrieval-Augmented Generation，在生成回复前先检索相关文档，将检索结果作为上下文提供给 LLM 的技术。
**英文**：RAG (Retrieval-Augmented Generation)
**相关章节**：第 8 章、第 19 章
**示例**：用户问"退货政策"，向量搜索找到相关文档，LLM 基于文档生成答案
**相关术语**：[Embedding](#embedding向量嵌入)、[Vector Database](#vector-database向量数据库)

### Rate Limiting（速率限制）
**定义**：限制单位时间内请求数量的流控机制。防止滥用、保护下游服务和控制成本。
**英文**：Rate Limiting
**相关章节**：第 23 章、第 24 章
**示例**：每个用户每分钟最多 10 次 LLM 调用，超过返回 429 Too Many Requests
**相关术语**：[Token Budget](#token-budget-token-预算)、[Backpressure](#backpressure背压)

### ReAct（推理-行动循环）
**定义**：Reason-Act Loop，Agent 循环执行"推理 → 行动 → 观察"的决策模式。LLM 生成推理过程和下一步行动，执行后观察结果。
**英文**：ReAct (Reason-Act Loop)
**相关章节**：第 2 章、第 3 章
**示例**：思考：需要查天气 → 行动：调用 weather_api → 观察：北京 15°C → 思考：回答用户
**相关术语**：[Agent](#agent智能体)、[Tool Use](#tool-use工具调用)

### Reflection（反思）
**定义**：Agent 评估自身输出质量、识别错误并改进的自我优化模式。通过多轮迭代提高结果准确性。
**英文**：Reflection
**相关章节**：第 11 章
**示例**：Agent 生成代码后，自我检查"是否有语法错误？测试覆盖率如何？"，修复问题
**相关术语**：[Chain-of-Thought](#chain-of-thoughtcot思维链)、[Debate](#debate辩论模式)

---

## S - 会话与技能

### Sandbox（沙箱）
**定义**：隔离执行环境，限制代码访问系统资源的安全机制。在 Agent 系统中用于安全地运行不受信任的工具代码。
**英文**：Sandbox
**相关章节**：第 25 章
**示例**：WASI 沙箱中运行用户自定义 Python 工具，无法访问文件系统或网络
**相关术语**：[WASI](#wasiwasm-sandbox-wasm-沙箱)、[Guardrails](#guardrails护栏)

### Session（会话）
**定义**：用户与 Agent 的一次完整交互过程，包含多轮对话和相关上下文。会话可持久化以支持跨次对话。
**英文**：Session
**相关章节**：第 7 章、第 9 章
**示例**：用户登录后的对话历史、偏好设置、未完成任务都绑定到同一个 session_id
**相关术语**：[Memory](#memory记忆系统)、[Context Window](#context-window上下文窗口)

### Skills（技能）
**定义**：可复用的 Agent 能力模块，封装特定领域的提示词、工具和工作流。可跨 Agent 共享和组合。
**英文**：Skills
**相关章节**：第 5 章
**示例**：`code_review` 技能包含代码分析提示词、静态检查工具和格式化工具
**相关术语**：[Plugins](#plugins插件)、[MCP](#mcp模型上下文协议)

### Streaming（流式响应）
**定义**：LLM 逐步生成内容，客户端实时接收部分结果的模式。改善用户体验，减少首字延迟。
**英文**：Streaming
**相关章节**：第 3 章、第 22 章
**示例**：ChatGPT 式的逐字输出，用户无需等待完整响应即可看到进展
**相关术语**：[Asynchronous Workflow](#asynchronous-workflow异步工作流)

### Summarization（摘要压缩）
**定义**：将长文本压缩为简短摘要的技术。在 Agent 系统中用于应对上下文窗口限制和降低 Token 成本。
**英文**：Summarization
**相关章节**：第 7 章
**示例**：将 1000 轮历史对话压缩为 "用户咨询退款流程，已解决" 的摘要
**相关术语**：[Context Window](#context-window上下文窗口)、[Memory](#memory记忆系统)

### Supervisor（监督者模式）
**定义**：一个 Agent 作为协调者，分配任务给专业 Agent 并汇总结果的多 Agent 架构模式。
**英文**：Supervisor Pattern
**相关章节**：第 15 章
**示例**：Supervisor Agent 将"写文章"分解为研究、写作、校对任务，分配给三个专员
**相关术语**：[Orchestrator](#orchestrator编排器)、[Handoff](#handoff任务交接)

---

## T - Token 与工具

### Temporal（工作流引擎）
**定义**：分布式、持久化的工作流引擎，支持长时间运行任务、确定性重放和自动重试。适合复杂 Agent 编排。
**英文**：Temporal (Workflow Engine)
**相关章节**：第 21 章
**示例**：定义"每日报告"工作流，即使服务重启也能从断点继续执行
**相关术语**：[DAG](#dag有向无环图)、[Orchestrator](#orchestrator编排器)

### Token（令牌）
**定义**：LLM 处理的文本最小单位，约 0.75 个英文单词或 0.5 个中文字符。LLM 计费和上下文窗口都以 Token 计量。
**英文**：Token
**相关章节**：第 7 章、第 23 章
**示例**：文本 "Hello World" 约等于 2 个 Token
**相关术语**：[Context Window](#context-window上下文窗口)、[Token Budget](#token-budget-token-预算)

### Token Budget（Token 预算）
**定义**：对单次请求、会话或用户的 Token 使用上限进行管理和分配的机制。控制成本和防止滥用。
**英文**：Token Budget
**相关章节**：第 23 章、第 24 章
**示例**：免费用户每天 10K Token，付费用户 1M Token，超限后降级或拒绝服务
**相关术语**：[Rate Limiting](#rate-limiting速率限制)、[Cost Optimization](#cost-optimization成本优化)

### Tool Use（工具调用）
**定义**：Agent 通过调用外部工具（API、数据库、计算器等）扩展能力的机制。是 Agent 与环境交互的核心方式。
**英文**：Tool Use
**相关章节**：第 3 章、第 4 章
**示例**：调用 `search_api("Claude 3.5")` 获取最新信息，调用 `calculator(23*47)` 计算结果
**相关术语**：[Function Calling](#function-calling函数调用)、[MCP](#mcp模型上下文协议)

### Tracing（分布式追踪）
**定义**：跟踪请求在分布式系统中的完整路径，记录每个服务的耗时和状态。在 Agent 系统中用于调试复杂调用链。
**英文**：Tracing (Distributed Tracing)
**相关章节**：第 22 章
**示例**：Jaeger 显示请求经过 Gateway(20ms) → Orchestrator(50ms) → Agent(300ms) → LLM(2s)
**相关术语**：[Observability](#observability可观测性)、[Logging](#logging日志)

### Tree-of-Thoughts（ToT，思维树）
**定义**：将问题解空间建模为树，探索多条推理路径，通过回溯和剪枝找到最优解的高级推理模式。
**英文**：Tree-of-Thoughts (ToT)
**相关章节**：第 17 章
**示例**：解数学题时，尝试 3 种方法，每种方法展开 2 步，评估选择最佳路径继续
**相关术语**：[Chain-of-Thought](#chain-of-thoughtcot思维链)、[Planning](#planning规划)

---

## V - 向量数据库

### Vector Database（向量数据库）
**定义**：专门存储和检索高维向量的数据库。支持相似度搜索，常用于 RAG 和语义检索。
**英文**：Vector Database
**相关章节**：第 8 章、第 19 章
**示例**：Pinecone、Qdrant 存储文档 Embedding，毫秒级返回最相似的 Top-K 结果
**相关术语**：[Embedding](#embedding向量嵌入)、[RAG](#ragretrieval-augmented-generation检索增强生成)

---

## W - WASI 与工作流

### WASI（WASM 沙箱）
**定义**：WebAssembly System Interface，允许 WASM 模块安全访问文件、网络等系统资源的标准接口。用于隔离执行不受信任代码。
**英文**：WASI (WebAssembly System Interface)
**相关章节**：第 25 章
**示例**：Shannon 使用 WASI 运行用户自定义工具，限制只能访问指定目录和 API
**相关术语**：[Sandbox](#sandbox沙箱)、[Guardrails](#guardrails护栏)

### Workflow（工作流）
**定义**：定义任务执行顺序、依赖关系和错误处理的流程编排。在 Agent 系统中用于组织多步骤任务。
**英文**：Workflow
**相关章节**：第 14 章、第 21 章
**示例**：电商订单工作流：验证库存 → 扣款 → 发货 → 发送通知
**相关术语**：[DAG](#dag有向无环图)、[Temporal](#temporal工作流引擎)

---

## 附录使用说明

### 如何使用本术语表

1. **快速查找**：按字母顺序组织，方便定位术语
2. **理解概念**：每个术语包含简洁定义和实际示例
3. **深入学习**：通过"相关章节"链接跳转到详细内容
4. **扩展阅读**：通过"相关术语"链接理解术语之间的关系

### 术语更新说明

AI Agent 领域快速发展，部分术语的定义和最佳实践可能随时间变化。本术语表基于 2026 年初的行业共识编写。建议结合各章节的"时效性提示"和外部参考资料使用。

### 反馈与贡献

如发现术语定义不准确或需要补充新术语，欢迎通过 GitHub Issues 反馈。

---

## 索引参考

**按主题分类**：
- **Agent 基础**：Agent、ReAct、Tool Use、Function Calling、Prompt
- **推理模式**：CoT、ToT、Planning、Reflection、Debate
- **多 Agent**：Multi-Agent、Orchestrator、DAG、Supervisor、Handoff、P2P
- **扩展机制**：MCP、Skills、Hooks、Plugins
- **上下文记忆**：Context Window、Memory、Session、Summarization、RAG
- **生产架构**：Temporal、Observability、Logging、Tracing、Metrics
- **安全合规**：OPA、WASI、Guardrails、Sandbox
- **成本优化**：Token Budget、Rate Limiting、Caching、Batch Processing
- **容错机制**：Circuit Breaker、Backpressure、Fallback Strategy、Error Handling
- **前沿实践**：Computer Use、Agentic Coding、Background Agent

**按技术栈分类**：
- **LLM 相关**：Token、Context Window、Prompt、Streaming、Function Calling
- **数据存储**：Vector Database、Embedding、Caching
- **工作流引擎**：Temporal、DAG、Workflow、Deterministic Replay
- **可观测性**：Logging、Tracing、Metrics、Observability
- **安全隔离**：WASI、Sandbox、OPA、Guardrails

---

**本术语表涵盖全书 30 章核心概念，共 60+ 关键术语。建议配合各章节内容和代码示例深入理解。**
