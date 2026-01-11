# Appendix B: Pattern Selection Guide

> **This appendix helps you quickly decide "which pattern for which scenario," avoiding over-engineering or choosing the wrong tool.**

---

The most common problem in production isn't "not knowing how to implement a pattern"--it's "not knowing which pattern to use."

You might ask:

- "Is this task better suited for ReAct or Planning?"
- "When do I need Reflection?"
- "What's the difference between ToT and Debate?"
- "What problem does multi-Agent orchestration actually solve?"

This appendix uses decision trees, comparison tables, and anti-pattern warnings to help you find the right pattern in 5 minutes.

**Core Principle**: Prefer simple patterns, only upgrade when necessary. Complex patterns = high cost + slow speed + hard to debug.

---

## B.1 Single Agent Reasoning Pattern Selection

### Quick Decision Tree

```
Does your task need external tools? (search, API, file read/write, etc.)
│
├─ No → Does it need explicit reasoning process?
│        ├─ Yes → Chain-of-Thought (Chapter 12)
│        └─ No → Direct Prompt (no pattern needed)
│
└─ Yes → Does the task need to be broken into multiple steps?
          │
          ├─ No → ReAct (Chapter 2)
          │
          └─ Yes → Do the decomposed steps have clear dependencies?
                   │
                   ├─ No dependencies → ReAct (Chapter 2)
                   │
                   └─ Has dependencies → Planning (Chapter 10)
                                        ↓
                   After Planning, is quality satisfactory?
                   │
                   ├─ Yes → Done
                   │
                   └─ No → Need quality improvement or multiple perspectives?
                          │
                          ├─ Quality improvement → Reflection (Chapter 11)
                          │
                          └─ Multiple perspectives/controversial → Need to explore multiple paths?
                                                                  │
                                                                  ├─ Need exploration → Tree-of-Thoughts (Chapter 17)
                                                                  │
                                                                  └─ Need opposition → Debate (Chapter 18)
```

### Single Agent Pattern Comparison Matrix

| Pattern | Token Cost | Latency | Quality Gain | Use Cases | Not Suitable | Chapter |
|---------|-----------|---------|--------------|-----------|--------------|---------|
| **ReAct** | Medium (baseline) | Low (baseline) | Baseline | General tasks needing tools | Pure reasoning without tools | Ch 2 |
| **Planning** | High (+50%) | Medium (+30%) | +20% | Complex multi-step tasks needing decomposition | Single-step tasks | Ch 10 |
| **Reflection** | Very High (+100%) | High (+100%) | +30% | High-value outputs needing iterative improvement | Simple tasks/latency-sensitive | Ch 11 |
| **CoT** | Medium (+10%) | Low | +15% | Math reasoning/logical analysis | Tasks needing tool calls | Ch 12 |
| **ToT** | Very High (+200-400%) | Very High | +40% | Multi-path exploration/critical early decisions | Problems with clear solutions | Ch 17 |
| **Debate** | Extreme (+300-500%) | Extreme | +50% | Controversial topics/need multiple perspectives | Questions with objective answers | Ch 18 |
| **Research** | High (+80%) | High | +35% | Systematic research/report generation | Simple information queries | Ch 19 |

**Cost Notes**: Percentages relative to ReAct baseline. Actual costs vary by task complexity and configuration.

### When to Use ReAct

**Use Cases**:
- Need to call external tools (search, APIs, file operations)
- Task is relatively simple, doesn't need complex planning
- Need transparent reasoning process (auditable)
- Cost-sensitive, need fast response

**Not Suitable**:
- Pure reasoning tasks without external tools
- Task too complex, needs advance planning
- Need high-quality output, single generation isn't enough

**Cost Considerations**:
- Token: ~3000-8000 per task
- Latency: ~10-30 seconds
- Low retry cost on failure

**Examples**:
```
✓ "Help me figure out why the API is returning 500 errors"
✓ "Search for the latest Python 3.12 features and summarize"
✗ "Research this company and write a 5000-word report" (use Planning + Research)
✗ "Should this system use microservices or monolith?" (use Debate)
```

### When to Use Planning

**Use Cases**:
- Task can be decomposed into 3+ independent subtasks
- Subtasks have dependencies
- Need parallel execution for efficiency
- Output needs structured organization

**Not Suitable**:
- Simple tasks completable in one step
- Tasks that can't be clearly decomposed
- Latency-sensitive real-time scenarios

