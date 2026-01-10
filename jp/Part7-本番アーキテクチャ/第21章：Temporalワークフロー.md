# 第 21 章：Temporal ワークフロー

> **Temporal を使うと、ワークフローがデータベーストランザクションみたいに信頼できるようになるよ。実行した場所まで保存されて、クラッシュしても途中から復旧できる。自分で状態マシンを書く必要がないんだ。**
> **でも魔法じゃないからね。Activity と Workflow の違いを理解して、バージョンゲーティングをいつ使うべきか分かってないと、ハマるよ。**

---

> **5 分間クイックパス**
>
> 1. Workflow = 確定的ロジック（リプレイ可能）、Activity = 副作用のある操作（リプレイ不可）
> 2. 断点継続：クラッシュ後に Workflow が履歴イベントをリプレイし、Activity の結果はイベントログから復元
> 3. 確定性の要件：time.Now()、rand、goroutine は禁止。workflow.* API だけを使う
> 4. バージョンゲーティング：workflow.GetVersion() で新旧コードを共存させて、スムーズにアップグレード
> 5. タイムアウト設定：StartToClose（単発）+ ScheduleToClose（合計）+ HeartbeatTimeout
>
> **10 分間パス**：21.1-21.3 -> 21.5 -> Shannon Lab

---

深夜 3 時、エージェント（Agent）が深い調査タスクを実行中だったとする。検索 API を 15 回呼び出して、2 分かかった。

そしたらサーバーが OOM でクラッシュした。

翌朝、ユーザーが聞いてくる：「私の調査レポートは？」

ログを調べると、タスクは完全に消えてた。あの 15 回の検索、全部無駄になったわけ。

これが Temporal が必要な理由だよ。

---

## 21.1 なぜワークフローエンジンが必要なのか

### 従来の方法の問題点

ワークフローエンジンがない時代、長時間実行するタスクをどう処理してた？

3 つの従来の方法とその問題点を比較してみよう：

```python
# ========== 方法 1：同期実行 ==========
def deep_research(query):
    for topic in decompose(query):
        result = search(topic)        # クラッシュ時、完了分が全部消える
        results.append(result)
    return synthesize(results)
# 問題：8 番目のサブタスクでプロセスがクラッシュしたら、前の 7 つが全部消える

# ========== 方法 2：データベース状態マシン ==========
def deep_research_with_state(query, task_id):
    state = db.get(task_id) or {"completed": [], "results": []}
    for i, topic in enumerate(decompose(query)):
        if i in state["completed"]: continue
        state["results"].append(search(topic))
        state["completed"].append(i)
        db.update(state)              # 毎ステップ保存するけど、コードが複雑に
    return synthesize(state["results"])
# 問題：タスクごとに状態マシンを書く必要あり、シリアライズでミスりやすい、並行処理が難しい、統一監視がない

# ========== 方法 3：メッセージキュー + Worker ==========
def start_research(query):           # プロデューサー
    for topic in decompose(query): queue.push({"topic": topic})
def worker():                        # コンシューマー
    while True:
        msg = queue.pop()
        db.save_result(msg["task_id"], search(msg["topic"]))
# 問題：結果の集約に追加ロジックが必要、リトライ戦略がバラバラ、依存関係を表現しにくい、デバッグ時にグローバル状態が見えない
```

### Temporal はどう解決するのか

Temporal のコアとなる考え方：**コードをデータとして永続化する**。

状態のスナップショットを保存するんじゃなくて、すべての決定ポイントを記録するんだ。実行した場所まで記録される。クラッシュしたら、これらの記録をリプレイして、自動的に以前の位置に復旧する。

```go
// 君が書くコード
func DeepResearchWorkflow(ctx workflow.Context, query string) (string, error) {
    topics := decompose(query)
    var results []string
    for _, topic := range topics {
        // Temporal がこの呼び出しの結果を自動で永続化
        var result string
        workflow.ExecuteActivity(ctx, SearchActivity, topic).Get(ctx, &result)
        results = append(results, result)
    }
    return synthesize(results), nil
}
```

