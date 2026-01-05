# 第 17 章：Tree-of-Thoughts

> **ToT 让 Agent 同时探索多条推理路径，通过评分和剪枝找到最优解——但它不是银弹；成本高、收敛慢，只适合真正需要"试错"的场景。**

---

## 17.1 先说结论：什么时候该用 ToT

先看一个真实案例：

2024 年中，我在帮一家技术咨询公司做架构评审 Agent。客户会提交系统设计方案，Agent 需要找出潜在问题并给出改进建议。

有一次，客户提交了一个支付系统设计：微服务架构、每个服务独立数据库、用 Kafka 做异步通信。

我让 Agent 用 Chain-of-Thought 分析。它的推理是这样的：

```
Step 1: 微服务架构可以提高可扩展性 ✓
Step 2: 独立数据库避免单点故障 ✓
Step 3: Kafka 解耦服务，提升吞吐量 ✓
Step 4: 设计合理，无明显问题
```

评审通过，客户开始实施。

两个月后，客户回来投诉：**分布式事务处理太复杂了！** 订单创建、库存扣减、支付确认，三个服务各有数据库，如何保证一致性？用 2PC 性能差，用 Saga 复杂度爆炸。

我回头看 Agent 的推理，发现问题在 Step 2——"独立数据库"这个决策在第一步就埋下了坑，但 CoT 没有探索其他可能性（比如共享数据库、CQRS）。它沿着一条路走到底，看起来每一步都合理，但整体方案有致命缺陷。

**这就是 CoT 的问题——它是单行道，一旦早期决策错误，后面全废。** 而 Tree-of-Thoughts（ToT）可以同时探索多条路径，通过对比评分发现更优方案。

但 ToT 不是免费的。成本是 CoT 的 3-5 倍，而且不是所有问题都需要它。

### 什么问题适合 ToT？

我见过太多人把 ToT 当成万能药。

**不是的。**

ToT 适合的场景有三个共同特征：

1. **解空间大**：问题有多种可能的解决路径
2. **早期决策影响大**：前几步走错，后面全废
3. **可以评估中间状态**：能判断某条路"有没有戏"

| 场景 | 为什么适合 | 为什么不适合 |
|------|-----------|-------------|
| 24 点游戏 | 多种运算组合，需要尝试 | 简单加减乘除就能算出来的 |
| 复杂数学证明 | 多种证明思路，需要比较 | 有标准解法的计算题 |
| 系统架构设计 | 多种架构方案，需要权衡 | 已有成熟模板的 CRUD 系统 |
| 策略规划 | 多种策略，需要模拟推演 | 只有一种明显正确做法的 |
| 代码调试（复杂 bug）| 多种可能原因，需要排查 | 错误信息明确指向某行的 |

一个简单的判断方法：

> 如果你自己解决这个问题时，会在纸上画树状图、列多个方案比较——那大概率适合 ToT。
> 如果你心里已经有答案，只是需要 Agent 帮你执行——那 CoT 就够了。

---

## 17.2 CoT vs ToT：核心差异

Chain-of-Thought 是线性思考：

```
Step 1 → Step 2 → Step 3 → 卡住了...
         ↓
      （没法回头）
```

Tree-of-Thoughts 是树形思考：

```
                ┌── 缓存优先方案 ──> 评估 0.85 → 继续探索
                │
起点 ──> 选择 ──┼── 微服务拆分方案 ──> 评估 0.45 → 剪掉
                │
                └── 边缘计算方案 ──> 评估 0.72 → 保留观察
```

核心差异在于三点：

| 维度 | CoT | ToT |
|------|-----|-----|
| **路径数量** | 单路径 | 多路径并行 |
| **错误恢复** | 走错难回头 | 可以剪枝、回溯 |
| **成本** | 相对低 | 成倍增加 |
| **收敛速度** | 快（一条路走到底）| 慢（需要探索 + 比较）|

说实话，我觉得 ToT 最大的价值不是"找到更好的答案"，而是"避免掉进坑里"。

CoT 的问题是：模型可能在第一步就走错了，但它会继续自信地往下推，最后给你一个"看起来合理但完全错误"的答案。

ToT 至少会尝试多条路，如果某条路分数明显比其他低，你就知道那个方向可能有问题。

---

## 17.3 思维树的核心结构

### 节点定义

每个节点代表一个"思考步骤"：

```go
type ThoughtNode struct {
    ID          string           // 节点 ID
    Thought     string           // 当前思考内容
    Score       float64          // 评估分数 (0-1)
    Depth       int              // 树深度
    ParentID    string           // 父节点
    Children    []*ThoughtNode   // 子节点
    TokensUsed  int              // 消耗的 token
    IsTerminal  bool             // 是否终止（找到答案或死胡同）
    Explanation string           // 解释（为什么走这条路）
}
```

