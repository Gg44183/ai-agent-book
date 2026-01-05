# 第 6 章：Hooks 与事件系统

> **Hooks 是 Agent 的神经系统——让你在不改核心代码的情况下，观测执行状态、插入自定义逻辑。但 Hook 太多或太慢，会把整个 Agent 拖垮。**

---

你的 Agent 在生产环境跑着，突然用户来问：

> "它现在在干嘛？怎么这么慢？"

你打开日志，发现全是乱糟糟的 print 语句。根本看不出 Agent 执行到哪一步了。

这就是没有 Hooks 系统的痛苦。

---

## 6.1 Hooks 解决什么问题？

三个字：**看、管、扩**。

1. **看**（可观测性）：Agent 在做什么？执行到哪一步了？
2. **管**（可控性）：能不能在关键节点暂停，让人确认后再继续？
3. **扩**（可扩展性）：能不能在不改核心代码的情况下加功能？

没有 Hooks，你需要在每个执行点手动加日志、用轮询检查状态、改核心代码加功能。

有了 Hooks：

```
Agent执行流程                    Hook触发点
──────────────────────────────────────────────
开始工作流       ────────▶     WORKFLOW_STARTED
  ↓
Agent开始思考    ────────▶     AGENT_THINKING
  ↓
调用工具        ────────▶     TOOL_INVOKED
  ↓
获得结果        ────────▶     TOOL_OBSERVATION
  ↓
需要审批？      ────────▶     APPROVAL_REQUESTED
  ↓                           (等待人工决策)
继续执行        ────────▶     APPROVAL_DECISION
  ↓
Agent完成       ────────▶     AGENT_COMPLETED
```

你可以订阅任何一个事件点，做自己想做的事——记日志、发通知、暂停流程、人工审批。

---

## 6.2 Shannon 的事件类型体系

Shannon 定义了一套完整的事件类型。我把它分几类：

### 工作流生命周期

```go
StreamEventWorkflowStarted   = "WORKFLOW_STARTED"
StreamEventWorkflowCompleted = "WORKFLOW_COMPLETED"
StreamEventAgentStarted      = "AGENT_STARTED"
StreamEventAgentCompleted    = "AGENT_COMPLETED"
```

### 执行状态

```go
StreamEventToolInvoked    = "TOOL_INVOKED"     // 工具调用
StreamEventToolObs        = "TOOL_OBSERVATION" // 工具返回
StreamEventAgentThinking  = "AGENT_THINKING"   // 思考中
StreamEventErrorOccurred  = "ERROR_OCCURRED"   // 出错了
```

### 工作流控制

```go
StreamEventWorkflowPaused     = "WORKFLOW_PAUSED"     // 暂停
StreamEventWorkflowResumed    = "WORKFLOW_RESUMED"    // 恢复
StreamEventWorkflowCancelled  = "WORKFLOW_CANCELLED"  // 取消
StreamEventApprovalRequested  = "APPROVAL_REQUESTED"  // 请求审批
StreamEventApprovalDecision   = "APPROVAL_DECISION"   // 审批结果
```

### LLM 事件

```go
StreamEventLLMPrompt  = "LLM_PROMPT"  // 发给 LLM 的内容
StreamEventLLMPartial = "LLM_PARTIAL" // LLM 增量输出
StreamEventLLMOutput  = "LLM_OUTPUT"  // LLM 最终输出
```

为什么要分这么细？

因为不同场景需要不同的事件。前端展示进度用 `AGENT_THINKING`、`TOOL_INVOKED`；调试 LLM 用 `LLM_PROMPT`、`LLM_OUTPUT`；做审计用 `WORKFLOW_COMPLETED`。

---

## 6.3 事件怎么发出去？

每个事件长这样：

```go
type EmitTaskUpdateInput struct {
    WorkflowID string                 // 关联到哪个工作流
    EventType  StreamEventType        // 什么类型的事件
    AgentID    string                 // 哪个 Agent 发的
    Message    string                 // 人类可读的描述
    Timestamp  time.Time              // 什么时候
    Payload    map[string]interface{} // 额外数据
}
```

发送逻辑：

