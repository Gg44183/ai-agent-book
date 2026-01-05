# 第 18 章：Debate 模式

> **Debate 让多个 Agent 从不同立场辩论，通过对抗暴露论证弱点——但它不是万能的；设计不好的辩论只是在浪费 Token。**

---

你问 Agent：

> "AI 会取代人类工作吗？"

它回答：

> "是的，AI 会取代很多工作，但也会创造新工作。历史上每次技术革命都是如此。总体来说，我们应该保持乐观..."

一个两边讨好、没有立场的答案。

问题不是答案本身错了，而是它只从一个角度想了一次，就给了结论。它没有质疑自己，没有考虑反面，更没有在对抗中检验论证强度。

我第一次意识到这个问题的严重性，是在帮一个投资机构做行业研究。Agent 给出了一个看似合理的结论，但当我手动扮演"反方"追问时，它的论证一下就垮了。

**Debate 模式就是把这个"手动追问"自动化——让多个 Agent 从不同立场辩论，乐观派、怀疑派、实用派各抒己见，在对抗中暴露弱点，最终形成更可靠的结论。**

---

## 18.1 为什么单 Agent 回答争议性问题容易出问题

单 Agent 回答争议性问题时，有三个典型问题：

| 问题 | 表现 | 后果 |
|------|------|------|
| **确认偏差** | 倾向于第一个想到的答案 | 忽略反面证据 |
| **过度自信** | 缺乏质疑和反思 | 论证漏洞没人指出 |
| **视角单一** | 没有考虑其他立场 | 结论偏颇，适用性差 |

Debate 怎么解决这些问题？

```
乐观派：AI 将创造比它取代的更多高质量工作，历史上每次技术革命都验证了这一点...

怀疑派：等等，这次不一样。以前的自动化替代的是体力劳动，这次替代的是认知劳动。而且转型速度太快，社会来不及适应...

实用派：你们都有道理。关键问题不是"会不会取代"，而是"转型多快"和"谁来为再培训买单"...

主持人综合：AI 对就业的影响取决于三个变量：技术进步速度、政策响应速度、再培训体系效率...
```

核心价值：

1. **多视角覆盖**：避免盲点
2. **对抗性质疑**：暴露论证弱点
3. **共识形成**：综合后的结论更可靠

---

## 18.2 什么时候该用 Debate

Debate 不是万能的。用错场景，只是在浪费 Token。

| 场景 | 为什么适合 | 为什么不适合 |
|------|-----------|-------------|
| 政策分析 | 需要权衡多方利益 | 有明确对错的法规解读 |
| 投资决策 | 需要考虑多种市场情景 | 纯粹的数学计算 |
| 产品方向 | 需要平衡技术可行性和商业价值 | 已经有明确需求的功能实现 |
| 伦理讨论 | 需要多角度道德审视 | 有明确行业规范的合规问题 |
| 争议话题 | 需要呈现不同立场 | 有客观答案的事实性问题 |

一个简单的判断方法：

> 如果你把这个问题发到一个专业论坛，会不会引发激烈争论？
> 如果会，那大概率适合 Debate。
> 如果大家会给出一致的答案，那用 Debate 就是浪费。

---

## 18.3 Debate 的三阶段流程

Shannon 的 Debate 实现分三个阶段：

```
Phase 1: 初始立场陈述
┌──────────────────────────────────────────┐
│  Debater-1    Debater-2    Debater-3    │
│  (乐观派)     (怀疑派)     (实用派)      │
│     ↓           ↓           ↓           │
│  立场A        立场B        立场C         │
└──────────────────────────────────────────┘
              ↓
Phase 2: 多轮辩论
┌──────────────────────────────────────────┐
│  Round 1: 各方质疑对方、加强己方         │
│  Round 2: 回应质疑、寻找共同点           │
│  Round 3: 收敛观点、形成初步共识         │
└──────────────────────────────────────────┘
              ↓
Phase 3: 解决阶段
┌──────────────────────────────────────────┐
│  主持人综合 / 投票决定 / 直接合成        │
└──────────────────────────────────────────┘
```

---

## 18.4 核心配置

