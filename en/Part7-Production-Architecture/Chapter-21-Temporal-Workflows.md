# Chapter 21: Temporal Workflows

> **Temporal makes your workflows as reliable as database transactions—saving progress at each step, recovering from breakpoints after crashes, no need to write your own state machine.**
> **But it's not magic: you need to understand the difference between Activity and Workflow, know when to use version gating, or you'll hit pitfalls.**

---

> **Quick Path** (5 minutes to grasp the core)
>
> 1. Workflow = deterministic logic (replayable), Activity = side-effect operations (not replayable)
> 2. Checkpoint recovery: After crash, Workflow replays history events, Activity results recovered from event log
> 3. Determinism requirement: No time.Now(), rand, goroutine—only use workflow.* APIs
> 4. Version gating: workflow.GetVersion() lets old and new code coexist, smooth upgrades
> 5. Timeout configuration: StartToClose (single attempt) + ScheduleToClose (total duration) + HeartbeatTimeout
>
> **10-minute path**: 21.1-21.3 -> 21.5 -> Shannon Lab

---

3 AM, your Agent is executing a deep research task. Already called the search API 15 times, spent 2 minutes.

Then, the server OOM crashed.

Next morning the user asks: Where's my research report?

You check logs, discover the task is completely lost. Those 15 searches—all wasted.

This is why you need Temporal.

---

## 21.1 Why Do We Need a Workflow Engine

### Problems with Traditional Methods

Without a workflow engine, how do you handle long-running tasks?

Below compares three traditional methods and their problems:

```python
# ========== Method 1: Synchronous Execution ==========
def deep_research(query):
    for topic in decompose(query):
        result = search(topic)        # On crash, all completed work lost
        results.append(result)
    return synthesize(results)
# Problem: Process crashes at 8th subtask, first 7 all lost

# ========== Method 2: Database State Machine ==========
def deep_research_with_state(query, task_id):
    state = db.get(task_id) or {"completed": [], "results": []}
    for i, topic in enumerate(decompose(query)):
        if i in state["completed"]: continue
        state["results"].append(search(topic))
        state["completed"].append(i)
        db.update(state)              # Save each step, but code is complex
    return synthesize(state["results"])
# Problem: Each task needs state machine, serialization error-prone, concurrency hard to handle, no unified monitoring

# ========== Method 3: Message Queue + Worker ==========
def start_research(query):           # Producer
    for topic in decompose(query): queue.push({"topic": topic})
def worker():                        # Consumer
    while True:
        msg = queue.pop()
        db.save_result(msg["task_id"], search(msg["topic"]))
# Problem: Result aggregation needs extra logic, retry strategy scattered, dependencies hard to express, no global state when debugging
```

### How Temporal Solves It

Temporal's core idea: **Persist your code as data**.

Not saving state snapshots, but recording every decision point. Wherever execution reaches, record it there. After crash, replay these records, automatically recover to the previous position.

```go
// Your code
func DeepResearchWorkflow(ctx workflow.Context, query string) (string, error) {
    topics := decompose(query)
    var results []string
    for _, topic := range topics {
        // Temporal automatically persists this call's result
        var result string
        workflow.ExecuteActivity(ctx, SearchActivity, topic).Get(ctx, &result)
        results = append(results, result)
    }
    return synthesize(results), nil
}
```

Looks like regular code, but Temporal is working behind the scenes:
1. Records a checkpoint before each `ExecuteActivity`
2. Activity results are persisted to database
3. During replay after crash, completed Activities directly return cached results
4. Continues execution from the last checkpoint

---

## 21.2 Core Concepts

### Workflow vs Activity

This is Temporal's most important distinction:

| Concept | Workflow | Activity |
|---------|----------|----------|
| **Definition** | Orchestration logic, decides what to do | Actual work, executes it |
| **Must Be Deterministic** | Yes | No |
| **Can Have Side Effects** | No | Yes |
| **Auto Retry** | No (replay) | Yes |
| **Timeout Handling** | Overall timeout | Single attempt timeout + retry |

**Key Rule**: Workflow code must be **deterministic**.

Same input must produce same decision sequence. This is to ensure replay can recover to the correct state.

Which operations break determinism? Comparing correct vs incorrect approaches:

```go
// ========== Breaks Determinism (Wrong) ==========    // ========== Correct Approach ==========
time.Now()                                   // workflow.Now(ctx)
rand.Int()                                   // workflow.SideEffect(ctx, func() { return rand.Int() })
http.Get("...")                              // workflow.ExecuteActivity(ctx, FetchActivity, ...)
os.Getenv("...")                             // Pass through Workflow parameters
uuid.New()                                   // workflow.SideEffect(ctx, func() { return uuid.New() })
```

### Activity Retry Strategy

Activities can configure retry strategies. Below shows configuration for different scenarios:

```go
// ========== General Configuration (Complete Parameter Explanation) ==========
activityOptions := workflow.ActivityOptions{
    StartToCloseTimeout: 30 * time.Second,        // Single execution timeout
    RetryPolicy: &temporal.RetryPolicy{
        InitialInterval:    1 * time.Second,      // First retry interval
        BackoffCoefficient: 2.0,                  // Exponential backoff coefficient
        MaximumInterval:    30 * time.Second,     // Maximum retry interval
        MaximumAttempts:    3,                    // Maximum retry count
        NonRetryableErrorTypes: []string{         // Non-retryable errors
            "InvalidInputError", "PermissionDeniedError",
        },
    },
}

// ========== Scenario-Based Configuration (Shannon Practice) ==========
// Long-running Agent execution: allow longer timeout + heartbeat detection
agentOpts := workflow.ActivityOptions{
    StartToCloseTimeout: 120 * time.Second,
    HeartbeatTimeout:    30 * time.Second,        // Heartbeat timeout detects liveness
    RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 3},
}
// Quick operations: short timeout, fewer retries
quickOpts := workflow.ActivityOptions{
    StartToCloseTimeout: 10 * time.Second,
    RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 2},
}
ctx = workflow.WithActivityOptions(ctx, activityOptions)
```

---

## 21.3 Version Gating

This is where Temporal is easiest to trip up, and also one of the most used patterns in Shannon code.

### Problem and Solution

```go
// ========== Problem: Direct Code Modification Causes Replay Failure ==========
// v1 original version
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, ActivityA, ...)
    workflow.ExecuteActivity(ctx, ActivityB, ...)
    return nil
}
// v2 directly adding ActivityNew -> Running v1 workflows fail with Non-determinism error during replay

// ========== Solution: Use workflow.GetVersion ==========
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, ActivityA, ...)
    // Version gating: new workflows return 1, old workflows during replay return DefaultVersion (-1)
    if workflow.GetVersion(ctx, "add_activity_new", workflow.DefaultVersion, 1) >= 1 {
        workflow.ExecuteActivity(ctx, ActivityNew, ...)
    }
    workflow.ExecuteActivity(ctx, ActivityB, ...)
    return nil
}
```

### Shannon Practical Application and Naming Conventions

Shannon code extensively uses version gating (reference `strategies/research.go`):

```go
// ========== Feature Evolution Example: Hierarchical Memory vs Session Memory ==========
hierarchicalVersion := workflow.GetVersion(ctx, "memory_retrieval_v1", workflow.DefaultVersion, 1)
if hierarchicalVersion >= 1 && input.SessionID != "" {
    workflow.ExecuteActivity(ctx, activities.RetrieveHierarchicalMemoryActivity, ...).Get(ctx, &memoryResult)
} else if workflow.GetVersion(ctx, "session_memory_v1", workflow.DefaultVersion, 1) >= 1 {
    workflow.ExecuteActivity(ctx, activities.GetSessionMessagesActivity, ...).Get(ctx, &messages)
}

// ========== Conditional Enabling Example: Context Compression ==========
if workflow.GetVersion(ctx, "context_compress_v1", workflow.DefaultVersion, 1) >= 1 &&
   input.SessionID != "" && len(input.History) > 20 {
    // New version enables context compression
}

// ========== Version Naming Conventions ==========
// Good naming (feature name + version number)           // Bad naming
workflow.GetVersion(ctx, "memory_retrieval_v1", ...)    // "fix_bug_123" (too vague)
workflow.GetVersion(ctx, "context_compress_v1", ...)    // "v2" (doesn't describe feature)
workflow.GetVersion(ctx, "iterative_research_v1", ...)
```