**Cost Considerations**:
- Token: ~5000-15000 per task
- Latency: ~30-90 seconds
- Decomposition overhead: ~1000 tokens
- Synthesis overhead: ~1500 tokens

**Key Configuration**:
```go
MaxIterations:     3    // Max iterations for coverage evaluation
MinCoverage:       0.85 // Minimum coverage threshold
MaxSubtasks:       7    // Upper limit on subtask count
```

**Examples**:
```
✓ "Analyze Tesla's 2024 financial performance, including revenue, profit, competitor comparison"
✓ "Help me research OpenAI and write a complete competitive analysis report"
✗ "What's the weather today" (use ReAct)
✗ "Help me write a sorting function" (use ReAct)
```

### When to Use Reflection

**Use Cases**:
- High quality requirements for output (reports, documents, analysis)
- Clear evaluation criteria exist
- Not cost-sensitive, quality is priority
- Single generation quality is unstable

**Not Suitable**:
- Simple Q&A with low quality requirements
- Latency-sensitive (will double time)
- Cost-sensitive
- No objective evaluation criteria

**Cost Considerations**:
- Token: Initial cost x 2-3
- Latency: Initial latency x 2-3
- MaxRetries = 1-2 is sufficient

**Key Configuration**:
```go
MaxRetries:          2    // Maximum retry count
ConfidenceThreshold: 0.7  // Quality threshold (don't set too high)
Criteria: []string{
    "completeness",
    "correctness",
    "clarity",
}
```

**Examples**:
```
✓ "Generate API technical documentation" (high quality requirement)
✓ "Write an investment due diligence report" (accuracy critical)
✗ "Help me check tomorrow's weather" (not worth it)
✗ "Real-time chat response" (latency-sensitive)
```

### When to Use Chain-of-Thought

**Use Cases**:
- Math reasoning, logical analysis
- Need to show thinking process
- No external tools needed
- Reduce jumping errors

**Not Suitable**:
- Need to call external tools (use ReAct)
- Need to explore multiple paths (use ToT)
- Simple fact queries

**Cost Considerations**:
- Token: +10% vs direct answer
- Latency: Almost no increase
- Improves reasoning accuracy ~15%

**Examples**:
```
✓ "Solve this math problem: 24 game (3,8,8,3)"
✓ "Analyze this code's time complexity"
✗ "Search for latest AI news" (use ReAct)
✗ "Design a high-availability architecture" (use ToT or Debate)
```

### When to Use Tree-of-Thoughts

**Use Cases**:
- Large solution space with multiple possible paths
- Early decisions have big impact, hard to backtrack
- Can evaluate intermediate state quality
- Quality > cost/speed

**Not Suitable**:
- Clear standard solution exists
- Cost/latency sensitive
- Cannot evaluate intermediate states

**Cost Considerations**:
- Token: +200-400% (node count x single cost)
- Latency: +200-300%
- ExplorationBudget controls upper limit

**Key Configuration**:
```go
MaxDepth:          3    // Tree depth
BranchingFactor:   3    // Branches per node
PruningThreshold:  0.3  // Pruning threshold
ExplorationBudget: 15   // Max nodes to explore
```

**Examples**:
```
✓ "Design a payment system architecture supporting 1M QPS"
✓ "Find the root cause of this complex bug (multiple possible causes)"
✗ "Implement a standard user login feature" (has mature templates)
✗ "Query a database field value" (clear operation)
```

### When to Use Debate

**Use Cases**:
- Controversial topics without standard answers
- Need multi-perspective analysis
- Need adversarial questioning
- High-risk decisions requiring thorough argumentation

**Not Suitable**:
- Has objectively correct answer
- Extremely cost/latency sensitive
- Simple decisions

**Cost Considerations**:
- Token: NumDebaters x MaxRounds x single cost
- Latency: Sequential execution of rounds
- 3 debaters x 2 rounds = 6x baseline cost

**Key Configuration**:
```go
NumDebaters:      3     // 2-5 debaters
MaxRounds:        2     // 2-3 rounds sufficient
Perspectives:     []string{"optimistic", "skeptical", "practical"}
RequireConsensus: false // Don't force consensus
VotingEnabled:    true  // Voting as fallback
```

**Examples**:
```
✓ "Will AI replace SaaS in 2025?"
✓ "Should the company build its own datacenter or use cloud?"
✗ "What's 2 + 2?" (objective answer)
✗ "What's Python's syntax?" (factual question)
```

---

## B.2 Multi-Agent Orchestration Pattern Selection

