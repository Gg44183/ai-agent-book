# 第 23 章：Token 预算控制

> **Token 预算是 Agent 系统的成本防火墙——它能大幅降低成本失控的概率，但不保证每次都精准；真正的保护来自多层防线的组合。**

---

> **⏱️ 快速通道**（5 分钟掌握核心）
>
> 1. 三级预算：Task（单任务）→ Session（会话累计）→ Agent（单 Agent）
> 2. 软硬限制：软限制警告但继续，硬限制立即终止
> 3. 预算分配策略：均分 or 按复杂度加权，预留 10-20% 综合缓冲
> 4. Token 估算有误差：实际 vs 估算可能差 20%+，用 tiktoken 提高精度
> 5. 超支处理：graceful degradation 优于直接报错
>
> **10 分钟路径**：23.1-23.3 → 23.6 → Shannon Lab

---

你部署了一个 Research Agent，用户提交了"深度分析全球 AI 市场"。

第二天早上查账单——$15。一个任务，$15。

原来 Agent 并行启动了 12 个子 Agent，每个调用 10 次 LLM，用的还是最贵的模型。总计 50 万 tokens，45 分钟。用户期望花 $0.50，实际花了 30 倍。

我第一次遇到这个问题时，真的吓了一跳。那时候我们上线了一个看起来很聪明的多 Agent 研究系统，结果一个周末跑完，账单比整个月预算还高。

问题出在哪？**没有预算控制。**

Agent 很勤奋，但它不知道什么叫"够了"。你不告诉它预算上限，它就会一直挖掘、一直思考、一直调用——直到任务完成或者你的钱花光。

这一章我们来解决这个问题。

---

## 23.1 为什么需要 Token 预算？

### 失控的后果

没有预算控制会发生什么：

| 问题 | 影响 | 实际案例 |
|------|------|----------|
| 单次任务成本失控 | 一个任务 $15，预期 $0.50 | Research Agent 无限递归搜索 |
| 月度账单震惊 | $5000，预期 $500 | 批量任务没设上限 |
| 无法预测运营成本 | 财务规划困难 | 无法给客户报价 |
| 用户体验差 | 等待 45 分钟 vs 预期 5 分钟 | 用户以为系统卡死了 |
| 级联失败 | 一个任务拖垮整个系统 | 共享资源被耗尽 |

### 三级预算控制

说实话，单靠一个预算数字是不够的。就像公司的财务管理——你不能只有年度预算，你还需要季度预算、项目预算、甚至差旅预算。

Agent 系统也一样，需要多级预算控制：

| 级别 | 控制对象 | 默认值 | 用途 |
|------|----------|--------|------|
| Task | 单次任务 | 10K tokens | 防止单任务失控 |
| Session | 会话累计 | 50K tokens | 控制交互成本 |
| Agent | 单 Agent 执行 | 按任务分配 | 公平分配资源 |

为什么要三级？

- **Task 级别**：防止"一个用户发一个变态任务拖垮系统"
- **Session 级别**：防止"一个用户发很多小任务累积成大账单"
- **Agent 级别**：防止"多 Agent 协作时某个 Agent 独占预算"

这三个级别互相配合，就像公司的财务三道防线：部门预算、项目预算、人员预算。

---

## 23.2 核心数据结构

### TokenBudget

这是 Shannon 中管理预算的核心结构。注意，这是一种典型实现方式，不是唯一的设计。

```go
type TokenBudget struct {
    // Task 级别预算
    TaskBudget     int `json:"task_budget"`
    TaskTokensUsed int `json:"task_tokens_used"`

    // Session 级别预算
    SessionBudget     int `json:"session_budget"`
    SessionTokensUsed int `json:"session_tokens_used"`

    // 成本追踪
    EstimatedCostUSD float64 `json:"estimated_cost_usd"`
    ActualCostUSD    float64 `json:"actual_cost_usd"`

    // 执行策略
    HardLimit        bool    `json:"hard_limit"`        // 超出时立即终止
    WarningThreshold float64 `json:"warning_threshold"` // 预警阈值（0.8 = 80%）
    RequireApproval  bool    `json:"require_approval"`  // 超出时等待审批
}
```