```go
func EmitTaskUpdate(ctx context.Context, in EmitTaskUpdateInput) error {
    // 1. 写日志
    logger.Info("streaming event",
        "workflow_id", in.WorkflowID,
        "type", string(in.EventType),
        "message", in.Message,
    )

    // 2. 发布到流
    streaming.Get().Publish(in.WorkflowID, streaming.Event{
        WorkflowID: in.WorkflowID,
        Type:       string(in.EventType),
        Message:    in.Message,
        Timestamp:  in.Timestamp,
    })

    return nil
}
```

注意这里是**双重发布**：同时写日志和发布到流。日志用来调试，流用来实时推送给前端。

---

## 6.4 流式事件管理器

Shannon 用 Redis Streams 做事件传输层。为什么用 Redis Streams？

1. **高吞吐**：每秒能处理几十万条消息
2. **持久化**：消息不会丢
3. **消费组**：多个消费者可以分担负载

Manager 的结构：

```go
type Manager struct {
    redis       *redis.Client
    dbClient    *db.Client     // PostgreSQL
    subscribers map[string]map[chan Event]*subscription
    capacity    int            // 容量限制
}
```

### 发布事件

```go
func (m *Manager) Publish(workflowID string, evt Event) {
    if m.redis != nil {
        // 1. 递增序列号（保证顺序）
        seq, _ := m.redis.Incr(ctx, m.seqKey(workflowID)).Result()
        evt.Seq = uint64(seq)

        // 2. 写入 Redis Stream，自动裁剪旧事件
        m.redis.XAdd(ctx, &redis.XAddArgs{
            Stream: m.streamKey(workflowID),
            MaxLen: int64(m.capacity),  // 容量限制
            Approx: true,
            Values: eventData,
        })

        // 3. 设置 TTL（24小时后自动清理）
        m.redis.Expire(ctx, streamKey, 24*time.Hour)
    }

    // 4. 重要事件持久化到 PostgreSQL
    if shouldPersistEvent(evt.Type) {
        select {
        case m.persistCh <- eventLog:
        default:
            // 队列满了就丢掉，不阻塞主流程
        }
    }
}
```

几个关键设计：

- **序列号**：确保事件有序
- **容量限制**：防止 Stream 无限增长
- **TTL**：24 小时后自动清理
- **非阻塞持久化**：队列满了就丢掉，不拖慢主流程

### 哪些事件要持久化？

```go
func shouldPersistEvent(eventType string) bool {
    switch eventType {
    // 需要持久化：重要事件
    case "WORKFLOW_COMPLETED", "WORKFLOW_FAILED",
         "TOOL_INVOKED", "TOOL_OBSERVATION",
         "LLM_OUTPUT", "BUDGET_THRESHOLD":
        return true

    // 不持久化：临时事件
    case "LLM_PARTIAL", "HEARTBEAT", "PING":
        return false

    // 默认持久化（保守策略）
    default:
        return true
    }
}
```

`LLM_PARTIAL` 是增量输出，一秒可能几十条，持久化没意义。`WORKFLOW_COMPLETED` 是最终结果，必须存。

---

## 6.5 工作流控制：暂停/恢复/取消

这是 Hooks 系统最强大的功能之一：**运行时控制工作流**。

Shannon 用 Temporal Signal 实现这个。Signal 是 Temporal 的一个特性，可以给正在运行的工作流发消息。

### 信号处理器

```go
type SignalHandler struct {
    State      *WorkflowControlState
    WorkflowID string
}

func (h *SignalHandler) Setup(ctx workflow.Context) {
    h.State = &WorkflowControlState{}

    // 注册三个信号通道
    pauseCh := workflow.GetSignalChannel(ctx, SignalPause)
    resumeCh := workflow.GetSignalChannel(ctx, SignalResume)
    cancelCh := workflow.GetSignalChannel(ctx, SignalCancel)

    // 后台协程监听信号
    workflow.Go(ctx, func(gCtx workflow.Context) {
        for {
            sel := workflow.NewSelector(gCtx)

            sel.AddReceive(pauseCh, func(c workflow.ReceiveChannel, more bool) {
                var req PauseRequest
                c.Receive(gCtx, &req)
                h.handlePause(gCtx, req)
            })

            sel.AddReceive(resumeCh, func(c workflow.ReceiveChannel, more bool) {
                var req ResumeRequest
                c.Receive(gCtx, &req)
                h.handleResume(gCtx, req)
            })

            sel.AddReceive(cancelCh, func(c workflow.ReceiveChannel, more bool) {
                var req CancelRequest
                c.Receive(gCtx, &req)
                h.handleCancel(gCtx, req)
            })

            sel.Select(gCtx)
        }
    })
}
```

