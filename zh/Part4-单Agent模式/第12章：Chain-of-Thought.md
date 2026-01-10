# 第 12 章：Chain-of-Thought

> **CoT 不是让 LLM 变聪明，而是让它把隐式推理变成显式步骤——这样你能看到它错在哪，也更容易把它拉回来。**

---

## 12.1 为什么需要逐步推理？

先看一个真实案例：

2024 年初，我在帮一个教育科技公司做数学辅导 Agent。功能很简单：学生输入数学题，Agent 给出解答和步骤。

测试阶段一切正常。简单的加减乘除、一元一次方程，准确率很高。

上线第一周，投诉来了。一个初中生问："小明有 15 个苹果，分给 3 个朋友，每人分到几个？然后他又买了 8 个，现在有几个？"

Agent 回答："11 个。"

错了。正确答案是 8 个（15 ÷ 3 = 5，5 - 5 = 0，0 + 8 = 8）。但 Agent 看到"15"和"3"，直接算了 15 - 3 - 1 = 11，完全没理解题意。

**这暴露了 LLM 的一个根本问题——它会"跳步"，直接给答案，而不是真正推理。** 对于多步骤的问题，这种跳步很容易出错。

一周后，我们在 Prompt 里加了一句："Please solve step by step"。同样的问题，Agent 开始这样回答：

```
步骤1：小明有 15 个苹果，分给 3 个朋友
步骤2：每人分到 15 ÷ 3 = 5 个
步骤3：小明分完后剩余 15 - 15 = 0 个
步骤4：又买了 8 个，现在有 0 + 8 = 8 个
答案：8 个
```

在我们的测试集上，准确率提升了约 40%。（注意：这是特定场景下的结果，实际效果因任务类型和模型而异，建议在你的评估集上测试。）

这就是 Chain-of-Thought（思维链）的价值——让 LLM 把推理过程外显，一步一步计算，而不是凭"直觉"猜答案。

### LLM 的默认行为

LLM 的默认行为是"一口气说完"——你问一句，它生成一整段。它不会停下来想"我刚才说的对不对"，也不会在生成过程中做计算。

这导致一个问题：**复杂推理容易出错。**

| 任务类型 | LLM 默认表现 | 问题 |
|----------|-------------|------|
| 多步数学 | 直接给答案 | 跳步，算错 |
| 逻辑推理 | 凭直觉猜 | 逻辑链断裂 |
| 因果分析 | 表面关联 | 因果倒置 |
| 调试代码 | 列举常见原因 | 没有真正分析 |

### CoT 的解决方案

Chain-of-Thought（思维链）的核心思想很简单：**让 LLM 一步一步思考，把中间过程写出来。**

看同一道题用 CoT 的效果：

```
让我一步步计算：
→ 步骤1: 小明最初有 5 个苹果
→ 步骤2: 给小红 2 个后，剩余 5 - 2 = 3 个
→ 步骤3: 从小华得到 3 个后，总共 3 + 3 = 6 个
→ 步骤4: 吃掉 1 个后，剩余 6 - 1 = 5 个

因此，小明现在有 5 个苹果。
```

这次对了。关键区别：**显式的推理过程迫使模型进行实际计算，而不是凭直觉猜测。**

### CoT 的价值

| 维度 | 无 CoT | 有 CoT |
|------|--------|--------|
| **准确性** | 靠模式匹配，复杂推理易出错 | 逐步验证，减少跳跃性错误 |
| **可解释性** | 黑盒输出，无法审计 | 透明过程，可追溯每一步 |
| **调试能力** | 错了不知道哪错了 | 可定位具体哪一步出错 |

但我要提醒一下：CoT 不是万能的。它能提高准确率，但不能保证正确。逐步推理的每一步仍然可能出错。

---

## 12.2 CoT 的学术背景

CoT 来自 2022 年的一篇论文：[Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)（Wei et al.）。

核心发现：

> **对于需要多步推理的任务，在 Prompt 中加入"let's think step by step"或提供推理示例，可以显著提升 LLM 的准确率。**

后来又有人发现了一个更简单的方法——Zero-shot CoT：只需要在问题后面加一句"Let's think step by step"，就能触发 LLM 的逐步推理。

这个发现很有意思：**LLM 其实有逐步推理的能力，只是需要被"提醒"才会用。**

---

## 12.3 CoT Prompt 设计

### 基础模板

最简单的 CoT Prompt：

```
问题：如果今天是周三，那么10天后是星期几？

请逐步思考，用 → 标记每个推理步骤。
最后给出结论，以"因此："开头。
```

