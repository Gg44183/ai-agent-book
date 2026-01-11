# Chapter 12: Chain-of-Thought

> **CoT doesn't make LLMs smarter—it turns implicit reasoning into explicit steps. This way you can see where it went wrong, and it's easier to pull it back on track.**

---

## 12.1 Why Do We Need Step-by-Step Reasoning?

First, a real case:

In early 2024, I was helping an edtech company build a math tutoring Agent. Simple functionality: students input math problems, Agent provides solutions and steps.

Everything worked fine during testing. Simple arithmetic, single-variable equations—high accuracy.

First week after launch, complaints came in. A middle schooler asked: "Xiao Ming has 15 apples. He splits them among 3 friends, how many does each get? Then he buys 8 more, how many does he have now?"

Agent answered: "11."

Wrong. The correct answer is 8 (15 / 3 = 5, each friend gets 5, Xiao Ming keeps 0, 0 + 8 = 8). But the Agent saw "15" and "3", computed 15 - 3 - 1 = 11, completely misunderstanding the question.

**This exposed a fundamental LLM problem—it "skips steps," giving answers directly rather than actually reasoning.** For multi-step problems, this step-skipping easily leads to errors.

A week later, we added one line to the prompt: "Please solve step by step." Same question, Agent started answering like this:

```
Step 1: Xiao Ming has 15 apples, splits among 3 friends
Step 2: Each gets 15 ÷ 3 = 5 apples
Step 3: After splitting, Xiao Ming has 15 - 15 = 0 apples left
Step 4: Buys 8 more, now has 0 + 8 = 8 apples
Answer: 8 apples
```

On our test set, accuracy improved by about 40%. (Note: this is the result from a specific scenario; actual effects vary by task type and model. We recommend testing on your own evaluation set.)

This is the value of Chain-of-Thought—making LLMs externalize their reasoning process, computing step by step rather than guessing answers by "intuition."

### LLM's Default Behavior

LLMs' default behavior is "say it all at once"—you ask one thing, they generate an entire paragraph. They don't stop to think "was what I just said correct," and they don't calculate during generation.

This causes a problem: **complex reasoning easily goes wrong.**

| Task Type | LLM Default Behavior | Problem |
|-----------|---------------------|---------|
| Multi-step math | Gives answer directly | Skips steps, calculates wrong |
| Logical reasoning | Guesses by intuition | Logic chain breaks |
| Causal analysis | Surface associations | Cause-effect reversed |
| Code debugging | Lists common causes | Doesn't actually analyze |

### CoT's Solution

Chain-of-Thought's core idea is simple: **Make the LLM think step by step, writing out intermediate steps.**

See the effect of CoT on the same problem:

```
Let me calculate step by step:
→ Step 1: Xiao Ming initially has 5 apples
→ Step 2: After giving 2 to Xiao Hong, 5 - 2 = 3 remaining
→ Step 3: After receiving 3 from Xiao Hua, total 3 + 3 = 6
→ Step 4: After eating 1, 6 - 1 = 5 remaining

Therefore, Xiao Ming now has 5 apples.
```

Got it right this time. Key difference: **Explicit reasoning process forces the model to actually compute, rather than guessing by intuition.**

### CoT's Value

| Dimension | Without CoT | With CoT |
|-----------|-------------|----------|
| **Accuracy** | Pattern matching, complex reasoning error-prone | Step-by-step verification, reduces jumping errors |
| **Explainability** | Black box output, can't audit | Transparent process, every step traceable |
| **Debug ability** | Wrong but don't know where | Can pinpoint which step went wrong |

But I should note: CoT isn't a cure-all. It can improve accuracy but can't guarantee correctness. Each step in step-by-step reasoning can still be wrong.

---

## 12.2 CoT's Academic Background

CoT comes from a 2022 paper: [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903) (Wei et al.).

Core finding:

> **For tasks requiring multi-step reasoning, adding "let's think step by step" to the prompt or providing reasoning examples can significantly improve LLM accuracy.**

Later, someone discovered an even simpler method—Zero-shot CoT: just add "Let's think step by step" after the question to trigger step-by-step reasoning in LLMs.