### 暂停和恢复

```go
func (h *SignalHandler) handlePause(ctx workflow.Context, req PauseRequest) {
    if h.State.IsPaused {
        return  // 已经暂停了
    }

    h.State.IsPaused = true
    h.State.PauseReason = req.Reason

    // 发送事件通知前端
    emitEvent(ctx, StreamEventWorkflowPausing, "Workflow pausing: "+req.Reason)

    // 传播给所有子工作流
    h.propagateSignalToChildren(ctx, SignalPause, req)
}
```

### 检查点机制

工作流不能在任意位置暂停，只能在"检查点"暂停。这是 Temporal 的限制，也是个合理的设计。

```go
func (h *SignalHandler) CheckPausePoint(ctx workflow.Context, checkpoint string) error {
    // 让出执行权，确保信号被处理
    _ = workflow.Sleep(ctx, 0)

    // 检查是否被取消
    if h.State.IsCancelled {
        return temporal.NewCanceledError("workflow cancelled")
    }

    // 检查是否被暂停
    if h.State.IsPaused {
        emitEvent(ctx, StreamEventWorkflowPaused, "Paused at: "+checkpoint)

        // 阻塞等待恢复（不是轮询！）
        _ = workflow.Await(ctx, func() bool {
            return !h.State.IsPaused || h.State.IsCancelled
        })
    }

    return nil
}
```

使用方式：

```go
func MyWorkflow(ctx workflow.Context, input Input) error {
    handler := &control.SignalHandler{...}
    handler.Setup(ctx)

    // 检查点 1
    if err := handler.CheckPausePoint(ctx, "before_research"); err != nil {
        return err
    }
    doResearch(ctx)

    // 检查点 2
    if err := handler.CheckPausePoint(ctx, "before_synthesis"); err != nil {
        return err
    }
    doSynthesis(ctx)

    return nil
}
```

在每个关键步骤前插入检查点，用户就可以在这些位置暂停工作流。

---

## 6.6 人工审批 Hook

对于高风险操作，你可能希望 Agent 先问问人。

### 审批策略

```go
type ApprovalPolicy struct {
    ComplexityThreshold float64  // 复杂度超过这个值就要审批
    TokenBudgetExceeded bool     // Token 超预算要审批
    RequireForTools     []string // 这些工具需要审批
}

func EvaluateApprovalPolicy(policy ApprovalPolicy, context map[string]interface{}) (bool, string) {
    // 检查复杂度
    if complexity := context["complexity_score"].(float64); complexity >= policy.ComplexityThreshold {
        return true, fmt.Sprintf("Complexity %.2f exceeds threshold", complexity)
    }

    // 检查危险工具
    if tools := context["tools_to_use"].([]string); containsAny(tools, policy.RequireForTools) {
        return true, "Dangerous tool requires approval"
    }

    return false, ""
}
```

### 请求审批

```go
func RequestAndWaitApproval(ctx workflow.Context, input TaskInput, reason string) (*HumanApprovalResult, error) {
    // 1. 发送审批请求
    var approval HumanApprovalResult
    workflow.ExecuteActivity(ctx, "RequestApproval", HumanApprovalInput{
        SessionID:      input.SessionID,
        WorkflowID:     workflow.GetInfo(ctx).WorkflowExecution.ID,
        Query:          input.Query,
        Reason:         reason,
    }).Get(ctx, &approval)

    // 2. 发送事件通知前端
    emitEvent(ctx, StreamEventApprovalRequested, "Approval requested: "+reason)

    // 3. 等待人工决策（最多 60 分钟）
    sigName := "human-approval-" + approval.ApprovalID
    ch := workflow.GetSignalChannel(ctx, sigName)
    timer := workflow.NewTimer(ctx, 60*time.Minute)

    var result HumanApprovalResult
    sel := workflow.NewSelector(ctx)
    sel.AddReceive(ch, func(c workflow.ReceiveChannel, more bool) {
        c.Receive(ctx, &result)
    })
    sel.AddFuture(timer, func(f workflow.Future) {
        result = HumanApprovalResult{Approved: false, Feedback: "timeout"}
    })
    sel.Select(ctx)

    // 4. 发送结果事件
    decision := "denied"
    if result.Approved {
        decision = "approved"
    }
    emitEvent(ctx, StreamEventApprovalDecision, "Approval "+decision)

    return &result, nil
}
```

