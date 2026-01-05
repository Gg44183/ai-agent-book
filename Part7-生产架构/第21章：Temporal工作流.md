# 第 21 章：Temporal 工作流

> **Temporal 让你的工作流像数据库事务一样可靠——执行到哪里就保存到哪里，崩溃了从断点恢复，不用自己写状态机。**
> **但它不是魔法：你需要理解 Activity 和 Workflow 的区别，知道什么时候用版本门控，否则会踩坑。**

---

> **⏱️ 快速通道**（5 分钟掌握核心）
>
> 1. Workflow = 确定性逻辑（可重放），Activity = 副作用操作（不可重放）
> 2. 断点续传：崩溃后 Workflow 重放历史事件，Activity 结果从事件日志恢复
> 3. 确定性要求：禁用 time.Now()、rand、goroutine，只用 workflow.* API
> 4. 版本门控：workflow.GetVersion() 让新老代码共存，平滑升级
> 5. 超时配置：StartToClose（单次）+ ScheduleToClose（总时长）+ HeartbeatTimeout
>
> **10 分钟路径**：21.1-21.3 → 21.5 → Shannon Lab

---

半夜 3 点，你的 Agent 正在执行一个深度研究任务。已经调用了 15 次搜索 API、花了 2 分钟。

然后，服务器 OOM 崩了。

第二天早上用户问：我的研究报告呢？

你查日志，发现任务彻底丢了。那 15 次搜索——全部白费。

这就是为什么你需要 Temporal。

---

## 21.1 为什么需要工作流引擎

### 传统方法的问题

没有工作流引擎的时候，你怎么处理长时间运行的任务？

下面对比三种传统方法及其问题：

```python
# ========== 方法 1：同步执行 ==========
def deep_research(query):
    for topic in decompose(query):
        result = search(topic)        # 崩溃时，已完成的全部丢失
        results.append(result)
    return synthesize(results)
# 问题：执行到第 8 个子任务时进程崩了，前面 7 个全部丢失

# ========== 方法 2：数据库状态机 ==========
def deep_research_with_state(query, task_id):
    state = db.get(task_id) or {"completed": [], "results": []}
    for i, topic in enumerate(decompose(query)):
        if i in state["completed"]: continue
        state["results"].append(search(topic))
        state["completed"].append(i)
        db.update(state)              # 每步保存，但代码复杂
    return synthesize(state["results"])
# 问题：每个任务都要写状态机、序列化易错、并发难处理、无统一监控

# ========== 方法 3：消息队列 + Worker ==========
def start_research(query):           # 生产者
    for topic in decompose(query): queue.push({"topic": topic})
def worker():                        # 消费者
    while True:
        msg = queue.pop()
        db.save_result(msg["task_id"], search(msg["topic"]))
# 问题：结果聚合需额外逻辑、重试策略分散、依赖关系难表达、调试时无全局状态
```

### Temporal 怎么解决

Temporal 的核心思路：**把你的代码当作数据来持久化**。

不是保存状态快照，而是记录每一个决策点。执行到哪里，就记录到哪里。崩溃后，重放这些记录，自动恢复到之前的位置。

```go
// 你写的代码
func DeepResearchWorkflow(ctx workflow.Context, query string) (string, error) {
    topics := decompose(query)
    var results []string
    for _, topic := range topics {
        // Temporal 自动持久化这个调用的结果
        var result string
        workflow.ExecuteActivity(ctx, SearchActivity, topic).Get(ctx, &result)
        results = append(results, result)
    }
    return synthesize(results), nil
}
```

看起来就是普通的代码，但 Temporal 在背后：
1. 每次 `ExecuteActivity` 前记录一个检查点
2. Activity 结果被持久化到数据库
3. 崩溃后重放时，已完成的 Activity 直接返回缓存结果
4. 从最后一个检查点继续执行

---

## 21.2 核心概念

### Workflow vs Activity

这是 Temporal 最重要的区分：

| 概念 | Workflow | Activity |
|------|----------|----------|
| **定义** | 编排逻辑，决定做什么 | 具体工作，实际执行 |
| **必须确定性** | 是 | 否 |
| **可以有副作用** | 否 | 是 |
| **自动重试** | 否（重放） | 是 |
| **超时处理** | 整体超时 | 单次超时+重试 |

**关键规则**：Workflow 代码必须是**确定性**的。

同样的输入，必须产生同样的决策序列。这是为了保证重放时能恢复到正确状态。

哪些操作会破坏确定性？对比正确与错误做法：

