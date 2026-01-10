# Part1: エージェント基礎

> LLMから自律型知能体へ - AIエージェントの本質を理解しよう

## 章一覧

| 章 | タイトル | コアクエスチョン |
|------|------|----------|
| 01 | エージェントとは何か | エージェント？普通のチャットボットとの違いは？ |
| 02 | ReAct循環 | エージェントはどう考えて、どう行動する？ |

## 学習目標

このPartを完了すると、次のことができるようになります：
- エージェントの定義と自律性のスペクトラムを理解する
- ReAct (Reason-Act-Observe) の基本循環をマスターする
- エージェントと従来のチャットボットの本質的な違いを区別する
- Shannonアーキテクチャの全体的な設計思想を理解する

## Shannonコード読解ガイド

```
Shannon/
├── docs/multi-agent-workflow-architecture.md  # アーキテクチャ概要
├── go/orchestrator/internal/workflows/strategies/react.go   # ReactWorkflow（ワークフロー層）
└── go/orchestrator/internal/workflows/patterns/react.go     # ReactLoop（パターン層）
```

## 前提知識

- LLM基本概念 (プロンプト、トークン、温度)
- 基本的なプログラミング能力 (Go/Python どちらか一方)