```go
type DebateConfig struct {
    NumDebaters      int      // 辩论者数量 (2-5)
    MaxRounds        int      // 最大辩论轮次
    Perspectives     []string // 不同视角列表
    RequireConsensus bool     // 是否要求达成共识
    ModeratorEnabled bool     // 是否启用主持人
    VotingEnabled    bool     // 是否启用投票机制
    ModelTier        string   // 模型层级
}
```

参数调优建议：

| 参数 | 建议值 | 理由 |
|------|--------|------|
| `NumDebaters` | 3 | 太少无对抗，太多难协调 |
| `MaxRounds` | 2-3 | 超过 3 轮容易陷入无限循环 |
| `Perspectives` | 对立+中立 | 确保形成真正对抗 |
| `RequireConsensus` | false | 强制共识可能导致无限循环 |
| `VotingEnabled` | true | 无法共识时用投票兜底 |

---

## 18.5 Phase 1：初始立场

并行让每个 Agent 从自己的视角陈述立场：

```go
func Debate(
    ctx workflow.Context,
    query string,
    context map[string]interface{},
    sessionID string,
    history []string,
    config DebateConfig,
    opts Options,
) (*DebateResult, error) {

    // 默认视角
    if len(config.Perspectives) == 0 {
        config.Perspectives = generateDefaultPerspectives(config.NumDebaters)
    }

    // 并行获取初始立场
    futures := make([]workflow.Future, config.NumDebaters)

    for i := 0; i < config.NumDebaters; i++ {
        perspective := config.Perspectives[i]
        agentID := fmt.Sprintf("debater-%d-%s", i+1, perspective)

        initialPrompt := fmt.Sprintf(
            "As a %s perspective, provide your position on: %s\n" +
            "Be specific and provide strong arguments.",
            perspective, query,
        )

        futures[i] = workflow.ExecuteActivity(ctx,
            activities.ExecuteAgent,
            activities.AgentExecutionInput{
                Query:     initialPrompt,
                AgentID:   agentID,
                Mode:      "debate",
                SessionID: sessionID,
            })
    }

    // 收集立场
    var positions []DebatePosition
    for i, future := range futures {
        var result AgentResult
        future.Get(ctx, &result)

        positions = append(positions, DebatePosition{
            AgentID:    fmt.Sprintf("debater-%d", i+1),
            Position:   result.Response,
            Arguments:  extractArguments(result.Response),
            Confidence: 0.5,  // 初始置信度
        })
    }

    // 继续 Phase 2...
}
```

Shannon 的默认视角生成：

```go
func generateDefaultPerspectives(num int) []string {
    perspectives := []string{
        "optimistic",   // 乐观派
        "skeptical",    // 怀疑派
        "practical",    // 实用派
        "innovative",   // 创新派
        "conservative", // 保守派
    }

    if num <= len(perspectives) {
        return perspectives[:num]
    }
    return perspectives
}
```

**重点**：视角设计是 Debate 成败的关键。

如果你设计的视角是 "positive"、"very-positive"、"somewhat-positive"——那不是辩论，是合唱团。

好的视角设计要形成真正的对抗：

| 话题类型 | 推荐视角组合 |
|----------|-------------|
| 技术选型 | 技术优先派 + 成本优先派 + 风险规避派 |
| 投资决策 | 激进派 + 保守派 + 套利派 |
| 产品方向 | 用户体验派 + 技术可行派 + 商业价值派 |
| 政策分析 | 受益方 + 受损方 + 中立方 |

---

## 18.6 Phase 2：多轮辩论

每个辩论者看到其他人的立场后，进行回应：

