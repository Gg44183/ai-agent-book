# 第 29 章：Background Agents

> **Background Agent 让任务脱离用户会话独立运行——定时调度、持续监控、故障自动恢复。这是 Agent 从"工具"进化到"员工"的关键一步，但也意味着失去实时人工监督，必须在设计时预埋足够的安全和预算控制。**

---

> **⏱️ 快速通道**（5 分钟掌握核心）
>
> 1. 核心价值：用户会话与任务执行解耦，支持长时任务和定时调度
> 2. Temporal 三件套：Schedule（定时）、Workflow（逻辑）、Activity（执行）
> 3. 无人值守必须预设：Token 预算上限 + 执行时间上限 + 告警阈值
> 4. 状态查询：用 Query 实时查进度，用 Signal 动态调整行为
> 5. 失败恢复：RetryPolicy 指数退避 + 最大重试次数 + 人工介入兜底
>
> **10 分钟路径**：29.1-29.3 → 29.5 → Shannon Lab

---

你的用户说："帮我每天早上 9 点生成一份 AI 行业新闻简报。"

传统方式，你得告诉他们设个闹钟，每天打开网页触发。或者写个 cron job，但那得有服务器，还得处理失败重试、日志监控...

Background Agent 改变了这个游戏。用户只需要说一句话，系统就会：
1. 创建一个定时调度任务
2. 每天早上 9 点自动触发 Agent 执行
3. 任务完成后发送通知
4. 如果失败，自动重试并告警
5. 用户随时可以暂停、恢复、查看历史

这不是简单的定时脚本——这是一个持续运行的 Agent，能自主处理意外情况，能记住上下文，能根据反馈调整行为。

但这也是最危险的 Agent 形态。用户不在线，谁来监督？运行 8 小时后 Token 爆了怎么办？任务卡死了如何自动恢复？

这一章，我们来看 Shannon 是如何用 Temporal 实现可靠的 Background Agent 的。

---

## 29.1 为什么需要 Background Agent

### 同步执行的局限

传统的 Agent 交互是同步的：用户发请求，等待结果，拿到响应。这在短任务上没问题，但遇到以下场景就力不从心了：

| 场景 | 执行时长 | 为什么同步不行 |
|------|----------|----------------|
| 深度研究报告 | 30 分钟 - 2 小时 | HTTP 超时、连接断开 |
| 数据分析任务 | 数小时 | 浏览器关闭、网络波动 |
| 定期监控 | 24/7 持续 | 用户不可能一直在线 |
| 批量处理 | 数小时到数天 | 需要断点续传 |

同步执行的问题：

```
用户会话                  任务执行
    |                        |
    ├──────启动任务──────────>|
    |                        |── 执行中...
    |                        |── 执行中...
    |<──────等待...          |
    |                        |── 执行中...
    X 连接断开               |── 继续执行?
                             |── 结果丢失!
```

### Background Agent 的核心特征

Background Agent 打破了用户会话和任务执行的绑定：

```
用户会话                    后台系统                     任务执行
    |                          |                           |
    ├────创建调度任务─────────>|                           |
    |<───返回任务ID (立即)────|                           |
    |                          |                           |
    X 用户离线                 |                           |
                               |                           |
    ～～～ 到达调度时间 ～～～  |                           |
                               ├──────触发执行────────────>|
                               |                           |── 执行...
                               |<─────记录状态─────────────|
                               |                           |── 执行...
                               |<─────执行完成─────────────|
                               |                           |
    ～～～ 用户上线 ～～～     |                           |
    |                          |                           |
    ├────查询结果─────────────>|                           |
    |<───返回历史记录─────────|                           |
```

![Background Agent 异步解耦流程](assets/background-agent-async-flow.svg)

核心特征：

1. **解耦**：用户会话和任务执行完全分离
2. **持久化**：任务状态存储在数据库/工作流引擎中
3. **调度**：支持 Cron 表达式定时触发
4. **可观测**：任务状态、进度、结果可随时查询
5. **容错**：失败自动重试、断点续传

---

## 29.2 架构设计：Temporal + Schedule Manager

Shannon 的 Background Agent 基于 Temporal 工作流引擎实现。选择 Temporal 是因为它提供了：