---

## 21.4 Signals and Queries

While a Workflow is running, you might need to interact with it:
- **Signal**: Send message to Workflow, trigger behavior change
- **Query**: Get Workflow's current state, doesn't change execution

### Signal Example: Pause/Resume

Shannon uses signals to implement pause/resume:

```go
// control/handler.go
type SignalHandler struct {
    paused        bool
    pauseCh       workflow.Channel
    resumeCh      workflow.Channel
    cancelCh      workflow.Channel
}

func (h *SignalHandler) Setup(ctx workflow.Context) {
    version := workflow.GetVersion(ctx, "pause_resume_v1", workflow.DefaultVersion, 1)
    if version < 1 {
        return  // Old versions don't support
    }

    // Register signal channels
    pauseSig := workflow.GetSignalChannel(ctx, "pause")
    resumeSig := workflow.GetSignalChannel(ctx, "resume")
    cancelSig := workflow.GetSignalChannel(ctx, "cancel")

    // Register query handler
    workflow.SetQueryHandler(ctx, "get_status", func() (string, error) {
        if h.paused {
            return "paused", nil
        }
        return "running", nil
    })

    // Background goroutine handles signals
    workflow.Go(ctx, func(ctx workflow.Context) {
        for {
            selector := workflow.NewSelector(ctx)
            selector.AddReceive(pauseSig, func(ch workflow.ReceiveChannel, more bool) {
                h.paused = true
            })
            selector.AddReceive(resumeSig, func(ch workflow.ReceiveChannel, more bool) {
                h.paused = false
            })
            selector.Select(ctx)
        }
    })
}

// Check pause state in workflow
func (h *SignalHandler) WaitIfPaused(ctx workflow.Context) {
    for h.paused {
        workflow.Sleep(ctx, 1*time.Second)
    }
}
```

### Sending Signals

```go
// External signal sending
client.SignalWorkflow(ctx, workflowID, runID, "pause", nil)

// Via HTTP API call
// POST /api/v1/workflows/{workflowID}/signal
// { "signal_name": "pause" }
```

---

## 21.5 Worker Startup and Priority Queues

### Startup Process

Shannon's Worker startup has comprehensive retry mechanisms:

```go
// TCP pre-check (quick determination if service is reachable)
for i := 1; i <= 60; i++ {
    c, err := net.DialTimeout("tcp", host, 2*time.Second)
    if err == nil {
        _ = c.Close()
        break
    }
    logger.Warn("Waiting for Temporal TCP endpoint",
        zap.String("host", host), zap.Int("attempt", i))
    time.Sleep(1 * time.Second)
}

// SDK connection retry (heavier operation, use exponential backoff)
var tClient client.Client
var err error
for attempt := 1; ; attempt++ {
    tClient, err = client.Dial(client.Options{
        HostPort: host,
        Logger:   temporal.NewZapAdapter(logger),
    })
    if err == nil {
        break
    }
    delay := time.Duration(min(attempt, 15)) * time.Second
    logger.Warn("Temporal not ready, retrying",
        zap.Int("attempt", attempt),
        zap.Duration("sleep", delay),
        zap.Error(err))
    time.Sleep(delay)
}
```

### Priority Queues

Shannon supports multi-queue mode, different priority tasks go to different queues:

```go
if priorityQueues {
    _ = startWorker("shannon-tasks-critical", 12, 12)
    _ = startWorker("shannon-tasks-high", 10, 10)
    w = startWorker("shannon-tasks", 8, 8)
    _ = startWorker("shannon-tasks-low", 4, 4)
} else {
    w = startWorker("shannon-tasks", 10, 10)
}
```

Typical uses for priority queues:

| Queue | Concurrency | Use Case |
|-------|-------------|----------|
| critical | 12 | Real-time requests where user is waiting |
| high | 10 | Important but can wait a bit |
| normal | 8 | Regular background tasks |
| low | 4 | Report generation, data cleanup, etc. |

---

## 21.6 Fire-and-Forget Pattern

For operations that don't affect the main flow (like logging, metrics reporting), use Fire-and-Forget:

```go
// Persist Agent execution result (fire-and-forget)
func persistAgentExecution(ctx workflow.Context, workflowID, agentID, input string,
                           result activities.AgentExecutionResult) {
    // Short timeout + no retry
    persistCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: 5 * time.Second,
        RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 1},
    })

    // Don't wait for result
    workflow.ExecuteActivity(
        persistCtx,
        activities.PersistAgentExecutionStandalone,
        activities.PersistAgentExecutionInput{
            WorkflowID: workflowID,
            AgentID:    agentID,
            Input:      input,
            Output:     result.Response,
            TokensUsed: result.TokensUsed,
        },
    )
    // Note: No .Get() call, don't wait for completion
}
```

Use cases:
- Logging
- Metrics reporting
- Audit trail
- Cache warming

---

## 21.7 Parallel Execution Patterns

Below shows three parallel execution patterns:

```go
// ========== Pattern 1: Basic Parallel (Wait for All) ==========
futures := make([]workflow.Future, len(subtasks))
for i, subtask := range subtasks {
    futures[i] = workflow.ExecuteActivity(ctx, activities.ExecuteAgent, subtask.Query)
}
for i, f := range futures {
    f.Get(ctx, &results[i])  // Wait in order
}

// ========== Pattern 2: Selector (First Complete First Processed) ==========
futures := make(map[string]workflow.Future)
for _, topic := range topics {
    futures[topic] = workflow.ExecuteActivity(ctx, SearchActivity, topic)
}
for len(futures) > 0 {
    selector := workflow.NewSelector(ctx)
    for topic, f := range futures {
        t := topic  // Closure capture
        selector.AddFuture(f, func(f workflow.Future) {
            f.Get(ctx, &result)
            processResult(t, result)
            delete(futures, t)
        })
    }
    selector.Select(ctx)
}

// ========== Pattern 3: Timeout Control ==========
ctx, cancel := workflow.WithCancel(ctx)
defer cancel()
selector := workflow.NewSelector(ctx)
selector.AddFuture(workflow.ExecuteActivity(ctx, LongTask, input), func(f workflow.Future) {
    err = f.Get(ctx, &result)
})
selector.AddFuture(workflow.NewTimer(ctx, 5*time.Minute), func(f workflow.Future) {
    cancel()  // Timeout cancel
    err = errors.New("timeout")
})
selector.Select(ctx)
```

---

## 21.8 Child Workflows

Complex tasks can be decomposed into child workflows:

```go
func ParentWorkflow(ctx workflow.Context, topics []string) ([]string, error) {
    var results []string

    // Start child workflows in parallel
    var futures []workflow.Future
    for _, topic := range topics {
        childOpts := workflow.ChildWorkflowOptions{
            WorkflowID: fmt.Sprintf("research-%s", topic),
        }
        childCtx := workflow.WithChildOptions(ctx, childOpts)
        future := workflow.ExecuteChildWorkflow(childCtx, ResearchChildWorkflow, topic)
        futures = append(futures, future)
    }

    // Wait for all to complete
    for _, future := range futures {
        var result string
        if err := future.Get(ctx, &result); err != nil {
            return nil, err
        }
        results = append(results, result)
    }

    return results, nil
}
```

Benefits of child workflows:
- **Isolated Failures**: One child workflow failing doesn't affect others
- **Independent Retry**: Can configure retry strategies separately
- **Visualization**: Clear hierarchy display in Temporal UI
- **Parallel Execution**: Multiple child workflows can run concurrently

---

## 21.9 Time Travel Debugging

### Temporal Web UI

Temporal comes with a Web UI where you can see:
- Workflow list and status
- Event history for each workflow
- Activity execution details
- Retry counts and error messages

Visit `http://localhost:8088` to view.

### Debugging Steps

```
1. Open Temporal Web UI
2. Find the problematic workflow
3. View Event History
4. Locate failed Activity
5. Check input parameters and errors
6. Reproduce locally with same input
```

### Export and Replay

```bash
# Export execution history
temporal workflow show --workflow-id "task-123" --output json > history.json

# Local replay test
temporal workflow replay --workflow-id "task-123"
```

---

