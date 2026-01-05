# 第 15 章：Supervisor 模式

> **Supervisor 是多 Agent 系统的管理层——它不干具体活，但它决定谁来干、怎么协调、失败了怎么办。管理层的价值在于让团队整体产出大于个体之和，而不是管得越多越好。**

---

> **⏱️ 快速通道**（5 分钟掌握核心）
>
> 1. 超过 5 个子任务或需要动态调整团队时用 Supervisor
> 2. 邮箱系统让 Agent 之间异步通信，必须用非阻塞发送
> 3. 动态招募通过 Signal 触发，运行时可以加人
> 4. 智能容错：50%+1 失败阈值，部分失败不影响整体
> 5. **Human-in-the-Loop**：人类是层级最高的 Supervisor，关键决策需要升级
> 6. 简单任务还是交给 DAG，Supervisor 开销更大
>
> **10 分钟路径**：15.1-15.3 → 15.5 → 15.12 (HITL) → Shannon Lab

---

## 15.1 什么时候用 Supervisor？

前一章我们讲了 DAG 工作流——通过依赖图调度任务，该并行的并行，该等待的等待。DAG 很强大，但它有个假设：**任务结构是固定的**。

如果你能预先规划好"有哪些任务、谁依赖谁"，DAG 就够用了。但有些场景不是这样的。

我去年帮一个咨询公司做竞争分析 Agent。开始时，需求很明确：分析 5 家竞争对手的产品、定价、市场份额。我用 DAG 设计了 5 个并行的研究任务 + 1 个综合任务。

上线后，客户提了新需求："能不能在分析过程中，如果发现某家公司特别重要，自动深挖它的技术专利？"

这就超出 DAG 的能力了。DAG 的任务是固定的，不能中途"加人"。你无法在执行时说"嘿，这个公司值得深入研究，再派一个专利分析师"。

**Supervisor 模式就是为了解决这类问题——当任务结构需要动态调整、Agent 需要互相通信、或者团队规模较大时，需要一个"管理层"来协调。**

DAG 很强，但有局限。看对比：

| 场景 | DAG | Supervisor |
|------|-----|-----------|
| 预定义任务结构 | 擅长 | 同样支持 |
| 动态任务生成 | 不支持 | 运行时招募 |
| Agent 间通信 | 仅通过依赖传递 | 邮箱系统 |
| 任务数量 > 5 | 可能过载 | 层级管理 |
| 角色专业化 | 基础支持 | 动态角色分配 |
| 智能失败恢复 | 基础重试 | 阈值容错 |
| 执行时间 | 短（分钟级） | 可长（小时级） |

**触发 Supervisor 的条件**：

```go
switch {
case len(decomp.Subtasks) > 5 || hasDeps:
    // 子任务多或依赖复杂 → Supervisor
    return SupervisorWorkflow(ctx, input)
default:
    return DAGWorkflow(ctx, input)
}
```

简单说：5 个任务以下用 DAG，超过就上 Supervisor。

> **注意**：Supervisor 的开销比 DAG 大。邮箱系统、团队目录、动态招募......这些都有成本。如果你的任务就是「并行搜索 3 个公司」，用 DAG 就够了，没必要上 Supervisor。

判断标准：
- 任务数量 > 5？考虑 Supervisor
- 需要中途加人？用 Supervisor
- Agent 之间需要通信？用 Supervisor
- 任务结构完全固定？用 DAG

---

## 15.2 架构概览

Supervisor 的核心组件：

```
┌─────────────────────────────────────────────────────────────┐
│                     SupervisorWorkflow                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────┐     │
│  │                    Supervisor                       │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │     │
│  │  │  任务分解   │  │  团队管理   │  │  结果综合  │  │     │
│  │  └─────────────┘  └─────────────┘  └────────────┘  │     │
│  └────────────────────────────────────────────────────┘     │
│                              │                               │
│              ┌───────────────┼───────────────┐               │
│              ▼               ▼               ▼               │
│       ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│       │ Agent A  │    │ Agent B  │    │ Agent C  │   ...    │
│       │(研究员)   │    │(分析师)  │    │(写作者)  │          │
│       └──────────┘    └──────────┘    └──────────┘          │
│              │               │               │               │
│              └───────────────┼───────────────┘               │
│                              ▼                               │
│                      ┌──────────────┐                        │
│                      │   Mailbox    │  ← Agent 间通信        │
│                      └──────────────┘                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**三大能力**：

1. **团队管理**：招募、退役、角色分配
2. **邮箱通信**：Agent 之间异步消息
3. **智能容错**：部分失败不影响整体

**实现参考 (Shannon)**: [`go/orchestrator/internal/workflows/supervisor_workflow.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/supervisor_workflow.go) - SupervisorWorkflow 函数

---

## 15.3 邮箱系统

Agent 之间怎么通信？靠邮箱。

### 为什么需要邮箱？

DAG 模式下，Agent 之间只能通过依赖传递数据：A 完成 → 结果传给 B。

但有些场景需要更灵活的通信：

