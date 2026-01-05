# 第 2 章：ReAct 循环

> **ReAct 是 Agent 的心脏：思考一步，行动一步，观察结果，再思考下一步——它让 AI 更少瞎编、更像在干活。**  
> **但它不是正确保证；预算、终止条件和验收，才是生产里真正的底线。**

---

上一章说了，Agent 的核心是"自己干活"。

但具体怎么干？它脑子里在想什么？每一步是怎么决定的？

答案就是 **ReAct**：Reasoning + Acting，推理与行动的交织循环。

这是 Agent 最核心的运行机制。搞懂它，你就搞懂了 Agent 的心脏。

---

## 2.1 从一个真实场景说起

假设你让 Agent："帮我调查竞争对手的定价策略。"

### 普通 ChatGPT 的做法

> "定价策略通常包括成本加成法、竞争定价法、价值定价法..."

这段话不算错，但它在回避你的问题：你要的是“对手是谁、多少钱、有哪些套餐/限制”，不是“定价策略百科”。

它既不知道你的竞争对手是谁，也没有去查。所以你得到的是“听起来合理，但无法行动”的答案。

### ReAct Agent 的做法

```
[第 1 轮]
思考：用户说“竞争对手”，但没说是谁。先把对象确认，不然我查到的都是垃圾。
行动：向用户追问：你指的竞争对手是哪几家？（给我名字或官网）
观察：用户回复：主要看「竞争对手 A」

[第 2 轮]
思考：好，有名字了。先找它的官方定价页，优先官网，其次文档/FAQ。
行动：调用搜索工具，query="竞争对手 A pricing"
观察：找到官网定价页 + 两篇评测文章

[第 3 轮]
思考：去读官网定价页，拿到准确数字和套餐差异。
行动：调用网页读取工具，访问官网定价页
观察：基础版 $29/月，专业版 $99/月，企业版需联系销售；功能差异清单…

[第 4 轮]
思考：把信息整理成对比，并补上“我们 vs 它”的差异点。
行动：生成分析报告（附来源链接）
观察：报告生成完成

[输出]
竞争对手 A 定价策略分析：
- 基础版：$29/月，面向个人用户
- 专业版：$99/月，面向小团队
- 企业版：联系销售，面向大客户
采用阶梯定价策略，与我们的 $49/月 定价相比...
```

看到区别没？

它不是一口气蒙一个答案，而是**边想边做，做完再想**。

这就是 ReAct 的核心：**Reason（思考）→ Act（行动）→ Observe（观察）**，循环往复，直到任务完成。

> **注意**：ReAct 不是魔法。它做的不是"自动变正确"，而是把"猜"变成"查 + 证据 + 可回滚"。它依然会犯错，但你能看到它错在哪、也更容易把它拉回来。

---

## 2.2 ReAct 的本质：为什么这个循环有效？

### LLM 的天生缺陷

LLM 天生是"一口气说完"的——你问一句，它生成一整段。它不会停下来想"我刚才说的对不对"，也不会中途去查资料。

这导致两个问题：

| 问题 | 表现 | 后果 |
|------|------|------|
| **信息过时** | 只能用训练数据里的知识 | 回答可能是几个月前的旧信息 |
| **无法验证** | 说完就完，不会检查 | 编造细节，自信地胡说八道 |

### ReAct 怎么解决

ReAct 强迫 LLM 停下来：

```
不是：问题 → 一口气生成答案
而是：问题 → 想一步 → 做一步 → 看结果 → 再想 → 再做 → ... → 答案
```

这带来三个关键好处：

| 好处 | 说明 | 例子 |
|------|------|------|
| **可以获取新信息** | 不是只靠已有知识瞎编，可以去查 | 搜索最新的产品定价 |
| **可以修正错误** | 发现走错路，可以回头 | 搜索结果不对，换个关键词再搜 |
| **可以追溯过程** | 每一步都有记录，出了问题知道在哪 | Debug 时能看到哪一步出错 |

### 学术背景

ReAct 来自 2022 年的一篇论文：[ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)。

核心发现很简单：

> **推理和行动交织进行，比单独用任何一个都强。**