This discovery is interesting: **LLMs actually have step-by-step reasoning capability, they just need to be "reminded" to use it.**

---

## 12.3 CoT Prompt Design

### Basic Template

The simplest CoT prompt:

```
Question: If today is Wednesday, what day of the week will it be 10 days from now?

Please think step by step, marking each reasoning step with →.
Give your conclusion at the end, starting with "Therefore:"
```

### Shannon's Default Template

Shannon's CoT implementation is in `patterns/chain_of_thought.go`, with this default template:

```go
func buildChainOfThoughtPrompt(query string, config ChainOfThoughtConfig) string {
    if config.PromptTemplate != "" {
        return strings.ReplaceAll(config.PromptTemplate, "{query}", query)
    }

    // Default CoT template
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

**Implementation reference (Shannon)**: [`patterns/chain_of_thought.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/chain_of_thought.go) - buildChainOfThoughtPrompt function

### Domain-Specific Templates

Different scenarios need different CoT templates:

**Math-specific**:
```
Math problem: {query}

Please solve using this format:

【Analysis】First understand what the question asks
【Formulas】List the formulas needed
【Calculation】
  → Step 1: ...
  → Step 2: ...
【Verification】Check if the result is reasonable
【Answer】The final result is...
```

**Code debugging-specific**:
```
Debug problem: {query}

Please analyze systematically:

1. 【Symptom Description】The observed error phenomenon
2. 【Hypothesis List】Possible causes (ranked by likelihood)
   → Hypothesis A: ...
   → Hypothesis B: ...
3. 【Verification Process】Verify each hypothesis
4. 【Root Cause Analysis】Determine the real cause
5. 【Fix Solution】Provide the solution

Therefore: The root cause is... The fix method is...
```

**Logic reasoning-specific**:
```
Reasoning problem: {query}

Please reason through the logic chain:

【Known Conditions】
  - Condition 1: ...
  - Condition 2: ...

【Reasoning Process】
  → From condition 1, we can derive...
  → Combined with condition 2, we can further derive...
  → Therefore...

【Conclusion】...
```

---

## 12.4 Shannon's CoT Implementation

### Configuration Structure

```go
type ChainOfThoughtConfig struct {
    MaxSteps              int    // Maximum reasoning steps
    RequireExplanation    bool   // Whether to require explanation
    ShowIntermediateSteps bool   // Whether output includes intermediate steps
    PromptTemplate        string // Custom template
    StepDelimiter         string // Step delimiter, default "\n→ "
    ModelTier             string // Model tier
}
```

### Result Structure

```go
type ChainOfThoughtResult struct {
    FinalAnswer    string        // Final answer
    ReasoningSteps []string      // Reasoning step list
    TotalTokens    int           // Token consumption
    Confidence     float64       // Reasoning confidence (0-1)
    StepDurations  []time.Duration // Time per step
}
```