```
Agent A (研究员)                    Agent B (分析师)
     │                                    │
     │  发现：竞争对手刚发布新产品         │
     │                                    │
     │── "嘿，你可能要关注这个" ─────────►│
     │                                    │ 收到消息
     │                                    │ 调整分析重点
```

A 不是 B 的依赖，但 A 发现的信息对 B 有用。这就需要邮箱。

### 邮箱实现

```go
type MailboxMessage struct {
    From, To, Role, Content string
}

func SupervisorWorkflow(ctx workflow.Context, input TaskInput) (TaskResult, error) {
    var messages []MailboxMessage
    sig := workflow.GetSignalChannel(ctx, "mailbox_v1")
    msgChan := workflow.NewChannel(ctx)

    // 接收消息的协程（非阻塞发送，防止死锁）
    workflow.Go(ctx, func(ctx workflow.Context) {
        for {
            var msg MailboxMessage
            sig.Receive(ctx, &msg)
            // 关键：用 Selector + Default 实现非阻塞发送
            sel := workflow.NewSelector(ctx)
            sel.AddSend(msgChan, msg, func() {})
            sel.AddDefault(func() {})  // 通道满了就跳过，不阻塞
            sel.Select(ctx)
        }
    })

    // 查询处理器：返回副本防止竞态
    workflow.SetQueryHandler(ctx, "getMailbox", func() ([]MailboxMessage, error) {
        result := make([]MailboxMessage, len(messages))
        copy(result, messages)
        return result, nil
    })
    // ...
}
```

**为什么要非阻塞**：Temporal 单线程，接收协程阻塞会卡住整个工作流。

---

## 15.4 动态团队管理

Supervisor 最强大的能力：运行时招募或退役 Agent。

### 招募信号与团队目录

```go
type RecruitRequest struct {
    Description string  // 任务描述
    Role        string  // 期望角色
}

type AgentInfo struct {
    AgentID, Role, Status string  // Status: running/completed/failed
}

var teamAgents []AgentInfo

// 动态招募协程
recruitCh := workflow.GetSignalChannel(ctx, "recruit_v1")
workflow.Go(ctx, func(ctx workflow.Context) {
    for {
        var req RecruitRequest
        recruitCh.Receive(ctx, &req)

        // 1. 策略授权检查（可选）
        // 2. 启动子工作流
        future := workflow.ExecuteChildWorkflow(ctx, SimpleTaskWorkflow, TaskInput{
            Query: req.Description, Context: map[string]interface{}{"role": req.Role},
        })
        var res TaskResult
        future.Get(ctx, &res)

        // 3. 收集结果
        childResults = append(childResults, AgentExecutionResult{AgentID: "dynamic_" + req.Role, Response: res.Result})
    }
})

// 团队目录查询
workflow.SetQueryHandler(ctx, "listTeamAgents", func() ([]AgentInfo, error) {
    result := make([]AgentInfo, len(teamAgents))
    copy(result, teamAgents)
    return result, nil
})
```

招募流程：`Signal: recruit_v1` → 策略授权 → 启动子工作流 → 收集结果

---

## 15.5 智能失败处理

Agent 会失败。网络超时、LLM 出错、工具报错。

DAG 的处理方式是：重试 3 次，还不行就整个任务失败。

Supervisor 更聪明：**允许部分失败，但不能超过一半**。

### 50%+1 阈值

```go
failedTasks := 0
maxFailures := len(decomp.Subtasks)/2 + 1  // 50%+1：超过一半失败则中止
taskRetries := make(map[string]int)

for _, st := range decomp.Subtasks {
    for taskRetries[st.ID] < 3 {  // 每任务最多重试 3 次
        err := workflow.ExecuteActivity(ctx, ExecuteAgent, st).Get(ctx, &res)
        if err == nil { break }
        taskRetries[st.ID]++
    }
    if taskRetries[st.ID] >= 3 {
        failedTasks++
        if failedTasks >= maxFailures {
            return TaskResult{Success: false, ErrorMessage: "Too many failures"}, nil
        }
    }
}
// 阈值可配置：研究任务 50%、数据分析 20%、关键业务 0%
```

**为什么是 50%+1**：6 个任务，maxFailures=4。2 个失败继续；4 个失败中止（结果不可靠）。

---

## 15.6 角色分配

每个 Agent 分配不同角色，让它们专注自己的领域。

### 角色分配机制

LLM 分解时指定角色，Supervisor 读取并分配到 Agent：

```go
// LLM 分解结果：{"subtasks": [...], "agent_types": ["researcher", "analyst"]}

for i, st := range decomp.Subtasks {
    role := "generalist"  // 默认角色
    if i < len(decomp.AgentTypes) && decomp.AgentTypes[i] != "" {
        role = decomp.AgentTypes[i]  // LLM 指定的角色
    }
    childCtx["role"] = role
    teamAgents = append(teamAgents, AgentInfo{AgentID: agentName, Role: role})
    // 角色注入到 system prompt："你是一个 researcher，你的专长是信息搜集..."
}
```