- 只推理不行动（Chain-of-Thought）：想得很好，但拿不到新信息
- 只行动不推理（直接 Function Calling）：瞎调工具，不知道为什么调
- 推理 + 行动（ReAct）：想清楚为什么要做，做完看结果，再决定下一步

这个洞察奠定了现代 Agent 的基础。

---

## 2.3 三个阶段，逐个拆解

```
┌──────────────────────────────────────────────────────────────┐
│                        ReAct 循环                            │
│                                                              │
│    ┌──────────┐     ┌──────────┐     ┌──────────┐           │
│    │  REASON  │ ──▶ │   ACT    │ ──▶ │ OBSERVE  │           │
│    │   思考    │     │   行动    │     │   观察    │           │
│    └──────────┘     └──────────┘     └──────────┘           │
│         ▲                                  │                 │
│         │                                  │                 │
│         └────────── 循环继续 ◀──────────────┘                 │
│                                                              │
│    终止条件：任务完成 / 达到最大轮数 / 无新进展                  │
└──────────────────────────────────────────────────────────────┘
```

### Reason（思考）

分析现在的情况，决定下一步干什么。

```
输入：用户目标 + 历史观察结果
输出：下一步要做什么，为什么
```

**关键原则：只想一步。** 不要让 LLM 想太远，它会发散。告诉它："基于当前信息，你下一个动作是什么？"

### Act（行动）

调用工具，执行动作。

```
输入：思考阶段决定的动作
输出：执行结果
```

> **提示**：一轮只推进一个关键动作。前期调试时尤其重要：动作越小，越容易定位问题。等你把流程跑顺了，再考虑并行调用工具提速。

常见的行动类型：
- 追问/确认（Clarify）
- 搜索（Web Search）
- 读文件（Read File）
- 写文件（Write File）
- 调用 API（HTTP Request）
- 执行代码（Code Execution）

### Observe（观察）

记录执行结果，喂给下一轮的 Reason 阶段。

```
输入：行动的执行结果
输出：结构化的观察记录
```

**关键原则：观察要客观。** 不要在这个阶段做判断，只记录事实。判断留给下一轮的 Reason。

---

## 2.4 什么时候该停？

这是 ReAct 最关键的问题之一。停太早，任务没完成；停太晚，Token 烧光了。

### 六种终止条件（两类是硬护栏）

| 条件 | 说明 | 优先级 |
|------|------|--------|
| **用户中断** | 用户主动停止 | 最高 |
| **任务完成** | LLM 明确表示任务已完成 | 高 |
| **预算耗尽** | 达到 token/成本上限，强制停止 | 高（护栏） |
| **超时** | 达到端到端时延上限，强制停止 | 高（护栏） |
| **结果收敛** | 连续两次观察结果很像，没有新进展 | 中 |
| **最大轮数** | 达到预设的 MaxIterations | 兜底 |

### 判断逻辑示例

```go
func shouldStop(state *ReactState) bool {
    // 0. 用户中断，立刻停
    if state.UserCanceled {
        return true
    }

    // 0.5 预算/超时是硬护栏
    if state.TokensUsed >= state.Config.TokenBudget {
        return true
    }
    if time.Since(state.StartTime) > state.Config.Timeout {
        return true
    }

    // 1. 达到最大轮数，强制停止
    if state.Iteration >= state.Config.MaxIterations {
        return true
    }

    // 2. 还没到最小轮数，继续
    if state.Iteration < state.Config.MinIterations {
        return false
    }

    // 3. LLM 明确说完成了
    if containsCompletionSignal(state.LastThought) {
        return true
    }

    // 4. 结果收敛，没有新进展
    if isResultConverged(state.Observations) {
        return true
    }

    return false
}
```

---

## 2.5 一个完整的调试案例

假设用户说："帮我查查为什么 API 返回 500 错误。"

### 循环执行过程

```
[第 1 轮]
思考：500 错误通常是服务端问题，需要看日志
行动：执行 tail -n 50 /var/log/app.log
观察：发现 NullPointerException at UserService.java:45

[第 2 轮]
思考：找到 NPE 了，位置在 UserService.java 第 45 行，去看看代码
行动：读取 src/UserService.java 第 40-50 行
观察：第 45 行是 user.getProfile().getName()

[第 3 轮]
思考：问题清楚了，user.getProfile() 可能返回 null，没做空值检查
行动：生成修复建议
观察：建议在访问前添加空值判断

[第 4 轮]
思考：问题定位了，方案也有了，任务完成
行动：输出最终报告
观察：报告生成完成

[终止：LLM 表示任务完成]
```