### Quick Decision Tree

```
Does your task need multiple Agents to collaborate?
│
├─ No → Use single Agent patterns (see B.1)
│
└─ Yes → Can the task be fully planned in advance?
          │
          ├─ Can be fully planned → Do subtasks have dependencies?
          │                          │
          │                          ├─ No dependencies → Parallel (Chapter 14)
          │                          │
          │                          ├─ Partial dependencies → DAG (Chapter 14)
          │                          │
          │                          └─ All sequential → Sequential (Chapter 14)
          │
          └─ Cannot be fully planned → Need dynamic adjustments?
                                       │
                                       ├─ Yes → Supervisor (Chapter 15)
                                       │
                                       └─ No → Need task handoff between Agents?
                                               │
                                               ├─ Yes → Handoff (Chapter 16)
                                               │
                                               └─ No → DAG (Chapter 14)
```

### Multi-Agent Pattern Comparison Matrix

| Pattern | Scheduling Complexity | Coordination Cost | Use Cases | Not Suitable | Chapter |
|---------|----------------------|------------------|-----------|--------------|---------|
| **Parallel** | Low | Low | Independent subtasks executing in parallel | Tasks with dependencies | Ch 14 |
| **Sequential** | Low | Medium | Strict sequential dependencies | Tasks that can parallelize | Ch 14 |
| **DAG** | Medium | Medium | Partial parallelism + dependencies | Cannot determine dependencies | Ch 14 |
| **Supervisor** | High | High | Dynamic task assignment | Can be planned in advance | Ch 15 |
| **Handoff** | Low | Medium | Specialized division of labor | No specialization needed | Ch 16 |

### When to Use Parallel Execution

**Use Cases**:
- 3+ completely independent subtasks
- Can execute simultaneously for efficiency
- No data dependencies between tasks

**Not Suitable**:
- Tasks have dependencies
- Too few tasks (1-2)
- Later tasks need earlier results

**Cost Considerations**:
- Parallelism: MaxConcurrency = 3-5 (personal accounts)
- Parallelism: MaxConcurrency = 5-10 (team accounts)
- Time savings: ~40-60% vs sequential

**Examples**:
```
✓ "Search stock prices for Tesla, BYD, and Rivian"
✓ "Translate this text to English, Japanese, and French"
✗ "First search stock price, then calculate gain" (has dependency, use Sequential)
✗ "Search one company's info" (only 1 task, no need for parallel)
```

### When to Use Sequential Execution

**Use Cases**:
- Later tasks clearly depend on earlier results
- Need result passing
- Logical chain progression

**Not Suitable**:
- Tasks can parallelize
- No dependencies between tasks

**Cost Considerations**:
- Latency: Sum of all task latencies
- PassPreviousResults: adds ~10% tokens
- Suitable for: 3-5 sequential steps

**Examples**:
```
✓ "First search Tesla stock price, then calculate gain, finally generate analysis"
✓ "Read file → Analyze data → Generate report"
✗ "Search three companies' stock prices" (can parallelize, use Parallel)
```

### When to Use DAG Workflows

**Use Cases**:
- 5+ subtasks, some can parallelize
- Clear dependency relationships
- Need smart scheduling for efficiency
- Can plan dependencies in advance

**Not Suitable**:
- All tasks are independent (use Parallel)
- All tasks are sequential (use Sequential)
- Cannot determine dependencies (use Supervisor)
- Too few tasks (<3)

**Cost Considerations**:
- Scheduling overhead: ~5% extra tokens
- Time savings: ~20-40% vs pure sequential
- Suitable for: 5-15 subtasks

**Key Configuration**:
```go
MaxConcurrency:         5      // Maximum concurrency
DependencyWaitTimeout:  360    // Dependency wait timeout (seconds)
PassDependencyResults:  true   // Pass dependency results
```

**Examples**:
```
✓ "Analyze Tesla financials: Get financials(A) + Get competitors(B) → Calculate growth(C, depends on A) + Calculate margin(D, depends on A) → Comparative analysis(E, depends on A,B,C,D)"
✗ "Search three companies" (no dependencies, use Parallel)
✗ "Dynamically decide next step" (cannot plan in advance, use Supervisor)
```

### When to Use Supervisor Pattern

**Use Cases**:
- Uncertain task count/types
- Need dynamic task assignment
- Agents need real-time communication
- Partial failures need intelligent degradation

**Not Suitable**:
- Tasks can be fully planned in advance
- Simple fixed processes
- Extremely cost-sensitive