| 角色 | 擅长 | 倾向 |
|------|------|------|
| researcher | 信息搜集 | 全面、详尽 |
| analyst | 数据分析 | 数字、趋势 |
| strategist | 战略规划 | 高层视角、长期 |
| writer | 内容创作 | 可读性、结构 |

---

## 15.7 从历史学习

Supervisor 可以从历史执行中学习，提供更好的分解建议。

### 记忆检索与应用

```go
if input.SessionID != "" {
    // 1. 获取历史记忆
    var memory *SupervisorMemoryContext
    workflow.ExecuteActivity(ctx, "FetchSupervisorMemory", input.SessionID).Get(ctx, &memory)

    // 2. 创建建议器，检索相似任务的历史分解策略
    advisor := NewDecompositionAdvisor(memory)
    suggestion := advisor.SuggestDecomposition(input.Query)

    // 3. 高置信度时直接应用历史策略
    if suggestion.Confidence > 0.8 {
        decomp.ExecutionStrategy = suggestion.Strategy
    }
}
// 学习效果：「分析 AI Agent 市场」成功 → 「分析 RPA 市场」自动复用类似策略
```

---

## 15.8 与 DAG 的协作

Supervisor 不是要取代 DAG。简单任务还是交给 DAG 处理更高效。

```go
// 判断是否简单任务
simpleByShape := len(decomp.Subtasks) == 0 ||
                 (len(decomp.Subtasks) == 1 && !needsTools)
isSimpleTask := (decomp.ComplexityScore < 0.3) && simpleByShape

if isSimpleTask {
    // 委托给 DAGWorkflow
    dagFuture := workflow.ExecuteChildWorkflow(ctx, DAGWorkflow, strategiesInput)

    var childExec workflow.Execution
    dagFuture.GetChildWorkflowExecution().Get(ctx, &childExec)
    controlHandler.RegisterChildWorkflow(childExec.ID)

    dagFuture.Get(ctx, &strategiesResult)
    controlHandler.UnregisterChildWorkflow(childExec.ID)

    return convertFromStrategiesResult(strategiesResult), nil
}
```

**分工**：

| 任务类型 | 处理者 | 原因 |
|----------|--------|------|
| 简单任务（1-2 步） | SimpleTask | 开销最小 |
| 中等任务（3-5 步） | DAG | 并行效率高 |
| 复杂任务（6+ 步） | Supervisor | 需要团队管理 |
| 动态任务 | Supervisor | 可以中途加人 |
| 需要通信 | Supervisor | 有邮箱系统 |

类比一下：Supervisor 是「部门经理」，DAG 是「项目组长」。

简单任务，项目组长自己带几个人就能搞定。
复杂任务，需要部门经理协调多个项目组、动态调配资源。

---

## 15.9 实战：多层级市场分析

场景：

```
用户：对 AI Agent 市场进行完整的竞争分析
```

### Supervisor 分解

```
├── 市场规模调研 (Agent A - 研究员)
├── 竞争对手识别 (Agent B - 分析师)
├── 产品对比分析 (Agent C - 产品专家)
│   └── 可能动态招募: 定价分析师
├── 技术趋势分析 (Agent D - 技术专家)
├── SWOT 综合 (Agent E - 战略分析师)
│   └── 依赖: A, B, C, D 的结果
└── 报告生成 (Agent F - 写作者)
    └── 依赖: E 的结果
```

### 执行流程

```
t0:  Supervisor 启动
     ├── 分解任务 → 6 个子任务
     ├── 初始化邮箱系统
     ├── 注册团队目录
     └── 设置控制信号处理器

t1:  并行启动 A, B, C, D（无依赖的任务）
     ├── Agent A (market-research): 市场规模调研
     ├── Agent B (competitor-scan): 竞争对手识别
     ├── Agent C (product-compare): 产品对比分析
     └── Agent D (tech-trend): 技术趋势分析

t2:  Agent C 发现需要定价分析
     ├── 发送 recruit_v1 信号
     │   {Description: "深度分析各产品定价策略", Role: "pricing_analyst"}
     ├── Supervisor 收到信号
     ├── 策略授权检查通过
     └── 动态招募 Agent C' (pricing-deep)

t3:  A, B 完成
     ├── 结果存入 childResults
     ├── 标记 completedTasks["market-research"] = true
     └── 发送邮箱消息给 E（可选）

t4:  C, C', D 完成
     └── 所有前置任务完成

t5:  Agent E 启动 (SWOT 分析)
     ├── 依赖检查：A, B, C, D 全部完成
     ├── 上下文注入前置结果
     └── 综合分析

t6:  E 完成 → F 启动 (报告生成)

t7:  F 完成 → Supervisor 综合
     ├── 收集所有 childResults
     ├── 预处理（去重、过滤）
     └── 返回最终报告
```

### 时间对比

```
Supervisor 模式：
├── A, B, C, D 并行: 20s
├── C' (动态招募): 10s (与 C、D 并行)
├── E (SWOT): 15s
├── F (报告): 10s
└── 总计: ~45s

如果串行：
├── A: 15s
├── B: 12s
├── C: 18s
├── D: 15s
├── E: 15s
├── F: 10s
└── 总计: ~85s

节省: ~47%
```