- 原生的调度能力（Schedule）
- 持久化的工作流状态
- 自动重试和故障恢复
- 可观测性和审计日志

### 核心组件

```
+-----------------------------------------------------------+
|                    Orchestrator (Go)                       |
|                                                           |
|  +-----------------------------------------------------+  |
|  |              Schedule Manager                        |  |
|  |  - CreateSchedule()  创建定时任务                     |  |
|  |  - PauseSchedule()   暂停任务                        |  |
|  |  - ResumeSchedule()  恢复任务                        |  |
|  |  - DeleteSchedule()  删除任务                        |  |
|  |  - ListSchedules()   列出任务                        |  |
|  +-------------------------+---------------------------+  |
+-----------------------------|-----------------------------+
                              |
           +------------------+------------------+
           v                  v                  v
    +------------+     +------------+     +------------+
    | PostgreSQL |     |  Temporal  |     |   Worker   |
    | (元数据)    |     | (调度引擎) |     | (执行器)   |
    +------------+     +------------+     +------------+
```

**Schedule Manager**：管理调度任务的生命周期，强制执行业务规则（配额、预算、最小间隔）。

**PostgreSQL**：存储调度元数据、执行历史、用户配置。

**Temporal**：实际的调度引擎，负责按 Cron 表达式触发工作流。

**Worker**：执行具体的 Agent 任务。

### 为什么需要两层存储？

你可能会问：Temporal 已经存储了调度信息，为什么还要 PostgreSQL？

因为它们负责不同的事情：

| 存储层 | 负责内容 | 查询需求 |
|--------|----------|----------|
| **Temporal** | 工作流状态、调度触发 | 由 Temporal 内部使用 |
| **PostgreSQL** | 业务元数据、用户配置、执行历史 | 用户 UI、分析报表、审计 |

比如，"查询某用户的所有调度任务" 这种操作，直接查 PostgreSQL 比遍历 Temporal 快得多。

---

## 29.3 创建定时任务

创建调度任务需要多重验证。以下是 Shannon 的实现：

```go
// 摘自 go/orchestrator/internal/schedules/manager.go

// CreateSchedule 创建新的定时任务
func (m *Manager) CreateSchedule(ctx context.Context, req *CreateScheduleInput) (*Schedule, error) {
    // 1. 验证 Cron 表达式
    schedule, err := m.cronParser.Parse(req.CronExpression)
    if err != nil {
        return nil, fmt.Errorf("%w: %v", ErrInvalidCronExpression, err)
    }

    // 2. 强制最小间隔
    if !m.validateMinInterval(req.CronExpression) {
        return nil, fmt.Errorf("%w: must be at least %d minutes",
            ErrIntervalTooShort, m.config.MinCronIntervalMins)
    }

    // 3. 检查用户配额
    count, err := m.dbOps.CountSchedulesByUser(ctx, req.UserID, req.TenantID)
    if err != nil {
        return nil, fmt.Errorf("failed to check schedule limit: %w", err)
    }
    if count >= m.config.MaxPerUser {
        return nil, fmt.Errorf("%w: %d/%d schedules",
            ErrScheduleLimitReached, count, m.config.MaxPerUser)
    }

    // 4. 验证预算限制
    if req.MaxBudgetPerRunUSD < 0 {
        return nil, fmt.Errorf("budget cannot be negative: $%.2f", req.MaxBudgetPerRunUSD)
    }
    if req.MaxBudgetPerRunUSD > m.config.MaxBudgetPerRunUSD {
        return nil, fmt.Errorf("%w: $%.2f > $%.2f", ErrBudgetExceeded,
            req.MaxBudgetPerRunUSD, m.config.MaxBudgetPerRunUSD)
    }

    // 5. 验证时区
    timezone := req.Timezone
    if timezone == "" {
        timezone = "UTC"
    }
    tz, err := time.LoadLocation(timezone)
    if err != nil {
        return nil, fmt.Errorf("%w: %s", ErrInvalidTimezone, timezone)
    }

    // 6. 生成 ID
    scheduleID := uuid.New()
    temporalScheduleID := fmt.Sprintf("schedule-%s", scheduleID.String())

    // 7. 在 Temporal 中创建调度
    _, err = m.temporalClient.ScheduleClient().Create(ctx, client.ScheduleOptions{
        ID: temporalScheduleID,
        Spec: client.ScheduleSpec{
            CronExpressions: []string{req.CronExpression},
            TimeZoneName:    timezone,
        },
        Action: &client.ScheduleWorkflowAction{
            Workflow:           "ScheduledTaskWorkflow",
            TaskQueue:          "shannon-tasks",
            WorkflowRunTimeout: time.Duration(req.TimeoutSeconds) * time.Second,
            Args: []interface{}{
                ScheduledTaskInput{
                    ScheduleID:         scheduleID.String(),
                    TaskQuery:          req.TaskQuery,
                    TaskContext:        req.TaskContext,
                    MaxBudgetPerRunUSD: req.MaxBudgetPerRunUSD,
                    UserID:             req.UserID.String(),
                    TenantID:           req.TenantID.String(),
                },
            },
        },
        Paused: false,
    })
    if err != nil {
        return nil, fmt.Errorf("failed to create Temporal schedule: %w", err)
    }

    // 8. 计算下次执行时间
    nextRun := schedule.Next(time.Now().In(tz))

    // 9. 持久化到数据库
    dbSchedule := &Schedule{
        ID:                 scheduleID,
        UserID:             req.UserID,
        TenantID:           req.TenantID,
        Name:               req.Name,
        CronExpression:     req.CronExpression,
        Timezone:           timezone,
        TaskQuery:          req.TaskQuery,
        MaxBudgetPerRunUSD: req.MaxBudgetPerRunUSD,
        TemporalScheduleID: temporalScheduleID,
        Status:             ScheduleStatusActive,
        NextRunAt:          &nextRun,
    }

    if err := m.dbOps.CreateSchedule(ctx, dbSchedule); err != nil {
        // 回滚：删除 Temporal 调度
        _ = m.temporalClient.ScheduleClient().GetHandle(ctx, temporalScheduleID).Delete(ctx)
        return nil, fmt.Errorf("failed to persist schedule: %w", err)
    }

    return dbSchedule, nil
}
```

