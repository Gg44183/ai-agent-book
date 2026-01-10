# Part 9: 最先端実践

> 最新のホットトピック：Computer Use、Agentic Coding、Background Agents、階層型モデル戦略

## 章一覧

| 章 | タイトル | コア課題 | Shannonとの関連 |
|------|------|----------|-------------|
| 27 | Computer Use | どうやってエージェントがブラウザやデスクトップを操作するか？ | `config/models.yaml` multimodal |
| 28 | Agentic Coding | どうやってコード生成エージェントを構築するか？ | `file_ops.py`, `wasi_sandbox.rs` |
| 29 | Background Agents | 非同期な長時間タスクをどう実装するか？ | `schedules/manager.go` |
| 30 | 階層型モデル戦略 | どうやって50～70%のコスト削減を実現するか？ | `config/models.yaml`, `manager.py` |

---

## 章節の概要

### 第27章：Computer Use

> エージェントが「目」と「手」を手に入れるとき：APIの呼び出しから実際のUIの操作へ

**コアコンテンツ**:
- **知覚-判断-実行ループ**：スクリーンショット理解 → 座標計算 → クリック/入力 → 結果検証
- **マルチモーダルモデル統合**：ビジョン理解がComputer Useの重要な能力
- **座標キャリブレーション**：異なる解像度とDPIスケーリングの差異を処理
- **セキュリティ対策**：危険エリア検出、入力コンテンツフィルタリング、OPAポリシー拡張
- **検証ループ**：毎回操作後にスクリーンショットで結果を検証し、失敗時に自動リトライ

**Shannonコード**：`config/models.yaml` (multimodal_models)、ツール拡張パターンを推奨

---

### 第28章：Agentic Coding

> エージェントをあなたのプログラミングパートナーに：コード生成から完全な開発ワークフローまで

**コアコンテンツ**:
- **安全なファイル操作**：ホワイトリストディレクトリ、パス検証、シンボリックリンク対策
- **WASI サンドボックス実行**：Fuel/Epoch制限、メモリ分離、タイムアウト制御
- **コード反思ループ**：生成 → レビュー → 改善の反復プロセス
- **マルチファイル編集調整**：アトミック変更、バックアップロールバック機構
- **Git統合**：ブランチ管理、自動コミット、PR作成

**Shannonコード**：`python/llm-service/llm_service/tools/builtin/file_ops.py`、`rust/agent-core/src/wasi_sandbox.rs`、`go/orchestrator/internal/workflows/patterns/reflection.go`

---

### 第29章：Background Agents

> タスクをバックグラウンドで実行し続ける：Temporalスケジューリングと定期実行タスクシステム

**コアコンテンツ**:
- **Temporal Schedule API**：ネイティブCronスケジューリング、一時停止/再開、タイムゾーン対応
- **リソース制限**：MaxPerUser (50)、MinCronInterval (60分)、MaxBudgetPerRunUSD ($10)
- **ScheduledTaskWorkflow**：ラッパーワークフロー、実行メタデータ記録（モデル、トークン、コスト）
- **孤立検出**：Temporalとデータベース状態の不整合を定期的に検出し、自動クリーンアップ
- **予算インジェクション**：各実行のコスト追跡と制限

**Shannonコード**：`schedules/manager.go`、`scheduled_task_workflow.go`

---

### 第30章：階層型モデル戦略

> インテリジェントルーティングで50～70%のコスト削減を実現：すべてのタスクが最強モデルを必要とするわけではないよね

**コアコンテンツ**:
- **3層アーキテクチャ**：Small (50%) / Medium (40%) / Large (10%) の目標分布
- **優先度ベースルーティング**：同じレベル内で複数Providerが優先度で選択され、自動フォールバック
- **複雑度分析**：タスク特性に基づいてモデル層を自動選択
- **能力マトリックス**：multimodal、thinking、coding、long_contextの能力マーク
- **サーキットブレーカー降級**：Circuit Breaker + 代替Providerへの自動降級
- **コスト追跡**：集中型の価格設定、リアルタイムコスト監視