三种执行模式，对应三种业务场景：

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `HardLimit` | 超出立即终止 | 成本敏感的批量任务 |
| `WarningThreshold` | 80% 时警告 | 交互式任务，给用户反应时间 |
| `RequireApproval` | 超出时暂停 | 高价值任务，需要人工决策 |

### BudgetTokenUsage

记录每次 Token 使用的详细信息。这里有个关键字段 `IdempotencyKey`——后面会详细讲为什么这个字段能救你一命。

```go
type BudgetTokenUsage struct {
    ID             string  `json:"id"`
    UserID         string  `json:"user_id"`
    SessionID      string  `json:"session_id"`
    TaskID         string  `json:"task_id"`
    AgentID        string  `json:"agent_id"`
    Model          string  `json:"model"`
    Provider       string  `json:"provider"`
    InputTokens    int     `json:"input_tokens"`
    OutputTokens   int     `json:"output_tokens"`
    TotalTokens    int     `json:"total_tokens"`
    CostUSD        float64 `json:"cost_usd"`
    IdempotencyKey string  `json:"idempotency_key,omitempty"`
}
```

---

## 23.3 BudgetManager 实现

### 核心结构

在 Shannon 中，BudgetManager 是预算控制的中枢。它管理三件事：预算检查、背压控制、熔断保护。

以下代码参考自 Shannon 的 `go/orchestrator/internal/budget/` 目录：

```go
type BudgetManager struct {
    db     *sql.DB
    logger *zap.Logger

    // 活跃会话的内存缓存
    sessionBudgets map[string]*TokenBudget
    userBudgets    map[string]*TokenBudget
    mu             sync.RWMutex

    // 默认预算
    defaultTaskBudget    int  // 10K tokens
    defaultSessionBudget int  // 50K tokens

    // 背压控制
    backpressureThreshold float64  // 80% 触发
    maxBackpressureDelay  int      // 最大延迟 ms

    // 速率限制
    rateLimiters map[string]*rate.Limiter

    // 熔断器
    circuitBreakers map[string]*CircuitBreaker

    // 幂等性追踪
    processedUsage map[string]bool
    idempotencyMu  sync.RWMutex
}
```

这个结构看起来复杂，其实核心就三层：

1. **预算层**：`sessionBudgets` + `userBudgets`
2. **保护层**：`rateLimiters` + `circuitBreakers`
3. **审计层**：`processedUsage`（防止重复计费）

### 预算检查

预算检查的核心逻辑参考自 Shannon 的实现：

```go
func (bm *BudgetManager) CheckBudget(ctx context.Context,
    userID, sessionID, taskID string, estimatedTokens int) (*BudgetCheckResult, error) {

    // Phase 1: 读锁检查已有预算
    bm.mu.RLock()
    userBudget, userExists := bm.userBudgets[userID]
    sessionBudget, sessionExists := bm.sessionBudgets[sessionID]
    bm.mu.RUnlock()

    // Phase 2: 不存在则创建默认预算（Double-check locking）
    if !userExists || !sessionExists {
        bm.mu.Lock()
        // Double-check 防止竞态
        if !sessionExists {
            sessionBudget = &TokenBudget{
                TaskBudget:       bm.defaultTaskBudget,
                SessionBudget:    bm.defaultSessionBudget,
                HardLimit:        false,
                WarningThreshold: 0.8,
            }
            bm.sessionBudgets[sessionID] = sessionBudget
        }
        bm.mu.Unlock()
    }

    result := &BudgetCheckResult{
        CanProceed:      true,
        RequireApproval: false,
        Warnings:        []string{},
    }

    // 检查 Task 级别预算
    if taskTokensUsed+estimatedTokens > taskBudget {
        if hardLimit {
            result.CanProceed = false
            result.Reason = fmt.Sprintf("Task budget exceeded: %d/%d tokens",
                taskTokensUsed+estimatedTokens, taskBudget)
        } else {
            result.RequireApproval = requireApproval
            result.Warnings = append(result.Warnings, "Task budget will be exceeded")
        }
    }

    return result, nil
}
```

关键设计点：

