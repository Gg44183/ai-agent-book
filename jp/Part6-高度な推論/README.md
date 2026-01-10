# Part 6: 高度な推論

> 複雑な意思決定シナリオ：Tree-of-Thoughts、複数エージェント間のDebate、研究成果の統合

## 章立て

| 章 | タイトル | コアクエスチョン |
|------|----------|----------|
| 17 | Tree-of-Thoughts | 複数の推論経路をどう探索するか？ |
| 18 | Debate モデル | 議論を通じて意思決定の質をいかに高めるか？ |
| 19 | Research Synthesis | 複数の情報源をいかに統合してレポートを生成するか？ |

## 学習目標

本Partを完了すれば、以下のことができるようになる：
- ToT (Tree-of-Thoughts) の探索と枝刈りを実装する
- 複数エージェント間のDebate機構を設計する
- 研究統合ワークフローを構築する
- 推論モデルに応じた最適なパターンを選択できる

## Shannon コード解説

```
Shannon/
├── go/orchestrator/internal/workflows/
│   ├── patterns/tot.go                 # Tree-of-Thoughts
│   ├── patterns/debate.go              # Debate モデル
│   └── research_workflow.go            # Research 統合
└── docs/pattern-usage-guide.md
```

## パターン選択ガイド

| シナリオ | 推奨パターン | 理由 |
|---------|-----------|------|
| 開放的な問題 | ToT | 複数の可能性を探索する必要がある |
| 議論の余地がある決定 | Debate | 多角的な論証が必要 |
| 情報収集 | Research | 複数ソースの並列処理 + 統合 |

## 前提知識

- Part 1～5 完了
- 意思決定理論の基礎知識
- 情報検索の基礎