見た目は普通のコードだよね。でも Temporal は裏で：
1. 各 `ExecuteActivity` の前にチェックポイントを記録
2. Activity の結果をデータベースに永続化
3. クラッシュ後のリプレイ時、完了した Activity は直接キャッシュ結果を返す
4. 最後のチェックポイントから実行を継続

---

## 21.2 コアコンセプト

### Workflow vs Activity

これが Temporal で最も重要な区別だよ：

| 概念 | Workflow | Activity |
|------|----------|----------|
| **定義** | オーケストレーションロジック、何をするか決める | 具体的な作業、実際に実行する |
| **確定性が必須** | はい | いいえ |
| **副作用を持てる** | いいえ | はい |
| **自動リトライ** | いいえ（リプレイ） | はい |
| **タイムアウト処理** | 全体タイムアウト | 単発タイムアウト+リトライ |

**重要なルール**：Workflow のコードは**確定的**でなければならない。

同じ入力に対して、同じ決定シーケンスを生成しないといけない。これはリプレイ時に正しい状態に復旧するためだよ。

どの操作が確定性を壊すのか？正しい方法と間違った方法を比較しよう：

```go
// ========== 確定性を壊す（間違い）==========    // ========== 正しい方法 ==========
time.Now()                                       // workflow.Now(ctx)
rand.Int()                                       // workflow.SideEffect(ctx, func() { return rand.Int() })
http.Get("...")                                  // workflow.ExecuteActivity(ctx, FetchActivity, ...)
os.Getenv("...")                                 // Workflow のパラメータとして渡す
uuid.New()                                       // workflow.SideEffect(ctx, func() { return uuid.New() })
```

### Activity のリトライ戦略

Activity にはリトライ戦略を設定できる。さまざまなシナリオの設定方法を見てみよう：

```go
// ========== 汎用設定（全パラメータの説明）==========
activityOptions := workflow.ActivityOptions{
    StartToCloseTimeout: 30 * time.Second,        // 単発実行タイムアウト
    RetryPolicy: &temporal.RetryPolicy{
        InitialInterval:    1 * time.Second,      // 初回リトライ間隔
        BackoffCoefficient: 2.0,                  // 指数バックオフ係数
        MaximumInterval:    30 * time.Second,     // 最大リトライ間隔
        MaximumAttempts:    3,                    // 最大リトライ回数
        NonRetryableErrorTypes: []string{         // リトライしないエラー
            "InvalidInputError", "PermissionDeniedError",
        },
    },
}

// ========== シナリオ別設定（Shannon の実践）==========
// 長時間 Agent 実行：長めのタイムアウト + ハートビート検出
agentOpts := workflow.ActivityOptions{
    StartToCloseTimeout: 120 * time.Second,
    HeartbeatTimeout:    30 * time.Second,        // ハートビートで生存確認
    RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 3},
}
// 高速操作：短いタイムアウト、少ないリトライ
quickOpts := workflow.ActivityOptions{
    StartToCloseTimeout: 10 * time.Second,
    RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 2},
}
ctx = workflow.WithActivityOptions(ctx, activityOptions)
```

---

## 21.3 バージョンゲーティング (Version Gating)

ここが Temporal で最もハマりやすいところで、Shannon のコードで最もよく使われているパターンの一つでもあるよ。

### 問題と解決策

```go
// ========== 問題：コードを直接変更するとリプレイが失敗する ==========
// v1 オリジナルバージョン
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, ActivityA, ...)
    workflow.ExecuteActivity(ctx, ActivityB, ...)
    return nil
}
// v2 で ActivityNew を直接追加 -> 実行中の v1 ワークフローがリプレイ時に Non-determinism エラー

// ========== 解決策：workflow.GetVersion を使う ==========
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, ActivityA, ...)
    // バージョンゲーティング：新しいワークフローは 1 を返し、古いワークフローのリプレイは DefaultVersion (-1) を返す
    if workflow.GetVersion(ctx, "add_activity_new", workflow.DefaultVersion, 1) >= 1 {
        workflow.ExecuteActivity(ctx, ActivityNew, ...)
    }
    workflow.ExecuteActivity(ctx, ActivityB, ...)
    return nil
}
```