**Shannonコード**：`config/models.yaml`、`llm_provider/manager.py`

---

## 学習目標

このPartを完了すれば、以下ができるようになる：

- [ ] Computer Useの知覚-判断-実行ループを理解する
- [ ] 安全なAgentic Codingワークフロー（サンドボックス + 反思）を設計する
- [ ] Temporal Schedule APIを使って定期バックグラウンドタスクを実装する
- [ ] 3層モデル戦略を設定して50～70%のコスト削減を実現する
- [ ] Research エージェントに最先端能力を追加する (v0.9)

---

## Shannonコード一覧

```
Shannon/
├── config/
│   └── models.yaml                    # 3層モデル設定、価格設定、能力マトリックス
├── go/orchestrator/
│   └── internal/
│       ├── schedules/
│       │   └── manager.go             # 定期実行タスクマネージャー (CRUD、リソース制限)
│       └── workflows/scheduled/
│           └── scheduled_task_workflow.go  # ラッパーワークフロー
├── python/llm-service/
│   ├── llm_provider/
│   │   └── manager.py                 # LLMマネージャー (ルーティング、サーキットブレーカー、フォールバック)
│   └── llm_service/tools/builtin/
│       ├── file_ops.py                # 安全なファイル読み書きツール
│       └── python_wasi_executor.py    # Pythonサンドボックス実行
└── rust/agent-core/src/sandbox/
    └── wasi_sandbox.rs                # WASIサンドボックス実装
```

---

## ホットトピック関連

| トピック | 代表的なプロダクト | Shannon実装 | 章 |
|------|----------|-------------|------|
| Computer Use | Claude Computer Use、Manus | マルチモーダル + ツール拡張 | Ch27 |
| Agentic Coding | Claude Code、Cursor、Windsurf | WASIサンドボックス + ファイルツール | Ch28 |
| Background Agents | Claude Code Ctrl+B | Temporal Schedule API | Ch29 |
| Cost Optimization | エンタープライズ向け低コスト化需要 | 階層型モデル戦略 | Ch30 |

---

## コスト削減の例

```
階層化なし (全部Large使用):
  1M リクエスト × $0.09/リクエスト = $90,000/月

階層化戦略 (50/40/10):
  Small:  500K × $0.0006  = $300
  Medium: 400K × $0.0018  = $720
  Large:  100K × $0.09    = $9,000
  合計: $10,020/月

節減: $79,980/月 (89%)
```

---

## 前提知識

- Part 1～8を完了（特にPart 7～8の本番アーキテクチャとエンタープライズ機能）
- ブラウザ自動化の基本 (Playwright/Puppeteer) - オプション
- Cron表現の基本 - オプション
- マルチモデルAPI経験 - オプション

---

## Research エージェント v0.9

このPartで扱う最先端能力モジュール：

| モジュール | 章 | 能力 |
|------|------|------|
| Computer Use | 第27章 | ウェブブラウジング、コンテンツ抽出 |
| Agentic Coding | 第28章 | 分析スクリプト生成 |
| Background Agents | 第29章 | 定期研究レポート |
| Tiered Models | 第30章 | インテリジェントモデル選択 |

**最終形態**:
```
ユーザー：「毎日朝9時にAI業界の日報を生成してほしい」

Research エージェント v0.9:
1. [Schedule] Cronスケジューリングタスク作成 (0 9 * * *)
2. [Tiered] Smallモデルで複雑度評価を実施
3. [Multi-Agent] 検索/分析/執筆を並列実行
4. [Browser] API未対応サイトにアクセスしてコンテンツ抽出
5. [Coding] データビジュアライゼーションスクリプト生成
6. [Budget] 各実行のコストを < $2 に制御
7. [Output] 構造化されたレポートを発送
```