1. **Double-check locking**：先读锁检查，不存在再写锁创建。这是高并发场景的标准模式
2. **预警机制**：达到阈值时发事件，不是等到超限才通知
3. **灵活执行**：硬限制/软限制/需审批三种模式

---

## 23.4 背压控制

### 什么是背压？

这是我觉得预算控制里最优雅的设计。

传统做法：预算用完了？拒绝请求。用户体验很差——前一秒还能用，后一秒突然不能用了。

背压做法：预算快用完了？**逐步减慢执行速度**。给用户反应时间，也给系统喘息空间。

就像高速公路——流量大的时候，不是直接关闭入口，而是通过红绿灯控制车流速度。

```
预算使用率 → 延迟策略
─────────────────────
< 80%     → 无延迟（畅通）
80-85%    → 50ms（轻微减速）
85-90%    → 300ms（明显减速）
90-95%    → 750ms（大幅减速）
95-100%   → 1500ms（严重拥堵）
>= 100%   → 5000ms（最大延迟）
```

### 实现

```go
func (bm *BudgetManager) CheckBudgetWithBackpressure(
    ctx context.Context, userID, sessionID, taskID string, estimatedTokens int,
) (*BackpressureResult, error) {

    // 先做常规预算检查
    baseResult, err := bm.CheckBudget(ctx, userID, sessionID, taskID, estimatedTokens)
    if err != nil {
        return nil, err
    }

    result := &BackpressureResult{
        BudgetCheckResult: baseResult,
    }

    // 计算使用率（包含预估的新 tokens）
    projectedUsage := sbTokensUsed + estimatedTokens
    usagePercent := float64(projectedUsage) / float64(sbBudgetLimit)

    // 超过阈值则启用背压
    if usagePercent >= bm.backpressureThreshold {
        result.BackpressureActive = true
        result.BackpressureDelay = bm.calculateBackpressureDelay(usagePercent)
    }

    result.BudgetPressure = bm.calculatePressureLevel(usagePercent)
    return result, nil
}

func (bm *BudgetManager) calculateBackpressureDelay(usagePercent float64) int {
    if usagePercent >= 1.0 {
        return bm.maxBackpressureDelay
    } else if usagePercent >= 0.95 {
        return 1500
    } else if usagePercent >= 0.9 {
        return 750
    } else if usagePercent >= 0.85 {
        return 300
    } else if usagePercent >= 0.8 {
        return 50
    }
    return 0
}
```

### Workflow 层应用背压

这里有个关键细节：**延迟必须在 Workflow 层做，不能在 Activity 层做**。

```go
func BudgetPreflight(ctx workflow.Context, input TaskInput, estimatedTokens int) (*budget.BackpressureResult, error) {
    actx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: 30 * time.Second,
    })

    var res budget.BackpressureResult
    err := workflow.ExecuteActivity(actx,
        constants.CheckTokenBudgetWithBackpressureActivity,
        activities.BudgetCheckInput{
            UserID:          input.UserID,
            SessionID:       input.SessionID,
            TaskID:          workflow.GetInfo(ctx).WorkflowExecution.ID,
            EstimatedTokens: estimatedTokens,
        }).Get(ctx, &res)

    if err != nil {
        return nil, err
    }

    // 关键：在 Workflow 层 Sleep，不在 Activity 层！
    if res.BackpressureActive && res.BackpressureDelay > 0 {
        logger.Info("Applying budget backpressure delay",
            "delay_ms", res.BackpressureDelay,
            "pressure_level", res.BudgetPressure,
        )
        if err := workflow.Sleep(ctx, time.Duration(res.BackpressureDelay)*time.Millisecond); err != nil {
            return nil, err
        }
    }

    return &res, nil
}
```

为什么必须在 Workflow 层 Sleep？

| 在哪里 Sleep | 后果 |
|-------------|------|
| Activity 层 `time.Sleep` | 阻塞 Worker 线程，Worker 数量有限，很快耗尽 |
| Workflow 层 `workflow.Sleep` | 确定性的，支持 Temporal 重放，可被取消 |

这是我见过最常见的坑之一。很多人第一反应是在 Activity 里 Sleep，结果 Worker 全被阻塞，系统瘫痪。

