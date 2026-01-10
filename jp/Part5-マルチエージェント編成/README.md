# Part 5: マルチエージェント編成

> 単一エージェントからマルチエージェントへ：編成、調整、協調

## 章立て

| 章 | タイトル | 核心課題 |
|------|------|----------|
| 13 | 編成の基礎 | 複数のエージェントが協力して働くにはどうするか |
| 14 | DAGワークフロー | タスクの依存関係をどう処理するか |
| 15 | Supervisorパターン | 動的なエージェントチームをどう管理するか |
| 16 | Handoffメカニズム | エージェント間でタスクと状態をどう受け渡すか |

## 学習目標

本Partを完了後、以下ができるようになります：
- Orchestrator編成アーキテクチャを設計する
- DAG (有向無環グラフ) ワークフローを実装する
- Supervisorパターンを使って動的なエージェントを管理する
- エージェント間のHandoffと状態の受け渡しを処理する

## Shannonコード案内

```
Shannon/
├── go/orchestrator/internal/workflows/
│   ├── orchestrator_router.go          # ルーティング判定
│   ├── dag_workflow.go                 # DAG実装
│   └── supervisor_workflow.go          # Supervisorパターン
└── docs/multi-agent-workflow-architecture.md
```

## コアアーキテクチャ

```
Orchestrator Router
    ├── SimpleTask (複雑度 < 0.3)
    ├── DAG (通常のマルチステップタスク)
    ├── React (ツール集約型)
    ├── Research (情報合成)
    └── Supervisor (> 5個のサブタスク)
```

## 前提知識

- Part 1-4の完了
- グラフ理論の基礎 (DAG、トポロジカルソート)
- 並行プログラミングの基礎