### 设计要点

1. **先 Temporal 后数据库**：如果数据库写入失败，回滚 Temporal 调度。反过来则更难回滚。

2. **多重验证**：Cron 语法、最小间隔、用户配额、预算限制、时区有效性——全部在创建时校验。

3. **预计算下次执行时间**：方便 UI 展示，不需要每次查询 Temporal。

### 最小间隔验证

防止用户创建过于频繁的调度（比如每分钟执行），这会耗尽资源和预算：

```go
// validateMinInterval 检查 Cron 表达式是否满足最小间隔
func (m *Manager) validateMinInterval(cronExpression string) bool {
    if m.config.MinCronIntervalMins <= 0 {
        return true // 无限制
    }

    schedule, err := m.cronParser.Parse(cronExpression)
    if err != nil {
        return false
    }

    // 计算下两次执行时间
    now := time.Now().In(time.UTC)
    next1 := schedule.Next(now)
    next2 := schedule.Next(next1)

    // 检查间隔是否满足最小要求
    intervalMinutes := next2.Sub(next1).Minutes()
    return intervalMinutes >= float64(m.config.MinCronIntervalMins)
}
```

---

## 29.4 暂停与恢复

用户可能需要临时暂停调度（比如出差期间不需要报告），之后再恢复。

### 暂停

```go
// PauseSchedule 暂停调度任务
func (m *Manager) PauseSchedule(ctx context.Context, scheduleID uuid.UUID, reason string) error {
    // 1. 获取调度
    dbSchedule, err := m.dbOps.GetSchedule(ctx, scheduleID)
    if err != nil {
        return fmt.Errorf("schedule not found: %w", err)
    }

    if dbSchedule.Status == ScheduleStatusPaused {
        return nil // 已暂停，幂等
    }

    // 2. 在 Temporal 中暂停
    handle := m.temporalClient.ScheduleClient().GetHandle(ctx, dbSchedule.TemporalScheduleID)
    if err := handle.Pause(ctx, client.SchedulePauseOptions{
        Note: reason,
    }); err != nil {
        return fmt.Errorf("failed to pause Temporal schedule: %w", err)
    }

    // 3. 更新数据库状态
    if err := m.dbOps.UpdateScheduleStatus(ctx, scheduleID, ScheduleStatusPaused); err != nil {
        return fmt.Errorf("failed to update schedule status: %w", err)
    }

    m.logger.Info("Schedule paused",
        zap.String("schedule_id", scheduleID.String()),
        zap.String("reason", reason),
    )

    return nil
}
```

