# 附录 B：模式选择指南 (Pattern Selection Guide)

> **本附录帮助你快速决定「什么场景用什么模式」，避免过度设计或选错工具。**

---

生产环境里最常见的问题不是"不知道怎么实现某个模式"，而是"不知道该用哪个模式"。

你可能会问：

- "这个任务适合 ReAct 还是 Planning？"
- "什么时候需要 Reflection？"
- "ToT 和 Debate 有什么区别？"
- "多 Agent 编排到底解决了什么问题？"

本附录通过决策树、对比表、反模式警示，帮你在 5 分钟内找到最合适的模式。

**核心原则**：优先选简单模式，只在必要时升级。复杂模式 = 高成本 + 慢速度 + 难调试。

---

## B.1 单 Agent 推理模式选择

### 快速决策树

```
你的任务需要外部工具吗？（搜索、API、文件读写等）
│
├─ 不需要 → 是否需要显式推理过程？
│             ├─ 需要 → Chain-of-Thought (第 12 章)
│             └─ 不需要 → 直接 Prompt（不需要模式）
│
└─ 需要 → 任务是否需要拆解成多步？
           │
           ├─ 不需要 → ReAct (第 2 章)
           │
           └─ 需要拆解 → 拆解后的步骤有明确依赖吗？
                        │
                        ├─ 没有依赖 → ReAct (第 2 章)
                        │
                        └─ 有依赖 → Planning (第 10 章)
                                    ↓
                        Planning 完成后，质量是否达标？
                        │
                        ├─ 达标 → 完成
                        │
                        └─ 不达标 → 需要提升质量还是需要多角度？
                                   │
                                   ├─ 提升质量 → Reflection (第 11 章)
                                   │
                                   └─ 多角度/有争议 → 需要探索多条路径吗？
                                                    │
                                                    ├─ 需要探索 → Tree-of-Thoughts (第 17 章)
                                                    │
                                                    └─ 需要对抗 → Debate (第 18 章)
```

### 单 Agent 模式对比矩阵

| 模式 | Token 成本 | 延迟 | 质量提升 | 适用场景 | 不适用场景 | 章节 |
|------|-----------|------|----------|----------|-----------|------|
| **ReAct** | 中 (基准) | 低 (基准) | 基准 | 需要工具调用的通用任务 | 纯推理无需工具 | 第 2 章 |
| **Planning** | 高 (+50%) | 中 (+30%) | +20% | 复杂多步任务需拆解 | 单步可完成的任务 | 第 10 章 |
| **Reflection** | 很高 (+100%) | 高 (+100%) | +30% | 高价值输出需迭代改进 | 简单任务/延迟敏感 | 第 11 章 |
| **CoT** | 中 (+10%) | 低 | +15% | 数学推理/逻辑分析 | 需要工具调用的任务 | 第 12 章 |
| **ToT** | 很高 (+200-400%) | 很高 | +40% | 多路径探索/早期决策关键 | 有明确解法的问题 | 第 17 章 |
| **Debate** | 极高 (+300-500%) | 极高 | +50% | 争议话题/需多视角 | 有客观答案的问题 | 第 18 章 |
| **Research** | 高 (+80%) | 高 | +35% | 系统性研究/报告生成 | 简单信息查询 | 第 19 章 |

**成本说明**：百分比相对于 ReAct 基准。实际成本受任务复杂度、配置参数影响。

### 何时使用 ReAct

**使用场景**：
- 需要调用外部工具（搜索、API、文件操作）
- 任务相对简单，不需要复杂规划
- 需要透明的推理过程（可审计）
- 成本敏感，需要快速响应

**不适用场景**：
- 纯推理任务，不需要外部工具
- 任务过于复杂，需要提前规划
- 需要高质量输出，单次生成不够

**成本考量**：
- Token: ~3000-8000 per task
- 延迟: ~10-30 秒
- 失败重试成本低

**示例**：
```
✓ "帮我查查为什么 API 返回 500 错误"
✓ "搜索最新的 Python 3.12 新特性并总结"
✗ "研究这家公司并写 5000 字报告" (用 Planning + Research)
✗ "这个系统应该用微服务还是单体？" (用 Debate)
```

### 何时使用 Planning

**使用场景**：
- 任务可以拆解成 3+ 个独立子任务
- 子任务之间有依赖关系
- 需要并行执行提升效率
- 输出需要结构化组织

