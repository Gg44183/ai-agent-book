# Part 9: 前沿实践

> 最新热点：Computer Use、Agentic Coding、Background Agents、分层模型策略

## 章节列表

| 章节 | 标题 | 核心问题 | Shannon关联 |
|------|------|----------|-------------|
| 27 | Computer Use | 如何让Agent操作浏览器和桌面？ | `config/models.yaml` multimodal |
| 28 | Agentic Coding | 如何构建代码生成Agent？ | `file_ops.py`, `wasi_sandbox.rs` |
| 29 | Background Agents | 如何实现异步长时任务？ | `schedules/manager.go` |
| 30 | 分层模型策略 | 如何优化50-70%的成本？ | `config/models.yaml`, `manager.py` |

---

## 章节摘要

### 第 27 章：Computer Use

> 当 Agent 获得"眼睛"和"手"：从调用 API 到操作真实界面

**核心内容**:
- **感知-决策-执行循环**: 截屏理解 → 坐标计算 → 点击/输入 → 结果验证
- **多模态模型集成**: 视觉理解是 Computer Use 的关键能力
- **坐标校准**: 处理不同分辨率和 DPI 缩放差异
- **安全防护**: 危险区域检测、输入内容过滤、OPA 策略扩展
- **验证循环**: 每次操作后截图验证结果，失败自动重试

**Shannon 代码**: `config/models.yaml` (multimodal_models), 建议工具扩展模式

---

### 第 28 章：Agentic Coding

> 让 Agent 成为你的编程伙伴：从代码生成到完整开发工作流

**核心内容**:
- **安全文件操作**: 白名单目录、路径验证、符号链接防护
- **WASI 沙箱执行**: Fuel/Epoch 限制、内存隔离、超时控制
- **代码反思循环**: 生成 → 审查 → 改进的迭代过程
- **多文件编辑协调**: 原子化变更、备份回滚机制
- **Git 集成**: 分支管理、自动提交、PR 创建

**Shannon 代码**: `python/llm-service/llm_service/tools/builtin/file_ops.py`, `rust/agent-core/src/wasi_sandbox.rs`, `go/orchestrator/internal/workflows/patterns/reflection.go`

---

### 第 29 章：Background Agents

> 让任务在后台持续运行：Temporal 调度与定时任务系统

**核心内容**:
- **Temporal Schedule API**: 原生 Cron 调度、暂停/恢复、时区支持
- **资源限制**: MaxPerUser (50)、MinCronInterval (60min)、MaxBudgetPerRunUSD ($10)
- **ScheduledTaskWorkflow**: 包装器工作流，记录执行元数据（模型、Token、成本）
- **孤儿检测**: 定期检测 Temporal 与数据库状态不一致，自动清理
- **预算注入**: 每次执行的成本追踪与限制

**Shannon 代码**: `schedules/manager.go`, `scheduled_task_workflow.go`

---

### 第 30 章：分层模型策略

> 智能路由实现 50-70% 成本降低：不是每个任务都需要最强模型

**核心内容**:
- **三层架构**: Small (50%) / Medium (40%) / Large (10%) 目标分布
- **优先级路由**: 同层级多 Provider 按优先级选择，自动 Fallback
- **复杂度分析**: 根据任务特征自动选择模型层级
- **能力矩阵**: multimodal、thinking、coding、long_context 能力标记
- **熔断降级**: Circuit Breaker + 自动降级到备选 Provider
- **成本追踪**: 集中式定价配置、实时成本监控

**Shannon 代码**: `config/models.yaml`, `llm_provider/manager.py`

---

## 学习目标

完成本 Part 后，你将能够：

- [ ] 理解 Computer Use 的感知-决策-执行循环
- [ ] 设计安全的 Agentic Coding 工作流（沙箱 + 反思）
- [ ] 使用 Temporal Schedule API 实现定时后台任务
- [ ] 配置三层模型策略实现 50-70% 成本降低
- [ ] 为 Research Agent 添加前沿能力 (v0.9)

---

## Shannon 代码导读

```
Shannon/
├── config/
│   └── models.yaml                    # 三层模型配置、定价、能力矩阵
├── go/orchestrator/
│   └── internal/
│       ├── schedules/
│       │   └── manager.go             # 定时任务管理器 (CRUD, 资源限制)
│       └── workflows/scheduled/
│           └── scheduled_task_workflow.go  # 包装器工作流
├── python/llm-service/
│   ├── llm_provider/
│   │   └── manager.py                 # LLM管理器 (路由, 熔断, Fallback)
│   └── llm_service/tools/builtin/
│       ├── file_ops.py                # 安全文件读写工具
│       └── python_wasi_executor.py    # Python沙箱执行
└── rust/agent-core/src/sandbox/
    └── wasi_sandbox.rs                # WASI沙箱实现
```

---

## 热门话题关联

| 话题 | 代表产品 | Shannon 实现 | 章节 |
|------|----------|-------------|------|
| Computer Use | Claude Computer Use, Manus | 多模态 + 工具扩展 | Ch27 |
| Agentic Coding | Claude Code, Cursor, Windsurf | WASI 沙箱 + 文件工具 | Ch28 |
| Background Agents | Claude Code Ctrl+B | Temporal Schedule API | Ch29 |
| Cost Optimization | 企业降本需求 | 三层模型策略 | Ch30 |

---

## 成本优化示例

```
不分层 (全用 Large):
  1M requests × $0.09/request = $90,000/月

分层策略 (50/40/10):
  Small:  500K × $0.0006  = $300
  Medium: 400K × $0.0018  = $720
  Large:  100K × $0.09    = $9,000
  总计: $10,020/月

节省: $79,980/月 (89%)
```

---

## 前置知识

- Part 1-8 完成（特别是 Part 7-8 的生产架构和企业级特性）
- 浏览器自动化基础 (Playwright/Puppeteer) - 可选
- Cron 表达式基础 - 可选
- 多模型 API 经验 - 可选

---

## Research Agent v0.9

本 Part 涵盖的前沿能力模块：

| 模块 | 章节 | 能力 |
|------|------|------|
| Computer Use | 第27章 | 网页浏览、内容提取 |
| Agentic Coding | 第28章 | 分析脚本生成 |
| Background Agents | 第29章 | 定时研究报告 |
| Tiered Models | 第30章 | 智能模型选择 |

**最终形态**:
```
用户: "每天早上9点生成AI行业日报"

Research Agent v0.9:
1. [Schedule] 创建 Cron 定时任务 (0 9 * * *)
2. [Tiered] 用 Small 模型做复杂度评估
3. [Multi-Agent] 并行执行搜索/分析/写作
4. [Browser] 访问无API网站提取内容
5. [Coding] 生成数据可视化脚本
6. [Budget] 控制每次执行成本 < $2
7. [Output] 发送结构化报告
```