Shannon 的实现在 [`patterns/tree_of_thoughts.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go)。核心思路很简单：每个节点有分数，分数低的被剪掉，分数高的继续探索。

### 配置参数

```go
type TreeOfThoughtsConfig struct {
    MaxDepth          int     // 最大深度，默认 3
    BranchingFactor   int     // 每节点分支数，2-4，默认 3
    PruningThreshold  float64 // 剪枝阈值，默认 0.3
    ExplorationBudget int     // 最大探索节点数，默认 15
    BacktrackEnabled  bool    // 是否允许回溯
    EvaluationMethod  string  // "scoring", "voting", "llm"
}
```

这些参数怎么调？我的经验是：

| 参数 | 小值效果 | 大值效果 | 建议起点 |
|------|----------|----------|----------|
| `MaxDepth` | 浅层快速，可能找不到深层解 | 深层精细，成本高 | 3 |
| `BranchingFactor` | 聚焦少数方向 | 发散探索更多可能 | 3 |
| `PruningThreshold` | 保留更多分支 | 激进剪枝，可能错过好方案 | 0.3 |
| `ExplorationBudget` | 省成本，覆盖不全 | 更全面，成本高 | 15 |

**关键公式**：最坏情况下的节点数 = `BranchingFactor^MaxDepth`

如果 BranchingFactor=3，MaxDepth=3，最坏情况是 27 个节点。所以 `ExplorationBudget` 设 15 是合理的——它会在探索完之前强制停止。

---

## 17.4 探索算法：Best-First Search

ToT 的核心是"优先探索看起来最有希望的节点"。Shannon 用的是 Best-First Search：

```go
func TreeOfThoughts(
    ctx workflow.Context,
    query string,
    context map[string]interface{},
    sessionID string,
    history []string,
    config TreeOfThoughtsConfig,
    opts Options,
) (*TreeOfThoughtsResult, error) {

    // 初始化根节点
    root := &ThoughtNode{
        ID:       "root",
        Thought:  query,
        Score:    1.0,
        Depth:    0,
        Children: make([]*ThoughtNode, 0),
    }

    // 探索队列（按分数排序）
    queue := []*ThoughtNode{root}
    thoughtsExplored := 0

    // 主循环
    for len(queue) > 0 && thoughtsExplored < config.ExplorationBudget {
        // 按分数排序，取最优节点
        sort.Slice(queue, func(i, j int) bool {
            return queue[i].Score > queue[j].Score
        })

        current := queue[0]
        queue = queue[1:]

        // 深度限制
        if current.Depth >= config.MaxDepth {
            current.IsTerminal = true
            continue
        }

        // 生成分支
        branches := generateBranches(ctx, current, query, config.BranchingFactor, ...)

        // 评估和剪枝
        for _, branch := range branches {
            branch.Score = evaluateThought(branch, query)

            // 剪枝低分分支
            if branch.Score < config.PruningThreshold {
                continue  // 直接丢弃
            }

            current.Children = append(current.Children, branch)

            if isTerminalThought(branch.Thought) {
                branch.IsTerminal = true
            } else {
                queue = append(queue, branch)
            }
        }

        thoughtsExplored++
    }

    // 找最优路径
    bestPath := findBestPath(root)
    return &TreeOfThoughtsResult{BestPath: bestPath, ...}, nil
}
```

核心设计点：

1. **Best-First**：每次选分数最高的节点扩展，而不是广度优先或深度优先
2. **预算控制**：`ExplorationBudget` 限制总节点数，防止成本失控
3. **动态剪枝**：低于阈值的分支直接丢弃，不浪费后续探索

---

## 17.5 如何评估一个"想法"

这是 ToT 最关键的部分。评估不准，整个树就废了。

### 启发式评分（快速但粗糙）

Shannon 默认用启发式评分：

```go
func evaluateThought(node *ThoughtNode, originalQuery string) float64 {
    score := 0.5  // 基础分
    thought := strings.ToLower(node.Thought)

    // 正向指标
    if strings.Contains(thought, "therefore") ||
       strings.Contains(thought, "solution") ||
       strings.Contains(thought, "answer") {
        score += 0.2  // 有结论倾向
    }

    if strings.Contains(thought, "because") ||
       strings.Contains(thought, "since") {
        score += 0.1  // 有逻辑连接
    }

    if strings.Contains(thought, "step") ||
       strings.Contains(thought, "first") {
        score += 0.1  // 有具体步骤
    }

    // 负向指标
    if strings.Contains(thought, "maybe") ||
       strings.Contains(thought, "perhaps") {
        score -= 0.1  // 模糊不确定
    }

    // 深度惩罚（偏好短路径）
    score -= float64(node.Depth) * 0.05

    return math.Max(0, math.Min(1, score))
}
```

这个评估器很"cheap"——它只看关键词，不理解语义。

优点：快，便宜。
缺点：可能被"套话"骗过（模型学会说 "therefore" 但没有真正推理）。

### LLM 评估（准确但贵）

复杂任务可以让另一个 LLM 来评估：

```go
// 概念示例：LLM 评估思维质量
func evaluateWithLLM(ctx workflow.Context, thought string, query string) float64 {
    prompt := fmt.Sprintf(`评估以下推理步骤的质量（0-1分）：

问题：%s
推理步骤：%s

评估标准：
1. 逻辑是否连贯
2. 是否朝着解决方案前进
3. 是否有明显错误

返回格式：{"score": 0.75, "reason": "..."}`, query, thought)

    response := callLLM(prompt)
    return parseScore(response)
}
```

这种方法更准确，但每个节点都要调一次 LLM，成本会翻倍。

### 我的建议

对于大多数场景，启发式评分 + 人工验收就够了。

只有当你发现启发式评分经常"选错路"时，才考虑换成 LLM 评估。

---

## 17.6 终止条件：什么时候停

ToT 需要知道什么时候算"找到答案"，什么时候算"死胡同"：

```go
func isTerminalThought(thought string) bool {
    lower := strings.ToLower(thought)

    // 解决方案指标
    solutionKeywords := []string{
        "the answer is",
        "therefore",
        "in conclusion",
        "final answer",
        "solution:",
    }
    for _, keyword := range solutionKeywords {
        if strings.Contains(lower, keyword) {
            return true
        }
    }

    // 死胡同指标
    deadEndKeywords := []string{
        "impossible",
        "cannot be solved",
        "no solution",
        "contradiction",
    }
    for _, keyword := range deadEndKeywords {
        if strings.Contains(lower, keyword) {
            return true
        }
    }

    return false
}
```

终止条件有两种：

1. **正向终止**：找到了答案（"the answer is..."）
2. **负向终止**：确认是死路（"impossible"）

负向终止很重要——它让 ToT 能快速放弃无望的分支，把资源集中在有希望的方向。

---

## 17.7 回溯机制：低置信度时怎么办

如果最优路径的置信度很低，Shannon 会尝试回溯探索备选：

```go
// 回溯逻辑
if config.BacktrackEnabled && result.Confidence < 0.5 && len(queue) > 0 {
    logger.Info("Backtracking to explore alternative paths")

    // 取队列中得分最高的 3 个备选
    alternatives := queue[:min(3, len(queue))]
    for _, alt := range alternatives {
        altPath := getPathToNode(alt, allNodes)
        altConfidence := calculatePathConfidence(altPath)

        if altConfidence > result.Confidence {
            result.BestPath = altPath
            result.Confidence = altConfidence
        }
    }
}
```

这个设计的核心思路是：如果最优解都不太确定，那可能是评估出了问题，不如再看看其他候选。

---

## 17.8 实战：研究角度探索

来看一个真实场景。

**任务**：分析 AI Agent 领域 2024 年的发展趋势

**配置**：

```go
config := TreeOfThoughtsConfig{
    MaxDepth:          3,
    BranchingFactor:   3,
    PruningThreshold:  0.4,
    ExplorationBudget: 12,
    BacktrackEnabled:  true,
}
```

**探索过程**：

```
Root: "分析 AI Agent 领域 2024 年的发展趋势"
├── 技术进展方向 (score: 0.75)
│   ├── 多模态能力 (score: 0.82) ← 最高分
│   ├── 推理能力提升 (score: 0.70)
│   └── 工具使用演进 (score: 0.68)
├── 产品落地方向 (score: 0.72)
│   ├── 企业级应用 (score: 0.78)
│   └── 开发者工具 (score: 0.65)
└── 生态发展方向 (score: 0.55)
    └── (分数 < 0.4，被剪枝)

最优路径: Root → 技术进展 → 多模态能力
置信度: 0.78
```

这个例子展示了 ToT 的优势：它不是只看"技术进展"就冲进去，而是先生成三个大方向，评估后发现"生态发展"方向信息太少，直接剪掉。

---

## 17.9 常见的坑

### 坑 1：分支爆炸

```go
// 错误：无限制探索
config := TreeOfThoughtsConfig{
    BranchingFactor:   5,
    ExplorationBudget: 0,  // 没有预算限制！
}
// 结果：5^3 = 125 个节点，Token 爆炸

// 正确：控制复杂度
config := TreeOfThoughtsConfig{
    BranchingFactor:   3,
    ExplorationBudget: 15,
    MaxDepth:          3,
}
```

### 坑 2：过度剪枝

```go
// 错误：阈值太高
config.PruningThreshold = 0.8
// 结果：几乎所有分支都被剪掉，只剩一条路（和 CoT 没区别）

// 正确：适度保留
config.PruningThreshold = 0.3
```

### 坑 3：评估偏差

启发式评估可能被模型"套话"骗过。模型学会在回答里加 "therefore"，但并没有真正推理。

**解决方法**：

```go
// 复杂任务用 LLM 评估
config.EvaluationMethod = "llm"

// 或者加入"内容检查"
func evaluateThought(node *ThoughtNode, query string) float64 {
    score := heuristicScore(node)

    // 检查是否真的推进了
    if !containsNewInfo(node.Thought, node.Parent.Thought) {
        score *= 0.5  // 惩罚原地踏步
    }

    return score
}
```

### 坑 4：忘记 Token 预算

ToT 的成本很容易失控。每个节点都是一次 LLM 调用。

```go
// 在 Shannon 里，预算是这样分配的
tokenBudgetPerThought := opts.BudgetAgentMax / config.ExplorationBudget
```

比如总预算 15000 tokens，探索 15 个节点，每个节点只有 1000 tokens 的配额。

---

## 17.10 ToT vs 其他推理模式

| 模式 | 适用场景 | 成本 | 收敛速度 |
|------|----------|------|----------|
| **CoT** | 有明确解法的问题 | 低 | 快 |
| **ToT** | 多路径探索、需要回溯 | 高（2-5x）| 慢 |
| **Reflection** | 迭代改进已有答案 | 中 | 中 |
| **Debate** | 争议性话题、多视角 | 高 | 中 |

**我的选择逻辑**：

1. 先用 CoT 试一次
2. 如果效果不好，看是"答案质量问题"还是"方向走错问题"
3. 质量问题 → Reflection
4. 方向问题 → ToT

---

## 小结

核心就一句话：**ToT 通过多路径探索找最优解——生成分支、评估打分、剪枝低分、选择最优**。

但它不是万能药。成本高、收敛慢，只适合真正需要"试错"的场景。

要点：

1. **Best-First Search**：按分数优先探索
2. **剪枝**：低于阈值的分支直接丢弃
3. **回溯**：低置信度时探索备选路径
4. **预算控制**：ExplorationBudget 限制成本
5. **评估方法**：启发式快但粗，LLM 准但贵

---

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- [`patterns/tree_of_thoughts.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go)：找 `TreeOfThoughts` 函数，看主循环怎么实现 Best-First Search；找 `evaluateThought` 看启发式评分逻辑