### Shannon での実際の使用例と命名規則

Shannon のコードではバージョンゲーティングを多用している（`strategies/research.go` を参照）：

```go
// ========== 機能進化の例：階層メモリ vs セッションメモリ ==========
hierarchicalVersion := workflow.GetVersion(ctx, "memory_retrieval_v1", workflow.DefaultVersion, 1)
if hierarchicalVersion >= 1 && input.SessionID != "" {
    workflow.ExecuteActivity(ctx, activities.RetrieveHierarchicalMemoryActivity, ...).Get(ctx, &memoryResult)
} else if workflow.GetVersion(ctx, "session_memory_v1", workflow.DefaultVersion, 1) >= 1 {
    workflow.ExecuteActivity(ctx, activities.GetSessionMessagesActivity, ...).Get(ctx, &messages)
}

// ========== 条件付き有効化の例：コンテキスト圧縮 ==========
if workflow.GetVersion(ctx, "context_compress_v1", workflow.DefaultVersion, 1) >= 1 &&
   input.SessionID != "" && len(input.History) > 20 {
    // 新バージョンでコンテキスト圧縮を有効化
}

// ========== バージョン命名規則 ==========
// 良い命名（機能名 + バージョン番号）      // 悪い命名
workflow.GetVersion(ctx, "memory_retrieval_v1", ...)    // "fix_bug_123"（曖昧すぎる）
workflow.GetVersion(ctx, "context_compress_v1", ...)    // "v2"（機能の説明がない）
workflow.GetVersion(ctx, "iterative_research_v1", ...)
```

---

## 21.4 シグナルとクエリ

Workflow の実行中に、やり取りが必要になることがあるよね：
- **シグナル (Signal)**：Workflow にメッセージを送って、動作を変更させる
- **クエリ (Query)**：Workflow の現在の状態を取得する。実行は変えない

### シグナルの例：一時停止/再開

Shannon ではシグナルを使って一時停止/再開を実装している：

```go
// control/handler.go
type SignalHandler struct {
    paused        bool
    pauseCh       workflow.Channel
    resumeCh      workflow.Channel
    cancelCh      workflow.Channel
}

func (h *SignalHandler) Setup(ctx workflow.Context) {
    version := workflow.GetVersion(ctx, "pause_resume_v1", workflow.DefaultVersion, 1)
    if version < 1 {
        return  // 旧バージョンはサポートしない
    }

    // シグナルチャネルを登録
    pauseSig := workflow.GetSignalChannel(ctx, "pause")
    resumeSig := workflow.GetSignalChannel(ctx, "resume")
    cancelSig := workflow.GetSignalChannel(ctx, "cancel")

    // クエリハンドラを登録
    workflow.SetQueryHandler(ctx, "get_status", func() (string, error) {
        if h.paused {
            return "paused", nil
        }
        return "running", nil
    })

    // バックグラウンドの goroutine でシグナルを処理
    workflow.Go(ctx, func(ctx workflow.Context) {
        for {
            selector := workflow.NewSelector(ctx)
            selector.AddReceive(pauseSig, func(ch workflow.ReceiveChannel, more bool) {
                h.paused = true
            })
            selector.AddReceive(resumeSig, func(ch workflow.ReceiveChannel, more bool) {
                h.paused = false
            })
            selector.Select(ctx)
        }
    })
}

// ワークフロー内で一時停止状態をチェック
func (h *SignalHandler) WaitIfPaused(ctx workflow.Context) {
    for h.paused {
        workflow.Sleep(ctx, 1*time.Second)
    }
}
```

### シグナルの送信

```go
// 外部からシグナルを送信
client.SignalWorkflow(ctx, workflowID, runID, "pause", nil)

// HTTP API 経由で呼び出し
// POST /api/v1/workflows/{workflowID}/signal
// { "signal_name": "pause" }
```