### 最终输出

```
问题分析：
API 返回 500 错误是因为 UserService.java 第 45 行存在空指针异常。
当用户没有设置 profile 时，getProfile() 返回 null，
直接调用 getName() 导致 NullPointerException。

修复建议：
在访问 profile 前添加空值检查：
if (user.getProfile() != null) {
    String name = user.getProfile().getName();
}
或使用 Optional：
String name = Optional.ofNullable(user.getProfile())
    .map(Profile::getName)
    .orElse("Unknown");
```

四轮循环，问题定位了，方案也有了。

**如果没有 ReAct 呢？** LLM 可能会编一堆"常见的 500 原因"——数据库连接失败、内存不足、配置错误——但没一个是你实际的问题。

---

## 2.6 生产环境的关键配置

生产环境里，ReAct 不是“跑起来就行”，而是“跑得住、停得下、出了事能复盘”。

我喜欢用 Shannon 来讲这套配置，因为它把常见事故直接写进了代码里：你能在源码里看到它是怎么防“无限循环 / 过早收工 / 上下文爆炸 / 无证据结论”的。

在 Shannon 里，你主要会遇到两层参数：
- **循环参数**（ReAct 自己的形态）：`ReactConfig`
- **预算护栏**（跨模式通用）：`Options.BudgetAgentMax` + workflow/activity 的超时

Shannon 的 `ReactConfig` 定义在 [`patterns/react.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/react.go)（节选）：

```go
type ReactConfig struct {
    MaxIterations     int
    MinIterations     int
    ObservationWindow int
    // MaxObservations / MaxThoughts / MaxActions ...
}
```

而 Token 预算不放在 `ReactConfig` 里，而是放在通用的 `Options`（[`patterns/options.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/options.go)）：

```go
type Options struct {
    BudgetAgentMax int
    // ...
}
```

我挺喜欢这个分法：因为预算不是 ReAct 专属的，Chain-of-Thought、Debate、Tree-of-Thoughts 也都得受同一套预算约束。

### 为什么需要 MaxIterations？

我见过 Agent 卡在一个搜索结果上反复转，烧掉几万 token 还没停。

**真实案例**：Agent 搜索"Python 安装教程"，第一个结果是广告页，它读完发现没用，再搜，又是同一个广告页（因为搜索词没变），再读，又没用... 循环 20 次，什么都没产出。

所以必须有硬限制。生产环境建议 MaxIterations = 10-15。

### 为什么需要 MinIterations？

有些任务，Agent 第一轮就说"搞定了"，其实啥都没干。

**真实案例**：用户问"帮我查下明天北京天气"，Agent 回答"好的，明天北京天气晴，温度 25 度"——但它根本没调用天气 API，这是它编的。

强制 MinIterations = 1，确保至少做一次真正的工具调用。

### 为什么需要 ObservationWindow？

观察历史越积越长，上下文越来越大，token 费用失控。

```go
// 只保留最近 5 条观察
recentObservations := observations[max(0, len(observations)-5):]
```

老的观察可以压缩成摘要，保留关键信息，丢弃细节。

### Shannon 额外做了两件“很生产”的事