### 恢复

```go
// ResumeSchedule 恢复暂停的调度任务
func (m *Manager) ResumeSchedule(ctx context.Context, scheduleID uuid.UUID, reason string) (*time.Time, error) {
    // 1. 获取调度
    dbSchedule, err := m.dbOps.GetSchedule(ctx, scheduleID)
    if err != nil {
        return nil, fmt.Errorf("schedule not found: %w", err)
    }

    if dbSchedule.Status == ScheduleStatusActive {
        return dbSchedule.NextRunAt, nil // 已激活，返回下次执行时间
    }

    // 2. 在 Temporal 中恢复
    handle := m.temporalClient.ScheduleClient().GetHandle(ctx, dbSchedule.TemporalScheduleID)
    if err := handle.Unpause(ctx, client.ScheduleUnpauseOptions{
        Note: reason,
    }); err != nil {
        return nil, fmt.Errorf("failed to unpause Temporal schedule: %w", err)
    }

    // 3. 计算新的下次执行时间
    schedule, _ := m.cronParser.Parse(dbSchedule.CronExpression)
    tz, _ := time.LoadLocation(dbSchedule.Timezone)
    nextRun := schedule.Next(time.Now().In(tz))

    // 4. 更新数据库
    m.dbOps.UpdateScheduleStatus(ctx, scheduleID, ScheduleStatusActive)
    m.dbOps.UpdateScheduleNextRun(ctx, scheduleID, nextRun)

    return &nextRun, nil
}
```

### 幂等性

注意两个方法都是幂等的：
- 暂停一个已暂停的调度，直接返回成功
- 恢复一个已激活的调度，直接返回下次执行时间

这样调用方不需要先查询状态再决定是否操作。

---

## 29.5 Cron 表达式详解

Cron 是定时调度的标准语言。Shannon 使用标准的 5 字段格式：

```
+------------- 分钟 (0 - 59)
| +----------- 小时 (0 - 23)
| | +--------- 日期 (1 - 31)
| | | +------- 月份 (1 - 12)
| | | | +----- 星期 (0 - 6, 0=Sunday)
| | | | |
* * * * *
```

### 常用示例

| 表达式 | 含义 |
|--------|------|
| `0 9 * * *` | 每天早上 9 点 |
| `0 9 * * 1-5` | 周一到周五早上 9 点 |
| `0 */4 * * *` | 每 4 小时整点 |
| `0 0 1 * *` | 每月 1 日零点 |
| `30 8 * * 1` | 每周一早上 8:30 |
| `0 9,18 * * *` | 每天 9 点和 18 点 |

### 时区支持

时区是 Background Agent 的关键特性。用户说"每天 9 点"，他指的是他所在时区的 9 点，不是 UTC 的 9 点。

```go
// Temporal 调度支持时区
_, err = m.temporalClient.ScheduleClient().Create(ctx, client.ScheduleOptions{
    Spec: client.ScheduleSpec{
        CronExpressions: []string{"0 9 * * *"},
        TimeZoneName:    "Asia/Tokyo",  // 东京时间 9 点
    },
})
```

支持标准 IANA 时区名称：`America/New_York`、`Europe/London`、`Asia/Shanghai` 等。

---

## 29.6 预算与成本控制

Background Agent 在用户不在场时运行，成本控制更加重要。

### 三层预算控制

1. **系统级限制**：每次执行的最大预算（管理员配置）
2. **用户级预算**：用户设置的每次执行预算
3. **累计预算**：某个调度的总消耗上限（可选）