---

## 15.10 常见的坑

| 坑 | 问题描述 | 解决方案 |
|----|---------|----------|
| 信号通道阻塞 | `msgChan.Send` 阻塞整个工作流 | 用 Selector + Default 非阻塞发送 |
| 查询处理器竞态 | 直接返回切片不安全 | 返回副本：`copy(result, messages)` |
| 子工作流信号丢失 | 子工作流收不到暂停/取消信号 | 注册：`controlHandler.RegisterChildWorkflow(childExec.ID)` |
| 失败阈值太严格 | 1 个失败就中止 | 用 50%+1：`len(subtasks)/2 + 1` |
| 动态招募无限制 | 招太多 Agent | 限制团队规模：`maxTeamSize = 10` |
| 邮箱消息堆积 | 消息永远不清理，OOM | 限制数量，保留后一半 |

```go
// 典型错误 vs 正确做法
// 错误：msgChan.Send(ctx, msg)  // 可能阻塞
// 正确：sel.AddSend(msgChan, msg, func() {}); sel.AddDefault(func() {}); sel.Select(ctx)

// 错误：return messages  // Query Handler 竞态
// 正确：result := make([]T, len(messages)); copy(result, messages); return result
```

---

## 15.11 其他框架的实现

Supervisor/Manager 模式是多 Agent 协作的核心，各框架都有类似实现：

| 框架 | 实现 | 特点 |
|------|------|------|
| **AutoGen** | `GroupChatManager` | 对话驱动，自动选择发言者 |
| **CrewAI** | `Crew` + hierarchical | 角色定义清晰，流程化 |
| **LangGraph** | 自定义 Supervisor 节点 | 完全可控，灵活度高 |
| **OpenAI Swarm** | `handoff()` 机制 | 轻量级，Agent 自主交接 |

### AutoGen 示例

```python
from autogen import GroupChat, GroupChatManager

# 创建 Agent
researcher = AssistantAgent("researcher", llm_config=llm_config)
analyst = AssistantAgent("analyst", llm_config=llm_config)
writer = AssistantAgent("writer", llm_config=llm_config)

# 创建 GroupChat
groupchat = GroupChat(
    agents=[researcher, analyst, writer],
    messages=[],
    max_round=10,
    speaker_selection_method="auto"  # LLM 自动选择下一个发言者
)

# 创建 Manager（相当于 Supervisor）
manager = GroupChatManager(groupchat=groupchat, llm_config=llm_config)

# 启动对话
user_proxy.initiate_chat(manager, message="分析 AI Agent 市场")
```

### CrewAI 示例

```python
from crewai import Crew, Agent, Task, Process

# 定义 Agent
researcher = Agent(
    role="研究员",
    goal="搜集市场数据",
    backstory="你是一个资深市场研究员"
)

analyst = Agent(
    role="分析师",
    goal="分析数据洞察",
    backstory="你是一个数据分析专家"
)

# 定义任务
research_task = Task(description="调研 AI Agent 市场", agent=researcher)
analysis_task = Task(description="分析市场数据", agent=analyst)

# 创建 Crew（层级模式）
crew = Crew(
    agents=[researcher, analyst],
    tasks=[research_task, analysis_task],
    process=Process.hierarchical,  # 层级模式，有 Manager
    manager_llm=llm
)

result = crew.kickoff()
```

### 选择建议

| 场景 | 推荐框架 |
|------|----------|
| 对话式协作 | AutoGen |
| 流程化任务 | CrewAI |
| 完全自定义 | LangGraph |
| 生产级可靠性 | Shannon (Temporal) |

---

## 15.12 Human-in-the-Loop 集成

到目前为止，我们讲的 Supervisor 都是"全自动"的——Agent 组成团队，Supervisor 协调，最后输出结果。人类只在开始时提问、结束时看结果。

但生产环境里，这样不够。

我去年帮一个金融客户部署 Agent 系统，第一周就出事了。Agent 自动发了一封邮件给客户，内容基本正确，但措辞不太恰当。客户没投诉，但 CEO 很紧张："这种事能不能先让人看一眼再发？"

这就是 **Human-in-the-Loop (HITL)** 的核心需求：**人类作为层级架构中的最高决策者，在关键节点介入**。

### 15.12.1 人类在层级中的位置

重新看 Supervisor 的架构：

```
之前的架构（无人类节点）：

┌─────────────────────────────────────────────────────────────┐
│                    Supervisor (Agent)                        │
│                         │                                    │
│         ┌───────────────┼───────────────┐                   │
│         ▼               ▼               ▼                   │
│    ┌─────────┐    ┌─────────┐    ┌─────────┐               │
│    │ Worker  │    │ Worker  │    │ Worker  │               │
│    │ Agent A │    │ Agent B │    │ Agent C │               │
│    └─────────┘    └─────────┘    └─────────┘               │
└─────────────────────────────────────────────────────────────┘

问题：Supervisor 谁来管？高风险决策谁来批？
```