### Core Flow

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

    // 1. Set defaults
    if config.MaxSteps == 0 {
        config.MaxSteps = 5
    }
    if config.StepDelimiter == "" {
        config.StepDelimiter = "\n→ "
    }

    // 2. Build CoT Prompt
    cotPrompt := buildChainOfThoughtPrompt(query, config)

    // 3. Call LLM
    cotResult := executeAgent(ctx, cotPrompt, ...)

    // 4. Parse reasoning steps
    steps := parseReasoningSteps(cotResult.Response, config.StepDelimiter)

    // 5. Extract final answer
    answer := extractFinalAnswer(cotResult.Response, steps)

    // 6. Calculate confidence
    confidence := calculateReasoningConfidence(steps, cotResult.Response)

    // 7. Request clarification if low confidence (optional)
    if config.RequireExplanation && confidence < 0.7 {
        // Use half budget to regenerate clearer explanation
        clarificationResult := requestClarification(ctx, query, steps)
        // Update result...
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

## 12.5 Step Parsing

The reasoning process generated by an LLM needs to be parsed into a structured list of steps. Shannon's implementation:

```go
func parseReasoningSteps(response, delimiter string) []string {
    lines := strings.Split(response, "\n")
    steps := []string{}

    for _, line := range lines {
        line = strings.TrimSpace(line)
        // Recognize step markers
        if strings.HasPrefix(line, "→") ||
           strings.HasPrefix(line, "Step") ||
           strings.HasPrefix(line, "1.") ||
           strings.HasPrefix(line, "2.") ||
           strings.HasPrefix(line, "3.") ||
           strings.HasPrefix(line, "•") {
            steps = append(steps, line)
        }
    }

    // Fallback strategy: when no explicit markers, split by sentences
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

Parsing priority:
1. Explicit markers (→, Step, number.)
2. Symbol markers (•)
3. Fallback: split by sentences

### Extracting Final Answer

```go
func extractFinalAnswer(response string, steps []string) string {
    // Look for conclusion markers
    markers := []string{
        "Therefore:",
        "Final Answer:",
        "The answer is:",
    }

    lower := strings.ToLower(response)
    for _, marker := range markers {
        if idx := strings.Index(lower, strings.ToLower(marker)); idx != -1 {
            answer := response[idx+len(marker):]
            // Take until next blank line
            if endIdx := strings.Index(answer, "\n\n"); endIdx > 0 {
                answer = answer[:endIdx]
            }
            return strings.TrimSpace(answer)
        }
    }

    // Fallback: use last step
    if len(steps) > 0 {
        return steps[len(steps)-1]
    }

    // Further fallback: last paragraph
    paragraphs := strings.Split(response, "\n\n")
    if len(paragraphs) > 0 {
        return paragraphs[len(paragraphs)-1]
    }

    return response
}
```

Multiple fallback strategies here ensure that even if the LLM doesn't output in the expected format, a meaningful answer can still be extracted.

---

## 12.6 Confidence Evaluation

Reasoning quality can be quantitatively evaluated. Shannon's implementation:

```go
func calculateReasoningConfidence(steps []string, response string) float64 {
    confidence := 0.5 // Base score

    // Step sufficiency: >=3 steps adds points
    if len(steps) >= 3 {
        confidence += 0.2
    }

    // Logical connectors
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

    // Structured markers
    if strings.Contains(response, "Step") || strings.Contains(response, "→") {
        confidence += 0.1
    }

    // Clear conclusion
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

Confidence formula (this is a heuristic I designed for discussion purposes, not an academic standard):

```
Confidence = 0.5 (base)
           + 0.2 (steps >= 3)
           + 0.15 (logical words >= 3)
           + 0.1 (structured markers)
           + 0.05 (clear conclusion)
           ────────────────
           Max 1.0
```

---

## 12.7 Low Confidence Handling

When `RequireExplanation=true` and confidence is below 0.7, Shannon requests clarification:

```go
if config.RequireExplanation && confidence < 0.7 {
    clarificationPrompt := fmt.Sprintf(
        "The previous reasoning for '%s' had unclear steps. "+
        "Please provide a clearer step-by-step explanation:\n%s",
        query,
        strings.Join(steps, config.StepDelimiter),
    )

    // Use half budget to regenerate
    clarifyResult := executeAgentWithBudget(ctx, clarificationPrompt, opts.BudgetAgentMax/2)

    // Update result
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

Clarification strategy:
- Use original steps as reference
- Use half the budget (control costs)
- Request clearer explanation

---

## 12.8 CoT vs Tree-of-Thoughts

CoT is linear: one step follows another, no going back.

Tree-of-Thoughts (ToT) is tree-shaped: each step can have multiple branches, with backtracking.

| Feature | Chain-of-Thought | Tree-of-Thoughts |
|---------|------------------|------------------|
| **Structure** | Linear chain | Branching tree |
| **Exploration** | Single path | Multiple paths in parallel |
| **Backtracking** | Not supported | Supported |
| **Token consumption** | Lower | Higher (3-10x) |
| **Use case** | Deterministic reasoning | Exploratory problems |

### When to Use ToT?

```
Does the problem have multiple possible solution paths?
├─ No → Use CoT (single path is enough)
└─ Yes → Need to compare different approaches?
         ├─ No → Use CoT (pick one randomly)
         └─ Yes → Use ToT (systematic exploration)
```

ToT is covered in detail in Chapter 17. For now, just know: **CoT is sufficient for most scenarios**.

---

## 12.9 Common Pitfalls

### Pitfall 1: Over-decomposition

**Symptom**: Simple problems are forced into too many steps, verbose output.

```
// Asked "What is 2+3?"
→ Step 1: Identify problem type—this is an addition problem
→ Step 2: Determine operands—2 and 3
→ Step 3: Review definition of addition
→ Step 4: Execute calculation 2 + 3 = 5
→ Step 5: Verify result
Therefore: 5
```

**Solution**: Dynamically adjust MaxSteps based on complexity:

```go
func adaptiveMaxSteps(query string) int {
    complexity := estimateComplexity(query)
    if complexity < 0.3 {
        return 2  // Simple problem
    } else if complexity < 0.7 {
        return 5  // Medium
    }
    return 8      // Complex
}
```

### Pitfall 2: Confusing Reasoning with Facts

**Symptom**: CoT generates steps that "look reasonable" but are based on wrong facts.

```
→ Step 1: Tesla became the world's most valuable automaker in 2020 (wrong fact)
→ Step 2: Therefore its sales should also be highest (wrong reasoning)
```

The problem: logic is correct, but premises are wrong, so conclusion is wrong.

**Solution**: Use CoT with tools to verify key facts:

```
When reasoning, follow these principles:
1. When specific data is involved, mark [needs verification]
2. Distinguish "reasoning" from "factual statements"
3. If uncertain about a fact, explicitly state it
```

### Pitfall 3: Inflated Confidence

**Symptom**: Model uses many logical connectors, but actual reasoning quality is poor.

For example, circular reasoning:
```
→ Step 1: A is true because B is true
→ Step 2: B is true because A is true
Therefore: Both A and B are true
```

Using "because" adds confidence points, but this is invalid reasoning.

**Solution**: Add semantic detection:

```go
func enhancedConfidence(steps []string, response string) float64 {
    base := calculateConfidence(steps, response)

    // Check for circular reasoning
    if hasCircularReasoning(steps) {
        base -= 0.3
    }

    // Check logical coherence between steps
    if !hasLogicalCoherence(steps) {
        base -= 0.2
    }

    return max(0, min(1.0, base))
}
```

### Pitfall 4: Inconsistent Format

**Symptom**: LLM sometimes uses "→", sometimes "Step", sometimes numbers—parsing fails.

**Solution**: Clearly specify format in prompt, and support multiple formats in parsing (Shannon already does this).

---

## 12.10 When to Use CoT?

Not all tasks need CoT.

| Task Type | Use CoT? | Reason |
|-----------|----------|--------|
| Simple calculation | No | Direct computation is faster |
| Fact lookup | No | Direct lookup is more accurate |
| Multi-step math | Yes | Reduce calculation errors |
| Logical reasoning | Yes | Externalize reasoning chain |
| Causal analysis | Yes | Trace cause-effect relationships |
| Code debugging | Yes | Systematic investigation |
| Creative writing | No | Would limit creativity |
| Real-time conversation | Depends | Latency vs accuracy trade-off |

**Rule of thumb**:

- Needs "derivation" → use
- Needs "auditable process" → use
- Simple and direct → don't use
- Creative → don't use
- Latency sensitive → use carefully

---

## 12.11 How Do Other Frameworks Do It?

CoT is a universal pattern; everyone has implementations:

| Framework/Paper | Implementation | Characteristics |
|-----------------|---------------|-----------------|
| **Zero-shot CoT** | "Let's think step by step" | Simplest, one sentence triggers |
| **Few-shot CoT** | Provide reasoning examples | More controllable, but needs manual design |
| **Self-Consistency** | Multiple CoT + voting | More accurate, but expensive |
| **LangChain** | CoT Prompt templates | Easy to integrate |
| **OpenAI o1/o3** | Built-in multi-step reasoning (black box) | Internal mechanism opaque, no manual triggering needed |

Core logic is the same: make LLM write out reasoning process.

Differences are in:
- Triggering method (zero-shot vs few-shot)
- Format constraints (free vs structured)
- Quality assurance (single vs multi-vote)

---

## 12.12 Relationship with ReAct

You might ask: what's the difference between CoT and ReAct?

| Dimension | CoT | ReAct |
|-----------|-----|-------|
| **Core goal** | Externalize reasoning process | Reasoning + action loop |
| **Uses tools** | No (pure reasoning) | Yes (think while doing) |
| **Output** | Reasoning steps + answer | Multi-round thought/action/observation |
| **Use case** | Problems requiring computation/reasoning | Tasks requiring external information |

Simply put:

- **CoT**: Think clearly then answer (doesn't need external information)
- **ReAct**: Think while searching and doing (needs external information)

They can be combined: use CoT during ReAct's "thinking" phase.

---

## Key Takeaways

1. **CoT Essence**: Externalize thinking, force model to reason step by step
2. **Prompt Design**: Clear instruction "step by step" + format convention (→, Step)
3. **Step Parsing**: Recognize markers + multi-layer fallback strategy
4. **Confidence Evaluation**: Step count + logical words + structured markers (heuristic, not academic standard)
5. **Use Cases**: Multi-step reasoning, needs audit trail; not for simple tasks, creative work

---

## Shannon Lab (10-Minute Quick Start)

This section helps you map this chapter's concepts to Shannon source code in 10 minutes.

### Required Reading (1 file)

- [`patterns/chain_of_thought.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/chain_of_thought.go): Find the `ChainOfThought` function, see how it uses `buildChainOfThoughtPrompt` to build prompts, `parseReasoningSteps` to parse reasoning steps, and `calculateReasoningConfidence` to evaluate confidence

### Optional Deep Dive (2 files, choose based on interest)

- [`patterns/tree_of_thoughts.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go): Compare implementation differences between ToT and CoT
- Try it yourself in ChatGPT/Claude: same math problem, with and without "Let's think step by step"—what's different about the answers?

---

## Exercises

### Exercise 1: Design CoT Templates

Design specialized CoT prompt templates for these scenarios:

1. **Legal reasoning**: Determine whether an action is illegal
2. **Medical diagnosis**: Infer possible diseases from symptoms
3. **Financial analysis**: Evaluate investment value of a stock

Each template should include:
- Problem description placeholder
- Format requirements for reasoning steps
- Format requirements for conclusion

### Exercise 2: Source Code Reading

Read the `parseReasoningSteps` function in `patterns/chain_of_thought.go`:

1. What step marker formats does it support?
2. How does it fall back when LLM doesn't use any markers?
3. Why does fallback limit to maximum 5 steps?

### Exercise 3 (Advanced): Design Circular Reasoning Detection

Design a `hasCircularReasoning` function:

- Input: List of reasoning steps
- Output: Whether circular reasoning exists

Think about:
- What patterns count as "circular reasoning"?
- What method to use for detection? (keyword matching? semantic similarity?)
- Is there false positive risk?

---

## Want to Go Deeper?

- [Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903) - Wei et al., 2022, original paper
- [Zero-shot CoT: "Let's think step by step"](https://arxiv.org/abs/2205.11916) - Simplest CoT triggering method
- [Self-Consistency Decoding](https://arxiv.org/abs/2203.11171) - Multiple CoT + voting improves accuracy
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601) - Tree-shaped extension of CoT

---

## Next Chapter Preview

That concludes Part 4 (Single Agent Patterns). We learned three core patterns:

- **Planning**: Decompose complex tasks into subtasks
- **Reflection**: Evaluate output quality, retry if not meeting standards
- **Chain-of-Thought**: Externalize reasoning process, reduce jumping errors

But what a single Agent can do is limited. When tasks are complex enough, you need multiple Agents to collaborate.

That's what Part 5 is about—**Multi-Agent Orchestration**.

Next chapter we'll start with orchestration basics: when a single Agent isn't enough, how to have multiple Agents divide work? Who decides who does what? What happens when something fails?