```go
// 系统配置
type Config struct {
    MaxPerUser          int     // 每用户最大调度数 (默认: 50)
    MinCronIntervalMins int     // 最小执行间隔 (默认: 60分钟)
    MaxBudgetPerRunUSD  float64 // 每次执行最大预算 (默认: $10)
}

// 创建时验证
if req.MaxBudgetPerRunUSD > m.config.MaxBudgetPerRunUSD {
    return nil, fmt.Errorf("%w: $%.2f > $%.2f", ErrBudgetExceeded,
        req.MaxBudgetPerRunUSD, m.config.MaxBudgetPerRunUSD)
}
```

### 预算注入到工作流

```go
// ScheduledTaskWorkflow 中注入预算
if input.MaxBudgetPerRunUSD > 0 {
    if taskInput.Context == nil {
        taskInput.Context = make(map[string]interface{})
    }
    taskInput.Context["max_budget_usd"] = input.MaxBudgetPerRunUSD
}
```

主工作流会检查这个预算并在超限时停止执行。

### 成本追踪

每次执行后记录成本，便于分析和告警：

```go
// 执行完成后记录
workflow.ExecuteActivity(activityCtx, "RecordScheduleExecutionComplete",
    RecordScheduleExecutionCompleteInput{
        ScheduleID: scheduleID,
        TaskID:     childWorkflowID,
        Status:     status,
        TotalCost:  totalCost,  // 从子工作流提取
        ErrorMsg:   errorMsg,
    },
).Get(ctx, nil)
```

---

## 29.7 孤儿检测与清理

数据库和 Temporal 的状态可能不一致。比如：
- 有人手动在 Temporal UI 删除了调度
- 数据库迁移时数据丢失
- 网络问题导致创建流程中断

需要定期检测和清理：

```go
// VerifyScheduleExists 检查调度是否在 Temporal 中存在
func (m *Manager) VerifyScheduleExists(ctx context.Context, schedule *Schedule) (bool, error) {
    if schedule.Status != ScheduleStatusActive && schedule.Status != ScheduleStatusPaused {
        return true, nil // 只验证活跃/暂停的调度
    }

    handle := m.temporalClient.ScheduleClient().GetHandle(ctx, schedule.TemporalScheduleID)
    _, err := handle.Describe(ctx)
    if err != nil {
        if strings.Contains(err.Error(), "not found") {
            m.logger.Warn("Detected orphaned schedule - Temporal schedule not found",
                zap.String("schedule_id", schedule.ID.String()),
                zap.String("temporal_id", schedule.TemporalScheduleID),
            )
            // 在数据库中标记为已删除
            m.dbOps.UpdateScheduleStatus(ctx, schedule.ID, ScheduleStatusDeleted)
            return false, nil
        }
        // 其他错误不确定状态，假设存在
        return true, nil
    }
    return true, nil
}

// DetectAndCleanOrphanedSchedules 批量检测孤儿调度
func (m *Manager) DetectAndCleanOrphanedSchedules(ctx context.Context) ([]uuid.UUID, error) {
    schedules, err := m.dbOps.GetAllActiveSchedules(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed to get active schedules: %w", err)
    }

    var orphanedIDs []uuid.UUID
    for _, schedule := range schedules {
        exists, err := m.VerifyScheduleExists(ctx, schedule)
        if err != nil {
            continue
        }
        if !exists {
            orphanedIDs = append(orphanedIDs, schedule.ID)
        }
    }

    if len(orphanedIDs) > 0 {
        m.logger.Info("Cleaned up orphaned schedules",
            zap.Int("count", len(orphanedIDs)),
        )
    }

    return orphanedIDs, nil
}
```

建议通过另一个定时任务每天运行一次孤儿检测。

---

## 29.8 安全考量

Background Agent 在用户不在场时运行，安全风险更高。

### 风险矩阵

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **预算失控** | 后台任务消耗大量 Token | 每次执行预算限制 |
| **无限循环** | Agent 陷入重试循环 | 最大重试次数、执行超时 |
| **权限滥用** | 定时任务执行敏感操作 | 操作审计、权限最小化 |
| **资源耗尽** | 太多调度任务同时运行 | 用户配额、最小间隔 |
| **状态不一致** | 数据库和 Temporal 不同步 | 孤儿检测、状态校验 |

### 操作审计

每次执行都应该有完整的审计记录：