加入人类节点后：

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│                    ┌──────────────┐                          │
│                    │    Human     │ ← 层级最高               │
│                    │  (Approver)  │   战略决策 / 高风险审批   │
│                    └──────┬───────┘                          │
│                           │ escalate / approve               │
│                    ┌──────▼───────┐                          │
│                    │  Supervisor  │ ← 战术层                 │
│                    │   (Agent)    │   任务协调 / 资源分配     │
│                    └──────┬───────┘                          │
│              ┌────────────┼────────────┐                     │
│              ▼            ▼            ▼                     │
│        ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│        │ Worker  │  │ Worker  │  │ Worker  │ ← 执行层       │
│        │ Agent A │  │ Agent B │  │ Agent C │                │
│        └─────────┘  └─────────┘  └─────────┘                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**人类不是"旁观者"，而是层级架构中的一个节点**——只是这个节点响应慢、成本高，所以只在必要时调用。

### 15.12.2 升级触发器（Escalation Triggers）

什么情况下需要升级给人类？不能靠"感觉"，要有明确的触发条件。

```go
// 概念示例：升级触发器配置
type EscalationTriggers struct {
    // 置信度触发
    ConfidenceThreshold float64 `json:"confidence_threshold"` // 低于此值升级，如 0.6

    // 成本触发
    SingleActionCostLimit float64 `json:"single_action_cost_limit"` // 单次操作成本上限，如 $1.00

    // 敏感操作触发
    SensitiveActions []string `json:"sensitive_actions"` // 如 ["delete", "publish", "pay", "send_email"]

    // 失败触发
    ConsecutiveFailures int `json:"consecutive_failures"` // 连续失败次数，如 3

    // 超时触发
    DecisionTimeout time.Duration `json:"decision_timeout"` // Agent 决策超时
}

// 判断是否需要升级
func (s *Supervisor) ShouldEscalate(ctx context.Context, result AgentResult) (bool, string) {
    // 1. 置信度检查
    if result.Confidence < s.triggers.ConfidenceThreshold {
        return true, fmt.Sprintf("置信度过低: %.2f < %.2f", result.Confidence, s.triggers.ConfidenceThreshold)
    }

    // 2. 敏感操作检查
    for _, action := range s.triggers.SensitiveActions {
        if result.ProposedAction == action {
            return true, fmt.Sprintf("敏感操作需要审批: %s", action)
        }
    }

    // 3. 成本检查
    if result.EstimatedCost > s.triggers.SingleActionCostLimit {
        return true, fmt.Sprintf("成本超限: $%.2f > $%.2f", result.EstimatedCost, s.triggers.SingleActionCostLimit)
    }

    // 4. 连续失败检查
    if s.consecutiveFailures >= s.triggers.ConsecutiveFailures {
        return true, fmt.Sprintf("连续失败 %d 次", s.consecutiveFailures)
    }

    return false, ""
}
```

**触发条件优先级**：

| 触发器 | 阈值示例 | 优先级 | 说明 |
|--------|----------|--------|------|
| 敏感操作 | delete/pay/publish | 最高 | 不可逆，必须人工确认 |
| 成本超限 | > $1.00 | 高 | 防止失控消耗 |
| 置信度过低 | < 0.6 | 中 | Agent 自己不确定 |
| 连续失败 | > 3 次 | 中 | 可能陷入死循环 |
| 超出范围 | 无匹配工具 | 低 | Agent 能力不足 |

> **注意**：不是所有升级都需要人类回应。有些可以配置"超时自动拒绝"，有些可以"超时自动通过"。关键操作建议"超时自动拒绝"。

### 15.12.3 三种 HITL 模式

根据人类参与程度，HITL 有三种模式：

| 模式 | 人类参与度 | Agent 自主度 | 适用场景 |
|------|-----------|-------------|---------|
| **Human-in-Command** | 每步确认 | 最低 | 高风险、新系统、建立信任期 |
| **Human-in-the-Loop** | 关键节点审批 | 中等 | 生产环境常态 |
| **Human-on-the-Loop** | 监控，按需介入 | 最高 | 低风险、高信任、成熟系统 |

```go
type HITLMode string

const (
    ModeHumanInCommand HITLMode = "human_in_command"  // 每步确认
    ModeHumanInTheLoop HITLMode = "human_in_the_loop" // 关键节点
    ModeHumanOnTheLoop HITLMode = "human_on_the_loop" // 监控模式
)

// 根据模式决定是否需要审批
func (s *Supervisor) NeedsApproval(mode HITLMode, action AgentAction) bool {
    switch mode {
    case ModeHumanInCommand:
        return true  // 每个动作都要审批
    case ModeHumanInTheLoop:
        return action.IsSensitive || action.Cost > s.triggers.SingleActionCostLimit
    case ModeHumanOnTheLoop:
        return false // 只监控，不主动打断
    }
    return false
}
```

**选择建议**：