**不适用场景**：
- 单步可完成的简单任务
- 任务无法明确拆解
- 延迟敏感的实时场景

**成本考量**：
- Token: ~5000-15000 per task
- 延迟: ~30-90 秒
- 分解开销: ~1000 tokens
- 综合开销: ~1500 tokens

**关键配置**：
```go
MaxIterations:     3    // 覆盖度评估最大迭代
MinCoverage:       0.85 // 最低覆盖度阈值
MaxSubtasks:       7    // 子任务数量上限
```

**示例**：
```
✓ "分析特斯拉 2024 年财务表现，包括收入、利润、对比竞争对手"
✓ "帮我研究 OpenAI，写一份完整的竞争分析报告"
✗ "今天天气怎么样" (用 ReAct)
✗ "帮我写个排序函数" (用 ReAct)
```

### 何时使用 Reflection

**使用场景**：
- 输出质量要求高（报告、文档、分析）
- 有明确的评估标准
- 成本不敏感，质量优先
- 单次生成质量不稳定

**不适用场景**：
- 简单问答，质量要求不高
- 延迟敏感（会翻倍时间）
- 成本敏感
- 没有客观评估标准

**成本考量**：
- Token: 初始成本 x 2-3
- 延迟: 初始延迟 x 2-3
- MaxRetries = 1-2 足够

**关键配置**：
```go
MaxRetries:          2    // 最多重试次数
ConfidenceThreshold: 0.7  // 质量阈值（不要设太高）
Criteria: []string{
    "completeness",
    "correctness",
    "clarity",
}
```

**示例**：
```
✓ "生成 API 技术文档"（高质量要求）
✓ "写一份投资尽调报告"（准确性关键）
✗ "帮我查一下明天天气"（不值得）
✗ "实时聊天回复"（延迟敏感）
```

### 何时使用 Chain-of-Thought

**使用场景**：
- 数学推理、逻辑分析
- 需要展示思考过程
- 不需要外部工具
- 减少跳跃性错误

**不适用场景**：
- 需要调用外部工具（用 ReAct）
- 需要探索多条路径（用 ToT）
- 简单事实查询

**成本考量**：
- Token: +10% vs 直接回答
- 延迟: 几乎不增加
- 提升推理准确性 ~15%

**示例**：
```
✓ "解这道数学题：24 点游戏 (3,8,8,3)"
✓ "分析这段代码的时间复杂度"
✗ "搜索最新的 AI 新闻" (用 ReAct)
✗ "设计一个高可用架构" (用 ToT 或 Debate)
```

### 何时使用 Tree-of-Thoughts

**使用场景**：
- 解空间大，有多条可能路径
- 早期决策影响大，走错难回头
- 可以评估中间状态质量
- 质量 > 成本/速度

**不适用场景**：
- 有明确标准解法
- 成本/延迟敏感
- 无法评估中间状态

**成本考量**：
- Token: +200-400% (节点数 x 单次成本)
- 延迟: +200-300%
- ExplorationBudget 控制上限

**关键配置**：
```go
MaxDepth:          3    // 树深度
BranchingFactor:   3    // 每节点分支数
PruningThreshold:  0.3  // 剪枝阈值
ExplorationBudget: 15   // 最多探索节点数
```

**示例**：
```
✓ "设计一个支撑 100 万 QPS 的支付系统架构"
✓ "找到这个复杂 bug 的根因（多个可能原因）"
✗ "实现一个标准的用户登录功能" (有成熟模板)
✗ "查询数据库某个字段的值" (明确操作)
```

### 何时使用 Debate

**使用场景**：
- 争议性话题，没有标准答案
- 需要多角度分析
- 需要对抗性质疑
- 决策风险高，需要充分论证

**不适用场景**：
- 有客观正确答案
- 成本/延迟极度敏感
- 简单决策

**成本考量**：
- Token: NumDebaters x MaxRounds x 单次成本
- 延迟: 串行执行各轮
- 3 辩手 x 2 轮 = 6x 基准成本

**关键配置**：
```go
NumDebaters:      3     // 2-5 个辩论者
MaxRounds:        2     // 2-3 轮足够
Perspectives:     []string{"optimistic", "skeptical", "practical"}
RequireConsensus: false // 不强制共识
VotingEnabled:    true  // 投票兜底
```

**示例**：
```
✓ "AI 会在 2025 年取代 SaaS 吗？"
✓ "公司应该自建机房还是用云？"
✗ "2 + 2 等于几？" (客观答案)
✗ "Python 的语法是什么？" (事实性问题)
```