---

## 23.5 熔断器模式

### 设计

背压是"减速"，熔断是"紧急刹车"。

当某用户连续触发预算超限，说明要么用户在滥用，要么系统有 bug。无论哪种情况，继续让请求进来都不是好主意。

```go
type CircuitBreaker struct {
    failureCount    int32
    lastFailureTime time.Time
    state           string // "closed", "open", "half-open"
    config          CircuitBreakerConfig
    successCount    int32
    mu              sync.RWMutex
}

type CircuitBreakerConfig struct {
    FailureThreshold int           // 触发熔断的失败次数
    ResetTimeout     time.Duration // 熔断后多久尝试恢复
    HalfOpenRequests int           // 半开状态允许的测试请求数
}
```

### 状态转换

![熔断器状态机](assets/circuit-breaker-states.svg)

这个模式来自微服务领域的经典设计。Netflix 的 Hystrix 让它变得流行，现在几乎是分布式系统的标配。

### 实现

```go
func (bm *BudgetManager) CheckBudgetWithCircuitBreaker(
    ctx context.Context, userID, sessionID, taskID string, estimatedTokens int,
) (*BackpressureResult, error) {

    // 先检查熔断器状态
    state := bm.GetCircuitState(userID)
    if state == "open" {
        return &BackpressureResult{
            BudgetCheckResult: &BudgetCheckResult{
                CanProceed: false,
                Reason:     "Circuit breaker is open due to repeated failures",
            },
            CircuitBreakerOpen: true,
        }, nil
    }

    // 半开状态下只允许有限请求
    if state == "half-open" {
        cb := bm.circuitBreakers[userID]
        if int(atomic.LoadInt32(&cb.successCount)) >= cb.config.HalfOpenRequests {
            return &BackpressureResult{
                BudgetCheckResult: &BudgetCheckResult{
                    CanProceed: false,
                    Reason:     "Circuit breaker in half-open state, test quota exceeded",
                },
                CircuitBreakerOpen: true,
            }, nil
        }
    }

    return bm.CheckBudgetWithBackpressure(ctx, userID, sessionID, taskID, estimatedTokens)
}
```

---

## 23.6 成本计算

### 分离输入/输出计费

这是我踩过的一个大坑。

最开始我们用的是"总 Token 数 × 单价"来计算成本。结果发现账单和实际差了 30%。

原因：不同模型的输入和输出价格差异很大。

| 模型 | Input/1K | Output/1K | 比例 |
|------|----------|-----------|------|
| GPT-4 | $0.03 | $0.06 | 1:2 |
| Claude Sonnet | $0.003 | $0.015 | 1:5 |

输出 tokens 通常比输入贵 2-5 倍。如果你合并计费，要么低估成本（用输入价格），要么高估成本（用输出价格）。

> LLM 定价变化频繁，以上仅为示意。具体价格请查阅各服务商官方定价页面。

### 定价配置

Shannon 使用 YAML 配置管理定价，这样价格变化时只需要更新配置：

```yaml
# config/models.yaml
pricing:
  defaults:
    combined_per_1k: 0.005

  models:
    openai:
      gpt-4o:
        input_per_1k: 0.0025
        output_per_1k: 0.010
      gpt-4o-mini:
        input_per_1k: 0.00015
        output_per_1k: 0.0006
    anthropic:
      claude-sonnet-4:
        input_per_1k: 0.003
        output_per_1k: 0.015
```

### 计算逻辑

Shannon 的定价计算参考 `go/orchestrator/internal/pricing/pricing.go`：

```go
func CostForSplit(model string, inputTokens, outputTokens int) float64 {
    if inputTokens < 0 {
        inputTokens = 0
    }
    if outputTokens < 0 {
        outputTokens = 0
    }

    cfg := get()
    for _, models := range cfg.Pricing.Models {
        if m, ok := models[model]; ok {
            in := m.InputPer1K
            out := m.OutputPer1K
            if in > 0 && out > 0 {
                return (float64(inputTokens)/1000.0)*in +
                       (float64(outputTokens)/1000.0)*out
            }
        }
    }
    // 回退到默认定价
    return float64(inputTokens+outputTokens) * DefaultPerToken()
}
```