```go
// ========== 破坏确定性（错误）==========    // ========== 正确做法 ==========
time.Now()                                   // workflow.Now(ctx)
rand.Int()                                   // workflow.SideEffect(ctx, func() { return rand.Int() })
http.Get("...")                              // workflow.ExecuteActivity(ctx, FetchActivity, ...)
os.Getenv("...")                             // 通过 Workflow 参数传入
uuid.New()                                   // workflow.SideEffect(ctx, func() { return uuid.New() })
```

### Activity 的重试策略

Activity 可以配置重试策略。以下展示不同场景的配置方式：

```go
// ========== 通用配置（完整参数说明）==========
activityOptions := workflow.ActivityOptions{
    StartToCloseTimeout: 30 * time.Second,        // 单次执行超时
    RetryPolicy: &temporal.RetryPolicy{
        InitialInterval:    1 * time.Second,      // 首次重试间隔
        BackoffCoefficient: 2.0,                  // 指数退避系数
        MaximumInterval:    30 * time.Second,     // 最大重试间隔
        MaximumAttempts:    3,                    // 最大重试次数
        NonRetryableErrorTypes: []string{         // 不重试的错误
            "InvalidInputError", "PermissionDeniedError",
        },
    },
}

// ========== 场景化配置（Shannon 实践）==========
// 长时 Agent 执行：允许更长超时 + 心跳检测
agentOpts := workflow.ActivityOptions{
    StartToCloseTimeout: 120 * time.Second,
    HeartbeatTimeout:    30 * time.Second,        // 心跳超时检测活性
    RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 3},
}
// 快速操作：短超时，少重试
quickOpts := workflow.ActivityOptions{
    StartToCloseTimeout: 10 * time.Second,
    RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 2},
}
ctx = workflow.WithActivityOptions(ctx, activityOptions)
```

---

## 21.3 版本门控 (Version Gating)

这是 Temporal 最容易踩坑的地方，也是 Shannon 代码中使用最多的模式之一。

### 问题与解决方案

```go
// ========== 问题：直接修改代码导致重放失败 ==========
// v1 原始版本
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, ActivityA, ...)
    workflow.ExecuteActivity(ctx, ActivityB, ...)
    return nil
}
// v2 直接加 ActivityNew → 正在运行的 v1 工作流重放时报 Non-determinism 错误

// ========== 解决方案：使用 workflow.GetVersion ==========
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, ActivityA, ...)
    // 版本门控：新工作流返回 1，旧工作流重放返回 DefaultVersion (-1)
    if workflow.GetVersion(ctx, "add_activity_new", workflow.DefaultVersion, 1) >= 1 {
        workflow.ExecuteActivity(ctx, ActivityNew, ...)
    }
    workflow.ExecuteActivity(ctx, ActivityB, ...)
    return nil
}
```

### Shannon 实际应用与命名规范

Shannon 代码中大量使用版本门控（参考 `strategies/research.go`）：

```go
// ========== 功能演进示例：分层记忆 vs 会话记忆 ==========
hierarchicalVersion := workflow.GetVersion(ctx, "memory_retrieval_v1", workflow.DefaultVersion, 1)
if hierarchicalVersion >= 1 && input.SessionID != "" {
    workflow.ExecuteActivity(ctx, activities.RetrieveHierarchicalMemoryActivity, ...).Get(ctx, &memoryResult)
} else if workflow.GetVersion(ctx, "session_memory_v1", workflow.DefaultVersion, 1) >= 1 {
    workflow.ExecuteActivity(ctx, activities.GetSessionMessagesActivity, ...).Get(ctx, &messages)
}

// ========== 条件启用示例：上下文压缩 ==========
if workflow.GetVersion(ctx, "context_compress_v1", workflow.DefaultVersion, 1) >= 1 &&
   input.SessionID != "" && len(input.History) > 20 {
    // 新版本启用上下文压缩
}

// ========== 版本命名规范 ==========
// 好的命名（功能名 + 版本号）           // 不好的命名
workflow.GetVersion(ctx, "memory_retrieval_v1", ...)    // "fix_bug_123"（太模糊）
workflow.GetVersion(ctx, "context_compress_v1", ...)    // "v2"（没有描述功能）
workflow.GetVersion(ctx, "iterative_research_v1", ...)
```

---

## 21.4 信号和查询

Workflow 运行时，你可能需要与它交互：
- **信号 (Signal)**：向 Workflow 发送消息，触发行为变化
- **查询 (Query)**：获取 Workflow 当前状态，不改变执行

### 信号示例：暂停/恢复

Shannon 使用信号实现暂停恢复：

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
        return  // 旧版本不支持
    }

    // 注册信号通道
    pauseSig := workflow.GetSignalChannel(ctx, "pause")
    resumeSig := workflow.GetSignalChannel(ctx, "resume")
    cancelSig := workflow.GetSignalChannel(ctx, "cancel")

    // 注册查询处理器
    workflow.SetQueryHandler(ctx, "get_status", func() (string, error) {
        if h.paused {
            return "paused", nil
        }
        return "running", nil
    })

    // 后台协程处理信号
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