```go
type ScheduleExecution struct {
    ID          uuid.UUID
    ScheduleID  uuid.UUID
    StartedAt   time.Time
    CompletedAt *time.Time
    Status      string    // RUNNING, COMPLETED, FAILED
    TotalCost   float64
    ErrorMsg    *string
    Metadata    map[string]interface{}
}
```

### 敏感操作限制

后台任务不应该执行某些敏感操作（至少不能没有额外授权）：

```python
# 概念示例：后台任务的操作限制

BACKGROUND_RESTRICTED_OPERATIONS = [
    "delete_data",        # 删除数据
    "send_email",         # 发送邮件（可能是垃圾邮件）
    "make_purchase",      # 购买操作
    "modify_permissions", # 修改权限
]

def check_background_operation(operation: str, is_background: bool) -> bool:
    if is_background and operation in BACKGROUND_RESTRICTED_OPERATIONS:
        raise BackgroundOperationRestricted(
            f"Operation '{operation}' is not allowed in background tasks. "
            f"Please trigger manually with user confirmation."
        )
    return True
```

---

## 29.9 实战示例

### 示例 1：每日新闻简报

```python
# 概念示例：创建每日新闻简报

async def create_daily_news_schedule(
    topic: str,
    user_id: str,
    timezone: str = "UTC",
) -> dict:
    """创建每日新闻简报定时任务"""

    request = {
        "name": f"Daily News: {topic}",
        "cron_expression": "0 9 * * *",  # 每天9点
        "timezone": timezone,
        "task_query": f"""
Generate a daily news digest about {topic}.

Include:
1. Top 5 news from the past 24 hours
2. Key insights and trends
3. Notable quotes or data points
4. Links to original sources

Format: Markdown, suitable for email newsletter.
""",
        "task_context": {
            "output_format": "markdown",
            "max_sources": 10,
        },
        "max_budget_per_run_usd": 2.0,
        "timeout_seconds": 600,
        "user_id": user_id,
    }

    return await schedule_client.create(request)
```

### 示例 2：竞品监控

```python
# 概念示例：竞品网站监控

async def create_competitor_monitor(
    competitor_urls: List[str],
    user_id: str,
) -> dict:
    """创建竞品监控定时任务"""

    request = {
        "name": "Competitor Price Monitor",
        "cron_expression": "0 */6 * * *",  # 每6小时
        "timezone": "UTC",
        "task_query": f"""
Monitor these competitor websites for changes:
{chr(10).join(competitor_urls)}

Report:
1. Any price changes detected
2. New products or features
3. Marketing message changes
4. Compare with previous check

If significant changes detected, flag as ALERT.
""",
        "task_context": {
            "previous_state_key": "competitor_state",  # 持久化状态
            "alert_threshold": "significant",
        },
        "max_budget_per_run_usd": 3.0,
        "timeout_seconds": 900,
        "user_id": user_id,
    }

    return await schedule_client.create(request)
```

### 示例 3：每周汇总报告

```python
# 概念示例：每周汇总

async def create_weekly_summary(
    topics: List[str],
    user_id: str,
) -> dict:
    """创建每周汇总报告"""

    request = {
        "name": "Weekly AI Industry Summary",
        "cron_expression": "0 9 * * 1",  # 每周一9点
        "timezone": "America/New_York",
        "task_query": f"""
Generate a comprehensive weekly summary for:
{', '.join(topics)}

Include:
1. Major announcements and releases
2. Funding and acquisitions
3. Research paper highlights
4. Industry trends analysis
5. Predictions for next week
""",
        "max_budget_per_run_usd": 5.0,
        "timeout_seconds": 1800,
        "user_id": user_id,
    }

    return await schedule_client.create(request)
```

---

## 29.10 常见的坑

### 坑 1：时区混淆

用户说"每天 9 点"，但系统按 UTC 执行。

```go
// 错误：默认 UTC，用户不知道
cron := "0 9 * * *"  // 用户以为是本地9点，实际是 UTC 9点

// 正确：明确要求时区，并在返回中清楚说明
if req.Timezone == "" {
    req.Timezone = "UTC"
}
response.Timezone = req.Timezone
response.NextRunAt = schedule.Next(time.Now().In(tz))
response.NextRunLocal = response.NextRunAt.Format("2006-01-02 15:04 MST")
```