### Shannon 的默认模板

Shannon 的 CoT 实现在 `patterns/chain_of_thought.go`，默认模板是这样的：

```go
func buildChainOfThoughtPrompt(query string, config ChainOfThoughtConfig) string {
    if config.PromptTemplate != "" {
        return strings.ReplaceAll(config.PromptTemplate, "{query}", query)
    }

    // 默认 CoT 模板
    return fmt.Sprintf(`Please solve this step-by-step:

Question: %s

Think through this systematically:
1. First, identify what is being asked
2. Break down the problem into steps
3. Work through each step with clear reasoning
4. Show your work and explain your thinking
5. Arrive at the final answer

Use "→" to mark each reasoning step.
End with "Therefore:" followed by your final answer.`, query)
}
```

**实现参考 (Shannon)**: [`patterns/chain_of_thought.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/chain_of_thought.go) - buildChainOfThoughtPrompt 函数

### 领域定制模板

不同场景需要不同的 CoT 模板：

**数学专用**：
```
数学问题: {query}

请按以下格式解答：

【分析】首先理解题目要求
【公式】列出需要用到的公式
【计算】
  → 步骤1: ...
  → 步骤2: ...
【验证】检查计算结果是否合理
【答案】最终结果是...
```

**代码调试专用**：
```
调试问题: {query}

请系统性分析：

1. 【症状描述】观察到的错误现象
2. 【假设列表】可能的原因（按可能性排序）
   → 假设A: ...
   → 假设B: ...
3. 【验证过程】逐个验证假设
4. 【根因分析】确定真正原因
5. 【修复方案】给出解决方法

Therefore: 根因是... 修复方法是...
```

**逻辑推理专用**：
```
推理问题: {query}

请按逻辑链推理：

【已知条件】
  - 条件1: ...
  - 条件2: ...

【推理过程】
  → 由条件1，可得...
  → 结合条件2，进一步可得...
  → 因此...

【结论】...
```

---

## 12.4 Shannon 的 CoT 实现

### 配置结构

```go
type ChainOfThoughtConfig struct {
    MaxSteps              int    // 最大推理步骤数
    RequireExplanation    bool   // 是否强制要求解释
    ShowIntermediateSteps bool   // 输出是否包含中间步骤
    PromptTemplate        string // 自定义模板
    StepDelimiter         string // 步骤分隔符，默认 "\n→ "
    ModelTier             string // 模型层级
}
```

### 结果结构

```go
type ChainOfThoughtResult struct {
    FinalAnswer    string        // 最终答案
    ReasoningSteps []string      // 推理步骤列表
    TotalTokens    int           // Token 消耗
    Confidence     float64       // 推理置信度 (0-1)
    StepDurations  []time.Duration // 每步耗时
}
```

### 核心流程

```go
func ChainOfThought(
    ctx workflow.Context,
    query string,
    context map[string]interface{},
    sessionID string,
    history []string,
    config ChainOfThoughtConfig,
    opts Options,
) (*ChainOfThoughtResult, error) {

    // 1. 设置默认值
    if config.MaxSteps == 0 {
        config.MaxSteps = 5
    }
    if config.StepDelimiter == "" {
        config.StepDelimiter = "\n→ "
    }

    // 2. 构建 CoT Prompt
    cotPrompt := buildChainOfThoughtPrompt(query, config)

    // 3. 调用 LLM
    cotResult := executeAgent(ctx, cotPrompt, ...)

    // 4. 解析推理步骤
    steps := parseReasoningSteps(cotResult.Response, config.StepDelimiter)

    // 5. 提取最终答案
    answer := extractFinalAnswer(cotResult.Response, steps)

    // 6. 计算置信度
    confidence := calculateReasoningConfidence(steps, cotResult.Response)

    // 7. 低置信度时请求澄清（可选）
    if config.RequireExplanation && confidence < 0.7 {
        // 用一半预算重新生成更清晰的解释
        clarificationResult := requestClarification(ctx, query, steps)
        // 更新结果...
    }

    return &ChainOfThoughtResult{
        FinalAnswer:    answer,
        ReasoningSteps: steps,
        Confidence:     confidence,
        TotalTokens:    cotResult.TokensUsed,
    }, nil
}
```

---

## 12.5 步骤解析

LLM 生成的推理过程需要解析成结构化的步骤列表。Shannon 的实现：

```go
func parseReasoningSteps(response, delimiter string) []string {
    lines := strings.Split(response, "\n")
    steps := []string{}

    for _, line := range lines {
        line = strings.TrimSpace(line)
        // 识别步骤标记
        if strings.HasPrefix(line, "→") ||
           strings.HasPrefix(line, "Step") ||
           strings.HasPrefix(line, "1.") ||
           strings.HasPrefix(line, "2.") ||
           strings.HasPrefix(line, "3.") ||
           strings.HasPrefix(line, "•") {
            steps = append(steps, line)
        }
    }

    // 降级策略：没有明确标记时，按句子分割
    if len(steps) == 0 {
        segments := strings.Split(response, ". ")
        for _, seg := range segments {
            if len(strings.TrimSpace(seg)) > 20 {
                steps = append(steps, seg)
                if len(steps) >= 5 {
                    break
                }
            }
        }
    }

    return steps
}
```

解析优先级：
1. 显式标记（→, Step, 数字.）
2. 符号标记（•）
3. 降级：按句子分割

### 提取最终答案

```go
func extractFinalAnswer(response string, steps []string) string {
    // 查找结论标记
    markers := []string{
        "Therefore:",
        "Final Answer:",
        "The answer is:",
        "因此：",
        "结论：",
    }

    lower := strings.ToLower(response)
    for _, marker := range markers {
        if idx := strings.Index(lower, strings.ToLower(marker)); idx != -1 {
            answer := response[idx+len(marker):]
            // 取到下一个空行为止
            if endIdx := strings.Index(answer, "\n\n"); endIdx > 0 {
                answer = answer[:endIdx]
            }
            return strings.TrimSpace(answer)
        }
    }

    // 降级：用最后一个步骤
    if len(steps) > 0 {
        return steps[len(steps)-1]
    }

    // 再降级：最后一段
    paragraphs := strings.Split(response, "\n\n")
    if len(paragraphs) > 0 {
        return paragraphs[len(paragraphs)-1]
    }

    return response
}
```

这里有多层降级策略，确保即使 LLM 没有按照预期格式输出，也能提取出有意义的答案。

---

## 12.6 置信度评估

推理质量可以量化评估。Shannon 的实现：

```go
func calculateReasoningConfidence(steps []string, response string) float64 {
    confidence := 0.5 // 基础分

    // 步骤充分度：>=3 步加分
    if len(steps) >= 3 {
        confidence += 0.2
    }

    // 逻辑连接词
    logicalTerms := []string{
        "therefore", "because", "since", "thus",
        "consequently", "hence", "so", "implies",
    }
    lower := strings.ToLower(response)
    count := 0
    for _, term := range logicalTerms {
        count += strings.Count(lower, term)
    }
    if count >= 3 {
        confidence += 0.15
    }

    // 结构化标记
    if strings.Contains(response, "Step") || strings.Contains(response, "→") {
        confidence += 0.1
    }

    // 明确结论
    if strings.Contains(lower, "therefore") ||
       strings.Contains(lower, "final answer") {
        confidence += 0.05
    }

    if confidence > 1.0 {
        confidence = 1.0
    }

    return confidence
}
```

置信度公式（这是我为了讨论方便设计的一个 heuristic，不是学术标准）：

```
置信度 = 0.5 (基础)
       + 0.2 (步骤 >= 3)
       + 0.15 (逻辑词 >= 3)
       + 0.1 (结构化标记)
       + 0.05 (明确结论)
       ────────────────
       最高 1.0
```

---

## 12.7 低置信度处理

当 `RequireExplanation=true` 且置信度低于 0.7 时，Shannon 会请求澄清：

```go
if config.RequireExplanation && confidence < 0.7 {
    clarificationPrompt := fmt.Sprintf(
        "The previous reasoning for '%s' had unclear steps. "+
        "Please provide a clearer step-by-step explanation:\n%s",
        query,
        strings.Join(steps, config.StepDelimiter),
    )

    // 用一半预算重新生成
    clarifyResult := executeAgentWithBudget(ctx, clarificationPrompt, opts.BudgetAgentMax/2)

    // 更新结果
    if clarifyResult.Success {
        clarifiedSteps := parseReasoningSteps(clarifyResult.Response, delimiter)
        if len(clarifiedSteps) > 0 {
            result.ReasoningSteps = clarifiedSteps
            result.FinalAnswer = extractFinalAnswer(clarifyResult.Response, clarifiedSteps)
            result.Confidence = calculateReasoningConfidence(clarifiedSteps, clarifyResult.Response)
        }
        result.TotalTokens += clarifyResult.TokensUsed
    }
}
```

澄清策略：
- 把原始步骤作为参考
- 使用一半预算（控制成本）
- 请求更清晰的解释

---

## 12.8 CoT vs Tree-of-Thoughts

CoT 是线性的：一步接一步，没有回头路。

Tree-of-Thoughts (ToT) 是树形的：每一步可以有多个分支，可以回溯。

| 特性 | Chain-of-Thought | Tree-of-Thoughts |
|------|------------------|------------------|
| **结构** | 线性链 | 分支树 |
| **探索** | 单一路径 | 多路径并行 |
| **回溯** | 不支持 | 支持 |
| **Token 消耗** | 较低 | 较高 (3-10x) |
| **适用场景** | 确定性推理 | 探索性问题 |

### 什么时候用 ToT？

```
问题是否有多种可能的解决路径？
├─ 否 → 用 CoT（单一路径足够）
└─ 是 → 需要比较不同方案吗？
         ├─ 否 → 用 CoT（随机选一条）
         └─ 是 → 用 ToT（系统性探索）
```

ToT 在第 17 章详细讲。现在只需要知道：**大多数场景 CoT 就够了**。

---

## 12.9 常见的坑

### 坑 1：过度分步

**症状**：简单问题被强制分解为太多步骤，输出冗长。

```
// 问「2+3 等于多少」
→ 步骤1: 识别问题类型——这是加法问题
→ 步骤2: 确定操作数——2 和 3
→ 步骤3: 回顾加法定义
→ 步骤4: 执行计算 2 + 3 = 5
→ 步骤5: 验证结果
Therefore: 5
```

**解决**：根据复杂度动态调整 MaxSteps：

```go
func adaptiveMaxSteps(query string) int {
    complexity := estimateComplexity(query)
    if complexity < 0.3 {
        return 2  // 简单问题
    } else if complexity < 0.7 {
        return 5  // 中等
    }
    return 8      // 复杂
}
```

### 坑 2：推理与事实混淆

**症状**：CoT 生成的步骤"看起来合理"但基于错误事实。

```
→ 步骤1: 特斯拉于 2020 年成为全球市值最高的汽车公司（错误事实）
→ 步骤2: 因此其销量应该也是最高的（错误推理）
```

问题是：逻辑正确，但前提错误，结论也错误。

**解决**：CoT 配合工具使用，验证关键事实：

```
推理时请遵循以下原则：
1. 涉及具体数据时，标注 [需验证]
2. 区分「推理」和「事实陈述」
3. 如果不确定某个事实，明确说明
```

### 坑 3：置信度虚高

**症状**：模型用了很多逻辑连接词，但实际推理质量差。

比如循环论证：
```
→ 步骤1: A 是真的，因为 B 是真的
→ 步骤2: B 是真的，因为 A 是真的
Therefore: A 和 B 都是真的
```

用了 "because"，置信度会加分，但这是无效推理。

**解决**：加入语义检测：

```go
func enhancedConfidence(steps []string, response string) float64 {
    base := calculateConfidence(steps, response)

    // 检查循环论证
    if hasCircularReasoning(steps) {
        base -= 0.3
    }

    // 检查步骤之间的逻辑连贯性
    if !hasLogicalCoherence(steps) {
        base -= 0.2
    }

    return max(0, min(1.0, base))
}
```

### 坑 4：格式不一致

**症状**：LLM 有时用 "→"，有时用 "Step"，有时用数字，解析失败。

**解决**：在 Prompt 里明确格式，并在解析时支持多种格式（Shannon 已经这么做了）。

---

## 12.10 什么时候用 CoT？

不是所有任务都需要 CoT。

| 任务类型 | 用 CoT？ | 原因 |
|----------|---------|------|
| 简单计算 | 否 | 直接算更快 |
| 事实查询 | 否 | 直接查更准 |
| 多步数学 | 是 | 减少计算错误 |
| 逻辑推理 | 是 | 外显推理链 |
| 因果分析 | 是 | 追溯因果关系 |
| 代码调试 | 是 | 系统化排查 |
| 创意写作 | 否 | 会限制创造力 |
| 实时对话 | 视情况 | 延迟 vs 准确性权衡 |

**经验法则**：

- 需要"推导"的 → 用
- 需要"审计过程"的 → 用
- 简单直接的 → 不用
- 创意类的 → 不用
- 延迟敏感的 → 谨慎用

---

## 12.11 其他框架怎么做？

CoT 是通用模式，各家都有实现：

| 框架/论文 | 实现方式 | 特点 |
|-----------|----------|------|
| **Zero-shot CoT** | "Let's think step by step" | 最简单，一句话触发 |
| **Few-shot CoT** | 提供推理示例 | 更可控，但需要人工设计 |
| **Self-Consistency** | 多次 CoT + 投票 | 更准确，但成本高 |
| **LangChain** | CoT Prompt 模板 | 易于集成 |
| **OpenAI o1/o3** | 内置多步推理（黑盒） | 内部机制不透明，无需手动触发 |

核心逻辑都一样：让 LLM 把推理过程写出来。

差别在于：
- 触发方式（零样本 vs 少样本）
- 格式约束（自由 vs 结构化）
- 质量保证（单次 vs 多次投票）

---

## 12.12 与 ReAct 的关系

你可能会问：CoT 和 ReAct 有什么区别？

| 维度 | CoT | ReAct |
|------|-----|-------|
| **核心目标** | 外显推理过程 | 推理 + 行动循环 |
| **是否调用工具** | 否（纯推理） | 是（边想边做） |
| **输出** | 推理步骤 + 答案 | 多轮思考/行动/观察 |
| **适用场景** | 需要计算/推理的问题 | 需要获取外部信息的任务 |

简单说：

- **CoT**：想清楚再回答（不需要外部信息）
- **ReAct**：边想边查边做（需要外部信息）

它们可以结合：在 ReAct 的"思考"阶段使用 CoT。

---

## 划重点

1. **CoT 本质**：外化思维，强制模型逐步推理
2. **Prompt 设计**：明确指令 "step by step" + 格式约定（→, Step）
3. **步骤解析**：识别标记 + 多层降级策略
4. **置信度评估**：步骤数 + 逻辑词 + 结构化标记（heuristic，非学术标准）
5. **适用场景**：多步推理、需要审计过程；不适合简单任务、创意类

---

## Shannon Lab（10 分钟上手）

本节帮你在 10 分钟内把本章概念对应到 Shannon 源码。

### 必读（1 个文件）

- [`patterns/chain_of_thought.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/chain_of_thought.go)：找 `ChainOfThought` 函数，看它怎么用 `buildChainOfThoughtPrompt` 构建 Prompt、用 `parseReasoningSteps` 解析推理步骤、用 `calculateReasoningConfidence` 评估置信度