## 21.10 Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Direct External Service Calls | `http.Get()` breaks determinism | Use `workflow.ExecuteActivity()` |
| Forgetting Version Gating | Adding Activity causes old workflow replay failure | Wrap with `workflow.GetVersion()` |
| Activity Returns Large Data | Several MB of data impacts performance | Return path/reference instead of data itself |
| Infinite Loop | Event history bloat | Use `Continue-As-New` to restart workflow |
| Ignoring Cancellation | Resource leaks, can't gracefully exit | Check `ctx.Err()` in loops |

```go
// ========== Pitfall 1: Direct External Service Calls ==========
http.Get("...")                                       // Wrong: breaks determinism
workflow.ExecuteActivity(ctx, FetchActivity, ...).Get(ctx, &data)  // Correct

// ========== Pitfall 2: Forgetting Version Gating ==========
workflow.ExecuteActivity(ctx, NewActivity, ...)       // Wrong: old workflow replay fails
if workflow.GetVersion(ctx, "add_new", workflow.DefaultVersion, 1) >= 1 {
    workflow.ExecuteActivity(ctx, NewActivity, ...)   // Correct
}

// ========== Pitfall 3: Activity Returns Large Data ==========
return downloadLargeFile()                            // Wrong: 10MB data
path := saveLargeFile(); return path, nil             // Correct: only return path

// ========== Pitfall 4: Infinite Loop ==========
for { workflow.ExecuteActivity(ctx, Poll, ...) }      // Wrong: event history bloat
if iteration > 1000 { return workflow.NewContinueAsNewError(ctx, Workflow, 0) }  // Correct

// ========== Pitfall 5: Ignoring Cancellation ==========
for { doWork() }                                      // Wrong: can't gracefully exit
for { if ctx.Err() != nil { return ctx.Err() }; doWork() }  // Correct
```

---

## What This Chapter Covered

1. **Workflow vs Activity**: Workflow orchestrates decisions (must be deterministic), Activity actually executes (can have side effects)
2. **Version Gating**: Use `workflow.GetVersion` for code changes to ensure compatibility
3. **Signals and Queries**: Signals change behavior, queries get state
4. **Parallel Execution**: Start with Futures, process with Selectors
5. **Fire-and-Forget**: Non-critical persistence doesn't block main flow

---

## Shannon Lab (10-Minute Quickstart)

This section helps you map this chapter's concepts to Shannon source code in 10 minutes.

### Required Reading (1 file)

- `go/orchestrator/internal/workflows/strategies/research.go`: Search "GetVersion" to see how `workflow.GetVersion` is used—understand actual version gating patterns

### Optional Deep Dive (pick 2 based on interest)

- `go/orchestrator/internal/workflows/control/handler.go`: Signal handler implementation, understand pause/resume mechanism
- `go/orchestrator/internal/activities/agent.go`: See how Activity wraps LLM calls

---

## Exercises

### Exercise 1: Design Version Migration

Your workflow was originally like this:

```go
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, StepA)
    workflow.ExecuteActivity(ctx, StepB)
    return nil
}
```

Now you need to:
1. Add a StepC between A and B
2. Rename StepB to StepB2 (parameters also changed)

Write the new code that's compatible with old workflows.

### Exercise 2: Parallel + Timeout

Design a workflow that satisfies:
- Start 5 search tasks in parallel
- Overall timeout 2 minutes
- Proceed to next phase when any 3 complete
- Timeout or failed tasks don't block overall progress

Write the key code snippets.

### Exercise 3 (Advanced): Budget Middleware

Design a Token budget middleware that satisfies:
- Check remaining budget before each Activity call
- Deduct actual consumption after Activity completes
- Return specific error when budget exhausted
- Write pseudocode

---

## Further Reading

- [Temporal Official Documentation](https://docs.temporal.io/): Concept explanations and best practices
- [Temporal Workflow Versioning](https://docs.temporal.io/develop/go/versioning): Detailed version gating guide
- [Temporal in Production](https://docs.temporal.io/production-deployment): Production deployment configuration

---

## Next Chapter Preview

Temporal solved the "crash recovery" problem. But there's still one more: **the system is running, but you don't know what it's doing.**

User says "my task is so slow"—what's slow? LLM call slow? Search slow? Database slow?

Next chapter covers **Observability**: how to use the three pillars of metrics, logs, and traces to make your Agent system transparent like glass.

Ready? Let's move on.