---

## 23.7 幂等性：防止重试重复计费

这是我觉得 Token 预算里最容易被忽视，但最致命的问题。

### 问题场景

Temporal 的一个核心特性是自动重试。Activity 失败了？自动重试。网络抖动？自动重试。

但是，如果你的"记录 Token 使用"Activity 执行成功了，但在返回结果时网络断了，Temporal 会认为它失败了，然后重试。

结果：同一次 LLM 调用，被记录了两次。账单翻倍。

### 解决方案

Shannon 使用 IdempotencyKey 来防止重复计费，参考 `go/orchestrator/internal/activities/budget.go`：

```go
func (b *BudgetActivities) RecordTokenUsage(ctx context.Context, input TokenUsageInput) error {
    // 获取 Activity 信息用于生成幂等键
    info := activity.GetInfo(ctx)

    // 生成幂等键：WorkflowID + ActivityID + 尝试次数
    // 这保证同一次 Activity 的重试会生成相同的 Key
    idempotencyKey := fmt.Sprintf("%s-%s-%d",
        info.WorkflowExecution.ID, info.ActivityID, info.Attempt)

    usage := &budget.BudgetTokenUsage{
        UserID:         input.UserID,
        // ... 其他字段 ...
        IdempotencyKey: idempotencyKey,
    }

    err := b.budgetManager.RecordUsage(ctx, usage)
    // RecordUsage 内部会检查 IdempotencyKey 是否已存在，存在则跳过
    return err
}
```

关键点：

- `IdempotencyKey = WorkflowID + ActivityID + Attempt`
- Temporal 重试时生成相同 Key
- `RecordUsage` 检测到重复 Key 时跳过

---

## 23.8 预算感知的 Agent 执行

把前面讲的所有组件串起来，看看完整的预算感知执行流程。

以下参考 Shannon 的 `go/orchestrator/internal/activities/budget.go` 中的 `ExecuteAgentWithBudget` 函数：

```go
func (b *BudgetActivities) ExecuteAgentWithBudget(ctx context.Context,
    input BudgetedAgentInput) (*AgentExecutionResult, error) {

    // 1. 执行前检查预算
    budgetCheck, err := b.budgetManager.CheckBudget(
        ctx,
        input.UserID,
        input.AgentInput.SessionID,
        input.TaskID,
        input.MaxTokens,
    )
    if err != nil {
        return nil, fmt.Errorf("budget check failed: %w", err)
    }

    if !budgetCheck.CanProceed {
        return &AgentExecutionResult{
            AgentID: input.AgentInput.AgentID,
            Success: false,
            Error:   fmt.Sprintf("Budget exceeded: %s", budgetCheck.Reason),
        }, nil
    }

    // 2. 执行 Agent
    input.AgentInput.Context["max_tokens"] = input.MaxTokens
    input.AgentInput.Context["model_tier"] = input.ModelTier
    result, err := executeAgentCore(ctx, input.AgentInput, logger)
    if err != nil {
        return nil, fmt.Errorf("agent execution failed: %w", err)
    }

    // 3. 生成幂等键，防止重试重复计费
    info := activity.GetInfo(ctx)
    idempotencyKey := fmt.Sprintf("%s-%s-%d",
        info.WorkflowExecution.ID, info.ActivityID, info.Attempt)

    // 4. 记录实际使用量
    err = b.budgetManager.RecordUsage(ctx, &budget.BudgetTokenUsage{
        UserID:         input.UserID,
        SessionID:      input.AgentInput.SessionID,
        TaskID:         input.TaskID,
        AgentID:        input.AgentInput.AgentID,
        Model:          result.ModelUsed,
        Provider:       result.Provider,
        InputTokens:    result.InputTokens,
        OutputTokens:   result.OutputTokens,
        IdempotencyKey: idempotencyKey,
    })

    return &result, nil
}
```

这个流程覆盖了预算控制的完整生命周期：检查 → 执行 → 记录。

---

## 23.9 双路径记录：避免重复

Shannon 的一个重要设计是"双路径记录"——根据是否启用预算，决定在哪里记录 Token 使用。