### 选读深挖（2 个，按兴趣挑）

- [`patterns/tree_of_thoughts.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go)：对比 ToT 和 CoT 的实现差异
- 自己在 ChatGPT/Claude 里试一下：同一个数学问题，加不加 "Let's think step by step"，回答有什么不同？

---

## 练习

### 练习 1：设计 CoT 模板

为以下场景设计专用的 CoT Prompt 模板：

1. **法律推理**：判断一个行为是否违法
2. **医学诊断**：根据症状推测可能的疾病
3. **金融分析**：评估一只股票的投资价值

每个模板应该包含：
- 问题描述占位符
- 推理步骤的格式要求
- 结论的格式要求

### 练习 2：源码阅读

读 `patterns/chain_of_thought.go` 里的 `parseReasoningSteps` 函数：

1. 它支持哪些步骤标记格式？
2. 如果 LLM 没有用任何标记，它怎么降级处理？
3. 为什么降级时限制最多 5 个步骤？

### 练习 3（进阶）：设计循环论证检测

设计一个 `hasCircularReasoning` 函数：

- 输入：推理步骤列表
- 输出：是否存在循环论证

思考：
- 什么样的模式算"循环论证"？
- 用什么方法检测？（关键词匹配？语义相似度？）
- 有没有 false positive 的风险？

---

## 想深入？

- [Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903) - Wei et al., 2022，原始论文
- [Zero-shot CoT: "Let's think step by step"](https://arxiv.org/abs/2205.11916) - 最简单的 CoT 触发方式
- [Self-Consistency Decoding](https://arxiv.org/abs/2203.11171) - 多次 CoT + 投票提高准确率
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601) - CoT 的树形扩展

---

## 下一章预告

到这里，Part 4（单 Agent 模式）就结束了。我们学了三个核心模式：

- **Planning**：把复杂任务拆解成子任务
- **Reflection**：评估输出质量，不达标就重试
- **Chain-of-Thought**：把推理过程外显，减少跳跃性错误

但一个 Agent 能做的事情是有限的。当任务足够复杂，你需要多个 Agent 协作。

这就是 Part 5 的内容——**多 Agent 编排**。

下一章我们先讲编排基础：当单个 Agent 不够用时，如何让多个 Agent 分工协作？谁来决定谁做什么？失败了怎么办？