```go
for round := 1; round <= config.MaxRounds; round++ {
    roundFutures := make([]workflow.Future, len(positions))

    for i, debater := range positions {
        // 收集其他人的立场
        othersPositions := []string{}
        for j, other := range positions {
            if i != j {
                othersPositions = append(othersPositions,
                    fmt.Sprintf("%s argues: %s", other.AgentID, other.Position))
            }
        }

        responsePrompt := fmt.Sprintf(
            "Round %d: Consider these other perspectives:\n%s\n\n" +
            "As %s, respond with:\n" +
            "1. Counter-arguments to opposing views\n" +
            "2. Strengthen your position\n" +
            "3. Find any common ground\n",
            round, strings.Join(othersPositions, "\n"), debater.AgentID,
        )

        roundFutures[i] = workflow.ExecuteActivity(ctx,
            activities.ExecuteAgent,
            activities.AgentExecutionInput{
                Query:   responsePrompt,
                AgentID: debater.AgentID,
                Context: map[string]interface{}{
                    "round":           round,
                    "other_positions": othersPositions,
                },
            })
    }

    // 收集本轮回应，更新立场和置信度
    for i, future := range roundFutures {
        var result AgentResult
        future.Get(ctx, &result)

        positions[i].Position = result.Response
        positions[i].Confidence = calculateArgumentStrength(result.Response)
    }

    // 共识检测
    if config.RequireConsensus && checkConsensus(positions) {
        break
    }
}
```

### 论点强度评估

Shannon 用启发式方法评估论证强度：

```go
func calculateArgumentStrength(response string) float64 {
    strength := 0.5

    lower := strings.ToLower(response)

    // 证据支持 (+0.15)
    if strings.Contains(lower, "evidence") ||
       strings.Contains(lower, "study") ||
       strings.Contains(lower, "data") {
        strength += 0.15
    }

    // 逻辑结构 (+0.1)
    if strings.Contains(response, "therefore") ||
       strings.Contains(response, "because") {
        strength += 0.1
    }

    // 反驳对方 (+0.15)
    if strings.Contains(lower, "however") ||
       strings.Contains(lower, "although") {
        strength += 0.15
    }

    // 具体例证 (+0.1)
    if strings.Contains(lower, "for example") ||
       strings.Contains(lower, "such as") {
        strength += 0.1
    }

    return math.Min(1.0, strength)
}
```

这个评估器不完美——它只看关键词，不理解语义。但对于多数场景，够用了。

### 共识检测

检测多数是否趋于一致：

```go
func checkConsensus(positions []DebatePosition) bool {
    agreementCount := 0
    for _, pos := range positions {
        lower := strings.ToLower(pos.Position)
        if strings.Contains(lower, "agree") ||
           strings.Contains(lower, "consensus") ||
           strings.Contains(lower, "common ground") {
            agreementCount++
        }
    }
    // 多数同意则认为达成共识
    return agreementCount > len(positions)/2
}
```

---

## 18.7 Phase 3：解决阶段

三种解决方式：

```go
if config.ModeratorEnabled {
    // 主持人综合各方观点
    result.FinalPosition = moderateDebate(ctx, positions, query)
} else if config.VotingEnabled {
    // 投票决定
    result.FinalPosition, result.Votes = conductVoting(positions)
} else {
    // 直接合成最强论点
    result.FinalPosition = synthesizePositions(positions, query)
}
```

### 投票机制

基于置信度的投票：

```go
func conductVoting(positions []DebatePosition) (string, map[string]int) {
    votes := make(map[string]int)

    winner := positions[0]
    for _, pos := range positions {
        votes[pos.AgentID] = int(pos.Confidence * 100)
        if pos.Confidence > winner.Confidence {
            winner = pos
        }
    }

    return winner.Position, votes
}
```

### 立场综合

找到最强论点，合成最终结论：

```go
func synthesizePositions(positions []DebatePosition, query string) string {
    // 找最强立场
    strongest := positions[0]
    for _, pos := range positions {
        if pos.Confidence > strongest.Confidence {
            strongest = pos
        }
    }

    // 收集所有论点
    allArguments := []string{}
    for _, pos := range positions {
        allArguments = append(allArguments, pos.Arguments...)
    }

    // 构建综合
    synthesis := fmt.Sprintf("After debate on '%s':\n\n", query)
    synthesis += fmt.Sprintf("Strongest Position: %s\n\n", strongest.Position)
    synthesis += "Key Arguments:\n"
    for i, arg := range allArguments[:min(5, len(allArguments))] {
        synthesis += fmt.Sprintf("- %s\n", arg)
    }

    return synthesis
}
```

---

## 18.8 实战示例