**Cost Considerations**:
- Scheduling overhead: +20-30% tokens
- Supervisor decisions: ~1000 tokens per round
- Suitable for: Complex, dynamic scenarios

**Examples**:
```
✓ "Coordinate multiple expert Agents to solve complex technical problems (dynamically adjust strategy)"
✓ "Customer service system dynamically routes to different expert Agents"
✗ "Fixed data processing pipeline" (use DAG)
✗ "Simple parallel search" (use Parallel)
```

### When to Use Handoff Mechanism

**Use Cases**:
- Need specialized division of labor
- Clear task handoff points
- Context needs to be passed
- Similar to workflow transfer

**Not Suitable**:
- No specialization needed
- No handoff points
- Tasks completely independent

**Cost Considerations**:
- Handoff overhead: ~500 tokens per handoff
- Context passing: varies by size

**Examples**:
```
✓ "Customer Service Agent → Tech Support Agent → Refund Agent"
✓ "Requirements Analysis Agent → Design Agent → Implementation Agent"
✗ "Parallel search of three websites" (no handoff needed, use Parallel)
```

---

## B.3 Anti-Pattern Warnings

### When NOT to Use Multi-Agent

**Anti-Pattern 1: Multi-Agent for Its Own Sake**

```
Wrong Example:
Task: "Query today's weather"
Design: Agent-1 searches → Agent-2 parses → Agent-3 formats

Problem: 3 Agents = 3x cost, but single Agent can finish in 5 seconds

Correct Approach: Single ReAct Agent does it directly
```

**Anti-Pattern 2: Over-Decomposition**

```
Wrong Example:
Task: "Analyze a company"
Decomposition: 20 fine-grained subtasks (company name, founding date, CEO, funding, product1, product2...)

Problem: Coordination cost > execution cost

Correct Approach: 3-7 coarse-grained tasks (basic info, product matrix, funding history)
```

**Anti-Pattern 3: Forced Parallelism**

```
Wrong Example:
Task: "First search stock price, then calculate gain"
Forced parallel: Agent-1 searches price || Agent-2 calculates gain (no data!)

Problem: Dependencies ignored

Correct Approach: Sequential or DAG
```

**Anti-Pattern 4: Infinite Iteration**

```
Wrong Example:
Planning with RequirePerfectCoverage = true
MaxIterations = 100

Problem: Never reaches 100%, burns through tokens

Correct Approach: CoverageThreshold = 0.85, MaxIterations = 3
```

### Common Over-Engineering Traps

| Trap | Symptom | Consequence | Solution |
|------|---------|-------------|----------|
| **Tool worship** | Use new tools immediately | Adds complexity, unclear ROI | Evaluate ROI, pilot first |
| **Premature optimization** | Start with most complex pattern | Hard to debug, high cost | Start simple, upgrade gradually |
| **Config explosion** | Expose dozens of parameters | Users don't know how to tune | Provide 3 presets: fast/balanced/quality |
| **Blind trend-following** | Implement every paper | Academic results ≠ production results | Validate necessity first, then implement |

### Decision Principles

**Occam's Razor**:
> Do not multiply entities beyond necessity. If a simple pattern works, don't use a complex one.

**Progressive Enhancement**:
```
Level 1: Single LLM call
         ↓ Not good enough?
Level 2: ReAct (add tools)
         ↓ Task too complex?
Level 3: Planning (add decomposition)
         ↓ Quality not good enough?
Level 4: Reflection (add iteration)
         ↓ Need multiple perspectives?
Level 5: ToT / Debate (add exploration/opposition)
```

**80/20 Rule**:
- 80% of tasks: ReAct + Planning is enough
- 15% of tasks: Need Reflection
- 5% of tasks: Need ToT / Debate

---

## B.4 Cost-Quality-Latency Tradeoff Matrix

### Three-Dimensional Tradeoff

```
              Quality
               ▲
               │
    Debate ●   │
           │   │   ● ToT
  Research ●   │
           │   │
Reflection ●   │
           │   │
  Planning ●   │   ● DAG
           │   │
       CoT ●   │
           │   │
     ReAct ●───┼────────────────► Latency
               │
               │
               ▼
              Cost
```

### Scenario Optimization Strategies

**Latency-Sensitive Scenarios** (real-time chat, online queries)
- Prefer: ReAct, CoT
- Avoid: Reflection, ToT, Debate
- Config: MaxIterations = 5, Timeout = 30s