---

## B.2 多 Agent 编排模式选择

### 快速决策树

```
你的任务需要多个 Agent 协作吗？
│
├─ 不需要 → 用单 Agent 模式（见 B.1）
│
└─ 需要 → 任务是否可以提前完全规划？
           │
           ├─ 可以完全规划 → 子任务之间有依赖吗？
           │                 │
           │                 ├─ 没有依赖 → Parallel (第 14 章)
           │                 │
           │                 ├─ 部分依赖 → DAG (第 14 章)
           │                 │
           │                 └─ 全串行 → Sequential (第 14 章)
           │
           └─ 无法完全规划 → 需要动态调整吗？
                            │
                            ├─ 需要 → Supervisor (第 15 章)
                            │
                            └─ 不需要 → 任务间需要交接吗？
                                       │
                                       ├─ 需要 → Handoff (第 16 章)
                                       │
                                       └─ 不需要 → DAG (第 14 章)
```

### 多 Agent 模式对比矩阵

| 模式 | 调度复杂度 | 协调成本 | 适用场景 | 不适用场景 | 章节 |
|------|-----------|---------|----------|-----------|------|
| **Parallel** | 低 | 低 | 独立子任务并行执行 | 任务间有依赖 | 第 14 章 |
| **Sequential** | 低 | 中 | 严格顺序依赖 | 任务可并行 | 第 14 章 |
| **DAG** | 中 | 中 | 部分并行+依赖 | 无法确定依赖 | 第 14 章 |
| **Supervisor** | 高 | 高 | 动态任务分配 | 可提前规划 | 第 15 章 |
| **Handoff** | 低 | 中 | 专业化分工 | 无需专业化 | 第 16 章 |

### 何时使用 Parallel 执行

**使用场景**：
- 3+ 个完全独立的子任务
- 可以同时执行提升效率
- 任务间无数据依赖

**不适用场景**：
- 任务间有依赖关系
- 任务数量太少 (1-2 个)
- 后续任务需要前序结果

**成本考量**：
- 并行度: MaxConcurrency = 3-5（个人账号）
- 并行度: MaxConcurrency = 5-10（团队账号）
- 节省时间: ~40-60% vs 串行

**示例**：
```
✓ "搜索 Tesla、BYD、Rivian 三家公司的股价"
✓ "翻译这段文字成英语、日语、法语"
✗ "先搜股价，再计算涨幅" (有依赖，用 Sequential)
✗ "搜索一家公司的信息" (只有 1 个任务，不需要并行)
```

### 何时使用 Sequential 执行

**使用场景**：
- 后续任务明确依赖前序结果
- 需要结果传递
- 逻辑链式推进

**不适用场景**：
- 任务可以并行
- 任务间无依赖

**成本考量**：
- 延迟: 各任务延迟累加
- PassPreviousResults: 增加 ~10% token
- 适合: 3-5 个顺序步骤

**示例**：
```
✓ "先搜索特斯拉股价，然后计算涨幅，最后生成分析"
✓ "读取文件 → 分析数据 → 生成报告"
✗ "搜索三个公司的股价" (可并行，用 Parallel)
```

### 何时使用 DAG 工作流

**使用场景**：
- 5+ 个子任务，部分可并行
- 有明确的依赖关系
- 需要智能调度优化效率
- 可以提前规划依赖

**不适用场景**：
- 所有任务都独立（用 Parallel）
- 所有任务都串行（用 Sequential）
- 依赖关系无法确定（用 Supervisor）
- 任务数量太少 (<3 个)

**成本考量**：
- 调度开销: ~5% 额外 token
- 节省时间: ~20-40% vs 纯串行
- 适合: 5-15 个子任务

**关键配置**：
```go
MaxConcurrency:         5      // 最大并发数
DependencyWaitTimeout:  360    // 依赖等待超时（秒）
PassDependencyResults:  true   // 传递依赖结果
```

**示例**：
```
✓ "分析特斯拉财务：获取财报(A) + 获取竞品(B) → 计算增长(C,依赖A) + 计算利润率(D,依赖A) → 对比分析(E,依赖A,B,C,D)"
✗ "搜索三个公司" (无依赖，用 Parallel)
✗ "动态决定下一步" (无法提前规划，用 Supervisor)
```

### 何时使用 Supervisor 模式