---

## 21.5 Worker の起動と優先度キュー

### 起動フロー

Shannon の Worker 起動には完全なリトライ機構がある：

```go
// TCP 事前チェック（サービスが到達可能か素早く判定）
for i := 1; i <= 60; i++ {
    c, err := net.DialTimeout("tcp", host, 2*time.Second)
    if err == nil {
        _ = c.Close()
        break
    }
    logger.Warn("Waiting for Temporal TCP endpoint",
        zap.String("host", host), zap.Int("attempt", i))
    time.Sleep(1 * time.Second)
}

// SDK 接続リトライ（より重い操作なので、指数バックオフを使用）
var tClient client.Client
var err error
for attempt := 1; ; attempt++ {
    tClient, err = client.Dial(client.Options{
        HostPort: host,
        Logger:   temporal.NewZapAdapter(logger),
    })
    if err == nil {
        break
    }
    delay := time.Duration(min(attempt, 15)) * time.Second
    logger.Warn("Temporal not ready, retrying",
        zap.Int("attempt", attempt),
        zap.Duration("sleep", delay),
        zap.Error(err))
    time.Sleep(delay)
}
```

### 優先度キュー

Shannon はマルチキューモードをサポートしていて、優先度の異なるタスクは異なるキューを使う：

```go
if priorityQueues {
    _ = startWorker("shannon-tasks-critical", 12, 12)
    _ = startWorker("shannon-tasks-high", 10, 10)
    w = startWorker("shannon-tasks", 8, 8)
    _ = startWorker("shannon-tasks-low", 4, 4)
} else {
    w = startWorker("shannon-tasks", 10, 10)
}
```

優先度キューの典型的な用途：

| キュー | 並行数 | 用途 |
|------|--------|------|
| critical | 12 | ユーザーが待っているリアルタイムリクエスト |
| high | 10 | 重要だけど少し待てるタスク |
| normal | 8 | 通常のバックグラウンドタスク |
| low | 4 | レポート生成、データクリーンアップなど |

---

## 21.6 Fire-and-Forget パターン

メインフローに影響しない操作（ログ記録、メトリクス報告など）には、Fire-and-Forget を使える：

```go
// Agent 実行結果を永続化（fire-and-forget）
func persistAgentExecution(ctx workflow.Context, workflowID, agentID, input string,
                           result activities.AgentExecutionResult) {
    // 短いタイムアウト + リトライなし
    persistCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: 5 * time.Second,
        RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 1},
    })

    // 結果を待たない
    workflow.ExecuteActivity(
        persistCtx,
        activities.PersistAgentExecutionStandalone,
        activities.PersistAgentExecutionInput{
            WorkflowID: workflowID,
            AgentID:    agentID,
            Input:      input,
            Output:     result.Response,
            TokensUsed: result.TokensUsed,
        },
    )
    // 注意：.Get() を呼び出してない、完了を待たない
}
```

用途：
- ログ記録
- メトリクス報告
- 監査証跡
- キャッシュの事前ウォームアップ

---

## 21.7 並列実行パターン

3 つの並列実行パターンを見てみよう：

```go
// ========== パターン 1：基本的な並列（全部完了を待つ）==========
futures := make([]workflow.Future, len(subtasks))
for i, subtask := range subtasks {
    futures[i] = workflow.ExecuteActivity(ctx, activities.ExecuteAgent, subtask.Query)
}
for i, f := range futures {
    f.Get(ctx, &results[i])  // 順番に待つ
}

// ========== パターン 2：セレクタ（先に完了したものから処理）==========
futures := make(map[string]workflow.Future)
for _, topic := range topics {
    futures[topic] = workflow.ExecuteActivity(ctx, SearchActivity, topic)
}
for len(futures) > 0 {
    selector := workflow.NewSelector(ctx)
    for topic, f := range futures {
        t := topic  // クロージャでキャプチャ
        selector.AddFuture(f, func(f workflow.Future) {
            f.Get(ctx, &result)
            processResult(t, result)
            delete(futures, t)
        })
    }
    selector.Select(ctx)
}

// ========== パターン 3：タイムアウト制御 ==========
ctx, cancel := workflow.WithCancel(ctx)
defer cancel()
selector := workflow.NewSelector(ctx)
selector.AddFuture(workflow.ExecuteActivity(ctx, LongTask, input), func(f workflow.Future) {
    err = f.Get(ctx, &result)
})
selector.AddFuture(workflow.NewTimer(ctx, 5*time.Minute), func(f workflow.Future) {
    cancel()  // タイムアウトでキャンセル
    err = errors.New("timeout")
})
selector.Select(ctx)
```