**Cost-Sensitive Scenarios** (large-scale batch processing)
- Prefer: ReAct, Planning
- Avoid: ToT, Debate, Research
- Config: BudgetAgentMax = 5000, use small model for evaluation

**Quality-Sensitive Scenarios** (research reports, decision support)
- Prefer: Planning + Reflection, Research, Debate
- Cost: Can accept 2-5x
- Config: ConfidenceThreshold = 0.8, MaxRetries = 2

### Cost Estimation Formulas

**Single Agent Patterns**:
```
ReAct:      Base
Planning:   Base × (1 + 0.5 × NumSubtasks)
Reflection: Base × (1 + MaxRetries)
CoT:        Base × 1.1
ToT:        Base × ExplorationBudget
Debate:     Base × NumDebaters × MaxRounds
```

**Multi-Agent Patterns**:
```
Parallel:   Base × NumTasks / MaxConcurrency (time optimization)
Sequential: Base × NumTasks (time accumulation)
DAG:        Base × NumTasks × 0.6-0.8 (partial parallelism)
Supervisor: Base × (NumAgents + SupervisorOverhead)
```

---

## B.5 Quick Selection Reference Table

### Selection by Task Type

| Task Type | Recommended Pattern | Reason | Chapter |
|-----------|---------------------|--------|---------|
| **Simple information query** | ReAct | Direct tool calling | Ch 2 |
| **Complex research report** | Planning + Research | Needs decomposition and systematic research | Ch 10, 19 |
| **Tech stack decision** | Debate | Needs multi-perspective weighing | Ch 18 |
| **Architecture design** | ToT or Debate | Explore multiple options or adversarial discussion | Ch 17, 18 |
| **Code debugging** | ReAct or ToT | Simple use ReAct, complex use ToT | Ch 2, 17 |
| **Math reasoning** | CoT | Show reasoning process | Ch 12 |
| **Documentation generation** | Planning + Reflection | Structured + quality assurance | Ch 10, 11 |
| **Data analysis** | Planning + DAG | Decomposition + parallel processing | Ch 10, 14 |
| **Customer service routing** | Supervisor or Handoff | Dynamic assignment or specialization | Ch 15, 16 |
| **Workflow execution** | DAG | Fixed process + dependency management | Ch 14 |

### Selection by Constraints

| Primary Constraint | Recommended Patterns | Avoid Patterns |
|-------------------|---------------------|----------------|
| **Latency < 10s** | ReAct, CoT | ToT, Debate, Reflection |
| **Cost < $0.01** | ReAct (small model) | ToT, Debate |
| **Quality > 90%** | Planning + Reflection + Debate | Single ReAct |
| **Explainability** | CoT, ReAct | Black-box APIs |
| **Reliability** | DAG (with retry) | Single-point Agent |

### Selection by Team Maturity

| Team Stage | Recommended Start | Gradually Introduce | Defer |
|------------|------------------|---------------------|-------|
| **Exploration** | ReAct, CoT | Planning | ToT, Debate |
| **Growth** | Planning, Reflection | DAG, Supervisor | - |
| **Mature** | Full pattern suite | Custom patterns | - |

---

## B.6 Real-World Case Studies

### Case 1: Competitive Analysis

**Requirement**: Analyze 3 competitors, generate comparison report

**Initial Choice**: Planning (decomposition) + Parallel (parallel search)

**Optimization Direction**:
- High quality requirement → add Reflection
- Has controversy (e.g., tech roadmap comparison) → add Debate

**Final Solution**:
```
Planning (decompose into 3 subtasks)
  → Parallel (parallel search of 3 companies)
  → Synthesis (combine results)
  → Reflection (quality check)
```

**Cost Estimate**:
- Base (ReAct): ~5000 tokens
- Planning: +2500 tokens
- Parallel (x3): +10000 tokens
- Reflection: +5000 tokens
- Total: ~22500 tokens (~$0.03 with GPT-4o-mini)

### Case 2: Technical Architecture Design

**Requirement**: Design a high-concurrency payment system

**Initial Choice**: ToT (explore multiple architecture options)

**Optimization Direction**:
- Has clear constraints (e.g., cost, tech stack) → use Debate over ToT
- Need expert perspectives → configure 3 Perspectives (performance, cost, security)

**Final Solution**:
```
Debate (3 perspectives: performance-first, cost-first, security-first)
  → 2 rounds of debate
  → Moderator synthesizes
```

