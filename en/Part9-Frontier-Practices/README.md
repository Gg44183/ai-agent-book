# Part 9: Frontier Practices

> Latest hot topics: Computer Use, Agentic Coding, Background Agents, Tiered Model Strategy

## Chapter List

| Chapter | Title | Core Question | Shannon Reference |
|---------|-------|---------------|-------------------|
| 27 | Computer Use | How do Agents operate browsers and desktops? | `config/models.yaml` multimodal |
| 28 | Agentic Coding | How to build code generation Agents? | `file_ops.py`, `wasi_sandbox.rs` |
| 29 | Background Agents | How to implement async long-running tasks? | `schedules/manager.go` |
| 30 | Tiered Model Strategy | How to optimize 50-70% of costs? | `config/models.yaml`, `manager.py` |

---

## Chapter Summaries

### Chapter 27: Computer Use

> When Agents get "eyes" and "hands": From calling APIs to operating real interfaces

**Core Content**:
- **Perceive-Decide-Execute Loop**: Screenshot understanding -> Coordinate calculation -> Click/Input -> Result verification
- **Multimodal Model Integration**: Visual understanding is the key capability for Computer Use
- **Coordinate Calibration**: Handling different resolutions and DPI scaling differences
- **Safety Protection**: Dangerous zone detection, input content filtering, OPA policy extensions
- **Verification Loop**: Screenshot to verify results after each operation, auto-retry on failure

**Shannon Code**: `config/models.yaml` (multimodal_models), suggested tool extension patterns

---

### Chapter 28: Agentic Coding

> Making Agents your programming partner: From code generation to complete development workflows

**Core Content**:
- **Safe File Operations**: Whitelist directories, path validation, symlink protection
- **WASI Sandbox Execution**: Fuel/Epoch limits, memory isolation, timeout control
- **Code Reflection Loop**: Generate -> Review -> Improve iteration process
- **Multi-file Edit Coordination**: Atomic changes, backup rollback mechanism
- **Git Integration**: Branch management, auto-commit, PR creation

**Shannon Code**: `python/llm-service/llm_service/tools/builtin/file_ops.py`, `rust/agent-core/src/wasi_sandbox.rs`, `go/orchestrator/internal/workflows/patterns/reflection.go`

---

### Chapter 29: Background Agents

> Keep tasks running in the background: Temporal scheduling and scheduled task systems

**Core Content**:
- **Temporal Schedule API**: Native Cron scheduling, pause/resume, timezone support
- **Resource Limits**: MaxPerUser (50), MinCronInterval (60min), MaxBudgetPerRunUSD ($10)
- **ScheduledTaskWorkflow**: Wrapper workflow, records execution metadata (model, tokens, cost)
- **Orphan Detection**: Periodically detect Temporal and database state inconsistencies, auto-cleanup
- **Budget Injection**: Cost tracking and limits per execution

**Shannon Code**: `schedules/manager.go`, `scheduled_task_workflow.go`

---

### Chapter 30: Tiered Model Strategy

> Smart routing to achieve 50-70% cost reduction: Not every task needs the strongest model

**Core Content**:
- **Three-tier model strategy**: Small (50%) / Medium (40%) / Large (10%) target distribution
- **Priority Routing**: Multiple providers per tier selected by priority, auto-Fallback
- **Complexity Analysis**: Auto-select model tier based on task characteristics
- **Capability Matrix**: multimodal, thinking, coding, long_context capability markers
- **Circuit Breaking Degradation**: Circuit Breaker + auto-degradation to backup providers
- **Cost Tracking**: Centralized pricing configuration, real-time cost monitoring

**Shannon Code**: `config/models.yaml`, `llm_provider/manager.py`

---

## Learning Objectives

After completing this Part, you will be able to:

- [ ] Understand Computer Use's perceive-decide-execute loop
- [ ] Design safe Agentic Coding workflows (sandbox + reflection)
- [ ] Use Temporal Schedule API to implement scheduled background tasks
- [ ] Configure three-tier model strategy to achieve 50-70% cost reduction
- [ ] Add frontier capabilities to Research Agent (v0.9)

---

## Shannon Code Guide

```
Shannon/
├── config/
│   └── models.yaml                    # Three-tier model config, pricing, capability matrix
├── go/orchestrator/
│   └── internal/
│       ├── schedules/
│       │   └── manager.go             # Scheduled task manager (CRUD, resource limits)
│       └── workflows/scheduled/
│           └── scheduled_task_workflow.go  # Wrapper workflow
├── python/llm-service/
│   ├── llm_provider/
│   │   └── manager.py                 # LLM manager (routing, circuit breaking, Fallback)
│   └── llm_service/tools/builtin/
│       ├── file_ops.py                # Safe file read/write tools
│       └── python_wasi_executor.py    # Python sandbox execution
└── rust/agent-core/src/sandbox/
    └── wasi_sandbox.rs                # WASI sandbox implementation
```

---

## Hot Topic Correlations

| Topic | Representative Products | Shannon Implementation | Chapter |
|-------|------------------------|------------------------|---------|
| Computer Use | Claude Computer Use, similar UI agents | Multimodal + Tool extensions | Ch27 |
| Agentic Coding | Claude Code, Cursor, Windsurf | WASI sandbox + File tools | Ch28 |
| Background Agents | Claude Code Ctrl+B | Temporal Schedule API | Ch29 |
| Cost Optimization | Enterprise cost reduction needs | Three-tier model strategy | Ch30 |

---

## Cost Optimization Example

```
Without tiering (all Large):
  1M requests x $0.09/request = $90,000/month

With tiered strategy (50/40/10):
  Small:  500K x $0.0006  = $300
  Medium: 400K x $0.0018  = $720
  Large:  100K x $0.09    = $9,000
  Total: $10,020/month

Savings: $79,980/month (89%)
```

---

## Prerequisites

- Part 1-8 completed (especially Part 7-8 production architecture and enterprise features)
- Browser automation basics (Playwright/Puppeteer) - optional
- Cron expression basics - optional
- Multi-model API experience - optional

---

## Research Agent v0.9

Frontier capability modules covered in this Part:

| Module | Chapter | Capability |
|--------|---------|------------|
| Computer Use | Chapter 27 | Web browsing, content extraction |
| Agentic Coding | Chapter 28 | Analysis script generation |
| Background Agents | Chapter 29 | Scheduled research reports |
| Tiered Models | Chapter 30 | Smart model selection |

**Final Form**:
```
User: "Generate an AI industry daily report at 9 AM every day"

Research Agent v0.9:
1. [Schedule] Create Cron scheduled task (0 9 * * *)
2. [Tiered] Use Small model for complexity assessment
3. [Multi-Agent] Parallel search/analysis/writing
4. [Browser] Access websites without APIs for content extraction
5. [Coding] Generate data visualization scripts
6. [Budget] Control per-execution cost < $2
7. [Output] Send structured report
```