---

## 21.8 子ワークフロー

複雑なタスクは子ワークフローに分解できる：

```go
func ParentWorkflow(ctx workflow.Context, topics []string) ([]string, error) {
    var results []string

    // 子ワークフローを並列で起動
    var futures []workflow.Future
    for _, topic := range topics {
        childOpts := workflow.ChildWorkflowOptions{
            WorkflowID: fmt.Sprintf("research-%s", topic),
        }
        childCtx := workflow.WithChildOptions(ctx, childOpts)
        future := workflow.ExecuteChildWorkflow(childCtx, ResearchChildWorkflow, topic)
        futures = append(futures, future)
    }

    // 全部完了を待つ
    for _, future := range futures {
        var result string
        if err := future.Get(ctx, &result); err != nil {
            return nil, err
        }
        results = append(results, result)
    }

    return results, nil
}
```

子ワークフローのメリット：
- **障害の分離**：一つの子ワークフローが失敗しても他に影響しない
- **独立したリトライ**：個別にリトライ戦略を設定できる
- **可視化**：Temporal UI で階層関係がクリアに表示される
- **並列実行**：複数の子ワークフローを並行して実行できる

---

## 21.9 タイムトラベルデバッグ

### Temporal Web UI

Temporal には Web UI が付属していて、以下が見える：
- ワークフロー一覧と状態
- 各ワークフローのイベント履歴
- Activity 実行の詳細
- リトライ回数とエラー情報

`http://localhost:8088` にアクセスして確認できる。

### デバッグ手順

```
1. Temporal Web UI を開く
2. 問題のワークフローを見つける
3. Event History を確認
4. 失敗した Activity を特定
5. 入力パラメータとエラーをチェック
6. 同じ入力でローカル再現
```

### エクスポートとリプレイ

```bash
# 実行履歴をエクスポート
temporal workflow show --workflow-id "task-123" --output json > history.json

# ローカルでリプレイテスト
temporal workflow replay --workflow-id "task-123"
```

---

## 21.10 よくあるハマりポイント

| ハマりポイント | 問題 | 解決策 |
|----|------|----------|
| 外部サービスを直接呼び出す | `http.Get()` が確定性を壊す | `workflow.ExecuteActivity()` を使う |
| バージョンゲーティングを忘れる | Activity を追加すると旧ワークフローのリプレイが失敗 | `workflow.GetVersion()` でラップする |
| Activity が大きなデータを返す | 数 MB のデータがパフォーマンスに影響 | データ本体ではなくパス/参照を返す |
| 無限ループ | イベント履歴が膨張する | `Continue-As-New` でワークフローを再起動 |
| キャンセルを無視 | リソースリーク、グレースフルに終了できない | ループ内で `ctx.Err()` をチェック |