**任务**：分析 "AI Agent 会在 2025 年取代 SaaS 吗？"

**配置**：

```go
config := DebateConfig{
    NumDebaters:      3,
    MaxRounds:        2,
    Perspectives:     []string{"tech-optimist", "risk-aware", "market-focused"},
    RequireConsensus: false,
    VotingEnabled:    true,
}
```

**辩论过程**：

```
=== Phase 1: 初始立场 ===

tech-optimist (confidence: 0.75):
  AI Agents 能提供个性化、自动化的端到端解决方案。
  传统 SaaS 的通用界面和手动工作流将被淘汰。
  多个成功案例已经证明这一趋势不可逆转...

risk-aware (confidence: 0.80):
  当前 Agent 的可靠性和企业安全标准不足。
  SaaS 经过多年优化的稳定性难以替代。
  企业采纳周期通常需要 3-5 年...

market-focused (confidence: 0.70):
  关键是定价和商业模式的转变。
  Agent-as-a-Service 会是 SaaS 的演进而非替代。
  市场份额转移需要生态系统重建...

=== Phase 2: Round 1 ===

tech-optimist:
  回应 risk-aware 的安全顾虑，指出沙箱和策略控制的进展...
  但承认企业采纳确实需要时间...

risk-aware:
  承认技术进步，但强调合规和审计的现实约束...
  引用多个企业 IT 采购周期的数据...

market-focused:
  寻找共同点，预测混合模式将成为过渡期的主流...
  提出"Agent-enhanced SaaS"作为中间形态...

=== Phase 2: Round 2 ===

各方开始收敛，形成初步共识：
- 技术方向明确（Agent 是趋势）
- 时间表需要调整（2025 太乐观）
- 形态会是演进而非替代

=== Phase 3: 解决 ===

Votes: {tech-optimist: 75, risk-aware: 80, market-focused: 70}
Winner: risk-aware

Final Position:
AI Agent 将成为 SaaS 的增强层而非替代品。
短期内（2025）企业将谨慎采纳，主要在低风险场景试点。
完全替代需要解决可靠性、安全性、合规性三大问题，
预计需要 3-5 年的过渡期。
```

---

## 18.9 学习与持久化

辩论结果可以持久化，用于后续学习：

```go
workflow.ExecuteActivity(ctx, activities.PersistDebateConsensus,
    activities.PersistDebateConsensusInput{
        SessionID:        sessionID,
        Topic:            query,
        WinningPosition:  result.FinalPosition,
        ConsensusReached: result.ConsensusReached,
        Confidence:       bestConfidence,
        Positions:        positionTexts,
        Metadata: map[string]interface{}{
            "rounds":       result.Rounds,
            "num_debaters": config.NumDebaters,
        },
    })
```

Shannon 会记录：
- 哪些话题容易达成共识
- 哪些视角组合最有效
- 哪些论证模式最强

这些数据可以用来优化未来的辩论策略。

---

## 18.10 常见的坑

### 坑 1：假辩论

```go
// 错误：视角太相似，没有真正对抗
config.Perspectives = []string{"positive", "very-positive", "somewhat-positive"}
// 结果：三个 Agent 互相点头，没有任何质疑

// 正确：形成真正对抗
config.Perspectives = []string{"optimistic", "skeptical", "practical"}
```

这是最常见的错误。如果你发现辩论结果和单 Agent 回答差不多，那多半是视角设计出了问题。

### 坑 2：无限循环

```go
// 错误：强制共识 + 无限轮次
config := DebateConfig{
    RequireConsensus: true,
    MaxRounds:        100,  // 可能永远达不成共识
}
// 结果：Token 烧光也没结论

// 正确：合理限制 + 兜底机制
config := DebateConfig{
    RequireConsensus: false,
    MaxRounds:        3,
    VotingEnabled:    true,  // 无法共识就投票
}
```

### 坑 3：Token 爆炸

```go
// 错误：每轮累积全部历史
for round := 1; round <= config.MaxRounds; round++ {
    prompt := buildPrompt(fullDebateHistory)  // 越来越长！
}
// 结果：第 3 轮的上下文可能已经超过模型限制

// 正确：滑动窗口 + 摘要
recentHistory := debateHistory[max(0, len(debateHistory)-6):]
summary := summarizeHistory(debateHistory, maxTokens)
```