使用方式：

```go
func ResearchWorkflow(ctx workflow.Context, input TaskInput) error {
    // 评估是否需要审批
    needsApproval, reason := EvaluateApprovalPolicy(policy, context)

    if needsApproval {
        approval, err := RequestAndWaitApproval(ctx, input, reason)
        if err != nil {
            return err
        }

        if !approval.Approved {
            return fmt.Errorf("rejected: %s", approval.Feedback)
        }
    }

    // 继续执行...
    return executeResearch(ctx, input)
}
```

---

## 6.7 实战：Token 消耗监控 Hook

来写一个实用的 Hook：监控 Token 消耗，快超预算时发警告。

```go
type TokenUsageHook struct {
    WarningThreshold  float64 // 80%
    CriticalThreshold float64 // 95%
    TotalBudget       int
    CurrentUsage      int
}

func (h *TokenUsageHook) OnTokensUsed(ctx workflow.Context, tokensUsed int) error {
    h.CurrentUsage += tokensUsed
    ratio := float64(h.CurrentUsage) / float64(h.TotalBudget)

    if ratio >= h.CriticalThreshold {
        return emitEvent(ctx, StreamEventBudgetThreshold,
            fmt.Sprintf("CRITICAL: Token budget at %.0f%% (%d/%d)",
                ratio*100, h.CurrentUsage, h.TotalBudget),
            map[string]interface{}{"level": "critical", "ratio": ratio},
        )
    }

    if ratio >= h.WarningThreshold {
        return emitEvent(ctx, StreamEventBudgetThreshold,
            fmt.Sprintf("WARNING: Token budget at %.0f%%", ratio*100),
            map[string]interface{}{"level": "warning", "ratio": ratio},
        )
    }

    return nil
}
```

这个 Hook 在每次 LLM 调用后触发，检查 Token 消耗是否接近预算。80% 时发警告，95% 时发严重警告。

---

## 6.8 常见的坑

### 坑 1：阻塞式 Hook

Hook 执行时间太长，拖慢主流程。

```go
// 阻塞式 - 会拖慢主流程
result, err := publishEvent(ctx, event)
if err != nil {
    return err  // 失败就停止
}

// 非阻塞式 - 推荐
select {
case eventCh <- event:
    // 成功
default:
    logger.Warn("Event dropped - channel full")
}
```

### 坑 2：事件风暴

大量低价值事件淹没重要事件。

解决：事件分级，选择性持久化。`LLM_PARTIAL` 不存，`WORKFLOW_COMPLETED` 必存。

### 坑 3：状态不一致

Signal 处理和状态检查之间有竞态条件。

解决：在检查状态前用 `workflow.Sleep(ctx, 0)` 让出执行权，确保 Signal 被处理。

### 坑 4：子工作流信号丢失

暂停父工作流时，子工作流继续跑。

解决：信号传播机制。

```go
func (h *SignalHandler) handlePause(ctx workflow.Context, req PauseRequest) {
    h.State.IsPaused = true
    // 传播给所有子工作流
    h.propagateSignalToChildren(ctx, SignalPause, req)
}
```

---

## 6.9 其他框架怎么做？

| 框架 | Hook 机制 | 特点 |
|------|----------|------|
| **LangChain** | Callbacks | `on_llm_start`, `on_tool_end` 等回调 |
| **LangGraph** | Node hooks | 在图节点进入/退出时触发 |
| **CrewAI** | Step callback | 每个 Agent 步骤后回调 |
| **Claude Code** | Hooks 目录 | 用独立脚本实现，通过 stdin/stdout 通信 |
| **Shannon** | Temporal Signal + Redis Streams | 支持暂停/恢复/取消 |

差异主要在于：

| 维度 | 简单 Callbacks | Temporal 信号模式 |
|------|---------------|------------------|
| **状态管理** | 内存中 | 持久化 |
| **故障恢复** | 丢失 | 可恢复 |
| **暂停/恢复** | 很难实现 | 原生支持 |
| **复杂度** | 低 | 高 |

如果只需要日志和简单通知，Callbacks 够用。如果需要长时间运行、可中断、可恢复的工作流，Temporal 信号模式更合适。