// 在工作流中检查暂停状态
func (h *SignalHandler) WaitIfPaused(ctx workflow.Context) {
    for h.paused {
        workflow.Sleep(ctx, 1*time.Second)
    }
}
```

### 发送信号

```go
// 外部发送信号
client.SignalWorkflow(ctx, workflowID, runID, "pause", nil)

// 通过 HTTP API 调用
// POST /api/v1/workflows/{workflowID}/signal
// { "signal_name": "pause" }
```

---

## 21.5 Worker 启动和优先级队列

### 启动流程

Shannon 的 Worker 启动有完善的重试机制：

```go
// TCP 预检查（快速判断服务是否可达）
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

// SDK 连接重试（更重的操作，用指数退避）
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

### 优先级队列

Shannon 支持多队列模式，不同优先级的任务走不同队列：

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

优先级队列的典型用途：

| 队列 | 并发数 | 用途 |
|------|--------|------|
| critical | 12 | 用户正在等待的实时请求 |
| high | 10 | 重要但可以稍等的任务 |
| normal | 8 | 常规后台任务 |
| low | 4 | 报告生成、数据清理等 |

---

## 21.6 Fire-and-Forget 模式

对于不影响主流程的操作（如日志记录、指标上报），可以用 Fire-and-Forget：

```go
// 持久化 Agent 执行结果（fire-and-forget）
func persistAgentExecution(ctx workflow.Context, workflowID, agentID, input string,
                           result activities.AgentExecutionResult) {
    // 短超时 + 不重试
    persistCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: 5 * time.Second,
        RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 1},
    })

    // 不等待结果
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
    // 注意：没有 .Get() 调用，不等待完成
}
```

用途：
- 日志记录
- 指标上报
- 审计追踪
- 缓存预热

---

## 21.7 并行执行模式

以下展示三种并行执行模式：

```go
// ========== 模式 1：基本并行（等待所有完成）==========
futures := make([]workflow.Future, len(subtasks))
for i, subtask := range subtasks {
    futures[i] = workflow.ExecuteActivity(ctx, activities.ExecuteAgent, subtask.Query)
}
for i, f := range futures {
    f.Get(ctx, &results[i])  // 按顺序等待
}

// ========== 模式 2：选择器（先完成先处理）==========
futures := make(map[string]workflow.Future)
for _, topic := range topics {
    futures[topic] = workflow.ExecuteActivity(ctx, SearchActivity, topic)
}
for len(futures) > 0 {
    selector := workflow.NewSelector(ctx)
    for topic, f := range futures {
        t := topic  // 闭包捕获
        selector.AddFuture(f, func(f workflow.Future) {
            f.Get(ctx, &result)
            processResult(t, result)
            delete(futures, t)
        })
    }
    selector.Select(ctx)
}

// ========== 模式 3：超时控制 ==========
ctx, cancel := workflow.WithCancel(ctx)
defer cancel()
selector := workflow.NewSelector(ctx)
selector.AddFuture(workflow.ExecuteActivity(ctx, LongTask, input), func(f workflow.Future) {
    err = f.Get(ctx, &result)
})
selector.AddFuture(workflow.NewTimer(ctx, 5*time.Minute), func(f workflow.Future) {
    cancel()  // 超时取消
    err = errors.New("timeout")
})
selector.Select(ctx)
```

---

## 21.8 子工作流

复杂任务可以分解为子工作流：

```go
func ParentWorkflow(ctx workflow.Context, topics []string) ([]string, error) {
    var results []string

    // 并行启动子工作流
    var futures []workflow.Future
    for _, topic := range topics {
        childOpts := workflow.ChildWorkflowOptions{
            WorkflowID: fmt.Sprintf("research-%s", topic),
        }
        childCtx := workflow.WithChildOptions(ctx, childOpts)
        future := workflow.ExecuteChildWorkflow(childCtx, ResearchChildWorkflow, topic)
        futures = append(futures, future)
    }

    // 等待所有完成
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

子工作流的好处：
- **隔离失败**：一个子工作流失败不影响其他
- **独立重试**：可以单独配置重试策略
- **可视化**：在 Temporal UI 中清晰展示层级关系
- **并行执行**：多个子工作流可以并发运行

---

## 21.9 时间旅行调试

### Temporal Web UI

Temporal 自带 Web UI，可以看到：
- 工作流列表和状态
- 每个工作流的事件历史
- Activity 执行详情
- 重试次数和错误信息

访问 `http://localhost:8088` 查看。

### 调试步骤