**使用场景**：
- 任务数量/类型不确定
- 需要动态分配任务
- Agent 间需要实时通信
- 部分失败需要智能降级

**不适用场景**：
- 任务可以提前完全规划
- 简单固定流程
- 成本极度敏感

**成本考量**：
- 调度开销: +20-30% token
- Supervisor 决策: 每轮 ~1000 tokens
- 适合: 复杂、动态场景

**示例**：
```
✓ "协调多个专家 Agent 解决复杂技术问题（动态调整策略）"
✓ "客服系统动态路由到不同专家 Agent"
✗ "固定的数据处理流水线" (用 DAG)
✗ "简单的并行搜索" (用 Parallel)
```

### 何时使用 Handoff 机制

**使用场景**：
- 需要专业化分工
- 任务交接明确
- 上下文需要传递
- 类似工作流转

**不适用场景**：
- 不需要专业化
- 无交接点
- 任务完全独立

**成本考量**：
- 交接开销: 每次 ~500 tokens
- 上下文传递: 视大小而定

**示例**：
```
✓ "客服 Agent → 技术支持 Agent → 退款 Agent"
✓ "需求分析 Agent → 设计 Agent → 实现 Agent"
✗ "并行搜索三个网站" (无需交接，用 Parallel)
```

---

## B.3 反模式警示 (Anti-Patterns)

### 何时 NOT 使用多 Agent

**反模式 1：为了多 Agent 而多 Agent**

```
错误示例：
任务："查询今天天气"
设计：Agent-1 搜索 → Agent-2 解析 → Agent-3 格式化

问题：3 个 Agent = 3x 成本，但单 Agent 5 秒就能搞定

正确做法：单 ReAct Agent 直接完成
```

**反模式 2：过度拆分**

```
错误示例：
任务："分析一家公司"
拆分：20 个细粒度子任务（公司名、成立时间、CEO、融资、产品1、产品2...）

问题：协调成本 > 执行成本

正确做法：3-7 个粗粒度任务（基本信息、产品矩阵、融资历史）
```

**反模式 3：强行并行**

```
错误示例：
任务："先搜股价，再算涨幅"
强行并行：Agent-1 搜股价 || Agent-2 算涨幅（没数据！）

问题：依赖关系被忽略

正确做法：Sequential 或 DAG
```

**反模式 4：无限迭代**

```
错误示例：
Planning with RequirePerfectCoverage = true
MaxIterations = 100

问题：永远达不到 100%，Token 烧光

正确做法：CoverageThreshold = 0.85, MaxIterations = 3
```

### 常见过度设计陷阱

| 陷阱 | 表现 | 后果 | 解决方案 |
|------|------|------|----------|
| **工具崇拜** | 新工具一出就用 | 增加复杂度，收益不明显 | 评估 ROI，先小范围试点 |
| **过早优化** | 一开始就用最复杂模式 | 难调试，成本高 | 从简单模式开始，逐步升级 |
| **配置爆炸** | 暴露几十个配置参数 | 用户不知道怎么调 | 提供 3 套预设：fast/balanced/quality |
| **盲目跟风** | 看到论文就实现 | 学术效果 ≠ 生产效果 | 先验证必要性，再考虑实现 |

### 决策原则

**奥卡姆剃刀原则**：
> 如无必要，勿增实体。能用简单模式解决的，不要用复杂模式。

**渐进增强原则**：
```
Level 1: 单次 LLM 调用
         ↓ 效果不够？
Level 2: ReAct（加工具）
         ↓ 任务太复杂？
Level 3: Planning（加拆解）
         ↓ 质量不够？
Level 4: Reflection（加迭代）
         ↓ 需要多角度？
Level 5: ToT / Debate（加探索/对抗）
```

**80/20 法则**：
- 80% 的任务用 ReAct + Planning 就够
- 15% 的任务需要 Reflection
- 5% 的任务才需要 ToT / Debate

---

## B.4 成本-质量-延迟权衡矩阵

### 三维权衡

```
              质量
               ▲
               │
    Debate ●   │
           │   │   ● ToT
  Research ●   │
           │   │
Reflection ●   │
           │   │
  Planning ●   │   ● DAG
           │   │
       CoT ●   │
           │   │
     ReAct ●───┼────────────────► 延迟
               │
               │
               ▼
              成本
```

### 场景优化策略

**延迟敏感场景**（实时对话、在线查询）
- 优先：ReAct、CoT
- 避免：Reflection、ToT、Debate
- 配置：MaxIterations = 5, Timeout = 30s