---

## 6.10 Claude Code 的 Hooks：一种轻量实现

Claude Code 有一套简单但实用的 Hooks 机制，值得参考。

它把 Hooks 定义在 `.claude/hooks/` 目录下，用独立脚本实现：

```bash
.claude/
└── hooks/
    ├── pre-tool-use.sh     # 工具调用前
    ├── post-tool-use.sh    # 工具调用后
    ├── notification.sh     # 通知用户
    └── stop.sh             # Agent 停止时
```

调用方式很简单：通过 stdin 传入事件数据，脚本处理后返回。

```bash
# pre-tool-use.sh 示例
#!/bin/bash
# 读取 JSON 输入
read -r input
tool_name=$(echo "$input" | jq -r '.tool')

# 记录日志
echo "$(date): Tool called: $tool_name" >> ~/.claude/hooks.log

# 如果是危险工具，阻止执行
if [[ "$tool_name" == "shell_execute" ]]; then
    echo '{"action": "block", "reason": "Shell execution not allowed"}'
    exit 1
fi

# 允许执行
echo '{"action": "allow"}'
```

这种设计的优点：

| 优点 | 说明 |
|------|------|
| **语言无关** | 任何能写脚本的语言都行 |
| **隔离性** | Hook 是独立进程，崩溃不影响主程序 |
| **简单** | 不需要学框架，会写脚本就行 |

缺点是性能开销（每次调用要启动进程）和功能受限（不能持久化状态）。

---

## 本章要点回顾

1. **Hooks 解决三个问题**：看（可观测）、管（可控制）、扩（可扩展）
2. **事件分级很重要**——不是所有事件都要持久化，`LLM_PARTIAL` 不存，`WORKFLOW_COMPLETED` 必存
3. **暂停/恢复用 Temporal Signal**——不是轮询，是真正的阻塞等待
4. **人工审批是安全护栏**——基于策略触发，支持超时自动拒绝
5. **Hook 要非阻塞**——队列满了就丢，不能拖慢主流程

---

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- [`streaming/manager.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/streaming/manager.go)：看 `Publish` 方法和 `shouldPersistEvent` 函数，理解事件发布和分级逻辑

### 选读深挖（2 个，按兴趣挑）

- [`control/handler.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/control/handler.go)：看 `SignalHandler` 怎么处理暂停/恢复/取消
- [`control/signals.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/control/signals.go)：看信号类型定义

---

## 练习

### 练习 1：设计事件分级

假设你有以下事件类型，哪些应该持久化？为什么？

1. `USER_MESSAGE_RECEIVED`
2. `LLM_TOKEN_GENERATED`
3. `TOOL_EXECUTION_STARTED`
4. `TOOL_EXECUTION_COMPLETED`
5. `AGENT_ERROR`
6. `HEARTBEAT`

### 练习 2：实现一个简单的 Hook

用你熟悉的语言，实现一个"工具调用日志"Hook：

1. 每次工具调用时记录：时间、工具名、参数摘要
2. 写入文件（JSON Lines 格式）
3. 考虑：这个 Hook 应该是同步还是异步的？为什么？

### 练习 3（进阶）：设计审批策略

为一个"研究助手"Agent 设计审批策略：

1. 什么情况下需要人工审批？
2. 审批超时应该怎么处理（自动批准 vs 自动拒绝）？
3. 如何避免频繁打扰用户？

---

## 延伸阅读

- [Shannon Streaming Manager](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/streaming/manager.go) - 代码实现
- [Temporal Signals Documentation](https://docs.temporal.io/workflows#signal) - Temporal 信号机制
- [Redis Streams Documentation](https://redis.io/docs/data-types/streams/) - Redis Streams 入门
- [Claude Code Hooks](https://claude.ai/code) - Claude Code 的 Hooks 文档

---

## 下一章预告

Part 2"工具与扩展"到此结束。

我们学了四件事：
- **工具调用**：让 Agent 能"动手"
- **MCP 协议**：工具的"USB 接口"
- **Skills**：打包角色配置
- **Hooks**：观测和控制执行

接下来进入 Part 3"上下文与记忆"。

Agent 执行过程中会产生大量信息，但 LLM 的上下文窗口是有限的。怎么在有限的空间里塞进最有用的信息？

下一章我们来聊**上下文窗口管理**。