```
1. 打开 Temporal Web UI
2. 找到问题工作流
3. 查看 Event History
4. 定位失败的 Activity
5. 检查输入参数和错误
6. 使用相同输入本地重现
```

### 导出与重放

```bash
# 导出执行历史
temporal workflow show --workflow-id "task-123" --output json > history.json

# 本地重放测试
temporal workflow replay --workflow-id "task-123"
```

---

## 21.10 常见的坑

| 坑 | 问题 | 解决方案 |
|----|------|----------|
| 直接调用外部服务 | `http.Get()` 破坏确定性 | 使用 `workflow.ExecuteActivity()` |
| 忘记版本门控 | 新增 Activity 导致旧工作流重放失败 | 用 `workflow.GetVersion()` 包裹 |
| Activity 返回大数据 | 几 MB 数据影响性能 | 返回路径/引用而非数据本身 |
| 无限循环 | 事件历史膨胀 | 用 `Continue-As-New` 重启工作流 |
| 忽略取消 | 资源泄露，无法优雅退出 | 循环中检查 `ctx.Err()` |

```go
// ========== 坑 1：直接调用外部服务 ==========
http.Get("...")                                       // 错误：破坏确定性
workflow.ExecuteActivity(ctx, FetchActivity, ...).Get(ctx, &data)  // 正确

// ========== 坑 2：忘记版本门控 ==========
workflow.ExecuteActivity(ctx, NewActivity, ...)       // 错误：旧工作流重放失败
if workflow.GetVersion(ctx, "add_new", workflow.DefaultVersion, 1) >= 1 {
    workflow.ExecuteActivity(ctx, NewActivity, ...)   // 正确
}

// ========== 坑 3：Activity 返回大数据 ==========
return downloadLargeFile()                            // 错误：10MB 数据
path := saveLargeFile(); return path, nil             // 正确：只返回路径

// ========== 坑 4：无限循环 ==========
for { workflow.ExecuteActivity(ctx, Poll, ...) }      // 错误：事件历史膨胀
if iteration > 1000 { return workflow.NewContinueAsNewError(ctx, Workflow, 0) }  // 正确

// ========== 坑 5：忽略取消 ==========
for { doWork() }                                      // 错误：无法优雅退出
for { if ctx.Err() != nil { return ctx.Err() }; doWork() }  // 正确
```

---

## 这章说了什么

1. **Workflow vs Activity**：Workflow 编排决策（必须确定性），Activity 实际执行（可以有副作用）
2. **版本门控**：代码变更用 `workflow.GetVersion` 保证兼容性
3. **信号和查询**：信号改变行为，查询获取状态
4. **并行执行**：用 Future 启动、选择器处理
5. **Fire-and-Forget**：非关键持久化不阻塞主流程

---

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- `go/orchestrator/internal/workflows/strategies/research.go`：搜索 "GetVersion" 看 `workflow.GetVersion` 怎么用——理解实际的版本门控模式

### 选读深挖（2 个，按兴趣挑）

- `go/orchestrator/internal/workflows/control/handler.go`：信号处理器实现，理解暂停/恢复机制
- `go/orchestrator/internal/activities/agent.go`：看 Activity 怎么包装 LLM 调用

---

## 练习

### 练习 1：设计版本迁移

你的工作流原来是这样的：

```go
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, StepA)
    workflow.ExecuteActivity(ctx, StepB)
    return nil
}
```

现在需要：
1. 在 A 和 B 之间加一个 StepC
2. 把 StepB 改名为 StepB2（参数也变了）

写出兼容旧工作流的新代码。

### 练习 2：并行 + 超时

设计一个工作流，满足：
- 并行启动 5 个搜索任务
- 整体超时 2 分钟
- 任意 3 个完成就进入下一阶段
- 超时或失败的任务不阻塞整体

写出关键代码片段。

### 练习 3（进阶）：预算中间件

设计一个 Token 预算中间件，满足：
- 每次 Activity 调用前检查剩余预算
- Activity 完成后扣减实际消耗
- 预算耗尽时返回特定错误
- 写出伪代码

---

## 进一步阅读

- [Temporal 官方文档](https://docs.temporal.io/)：概念详解和最佳实践
- [Temporal Workflow Versioning](https://docs.temporal.io/develop/go/versioning)：版本门控详细指南
- [Temporal in Production](https://docs.temporal.io/production-deployment)：生产部署配置

---

## 下一章预告

Temporal 解决了"崩溃恢复"问题。但还有一个问题：**系统在跑，但你不知道它在干什么。**

用户说"我的任务好慢"——是哪里慢？LLM 调用慢？搜索慢？数据库慢？

下一章讲**可观测性**：如何用指标、日志、追踪三板斧，让你的 Agent 系统像玻璃一样透明。

准备好了？往下走。