```
新系统上线 ──────► Human-in-Command（1-2 周）
     │
     │ 完成 50+ 任务，无重大事故
     ▼
逐步放权 ──────► Human-in-the-Loop（常态）
     │
     │ 90 天内 99% 成功率
     ▼
高度信任 ──────► Human-on-the-Loop（可选）
```

### 15.12.4 中断与接管（Interrupt & Takeover）

用户随时可以"抢方向盘"。这是 HITL 最重要的能力之一。

**四种中断操作**：

| 操作 | 说明 | 状态变化 |
|------|------|---------|
| **暂停** (Pause) | 停止执行，保存状态 | Running → Paused |
| **恢复** (Resume) | 继续执行 | Paused → Running |
| **接管** (Takeover) | 人类完成剩余步骤 | Running → HumanControl |
| **取消** (Cancel) | 放弃任务，可选回滚 | Any → Cancelled |

```go
// 扩展 ControlHandler 支持人类接管
type ControlHandler struct {
    // ... 原有字段
    humanTakeover bool
    takeoverChan  workflow.Channel
}

// 人类接管信号处理
func (h *ControlHandler) HandleTakeover(ctx workflow.Context) {
    takeoverSig := workflow.GetSignalChannel(ctx, "human_takeover_v1")
    workflow.Go(ctx, func(ctx workflow.Context) {
        var req TakeoverRequest
        takeoverSig.Receive(ctx, &req)
        h.humanTakeover = true
        h.takeoverReason = req.Reason
        // 通知所有子工作流暂停
        h.PauseAllChildren(ctx)
    })
}

// 在执行循环中检查
func (s *Supervisor) executeWithHITL(ctx workflow.Context, task Task) (Result, error) {
    for {
        // 检查是否被人类接管
        if s.controlHandler.IsHumanTakeover() {
            return Result{
                Status:  "handed_to_human",
                Message: "任务已交给人类处理",
                Context: s.getCurrentContext(),  // 传递当前上下文
            }, nil
        }

        // 检查是否暂停
        if s.controlHandler.IsPaused() {
            workflow.Await(ctx, func() bool { return !s.controlHandler.IsPaused() })
        }

        // 正常执行...
    }
}
```

**接管时的上下文传递**：

```go
type TakeoverContext struct {
    // 当前状态
    CurrentStep     int                    `json:"current_step"`
    TotalSteps      int                    `json:"total_steps"`
    CompletedTasks  []string               `json:"completed_tasks"`
    PendingTasks    []string               `json:"pending_tasks"`

    // 已收集的信息
    IntermediateResults map[string]interface{} `json:"intermediate_results"`

    // 建议的下一步
    SuggestedNextAction string               `json:"suggested_next_action"`

    // 失败原因（如果因为失败而接管）
    FailureReason       string               `json:"failure_reason,omitempty"`
}
```

人类接管后，可以：
1. 查看 Agent 已完成的工作
2. 手动完成剩余步骤
3. 修改计划后让 Agent 继续
4. 直接取消任务

### 15.12.5 审批工作流实现

在 Temporal 中实现审批等待：

```go
// 审批请求
type ApprovalRequest struct {
    TaskID      string                 `json:"task_id"`
    Action      string                 `json:"action"`
    Description string                 `json:"description"`
    Risk        string                 `json:"risk"`       // low/medium/high/critical
    Context     map[string]interface{} `json:"context"`
    Timeout     time.Duration          `json:"timeout"`
}

// 审批响应
type ApprovalResponse struct {
    Approved  bool   `json:"approved"`
    Approver  string `json:"approver"`
    Comment   string `json:"comment,omitempty"`
    Timestamp int64  `json:"timestamp"`
}

// 等待人类审批（带超时）
func (s *Supervisor) WaitForApproval(ctx workflow.Context, req ApprovalRequest) (ApprovalResponse, error) {
    // 1. 发送通知（Slack/Email/Dashboard）
    workflow.ExecuteActivity(ctx, NotifyForApproval, req).Get(ctx, nil)

    // 2. 等待审批信号
    approvalCh := workflow.GetSignalChannel(ctx, "approval_response_v1")

    var response ApprovalResponse
    timeout := workflow.NewTimer(ctx, req.Timeout)

    sel := workflow.NewSelector(ctx)

    // 收到审批
    sel.AddReceive(approvalCh, func(c workflow.ReceiveChannel, more bool) {
        c.Receive(ctx, &response)
    })

    // 超时处理
    sel.AddFuture(timeout, func(f workflow.Future) {
        response = ApprovalResponse{
            Approved:  false,
            Comment:   "审批超时，自动拒绝",
            Timestamp: time.Now().Unix(),
        }
    })

    sel.Select(ctx)

    // 3. 记录审批结果
    workflow.ExecuteActivity(ctx, LogApprovalResult, req, response).Get(ctx, nil)

    return response, nil
}
```

**在 Supervisor 主循环中使用**：