1. **提前停（不是等烧完 MaxIterations）**：`shouldStopReactLoop` 会检测“结果收敛/没有新信息”，比如连续两次 observation 很像就停（源码里有个很便宜但有效的 `areSimilar`）
2. **要证据再收工**：在 research 模式下，它会检查 `toolExecuted`、`observations` 等条件，避免模型没查就说“完成了”

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- [`patterns/react.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/react.go)：搜 `Phase 1/2/3` + `shouldStopReactLoop`，把循环和"提前停"的理由对上号

### 选读深挖（2 个，按兴趣挑）

- [`patterns/options.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/options.go)：看 `BudgetAgentMax` 为什么归到通用 Options（而不是塞进 ReactConfig）
- [`strategies/react.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/strategies/react.go)：看 ReactWorkflow 怎么加载配置、注入 memory、再调用 ReactLoop 跑起来

---

## 2.7 常见的坑

### 坑 1：无限循环

**症状**：Agent 反复做同一件事，停不下来。

**原因**：搜索词不变，结果不变，但 Agent 没意识到自己在重复。

**解决**：
- 加 MaxIterations 硬限制
- 加相似性检测：如果连续两次观察结果高度相似，强制停止或换策略（Shannon 里就有 `areSimilar` 这种 cheap heuristic）
- 在 Prompt 里提醒："如果你发现结果和上一次一样，请换一个方法"

### 坑 2：过早放弃

**症状**：Agent 第一轮就说"完成了"，其实啥都没干。

**原因**：LLM 偷懒，直接用已有知识编答案。

**解决**：
- 加 MinIterations，强制至少做一次工具调用
- 在 Prompt 里明确："你必须使用工具获取信息，不能直接回答"

### 坑 3：Token 爆炸

**症状**：几轮下来，上下文长度暴涨，费用失控。

**原因**：每次观察都完整保留，历史越积越长。

**解决**：
- 限制 ObservationWindow，只看最近几条
- 对老的观察做摘要压缩
- 设置预算护栏（例如 Shannon 的 `Options.BudgetAgentMax`）

### 坑 4：思考和行动脱节

**症状**：LLM 想的是一回事，做的是另一回事。

**原因**：Reason 和 Act 阶段的 Prompt 没有衔接好。

**解决**：在 Act 阶段明确引用 Reason 的输出：

```
你刚才的思考是：{thought}
请根据这个思考，执行对应的行动。
```

---

## 2.8 其他框架怎么做？

ReAct 是通用模式，不是 Shannon 专属。各家都有实现：

| 框架 | 实现方式 | 特点 |
|------|----------|------|
| **LangChain** | `create_react_agent()` | 最广泛使用，生态丰富 |
| **LangGraph** | 状态图 + 节点 | 可视化调试，流程可控 |
| **OpenAI** | Function Calling | 原生支持，延迟低 |
| **Anthropic** | Tool Use | Claude 原生支持 |
| **AutoGPT** | 自定义循环 | 高度自主，但不稳定 |

核心逻辑都一样：思考 → 行动 → 观察 → 循环。

差别在于：
- 工具定义的格式（JSON Schema vs 自定义格式）
- 循环控制的粒度（框架控制 vs 用户控制）
- 生态集成（向量库、监控、持久化）

选哪个？看你的场景。快速原型用 LangChain，生产系统考虑 LangGraph 或自建。

---

## 2.9 本章要点回顾

1. **ReAct 定义**：Reasoning + Acting，推理与行动的交织循环
2. **三个阶段**：Reason（思考）→ Act（行动）→ Observe（观察）
3. **为什么有效**：让 LLM 能获取新信息、修正错误、追溯过程
4. **终止条件**：任务完成 / 结果收敛 / 最大轮数 / 用户中断 + 预算/超时护栏
5. **关键配置**：MaxIterations（防无限循环）、MinIterations（防偷懒）、ObservationWindow（控成本）+ Budget/Timeout（硬护栏）

---

## 2.10 延伸阅读

- **ReAct 论文**：[ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) - 原始论文，理解设计动机
- **LangChain ReAct**：[官方文档](https://python.langchain.com/docs/modules/agents/agent_types/react) - 最流行的实现
- **Chain-of-Thought 对比**：[Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903) - 理解"只推理不行动"的局限
- **Shannon Pattern Guide**：[`docs/pattern-usage-guide.md`](https://github.com/Kocoro-lab/Shannon/blob/main/docs/pattern-usage-guide.md)（从使用者视角看各类推理模式怎么配）

---

## 下一章预告

你可能会问：Agent 靠什么"做"？那些搜索、读文件、调 API 的能力从哪来？

这就是下一章的内容——**工具调用基础**。

工具是 Agent 的手脚。ReAct 告诉它怎么思考，工具让它能真正动手。

没有工具的 Agent，就像没有手的人——想得再好，也干不成事。