根据 Shannon 的 `docs/token-budget-tracking.md` 文档：

![Token Budget 工作流](assets/token-budget-workflow.svg)

为什么这么设计？因为如果不区分，**会出现重复记录**：

| 场景 | Activity 记录 | Pattern 记录 | 结果 |
|------|--------------|-------------|------|
| 预算启用 | 是 | 是 | 重复！ |
| 预算启用 | 是 | 否（Guard） | 正确 |
| 预算禁用 | 否 | 是 | 正确 |

Shannon 的解决方案是在 Pattern 层加 Guard：

```go
// Pattern 层的记录逻辑
if budgetPerAgent <= 0 {  // 只有预算禁用时才记录
    _ = workflow.ExecuteActivity(ctx, constants.RecordTokenUsageActivity, ...)
}
```

---

## 23.10 配置与监控

### 配置

```yaml
# config/shannon.yaml
session:
  token_budget_per_task: 10000
  token_budget_per_session: 50000

budget:
  backpressure:
    threshold: 0.8
    max_delay_ms: 5000

  circuit_breaker:
    failure_threshold: 5
    reset_timeout: "5m"
    half_open_requests: 3
```

### 监控指标

| 指标 | 类型 | 说明 |
|------|------|------|
| `budget_tokens_used` | Counter | 已使用 tokens |
| `budget_cost_usd` | Counter | 累计成本 |
| `budget_exceeded_total` | Counter | 超限次数 |
| `backpressure_delay_seconds` | Histogram | 背压延迟分布 |
| `circuit_breaker_state` | Gauge | 熔断器状态（0=closed, 1=half-open, 2=open） |

### 告警规则

```yaml
- alert: BudgetExceededRate
  expr: rate(budget_exceeded_total[5m]) > 0.1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High budget exceeded rate"
    description: "More than 10% of requests are exceeding budget"

- alert: CircuitBreakerOpen
  expr: circuit_breaker_state == 2  # 2 = open
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Circuit breaker open for user {{ $labels.user_id }}"
```

---

## 23.11 常见的坑

### 坑 1：遗忘幂等性

```go
// 错误：每次重试都记录
err = bm.RecordUsage(ctx, &budget.BudgetTokenUsage{
    TaskID: taskID,
    // 没有 IdempotencyKey!
})

// 正确：使用 Activity 信息生成幂等键
info := activity.GetInfo(ctx)
idempotencyKey := fmt.Sprintf("%s-%s-%d",
    info.WorkflowExecution.ID, info.ActivityID, info.Attempt)
```

这个坑我见过太多次了。一个系统运行了三个月都没问题，直到有一天网络抖动频繁，突然账单翻了两倍。

### 坑 2：Activity 层 Sleep

```go
// 错误：阻塞 Worker 线程
func (b *BudgetActivities) CheckWithBackpressure(...) {
    if delay > 0 {
        time.Sleep(time.Duration(delay) * time.Millisecond) // 阻塞!
    }
}

// 正确：在 Workflow 层使用 workflow.Sleep
if res.BackpressureDelay > 0 {
    workflow.Sleep(ctx, time.Duration(res.BackpressureDelay)*time.Millisecond)
}
```

Worker 线程是有限的。如果你在 Activity 里 Sleep，每个被延迟的请求都会占用一个 Worker 线程。当预算压力大的时候，恰恰是背压频繁触发的时候，恰恰是 Worker 最容易耗尽的时候。

### 坑 3：并发更新预算

```go
// 错误：读后写存在竞态
sessionBudget := bm.sessionBudgets[sessionID]
sessionBudget.TaskTokensUsed += tokens  // 不安全！

// 正确：持锁更新
bm.mu.Lock()
if sessionBudget, ok := bm.sessionBudgets[sessionID]; ok {
    sessionBudget.TaskTokensUsed += tokens
}
bm.mu.Unlock()
```

### 坑 4：只算总量不算比例

```go
// 错误：只看用了多少
if sessionBudget.TaskTokensUsed > 10000 {
    // 超预算
}

// 正确：算比例，支持动态预算
usagePercent := float64(sessionBudget.TaskTokensUsed) / float64(sessionBudget.TaskBudget)
if usagePercent >= 0.8 {
    // 接近预算
}
```

