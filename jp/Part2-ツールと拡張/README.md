# Part 2: ツールと拡張

> エージェントに真の実行能力を備わせよう：ツール呼び出し、MCPプロトコル、Skillsシステム

## 章一覧

| 章 | タイトル | コアクエスチョン |
|------|------|----------|
| 03 | ツール呼び出し基礎 | LLMが外部関数を呼び出すにはどうすればいい？ |
| 04 | MCPプロトコル詳解 | エージェントと外部システムの接続をどう標準化する？ |
| 05 | Skillsシステム | 再利用可能なエージェント能力をどう構築する？ |
| 06 | Hooksとプラグイン | エージェントのライフサイクルをどう拡張し、配布する？ |

## 学習目標

本Partを完了すると、以下ができるようになります：
- Function Callingツール定義を実装する
- MCP (Model Context Protocol) プロトコルアーキテクチャを理解する
- 再利用可能なSkillsシステムを設計する
- Hooksを使ってエージェント動作を拡張する

## Shannon コードガイド

```
Shannon/
├── python/llm-service/tools/           # ツール実装
├── python/llm-service/roles/presets.py # Skills プリセット
└── docs/pattern-usage-guide.md         # パターンガイド
```

## 関連ホットトピック

- **MCP**: Claude Code、Cursorなどのツール標準プロトコル
- **Hooks**: Claude Codeイベント駆動型拡張メカニズム
- **Plugins**: 能力パッケージングとコミュニティ共有

## 前提知識

- Part 1 完了
- JSON Schema基礎
- HTTP/gRPC基礎