**Cost Estimate**:
- Base: ~8000 tokens
- Debate (3 × 2 rounds): ~48000 tokens
- Total: ~56000 tokens (~$0.07 with GPT-4o-mini)

### Case 3: Customer Service Smart Routing

**Requirement**: Route user questions to different expert Agents

**Initial Choice**: Supervisor (dynamic routing)

**Optimization Direction**:
- Routing rules relatively fixed → use Handoff over Supervisor
- Save Supervisor decision cost

**Final Solution**:
```
Classification Agent (determine question type)
  → Handoff to corresponding expert
  → Expert Agent handles
```

**Cost Estimate**:
- Classification: ~1000 tokens
- Expert handling: ~5000 tokens
- Handoff overhead: ~500 tokens
- Total: ~6500 tokens per request

---

## B.7 Configuration Template Reference

### Fast Template (Latency Priority)

```yaml
mode: react
config:
  max_iterations: 5
  timeout_seconds: 30
  budget_agent_max: 5000

# Disabled
reflection_enabled: false
debate_enabled: false
tot_enabled: false
```

### Balanced Template

```yaml
mode: planning
config:
  max_iterations: 10
  timeout_seconds: 120
  budget_agent_max: 15000

  # Planning
  max_subtasks: 7
  min_coverage: 0.85

  # Reflection (optional)
  reflection_enabled: true
  reflection_max_retries: 1
  reflection_confidence_threshold: 0.7
```

### Quality Template (Quality Priority)

```yaml
mode: research
config:
  max_iterations: 15
  timeout_seconds: 300
  budget_agent_max: 30000

  # Research
  max_research_rounds: 3
  coverage_threshold: 0.9

  # Reflection
  reflection_enabled: true
  reflection_max_retries: 2
  reflection_confidence_threshold: 0.8

  # Debate (optional)
  debate_enabled: true
  debate_num_debaters: 3
  debate_max_rounds: 2
```

---

## B.8 Debugging Decision Flow

When task results aren't satisfactory:

```
Poor results?
│
├─ Did it call tools?
│  ├─ No → Check MinIterations, force tool use
│  └─ Yes → Continue
│
├─ Did it decompose the task?
│  ├─ No → Task complexity > 0.5? Consider enabling Planning
│  └─ Yes → Continue
│
├─ Is quality satisfactory?
│  ├─ No → Try Reflection (MaxRetries=1)
│  └─ Yes but controversial → Consider Debate
│
└─ Is cost acceptable?
   ├─ Too high → Lower MaxIterations, BudgetAgentMax
   └─ Acceptable → Continue tuning config parameters
```

---

## B.9 Summary and Decision Principles

### Golden Rules

1. **Occam's Razor**: Keep it simple when possible
2. **Progressive Enhancement**: Start with ReAct, upgrade gradually
3. **Evaluate ROI**: Does cost increase bring equivalent quality improvement
4. **Clarify Constraints**: Latency/cost/quality--optimize at most two

### Common Decision Paths

**90% of tasks**:
- ReAct (needs tools)
- CoT (pure reasoning)

**5% of tasks**:
- Planning (complex decomposition)
- DAG (parallel optimization)

**3% of tasks**:
- Reflection (high quality)
- Research (systematic research)

**2% of tasks**:
- ToT (multi-path exploration)
- Debate (controversial topics)

### Final Advice

**In production**:
- First get ReAct working end-to-end
- Identify bottlenecks (quality/efficiency/cost)
- Selectively introduce advanced patterns
- Continuously monitor ROI

**Don't chase perfection**:
- 85% quality might be good enough
- Last 15% improvement often needs 5x cost
- Diminishing returns apply

**Remember**:
> Correct pattern selection > perfect parameter tuning

---

## Further Reading

- **Chapter 2**: ReAct Loop - Foundational pattern
- **Chapter 10**: Planning Pattern - Task decomposition
- **Chapter 11**: Reflection Pattern - Quality assurance
- **Chapter 12**: Chain-of-Thought - Reasoning display
- **Chapter 14**: DAG Workflows - Multi-Agent orchestration
- **Chapter 17**: Tree-of-Thoughts - Multi-path exploration
- **Chapter 18**: Debate Pattern - Adversarial discussion
- **Chapter 23**: Token Budget Control - Cost management
- **Appendix A**: Case Studies (if available) - Real scenarios

---

**Next Step**: After selecting the appropriate pattern using this guide, refer to the corresponding chapter to learn implementation details.