用绝对值判断的问题是：不同用户可能有不同预算。VIP 用户 100K tokens，普通用户 10K tokens。如果你用绝对值判断，VIP 用户到 10K 就被限制了。

---

## 23.12 框架对比

Token 预算控制不是 Shannon 独有的概念。其他框架怎么做的？

| 框架 | 预算控制方式 | 背压支持 | 熔断器 | 幂等性 |
|------|-------------|---------|--------|--------|
| **Shannon** | 三级预算 | 内置 | 内置 | IdempotencyKey |
| **LangChain** | Callback 手动实现 | 无 | 无 | 无 |
| **LangGraph** | 可通过 State 实现 | 需自己写 | 需自己写 | 需自己写 |
| **CrewAI** | max_rpm/max_tokens | 有限 | 无 | 无 |
| **AutoGen** | 无内置 | 无 | 无 | 无 |

说实话，这是 Shannon 做得比较好的地方。大多数框架都把预算控制当作"用户自己解决的事情"，但在生产环境里，这恰恰是最容易出事的地方。

---

## 这章说了什么

1. **三级预算**：Task/Session/Agent 分层控制，像公司财务的三道防线
2. **背压控制**：接近上限时逐步减速而非直接拒绝，给用户和系统喘息空间
3. **熔断器**：连续超限时暂时拒绝，防止级联故障
4. **幂等性**：IdempotencyKey 防止 Temporal 重试导致重复计费
5. **分离计费**：输入/输出 tokens 分别计价，估算更准确

---

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- `go/orchestrator/internal/activities/budget.go`：看 `ExecuteAgentWithBudget` 函数的预算检查 → 执行 → 记录流程，理解 `idempotencyKey` 生成逻辑和为什么不在 Activity 层 Sleep

### 选读深挖（2 个，按兴趣挑）

- `docs/token-budget-tracking.md`：理解双路径记录的设计思路，搜索 "Guard Pattern" 看各个 Workflow Pattern 如何避免重复记录
- `go/orchestrator/internal/pricing/pricing.go`：看 `CostForSplit` 函数，理解为什么要分离 input/output 计费以及回退逻辑

---

## 练习

### 练习 1：设计预算超限的用户提示

用户任务快要超预算了（80%），你需要设计一条通知消息。要求包含：
- 当前使用量和预算上限
- 还能做什么/不能做什么
- 用户可以采取的行动

### 练习 2：源码理解

读 Shannon 的 `go/orchestrator/internal/activities/budget.go`：
1. 找到 `ExecuteAgentWithBudget` 函数，解释它为什么要在执行后才记录使用量（而不是执行前）
2. 如果 Agent 执行失败了，Token 使用会被记录吗？这合理吗？

### 练习 3（进阶）：设计"动态预算调整"功能

场景：VIP 用户在月初预算宽松，月底预算紧张。设计一个"动态预算"机制：
- 写出核心数据结构
- 描述调整预算的触发条件
- 考虑：如果正在执行的任务突然被"降低预算"了怎么办？

---

## 进一步阅读

- **Temporal 重试机制**：[Temporal Retry Policies](https://docs.temporal.io/retry-policies) - 理解为什么需要幂等性
- **Circuit Breaker 模式**：[微服务熔断模式](https://martinfowler.com/bliki/CircuitBreaker.html) - Martin Fowler 的经典文章
- **各 LLM 服务商定价页面**：价格经常变化，建议定期检查更新配置

---

## 下一章预告

Token 预算解决了"花多少钱"的问题。但还有一个问题：**谁能做什么？**

一个 Agent 能调用什么工具？能访问什么数据？能执行什么操作？这些不是预算问题，是权限问题。

下一章我们来聊 **OPA 策略治理**——用 Open Policy Agent 实现细粒度的访问控制。

想象一下：用户说"帮我删掉所有测试数据"。Agent 应该先检查：这个用户有删除权限吗？这个操作需要审批吗？这个时间段允许执行吗？

这些问题，OPA 来回答。

下一章见。