```go
func SupervisorWorkflow(ctx workflow.Context, input TaskInput) (TaskResult, error) {
    // ... 初始化 ...

    for _, subtask := range decomp.Subtasks {
        // 检查是否需要审批
        shouldEscalate, reason := supervisor.ShouldEscalate(ctx, subtask)

        if shouldEscalate {
            approval, err := supervisor.WaitForApproval(ctx, ApprovalRequest{
                TaskID:      subtask.ID,
                Action:      subtask.Action,
                Description: subtask.Description,
                Risk:        subtask.RiskLevel,
                Context:     subtask.Context,
                Timeout:     30 * time.Minute,  // 30 分钟超时
            })

            if err != nil || !approval.Approved {
                // 记录拒绝原因，跳过或终止
                if subtask.Required {
                    return TaskResult{Success: false, ErrorMessage: "必要任务被拒绝: " + reason}, nil
                }
                continue  // 非必要任务，跳过
            }
        }

        // 执行任务
        result, err := supervisor.ExecuteSubtask(ctx, subtask)
        // ...
    }
    // ...
}
```

### 15.12.6 信任升级机制

不能永远 Human-in-Command。随着系统证明自己可靠，应该逐步放权。

```go
type TrustLevel struct {
    Level           string  `json:"level"`            // novice/proficient/expert
    TasksCompleted  int     `json:"tasks_completed"`
    SuccessRate     float64 `json:"success_rate"`
    DaysSinceStart  int     `json:"days_since_start"`
    LastIncident    *time.Time `json:"last_incident,omitempty"`
}

// 信任升级条件
var trustUpgradeRules = map[string]struct {
    MinTasks       int
    MinSuccessRate float64
    MinDays        int
    NoIncidentDays int
}{
    "novice_to_proficient": {
        MinTasks:       50,
        MinSuccessRate: 0.90,
        MinDays:        7,
        NoIncidentDays: 7,
    },
    "proficient_to_expert": {
        MinTasks:       200,
        MinSuccessRate: 0.98,
        MinDays:        30,
        NoIncidentDays: 30,
    },
}

// 检查是否可以升级
func (t *TrustLevel) CanUpgrade() (bool, string) {
    var targetLevel string
    var rules struct {
        MinTasks       int
        MinSuccessRate float64
        MinDays        int
        NoIncidentDays int
    }

    switch t.Level {
    case "novice":
        targetLevel = "proficient"
        rules = trustUpgradeRules["novice_to_proficient"]
    case "proficient":
        targetLevel = "expert"
        rules = trustUpgradeRules["proficient_to_expert"]
    default:
        return false, "已是最高级别"
    }

    if t.TasksCompleted < rules.MinTasks {
        return false, fmt.Sprintf("任务数不足: %d/%d", t.TasksCompleted, rules.MinTasks)
    }
    if t.SuccessRate < rules.MinSuccessRate {
        return false, fmt.Sprintf("成功率不足: %.1f%%/%.1f%%", t.SuccessRate*100, rules.MinSuccessRate*100)
    }
    if t.DaysSinceStart < rules.MinDays {
        return false, fmt.Sprintf("运行天数不足: %d/%d", t.DaysSinceStart, rules.MinDays)
    }
    if t.LastIncident != nil {
        daysSinceIncident := int(time.Since(*t.LastIncident).Hours() / 24)
        if daysSinceIncident < rules.NoIncidentDays {
            return false, fmt.Sprintf("距上次事故天数不足: %d/%d", daysSinceIncident, rules.NoIncidentDays)
        }
    }

    return true, targetLevel
}
```

**信任级别对应的 HITL 模式**：

| 信任级别 | HITL 模式 | 审批范围 | 升级条件 |
|----------|-----------|---------|---------|
| Novice | Human-in-Command | 所有操作 | 初始状态 |
| Proficient | Human-in-the-Loop | 仅敏感操作 | 50 任务 + 90% 成功率 + 7 天无事故 |
| Expert | Human-on-the-Loop | 仅异常情况 | 200 任务 + 98% 成功率 + 30 天无事故 |

> **注意**：信任可以升级，也可以降级。一次严重事故可能让 Expert 直接降回 Novice。这是保护机制，不是惩罚。

### 15.12.7 LangGraph 的 interrupt() 对比

LangGraph 提供了原生的 `interrupt()` 函数实现 HITL：

```python
# LangGraph 的 interrupt() 模式
from langgraph.types import interrupt, Command

def human_approval_node(state):
    """需要人类审批的节点"""
    # 暂停执行，返回审批请求给前端
    human_response = interrupt({
        "question": "是否批准执行以下操作？",
        "action": state["proposed_action"],
        "risk_level": state["risk_assessment"],
        "context": state["context"]
    })

    if human_response["approved"]:
        return Command(goto="execute_action")
    else:
        return Command(
            goto="revise_plan",
            update={"feedback": human_response["feedback"]}
        )

# 构建图
graph = StateGraph(State)
graph.add_node("plan", plan_node)
graph.add_node("human_approval", human_approval_node)
graph.add_node("execute_action", execute_node)
graph.add_node("revise_plan", revise_node)

# 编译时启用 checkpointer（支持中断恢复）
app = graph.compile(checkpointer=MemorySaver())

# 执行到中断点
result = app.invoke(initial_state, config={"configurable": {"thread_id": "task-123"}})

# 恢复执行（带人类输入）
result = app.invoke(
    Command(resume={"approved": True, "feedback": ""}),
    config={"configurable": {"thread_id": "task-123"}}
)
```