### 坑 2：忘记回滚

创建调度时，如果数据库写入失败，忘记删除已创建的 Temporal 调度。

```go
// 错误：无回滚
_, err = m.temporalClient.ScheduleClient().Create(ctx, ...)
// ... Temporal 创建成功

err = m.dbOps.CreateSchedule(ctx, dbSchedule)
if err != nil {
    return nil, err  // Temporal 调度成了孤儿！
}

// 正确：失败时回滚
if err := m.dbOps.CreateSchedule(ctx, dbSchedule); err != nil {
    _ = m.temporalClient.ScheduleClient().GetHandle(ctx, temporalScheduleID).Delete(ctx)
    return nil, fmt.Errorf("failed to persist schedule: %w", err)
}
```

### 坑 3：删除只删数据库

```go
// 错误：只删除数据库记录
m.dbOps.DeleteSchedule(ctx, scheduleID)
// Temporal 调度继续运行，成了孤儿！

// 正确：先删 Temporal，再更新数据库
handle := m.temporalClient.ScheduleClient().GetHandle(ctx, dbSchedule.TemporalScheduleID)
handle.Delete(ctx)
m.dbOps.UpdateScheduleStatus(ctx, scheduleID, ScheduleStatusDeleted)
```

### 坑 4：无预算限制

```go
// 错误：用户可以设置任意预算
request.MaxBudgetPerRunUSD = 1000.0  // 每次执行消耗 $1000

// 正确：强制系统上限
if req.MaxBudgetPerRunUSD > m.config.MaxBudgetPerRunUSD {
    return nil, fmt.Errorf("%w: $%.2f > $%.2f", ErrBudgetExceeded,
        req.MaxBudgetPerRunUSD, m.config.MaxBudgetPerRunUSD)
}
```

---

## 29.11 回顾

1. **Background Agent 定义**：任务脱离用户会话独立运行，支持定时调度、暂停恢复
2. **双层存储**：Temporal 负责调度执行，PostgreSQL 负责业务查询
3. **多重验证**：Cron 语法、最小间隔、用户配额、预算限制
4. **时区支持**：用户期望本地时间，必须明确处理时区
5. **孤儿清理**：定期检测数据库和 Temporal 的不一致状态

---

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- `go/orchestrator/internal/schedules/manager.go`：Schedule Manager 的完整实现，包括创建、暂停、恢复、删除

### 选读深挖（2 个，按兴趣挑）

- `go/orchestrator/internal/workflows/scheduled/scheduled_task_workflow.go`：调度触发时执行的工作流包装器
- `config/models.yaml` 中的预算配置：了解如何设置系统级别的资源限制

---

## 练习

### 练习 1：设计告警调度

设计一个监控告警系统：
1. 每 5 分钟检查系统状态
2. 如果检测到异常，发送告警通知
3. 告警后进入"冷却期"，避免重复告警
4. 异常恢复后发送恢复通知

### 练习 2：实现执行历史查询

设计执行历史的 API 和存储：
1. 存储每次执行的开始时间、结束时间、状态、成本
2. 支持按调度 ID 查询历史
3. 支持按时间范围过滤
4. 计算某个调度的累计成本

### 练习 3（进阶）：级联暂停

设计一个系统，当某个调度连续失败 3 次后：
1. 自动暂停该调度
2. 发送通知给用户
3. 记录暂停原因
4. 用户恢复时检查失败原因是否已解决

---

## 进一步阅读

- **Temporal Schedules** - https://docs.temporal.io/workflows#schedule
- **Cron Expression** - https://crontab.guru/
- **IANA Time Zone Database** - https://www.iana.org/time-zones

---

## 下一章预告

Background Agent 按调度执行任务，但每次执行用什么模型？都用最贵的大模型？那成本太高了。

下一章讲 **分层模型策略**——如何通过智能的模型选择实现 50-70% 的成本降低。

核心思路很简单：简单任务用小模型，复杂任务用大模型。但实现起来没那么简单：
- 怎么判断任务的复杂度？
- 小模型失败了要不要升级到大模型？
- 不同类型的任务适合什么模型？

下一章，我们来看 Shannon 的分层模型路由策略。