```go
// ========== ハマりポイント 1：外部サービスを直接呼び出す ==========
http.Get("...")                                       // 間違い：確定性を壊す
workflow.ExecuteActivity(ctx, FetchActivity, ...).Get(ctx, &data)  // 正しい

// ========== ハマりポイント 2：バージョンゲーティングを忘れる ==========
workflow.ExecuteActivity(ctx, NewActivity, ...)       // 間違い：旧ワークフローのリプレイが失敗
if workflow.GetVersion(ctx, "add_new", workflow.DefaultVersion, 1) >= 1 {
    workflow.ExecuteActivity(ctx, NewActivity, ...)   // 正しい
}

// ========== ハマりポイント 3：Activity が大きなデータを返す ==========
return downloadLargeFile()                            // 間違い：10MB のデータ
path := saveLargeFile(); return path, nil             // 正しい：パスだけを返す

// ========== ハマりポイント 4：無限ループ ==========
for { workflow.ExecuteActivity(ctx, Poll, ...) }      // 間違い：イベント履歴が膨張
if iteration > 1000 { return workflow.NewContinueAsNewError(ctx, Workflow, 0) }  // 正しい

// ========== ハマりポイント 5：キャンセルを無視 ==========
for { doWork() }                                      // 間違い：グレースフルに終了できない
for { if ctx.Err() != nil { return ctx.Err() }; doWork() }  // 正しい
```

---

## この章で学んだこと

1. **Workflow vs Activity**：Workflow はオーケストレーションの決定（確定的である必要あり）、Activity は実際の実行（副作用を持てる）
2. **バージョンゲーティング**：コード変更時は `workflow.GetVersion` で互換性を保証
3. **シグナルとクエリ**：シグナルで動作を変更、クエリで状態を取得
4. **並列実行**：Future で起動、セレクタで処理
5. **Fire-and-Forget**：重要でない永続化はメインフローをブロックしない

---

## Shannon Lab（10 分間ハンズオン）

このセクションでは、この章のコンセプトを Shannon のソースコードに対応させてみよう。

### 必読（1 ファイル）

- `go/orchestrator/internal/workflows/strategies/research.go`："GetVersion" で検索して `workflow.GetVersion` の使い方を見てみよう。実際のバージョンゲーティングパターンを理解できる

### 選読（2 つ、興味に応じて選んで）

- `go/orchestrator/internal/workflows/control/handler.go`：シグナルハンドラの実装、一時停止/再開の仕組みを理解
- `go/orchestrator/internal/activities/agent.go`：Activity が LLM 呼び出しをどうラップしているか確認

---

## 演習

### 演習 1：バージョン移行を設計する

元のワークフローがこうなってたとする：

```go
func MyWorkflow(ctx workflow.Context) error {
    workflow.ExecuteActivity(ctx, StepA)
    workflow.ExecuteActivity(ctx, StepB)
    return nil
}
```

今、以下が必要：
1. A と B の間に StepC を追加
2. StepB を StepB2 にリネーム（パラメータも変わった）

旧ワークフローと互換性のある新コードを書いてみよう。

### 演習 2：並列 + タイムアウト

以下を満たすワークフローを設計してみて：
- 5 つの検索タスクを並列で起動
- 全体タイムアウトは 2 分
- 任意の 3 つが完了したら次のフェーズへ
- タイムアウトや失敗したタスクは全体をブロックしない

重要なコード断片を書いてみよう。

### 演習 3（上級）：トークン予算ミドルウェア

以下を満たすトークン予算ミドルウェアを設計してみて：
- 各 Activity 呼び出し前に残り予算をチェック
- Activity 完了後に実際の消費を差し引く
- 予算が尽きたら特定のエラーを返す
- 擬似コードを書いてみよう

---

## 参考資料

- [Temporal 公式ドキュメント](https://docs.temporal.io/)：コンセプトの詳細解説とベストプラクティス
- [Temporal Workflow Versioning](https://docs.temporal.io/develop/go/versioning)：バージョンゲーティングの詳細ガイド
- [Temporal in Production](https://docs.temporal.io/production-deployment)：本番デプロイの設定

---

## 次章の予告

Temporal で「クラッシュ復旧」の問題は解決した。でもまだ一つ問題がある：**システムは動いてるけど、何をやってるか分からない。**

ユーザーが「タスクが遅い」と言ってきた。どこが遅いの？LLM 呼び出し？検索？データベース？

次章では**オブザーバビリティ**について話すよ。メトリクス、ログ、トレースの三種の神器を使って、エージェントシステムをガラスのように透明にする方法を学ぼう。

準備できた？進もう。
