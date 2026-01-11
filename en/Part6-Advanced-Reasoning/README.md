# Part 6: Advanced Reasoning

> Complex decision scenarios: Tree-of-Thoughts, multi-Agent debate, research synthesis

## Chapter List

| Chapter | Title | Core Question |
|---------|-------|---------------|
| 17 | Tree-of-Thoughts | How to explore multiple reasoning paths? |
| 18 | Debate Pattern | How to improve decision quality through debate? |
| 19 | Research Synthesis | How to synthesize multi-source information into reports? |

## Learning Objectives

After completing this Part, you will be able to:
- Implement ToT (Tree-of-Thoughts) exploration and pruning
- Design multi-Agent debate mechanisms
- Build research synthesis workflows
- Choose appropriate reasoning patterns

## Shannon Code Guide

```
Shannon/
├── go/orchestrator/internal/workflows/
│   ├── patterns/tot.go                 # Tree-of-Thoughts
│   ├── patterns/debate.go              # Debate pattern
│   └── research_workflow.go            # Research synthesis
└── docs/pattern-usage-guide.md
```

## Pattern Selection Guide

| Scenario | Recommended Pattern | Reason |
|----------|---------------------|--------|
| Open-ended problems | ToT | Need to explore multiple possibilities |
| Controversial decisions | Debate | Multi-angle argumentation |
| Information gathering | Research | Multi-source parallel + synthesis |

## Prerequisites

- Part 1-5 completed
- Decision theory fundamentals
- Information retrieval basics