### 选读深挖（2 个，按兴趣挑）

- [`patterns/options.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/options.go)：理解 `BudgetAgentMax` 怎么分配给每个节点
- [`patterns/chain_of_thought.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/chain_of_thought.go)：对比 CoT 的实现比 ToT 简单多少，思考为什么

---

## 练习

### 练习 1：场景判断

判断以下场景适合 CoT 还是 ToT，并说明理由：

1. "帮我把这段 Python 代码翻译成 Go"
2. "设计一个能支撑 10 万日活的社区 App 架构"
3. "解释一下什么是微服务"
4. "帮我找到这个 bug 的根因（日志里有多个可疑点）"

### 练习 2：参数调优

如果你发现 ToT 探索了 15 个节点，但最优路径的置信度只有 0.4，你会怎么调整参数？给出至少两种可能的调整方向。

### 练习 3（进阶）：改进评估器

Shannon 的启发式评估器只看关键词。设计一个改进版：

1. 写出改进逻辑的伪代码
2. 考虑：怎么避免模型"套话"骗过评估器？
3. 思考：改进会增加多少成本？值得吗？

---

## 想深入？

- [Tree of Thoughts: Deliberate Problem Solving with Large Language Models](https://arxiv.org/abs/2305.10601) - Yao et al., 2023，ToT 原始论文
- [Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903) - CoT 论文，理解 ToT 的前身
- A*, BFS, DFS 搜索算法对比——ToT 本质是搜索问题

---

## 下一章预告

ToT 解决的是"多条路怎么选"的问题。但有时候，问题本身就是争议性的——不是找最优解，而是要听不同声音。

比如："AI 会取代人类工作吗？"

这种问题没有标准答案，不同视角会有不同结论。这时候，你需要的不是一棵思维树，而是一场**辩论**。

下一章我们来聊 **Debate 模式**——让多个 Agent 从不同立场辩论，在对抗中暴露弱点，综合形成更可靠的结论。

下一章见。