**成本敏感场景**（大规模批处理）
- 优先：ReAct、Planning
- 避免：ToT、Debate、Research
- 配置：BudgetAgentMax = 5000, 用小模型评估

**质量敏感场景**（研究报告、决策支持）
- 优先：Planning + Reflection、Research、Debate
- 成本：可以接受 2-5x
- 配置：ConfidenceThreshold = 0.8, MaxRetries = 2

### 成本估算公式

**单 Agent 模式**：
```
ReAct:      Base
Planning:   Base × (1 + 0.5 × NumSubtasks)
Reflection: Base × (1 + MaxRetries)
CoT:        Base × 1.1
ToT:        Base × ExplorationBudget
Debate:     Base × NumDebaters × MaxRounds
```

**多 Agent 模式**：
```
Parallel:   Base × NumTasks / MaxConcurrency (时间优化)
Sequential: Base × NumTasks (时间累加)
DAG:        Base × NumTasks × 0.6-0.8 (部分并行)
Supervisor: Base × (NumAgents + SupervisorOverhead)
```

---

## B.5 快速选择速查表

### 按任务类型选择

| 任务类型 | 推荐模式 | 原因 | 章节 |
|---------|---------|------|------|
| **简单信息查询** | ReAct | 直接工具调用 | 第 2 章 |
| **复杂研究报告** | Planning + Research | 需要拆解和系统性研究 | 第 10、19 章 |
| **技术选型决策** | Debate | 需要多视角权衡 | 第 18 章 |
| **架构设计** | ToT 或 Debate | 探索多种方案或对抗讨论 | 第 17、18 章 |
| **代码调试** | ReAct 或 ToT | 简单用 ReAct，复杂用 ToT | 第 2、17 章 |
| **数学推理** | CoT | 展示推理过程 | 第 12 章 |
| **文档生成** | Planning + Reflection | 结构化 + 质量保证 | 第 10、11 章 |
| **数据分析** | Planning + DAG | 拆解 + 并行处理 | 第 10、14 章 |
| **客服路由** | Supervisor 或 Handoff | 动态分配或专业化 | 第 15、16 章 |
| **工作流执行** | DAG | 固定流程 + 依赖管理 | 第 14 章 |

### 按约束条件选择

| 首要约束 | 推荐模式 | 避免模式 |
|---------|---------|---------|
| **延迟 < 10 秒** | ReAct、CoT | ToT、Debate、Reflection |
| **成本 < $0.01** | ReAct（小模型） | ToT、Debate |
| **质量 > 90%** | Planning + Reflection + Debate | 单次 ReAct |
| **可解释性** | CoT、ReAct | 黑盒 API |
| **可靠性** | DAG（失败重试）| 单点 Agent |

### 按团队成熟度选择

| 团队阶段 | 推荐起步 | 逐步引入 | 暂缓引入 |
|---------|---------|---------|---------|
| **探索期** | ReAct、CoT | Planning | ToT、Debate |
| **成长期** | Planning、Reflection | DAG、Supervisor | - |
| **成熟期** | 全套模式 | 自定义模式 | - |

---

## B.6 真实案例选型

### 案例 1：竞争对手分析

**需求**：分析 3 家竞争对手，生成对比报告

**初步选择**：Planning（拆解）+ Parallel（并行搜索）

**优化方向**：
- 质量要求高 → 加 Reflection
- 有争议性（如技术路线对比）→ 加 Debate

**最终方案**：
```
Planning (拆解为 3 个子任务)
  → Parallel (并行搜索 3 家公司)
  → Synthesis (综合)
  → Reflection (质量检查)
```

**成本估算**：
- Base (ReAct): ~5000 tokens
- Planning: +2500 tokens
- Parallel (x3): +10000 tokens
- Reflection: +5000 tokens
- Total: ~22500 tokens (~$0.03 with GPT-4o-mini)

### 案例 2：技术架构设计

**需求**：设计一个高并发支付系统

**初步选择**：ToT（探索多种架构方案）

**优化方向**：
- 有明确约束（如成本、技术栈）→ 用 Debate 而非 ToT
- 需要专家视角 → 配置 3 个 Perspective（性能、成本、安全）

**最终方案**：
```
Debate (3 视角: 性能优先、成本优先、安全优先)
  → 2 轮辩论
  → 主持人综合
```