**Temporal Signal vs LangGraph interrupt() 对比**：

| 特性 | Temporal Signal | LangGraph interrupt() |
|------|-----------------|----------------------|
| 状态持久化 | 原生支持 | 需要 Checkpointer |
| 超时处理 | Timer + Selector | 需要自己实现 |
| 分布式 | 天然分布式 | 单进程为主 |
| 学习曲线 | 较陡 | 较平缓 |
| 适用场景 | 生产级长时任务 | 原型和中小规模 |

### 15.12.8 实战：带审批的市场分析

回到 15.9 的市场分析例子，加入 HITL：

```
t0:  Supervisor 启动
     ├── 分解任务 → 6 个子任务
     └── 检测到"发布报告"是敏感操作，标记需要审批

t1:  并行执行 A, B, C, D（无需审批的任务）

t5:  Agent E (SWOT) 完成
     ├── 结果送人类审核（可选，高风险场景）
     └── 审批通过 / 修改建议

t6:  Agent F (报告生成) 完成
     ├── 触发升级：发布报告是敏感操作
     ├── 发送审批请求到 Slack
     │   "市场分析报告已生成，是否发布给客户？"
     │   [查看报告] [批准] [拒绝] [修改后重试]
     └── 等待人类响应（30 分钟超时）

t7:  人类点击 [批准]
     ├── 记录审批日志
     └── 执行发布

t8:  任务完成
     └── 更新信任指标：tasks_completed++, success_rate 更新
```

```go
// 市场分析中的敏感操作检测
func detectSensitiveActions(subtasks []Subtask) {
    sensitiveKeywords := []string{"publish", "send", "delete", "pay", "share"}

    for i := range subtasks {
        for _, kw := range sensitiveKeywords {
            if strings.Contains(strings.ToLower(subtasks[i].Action), kw) {
                subtasks[i].RequiresApproval = true
                subtasks[i].RiskLevel = "high"
                break
            }
        }
    }
}
```

---

## 这章讲完了

核心就一句话：**Supervisor 是多 Agent 的管理层——动态招人、协调通信、容忍失败，关键决策升级给人类**。

## 小结

1. **邮箱系统**：Agent 间异步通信，非阻塞发送
2. **动态团队**：运行时招募/退役，策略授权
3. **智能容错**：50%+1 失败阈值，部分失败继续
4. **角色分配**：LLM 指定 Agent 类型
5. **历史学习**：从历史执行学习分解策略
6. **Human-in-the-Loop**：人类是层级最高的节点，通过升级触发器、中断接管、信任升级实现人机协作

---

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- [`supervisor_workflow.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/supervisor_workflow.go)：找 `SupervisorWorkflow` 函数整体结构、`mailbox_v1` 信号处理、`recruit_v1` 动态招募实现、失败计数逻辑（`failedTasks` 和 `maxFailures`）

### 选读深挖（2 个，按兴趣挑）

- 团队目录实现：搜索 `teamAgents` 变量，理解 `listTeamAgents` 和 `findTeamAgentsByRole` 查询
- 与 DAG 的协作：搜索 `isSimpleTask` 判断逻辑，理解什么情况下委托给 DAGWorkflow

---

## 练习

### 练习 1：邮箱系统设计

设计一个场景：研究员发现重要信息，需要通知分析师调整分析重点。

要求：
1. 画出消息流程图
2. 写出 MailboxMessage 的内容
3. 分析师收到消息后怎么处理？

### 练习 2：失败阈值分析

假设有 8 个子任务，分析以下情况：

1. 2 个失败 → 工作流会继续还是中止？
2. 4 个失败 → 会怎样？
3. 如果是关键业务任务，你会怎么调整阈值？

### 练习 3（进阶）：动态招募策略

设计一个「智能招募」策略：

场景：产品对比分析时，发现需要深度分析定价策略。

要求：
1. 什么条件触发招募？
2. 怎么判断应该招募什么角色？
3. 怎么防止过度招募？

---

## 想深入？

- [Temporal Signals](https://docs.temporal.io/develop/go/message-passing#signals) - 理解信号机制
- [Temporal Query Handlers](https://docs.temporal.io/develop/go/message-passing#queries) - 查询处理器
- [AutoGen GroupChat](https://microsoft.github.io/autogen/docs/tutorial/conversation-patterns) - 对话式多 Agent

---

## 下一章预告

Supervisor 管理团队，DAG 调度任务。但还有一个问题没解决：

Agent A 完成后，怎么把数据精确传给 Agent B？

- 简单场景：把结果塞进上下文
- 复杂场景：工作空间共享
- 更复杂：P2P 协议

下一章讲 **Handoff 机制**——让 Agent 之间能精确可靠地传递数据和状态。

接下来我们看...