### 坑 4：忽略少数派

即使没达成共识，少数派的观点也可能有价值：

```go
if !result.ConsensusReached {
    result.MinorityPositions = extractMinorityViews(positions)
    // 可能是风险预警，不该忽视
}
```

我见过一个案例：怀疑派在辩论中"输"了，但它指出的安全风险后来真的发生了。

---

## 18.11 Debate vs 其他模式

| 模式 | 适用场景 | 结果特点 | 成本 |
|------|----------|----------|------|
| **Debate** | 有争议话题、需要多角度 | 综合多方观点，可能有分歧 | 高（N*M 次调用）|
| **ToT** | 探索解决路径 | 找到最优单一方案 | 高（树形探索）|
| **Reflection** | 改进已有回答 | 迭代优化同一方向 | 中（2-3 轮）|
| **Ensemble** | 提升鲁棒性 | 多数投票/加权平均 | 中（N 次并行）|

**我的选择逻辑**：

1. 问题有客观答案 → 不用 Debate
2. 问题有争议，需要多视角 → Debate
3. 已有答案但质量不够 → Reflection
4. 需要探索多条解决路径 → ToT

---

## 小结

核心就一句话：**Debate 让多个 Agent 从不同立场辩论，通过对抗暴露弱点，综合形成更可靠的结论**。

但它不是万能的。设计不好的辩论只是在浪费 Token。

要点：

1. **视角设计**：确保形成真正对抗，不是假辩论
2. **多轮收敛**：每轮回应对方、寻找共同点
3. **解决机制**：主持人综合、投票、或直接合成
4. **合理限制**：MaxRounds 和 VotingEnabled 防止无限循环
5. **保留少数派**：少数意见可能是重要预警

---

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- [`patterns/debate.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/debate.go)：找 `Debate` 函数，看三个 Phase 怎么串联；找 `calculateArgumentStrength` 看论点评估逻辑

### 选读深挖（2 个，按兴趣挑）

- [`activities/consensus_memory.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/activities/consensus_memory.go)：看 `PersistDebateConsensus`，理解辩论结果怎么持久化用于学习
- [`patterns/reflection.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/reflection.go)：对比 Debate（多视角对抗）和 Reflection（自我迭代）的区别

---

## 练习

### 练习 1：视角设计

为以下话题设计 3 个辩论视角，确保形成真正对抗：

1. "公司应该全面采用远程办公吗？"
2. "初创公司应该自建基础设施还是用云服务？"
3. "AI 生成的代码应该用在生产环境吗？"

### 练习 2：配置调优

如果你发现辩论进行了 3 轮，但三个 Agent 始终没有达成共识，而且最终投票结果非常接近（分数差不到 5%），你会怎么处理？

给出至少两种解决方案。

### 练习 3（进阶）：改进论点评估

Shannon 的 `calculateArgumentStrength` 只看关键词。设计一个改进版：

1. 增加哪些评估维度？
2. 怎么避免模型"套话"获取高分？
3. 成本收益分析：改进值得吗？

---

## 想深入？

- [Improving Factuality and Reasoning in Language Models through Multiagent Debate](https://arxiv.org/abs/2305.19118) - Du et al., 2023，Debate 模式的理论基础
- 博弈论基础：辩论作为零和博弈 vs 合作博弈
- 批判性思维：论证结构分析（前提、推理、结论）

---

## 下一章预告

Debate 解决的是"多视角怎么综合"的问题。但有时候，你需要的不是辩论，而是**系统性研究**。

比如："帮我研究一下这家公司，写一份完整的分析报告。"

这需要的是：并行调研多个维度、评估信息覆盖率、识别缺口并补充、最后综合成报告。

下一章我们来聊 **Research Synthesis**——如何将多源并行研究、覆盖率评估、迭代补充整合为高质量的综合报告。

这也是 Part 6 高级推理的最后一章。ToT、Debate、Research Synthesis 三个模式，覆盖了"探索"、"对抗"、"综合"三种高级推理需求。

第 19 章见。