**成本估算**：
- Base: ~8000 tokens
- Debate (3 × 2 轮): ~48000 tokens
- Total: ~56000 tokens (~$0.07 with GPT-4o-mini)

### 案例 3：客服智能路由

**需求**：根据用户问题路由到不同专家 Agent

**初步选择**：Supervisor（动态路由）

**优化方向**：
- 路由规则相对固定 → 用 Handoff 而非 Supervisor
- 节省 Supervisor 决策成本

**最终方案**：
```
分类 Agent (判断问题类型)
  → Handoff 到对应专家
  → 专家 Agent 处理
```

**成本估算**：
- 分类: ~1000 tokens
- 专家处理: ~5000 tokens
- Handoff 开销: ~500 tokens
- Total: ~6500 tokens per request

---

## B.7 配置模板参考

### Fast 模板（延迟优先）

```yaml
mode: react
config:
  max_iterations: 5
  timeout_seconds: 30
  budget_agent_max: 5000

# 不启用
reflection_enabled: false
debate_enabled: false
tot_enabled: false
```

### Balanced 模板（平衡）

```yaml
mode: planning
config:
  max_iterations: 10
  timeout_seconds: 120
  budget_agent_max: 15000

  # Planning
  max_subtasks: 7
  min_coverage: 0.85

  # Reflection (可选)
  reflection_enabled: true
  reflection_max_retries: 1
  reflection_confidence_threshold: 0.7
```

### Quality 模板（质量优先）

```yaml
mode: research
config:
  max_iterations: 15
  timeout_seconds: 300
  budget_agent_max: 30000

  # Research
  max_research_rounds: 3
  coverage_threshold: 0.9

  # Reflection
  reflection_enabled: true
  reflection_max_retries: 2
  reflection_confidence_threshold: 0.8

  # Debate (可选)
  debate_enabled: true
  debate_num_debaters: 3
  debate_max_rounds: 2
```

---

## B.8 调试决策流程

当任务效果不理想时：

```
效果不好？
│
├─ 是否调用了工具？
│  ├─ 没调用 → 检查 MinIterations，强制工具使用
│  └─ 调用了 → 继续
│
├─ 是否拆解了任务？
│  ├─ 没拆解 → 任务复杂度 > 0.5？考虑启用 Planning
│  └─ 拆解了 → 继续
│
├─ 质量是否达标？
│  ├─ 不达标 → 尝试 Reflection (MaxRetries=1)
│  └─ 达标但有争议 → 考虑 Debate
│
└─ 成本是否可接受？
   ├─ 太高 → 降低 MaxIterations、BudgetAgentMax
   └─ 可接受 → 继续优化配置参数
```

---

## B.9 总结与决策原则

### 黄金法则

1. **奥卡姆剃刀**：能简单就别复杂
2. **渐进增强**：从 ReAct 开始，逐步升级
3. **评估 ROI**：成本增加是否带来等价质量提升
4. **明确约束**：延迟/成本/质量，最多优化两个

### 常见决策路径

**90% 的任务**：
- ReAct (需要工具)
- CoT (纯推理)

**5% 的任务**：
- Planning (复杂拆解)
- DAG (并行优化)

**3% 的任务**：
- Reflection (高质量)
- Research (系统研究)

**2% 的任务**：
- ToT (多路径探索)
- Debate (争议话题)

### 最后的建议

**在生产环境中**：
- 先用 ReAct 跑通流程
- 识别瓶颈（质量/效率/成本）
- 针对性引入高级模式
- 持续监控 ROI

**不要追求完美**：
- 85% 的质量可能已经够用
- 最后 15% 的提升往往需要 5x 成本
- 边际收益递减

**记住**：
> 正确的模式选择 > 完美的参数调优

---

## 延伸阅读

- **第 2 章**：ReAct 循环 - 基础模式
- **第 10 章**：Planning 模式 - 任务拆解
- **第 11 章**：Reflection 模式 - 质量保证
- **第 12 章**：Chain-of-Thought - 推理展示
- **第 14 章**：DAG 工作流 - 多 Agent 编排
- **第 17 章**：Tree-of-Thoughts - 多路径探索
- **第 18 章**：Debate 模式 - 对抗讨论
- **第 23 章**：Token 预算控制 - 成本管理
- **附录 A**：案例研究（如有）- 真实场景

---

**下一步**：根据本指南选择合适的模式后，参考对应章节深入学习实现细节。
