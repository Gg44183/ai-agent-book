# Part 4: 単一エージェントパターン

> 単一エージェントの推論能力を深掘りする：Planning、Reflection、Chain-of-Thought

## 章一覧

| 章 | タイトル | 中心的な問い |
|------|----------|----------|
| 10 | Planningパターン | エージェントは複雑なタスクをどのように分解するか？ |
| 11 | Reflectionパターン | エージェントは自己評価と改善をどのように行うか？ |
| 12 | Chain-of-Thought | エージェントの推論プロセスをどのように表現するか？ |

## 学習目標

本Partを完了すれば、以下ができるようになるでしょう：
- タスク自動分解（Decomposition）の実装
- Reflection-改善ループの設計
- CoTの原理と最佳実践の理解
- 単一エージェントの能力限界の評価

## Shannonコード参照

```
Shannon/
├── go/orchestrator/internal/activities/
│   └── agent_activities.go             # /agent/decompose
├── go/orchestrator/internal/workflows/
│   └── patterns/                       # 推論パターンライブラリ
└── docs/pattern-usage-guide.md
```

## パターン比較

| パターン | 使用シーン | 複雑度 |
|------|----------|--------|
| Planning | マルチステップタスク | 中 |
| Reflection | 品質重視タスク | 中 |
| CoT | ロジック推論タスク | 低 |

## 前提知識

- Part 1～3 の完了
- Prompt Engineering の基礎
