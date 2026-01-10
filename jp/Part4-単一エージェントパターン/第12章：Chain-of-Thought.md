# 第12章：Chain-of-Thought

> **CoT は LLM を賢くするものじゃない。暗黙の推論を明示的なステップに変えるんだ。そうすれば、どこで間違えたかが見えるし、軌道修正もしやすくなる。**

---

## 12.1 なぜ段階的に推論する必要があるのか？

まず実例から見てみよう。

2024年の初め、ある教育テック企業で数学指導エージェント（Agent）の開発を手伝っていた。機能はシンプルで、生徒が数学の問題を入力すると、エージェントが解答とステップを返す。

テスト段階では問題なかった。簡単な四則演算や一次方程式なら、正答率はかなり高い。

ところがリリースして1週間、クレームが来た。ある中学生がこう聞いたんだ。「太郎くんはりんごを15個持っています。3人の友達に分けると、1人何個？その後8個買いました。今何個ありますか？」

エージェントの回答：「11個です。」

間違い。正解は8個だ（15 / 3 = 5、5 - 5 = 0、0 + 8 = 8）。でもエージェントは「15」と「3」を見て、15 - 3 - 1 = 11 みたいな計算をして、問題の意味を全然理解していなかった。

**これで LLM の根本的な問題が見えた。「飛ばし」をやるんだよ。実際に推論せず、いきなり答えを出す。** 複数ステップの問題だと、この飛ばしで簡単に間違える。

1週間後、プロンプトに一言追加した。「Please solve step by step」と。同じ問題で、エージェントはこう答えるようになった：

```
ステップ1：太郎くんはりんごを15個持っている。3人の友達に分ける
ステップ2：1人あたり 15 / 3 = 5 個
ステップ3：分け終わった後、太郎くんの残り 15 - 15 = 0 個
ステップ4：さらに8個買ったので、今は 0 + 8 = 8 個
答え：8個
```

正答率が一気に40%上がった。

これが Chain-of-Thought（思考の連鎖）の価値だ。LLM に推論プロセスを外に出させて、一歩ずつ計算させる。「直感」で答えを当てさせるんじゃなくてね。

### LLM のデフォルト動作

LLM のデフォルト動作は「一気に全部言う」だ。質問すると、一段落まるごと生成する。「さっき言ったこと合ってるかな」と立ち止まることもないし、生成中に計算することもない。

これが問題を引き起こす。**複雑な推論で間違えやすいんだ。**

| タスクの種類 | LLM のデフォルト動作 | 問題点 |
|-------------|---------------------|--------|
| 複数ステップの計算 | いきなり答えを出す | 飛ばし、計算ミス |
| 論理推論 | 直感で当てる | 論理の連鎖が途切れる |
| 因果分析 | 表面的な関連付け | 因果関係の逆転 |
| コードデバッグ | よくある原因を列挙 | 実際に分析していない |

### CoT の解決策

Chain-of-Thought（思考の連鎖）の核心はシンプルだ。**LLM に一歩ずつ考えさせて、途中経過を書き出させる。**

同じ問題を CoT で解いた例を見てみよう：

```
一歩ずつ計算していこう：
-> ステップ1: 太郎くんは最初5個のりんごを持っている
-> ステップ2: 花子さんに2個あげた後、残り 5 - 2 = 3 個
-> ステップ3: 次郎くんから3個もらった後、合計 3 + 3 = 6 個
-> ステップ4: 1個食べた後、残り 6 - 1 = 5 個

したがって、太郎くんは今5個のりんごを持っている。
```

今度は正解。キーポイントは：**明示的な推論プロセスが、モデルに実際の計算を強制する。直感で当てさせるんじゃなくて。**

### CoT の価値

| 観点 | CoT なし | CoT あり |
|------|----------|----------|
| **正確性** | パターンマッチング頼み、複雑な推論でミスしやすい | 段階的に検証、飛躍的なミスを減らす |
| **説明可能性** | ブラックボックス出力、監査不可 | 透明なプロセス、各ステップを追跡可能 |
| **デバッグ能力** | 間違っても、どこが間違いか分からない | どのステップでミスしたか特定できる |

ただ、一つ注意しておきたい。CoT は万能じゃない。正答率を上げられるけど、正しさを保証はできない。段階的推論の各ステップでも、まだ間違える可能性はある。

---

## 12.2 CoT の学術的背景

CoT は2022年の論文から来ている：[Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)（Wei et al.）

核心的な発見：

> **複数ステップの推論が必要なタスクでは、プロンプトに「let's think step by step」と入れるか、推論例を提供すると、LLM の正答率が大幅に上がる。**

その後、もっとシンプルな方法も発見された。Zero-shot CoT だ。質問の後に「Let's think step by step」と一言加えるだけで、LLM の段階的推論を引き出せる。

この発見は興味深い。**LLM には実は段階的推論の能力があって、ただ「リマインド」が必要なだけなんだ。**

---

## 12.3 CoT プロンプト設計

### 基本テンプレート

最もシンプルな CoT プロンプト：

```
問題：今日が水曜日なら、10日後は何曜日？

段階的に考えてください。各推論ステップには -> を付けてください。
最後に結論を出し、「したがって：」で始めてください。
```

### Shannon のデフォルトテンプレート

Shannon の CoT 実装は `patterns/chain_of_thought.go` にある。デフォルトテンプレートはこんな感じだ：

```go
func buildChainOfThoughtPrompt(query string, config ChainOfThoughtConfig) string {
    if config.PromptTemplate != "" {
        return strings.ReplaceAll(config.PromptTemplate, "{query}", query)
    }

    // デフォルト CoT テンプレート
    return fmt.Sprintf(`Please solve this step-by-step:

Question: %s

Think through this systematically:
1. First, identify what is being asked
2. Break down the problem into steps
3. Work through each step with clear reasoning
4. Show your work and explain your thinking
5. Arrive at the final answer

Use "→" to mark each reasoning step.
End with "Therefore:" followed by your final answer.`, query)
}
```

**実装参照 (Shannon)**: [`patterns/chain_of_thought.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/chain_of_thought.go) - buildChainOfThoughtPrompt 関数

### ドメイン別カスタムテンプレート

シーンによって異なる CoT テンプレートが必要だ：

**数学専用**：
```
数学の問題: {query}

以下の形式で解答してください：

【分析】まず問題の要求を理解する
【公式】使う公式を列挙する
【計算】
  -> ステップ1: ...
  -> ステップ2: ...
【検証】計算結果が妥当か確認する
【答え】最終結果は...
```

**コードデバッグ専用**：
```
デバッグの問題: {query}

体系的に分析してください：

1. 【症状の説明】観察されたエラー現象
2. 【仮説リスト】考えられる原因（可能性順）
   -> 仮説A: ...
   -> 仮説B: ...
3. 【検証プロセス】各仮説を順に検証
4. 【根本原因分析】真の原因を特定
5. 【修正案】解決方法を提示

Therefore: 根本原因は... 修正方法は...
```

**論理推論専用**：
```
推論の問題: {query}

論理の連鎖で推論してください：

【既知の条件】
  - 条件1: ...
  - 条件2: ...

【推論プロセス】
  -> 条件1から、...が導かれる
  -> 条件2と合わせると、さらに...が導かれる
  -> したがって...

【結論】...
```

---

## 12.4 Shannon の CoT 実装

### 設定構造

```go
type ChainOfThoughtConfig struct {
    MaxSteps              int    // 最大推論ステップ数
    RequireExplanation    bool   // 説明を必須にするか
    ShowIntermediateSteps bool   // 出力に中間ステップを含めるか
    PromptTemplate        string // カスタムテンプレート
    StepDelimiter         string // ステップ区切り文字、デフォルト "\n-> "
    ModelTier             string // モデル階層
}
```

### 結果構造

```go
type ChainOfThoughtResult struct {
    FinalAnswer    string        // 最終回答
    ReasoningSteps []string      // 推論ステップリスト
    TotalTokens    int           // トークン消費量
    Confidence     float64       // 推論の信頼度 (0-1)
    StepDurations  []time.Duration // 各ステップの所要時間
}
```

### コアフロー

```go
func ChainOfThought(
    ctx workflow.Context,
    query string,
    context map[string]interface{},
    sessionID string,
    history []string,
    config ChainOfThoughtConfig,
    opts Options,
) (*ChainOfThoughtResult, error) {

    // 1. デフォルト値を設定
    if config.MaxSteps == 0 {
        config.MaxSteps = 5
    }
    if config.StepDelimiter == "" {
        config.StepDelimiter = "\n-> "
    }

    // 2. CoT プロンプトを構築
    cotPrompt := buildChainOfThoughtPrompt(query, config)

    // 3. LLM を呼び出し
    cotResult := executeAgent(ctx, cotPrompt, ...)

    // 4. 推論ステップを解析
    steps := parseReasoningSteps(cotResult.Response, config.StepDelimiter)

    // 5. 最終回答を抽出
    answer := extractFinalAnswer(cotResult.Response, steps)

    // 6. 信頼度を計算
    confidence := calculateReasoningConfidence(steps, cotResult.Response)

    // 7. 低信頼度の場合は明確化を要求（オプション）
    if config.RequireExplanation && confidence < 0.7 {
        // 予算の半分で、より明確な説明を再生成
        clarificationResult := requestClarification(ctx, query, steps)
        // 結果を更新...
    }

    return &ChainOfThoughtResult{
        FinalAnswer:    answer,
        ReasoningSteps: steps,
        Confidence:     confidence,
        TotalTokens:    cotResult.TokensUsed,
    }, nil
}
```

---

## 12.5 ステップの解析

LLM が生成した推論プロセスは、構造化されたステップリストに解析する必要がある。Shannon の実装はこうだ：

```go
func parseReasoningSteps(response, delimiter string) []string {
    lines := strings.Split(response, "\n")
    steps := []string{}

    for _, line := range lines {
        line = strings.TrimSpace(line)
        // ステップマーカーを識別
        if strings.HasPrefix(line, "->") ||
           strings.HasPrefix(line, "Step") ||
           strings.HasPrefix(line, "1.") ||
           strings.HasPrefix(line, "2.") ||
           strings.HasPrefix(line, "3.") ||
           strings.HasPrefix(line, "•") {
            steps = append(steps, line)
        }
    }

    // フォールバック戦略：明確なマーカーがない場合、文で分割
    if len(steps) == 0 {
        segments := strings.Split(response, ". ")
        for _, seg := range segments {
            if len(strings.TrimSpace(seg)) > 20 {
                steps = append(steps, seg)
                if len(steps) >= 5 {
                    break
                }
            }
        }
    }

    return steps
}
```

解析の優先順位：
1. 明示的マーカー（->, Step, 数字.）
2. 記号マーカー（•）
3. フォールバック：文で分割

### 最終回答の抽出

```go
func extractFinalAnswer(response string, steps []string) string {
    // 結論マーカーを探す
    markers := []string{
        "Therefore:",
        "Final Answer:",
        "The answer is:",
        "したがって：",
        "結論：",
    }

    lower := strings.ToLower(response)
    for _, marker := range markers {
        if idx := strings.Index(lower, strings.ToLower(marker)); idx != -1 {
            answer := response[idx+len(marker):]
            // 次の空行まで取得
            if endIdx := strings.Index(answer, "\n\n"); endIdx > 0 {
                answer = answer[:endIdx]
            }
            return strings.TrimSpace(answer)
        }
    }

    // フォールバック：最後のステップを使用
    if len(steps) > 0 {
        return steps[len(steps)-1]
    }

    // さらにフォールバック：最後の段落
    paragraphs := strings.Split(response, "\n\n")
    if len(paragraphs) > 0 {
        return paragraphs[len(paragraphs)-1]
    }

    return response
}
```

ここには多層のフォールバック戦略がある。LLM が期待通りのフォーマットで出力しなくても、意味のある回答を抽出できるようにね。

---

## 12.6 信頼度評価

推論の品質は定量的に評価できる。Shannon の実装：

```go
func calculateReasoningConfidence(steps []string, response string) float64 {
    confidence := 0.5 // 基礎スコア

    // ステップの十分さ：3ステップ以上で加点
    if len(steps) >= 3 {
        confidence += 0.2
    }

    // 論理接続詞
    logicalTerms := []string{
        "therefore", "because", "since", "thus",
        "consequently", "hence", "so", "implies",
    }
    lower := strings.ToLower(response)
    count := 0
    for _, term := range logicalTerms {
        count += strings.Count(lower, term)
    }
    if count >= 3 {
        confidence += 0.15
    }

    // 構造化マーカー
    if strings.Contains(response, "Step") || strings.Contains(response, "->") {
        confidence += 0.1
    }

    // 明確な結論
    if strings.Contains(lower, "therefore") ||
       strings.Contains(lower, "final answer") {
        confidence += 0.05
    }

    if confidence > 1.0 {
        confidence = 1.0
    }

    return confidence
}
```

信頼度の計算式（これは議論のために設計したヒューリスティックで、学術的な標準じゃない）：

```
信頼度 = 0.5 (基礎)
       + 0.2 (ステップ >= 3)
       + 0.15 (論理語 >= 3)
       + 0.1 (構造化マーカー)
       + 0.05 (明確な結論)
       ────────────────
       最高 1.0
```

---

## 12.7 低信頼度の処理

`RequireExplanation=true` かつ信頼度が0.7未満の場合、Shannon は明確化を要求する：

```go
if config.RequireExplanation && confidence < 0.7 {
    clarificationPrompt := fmt.Sprintf(
        "The previous reasoning for '%s' had unclear steps. "+
        "Please provide a clearer step-by-step explanation:\n%s",
        query,
        strings.Join(steps, config.StepDelimiter),
    )

    // 予算の半分で再生成
    clarifyResult := executeAgentWithBudget(ctx, clarificationPrompt, opts.BudgetAgentMax/2)

    // 結果を更新
    if clarifyResult.Success {
        clarifiedSteps := parseReasoningSteps(clarifyResult.Response, delimiter)
        if len(clarifiedSteps) > 0 {
            result.ReasoningSteps = clarifiedSteps
            result.FinalAnswer = extractFinalAnswer(clarifyResult.Response, clarifiedSteps)
            result.Confidence = calculateReasoningConfidence(clarifiedSteps, clarifyResult.Response)
        }
        result.TotalTokens += clarifyResult.TokensUsed
    }
}
```

明確化の戦略：
- 元のステップを参考として渡す
- 予算の半分を使用（コスト管理）
- より明確な説明を要求

---

## 12.8 CoT vs Tree-of-Thoughts

CoT は線形だ。一歩ずつ進んで、後戻りはない。

Tree-of-Thoughts (ToT) はツリー形だ。各ステップで複数の分岐があり、バックトラックできる。

| 特性 | Chain-of-Thought | Tree-of-Thoughts |
|------|------------------|------------------|
| **構造** | 線形チェーン | 分岐ツリー |
| **探索** | 単一パス | 複数パス並列 |
| **バックトラック** | サポートなし | サポートあり |
| **トークン消費** | 比較的少ない | 比較的多い (3-10倍) |
| **適用シーン** | 確定的な推論 | 探索的な問題 |

### いつ ToT を使うべき？

```
問題に複数の解決パスがあるか？
├─ いいえ -> CoT を使う（単一パスで十分）
└─ はい -> 異なる方法を比較する必要があるか？
         ├─ いいえ -> CoT を使う（ランダムに1つ選ぶ）
         └─ はい -> ToT を使う（体系的に探索）
```

ToT は第17章で詳しく説明する。今は覚えておいてほしい：**ほとんどのシーンでは CoT で十分だ**。

---

## 12.9 よくある落とし穴

### 落とし穴1：過剰なステップ分割

**症状**：シンプルな問題が強制的に多くのステップに分解され、出力が冗長になる。

```
// 「2+3 は何？」と聞いたら
-> ステップ1: 問題の種類を識別——これは足し算の問題
-> ステップ2: オペランドを特定——2 と 3
-> ステップ3: 足し算の定義を復習
-> ステップ4: 計算を実行 2 + 3 = 5
-> ステップ5: 結果を検証
Therefore: 5
```

**解決策**：複雑さに応じて MaxSteps を動的に調整：

```go
func adaptiveMaxSteps(query string) int {
    complexity := estimateComplexity(query)
    if complexity < 0.3 {
        return 2  // シンプルな問題
    } else if complexity < 0.7 {
        return 5  // 中程度
    }
    return 8      // 複雑
}
```

### 落とし穴2：推論と事実の混同

**症状**：CoT が生成したステップは「もっともらしく見える」が、誤った事実に基づいている。

```
-> ステップ1: テスラは2020年に世界で時価総額最高の自動車会社になった（誤った事実）
-> ステップ2: したがって販売台数も最高のはずだ（誤った推論）
```

問題は：論理は正しいが、前提が間違っているので、結論も間違い。

**解決策**：CoT をツールと組み合わせて、重要な事実を検証：

```
推論する際は以下の原則に従ってください：
1. 具体的なデータを含む場合、[要検証] と注記する
2. 「推論」と「事実の陳述」を区別する
3. ある事実が不確かな場合、明確にそう述べる
```

### 落とし穴3：信頼度の水増し

**症状**：モデルが多くの論理接続詞を使っているが、実際の推論品質は低い。

例えば循環論法：
```
-> ステップ1: A は真である、なぜなら B が真だから
-> ステップ2: B は真である、なぜなら A が真だから
Therefore: A と B は両方とも真である
```

「because」を使っているから信頼度は加点されるが、これは無効な推論だ。

**解決策**：意味的なチェックを追加：

```go
func enhancedConfidence(steps []string, response string) float64 {
    base := calculateConfidence(steps, response)

    // 循環論法をチェック
    if hasCircularReasoning(steps) {
        base -= 0.3
    }

    // ステップ間の論理的一貫性をチェック
    if !hasLogicalCoherence(steps) {
        base -= 0.2
    }

    return max(0, min(1.0, base))
}
```

### 落とし穴4：フォーマットの不一致

**症状**：LLM が時々「->」を使い、時々「Step」を使い、時々数字を使うので、パースが失敗する。

**解決策**：プロンプトでフォーマットを明確にし、パース時に複数のフォーマットをサポート（Shannon は既にそうしている）。

---

## 12.10 いつ CoT を使うべき？

すべてのタスクに CoT が必要なわけじゃない。

| タスクの種類 | CoT を使う？ | 理由 |
|-------------|-------------|------|
| シンプルな計算 | いいえ | 直接計算した方が速い |
| 事実の検索 | いいえ | 直接検索した方が正確 |
| 複数ステップの数学 | はい | 計算ミスを減らす |
| 論理推論 | はい | 推論チェーンを可視化 |
| 因果分析 | はい | 因果関係を追跡 |
| コードデバッグ | はい | 体系的に調査 |
| クリエイティブライティング | いいえ | 創造性を制限してしまう |
| リアルタイム対話 | 状況による | レイテンシ vs 正確性のトレードオフ |

**経験則**：

- 「導出」が必要なら -> 使う
- 「プロセスの監査」が必要なら -> 使う
- シンプルで直接的なら -> 使わない
- クリエイティブ系なら -> 使わない
- レイテンシに敏感なら -> 慎重に使う

---

## 12.11 他のフレームワークはどうしてる？

CoT は汎用パターンだから、各社で実装がある：

| フレームワーク/論文 | 実装方法 | 特徴 |
|---------------------|----------|------|
| **Zero-shot CoT** | "Let's think step by step" | 最もシンプル、一言でトリガー |
| **Few-shot CoT** | 推論例を提供 | よりコントロール可能、ただし人手で設計が必要 |
| **Self-Consistency** | 複数回 CoT + 投票 | より正確、ただしコスト高 |
| **LangChain** | CoT プロンプトテンプレート | 統合しやすい |
| **OpenAI o1** | 内蔵推論（ブラックボックス） | 自動 CoT、コントロール不可 |

コアロジックはみんな同じ：LLM に推論プロセスを書き出させる。

違いは：
- トリガー方法（ゼロショット vs フューショット）
- フォーマット制約（自由 vs 構造化）
- 品質保証（単発 vs 複数回投票）

---

## 12.12 ReAct との関係

こう思うかもしれない。CoT と ReAct は何が違うの？

| 観点 | CoT | ReAct |
|------|-----|-------|
| **核心目標** | 推論プロセスを可視化 | 推論 + 行動ループ |
| **ツール呼び出し** | なし（純粋な推論） | あり（考えながら動く） |
| **出力** | 推論ステップ + 回答 | 複数ラウンドの思考/行動/観察 |
| **適用シーン** | 計算/推論が必要な問題 | 外部情報の取得が必要なタスク |

簡単に言うと：

- **CoT**：じっくり考えてから答える（外部情報は不要）
- **ReAct**：考えながら調べて行動する（外部情報が必要）

両者は組み合わせられる。ReAct の「思考」フェーズで CoT を使うんだ。

---

## 要点まとめ

1. **CoT の本質**：思考を外部化し、モデルに段階的推論を強制する
2. **プロンプト設計**：明確な指示「step by step」+ フォーマット規約（->, Step）
3. **ステップ解析**：マーカー識別 + 多層フォールバック戦略
4. **信頼度評価**：ステップ数 + 論理語 + 構造化マーカー（ヒューリスティック、学術標準ではない）
5. **適用シーン**：複数ステップ推論、プロセス監査が必要な場合。シンプルなタスク、クリエイティブ系には不向き

---

## Shannon Lab（10分で始める）

このセクションでは、10分で本章の概念を Shannon のソースコードに対応させる。

### 必読（1ファイル）

- [`patterns/chain_of_thought.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/chain_of_thought.go)：`ChainOfThought` 関数を見つけて、`buildChainOfThoughtPrompt` でプロンプトを構築し、`parseReasoningSteps` で推論ステップを解析し、`calculateReasoningConfidence` で信頼度を評価する方法を確認しよう

### 選読（興味に応じて2つ）

- [`patterns/tree_of_thoughts.go`](https://github.com/Kocoro-lab/Shannon/blob/main/go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go)：ToT と CoT の実装の違いを比較
- ChatGPT/Claude で自分で試してみよう：同じ数学の問題で、「Let's think step by step」を入れるか入れないかで、回答がどう変わるか？

---

## 練習

### 練習1：CoT テンプレートを設計

以下のシーン用に専用の CoT プロンプトテンプレートを設計しよう：

1. **法的推論**：ある行為が違法かどうかを判断
2. **医療診断**：症状から可能性のある病気を推測
3. **金融分析**：ある株式の投資価値を評価

各テンプレートには以下を含めること：
- 問題記述のプレースホルダー
- 推論ステップのフォーマット要件
- 結論のフォーマット要件

### 練習2：ソースコードリーディング

`patterns/chain_of_thought.go` の `parseReasoningSteps` 関数を読んで：

1. どのステップマーカーフォーマットをサポートしている？
2. LLM がマーカーを使わなかった場合、どうフォールバック処理する？
3. フォールバック時に最大5ステップに制限しているのはなぜ？

### 練習3（上級）：循環論法検出を設計

`hasCircularReasoning` 関数を設計しよう：

- 入力：推論ステップのリスト
- 出力：循環論法が存在するかどうか

考えてみよう：
- どんなパターンが「循環論法」になる？
- 何を使って検出する？（キーワードマッチング？意味的類似度？）
- 偽陽性のリスクはある？

---

## もっと深く学びたい？

- [Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903) - Wei et al., 2022、オリジナル論文
- [Zero-shot CoT: "Let's think step by step"](https://arxiv.org/abs/2205.11916) - 最もシンプルな CoT トリガー方法
- [Self-Consistency Decoding](https://arxiv.org/abs/2203.11171) - 複数回 CoT + 投票で正答率向上
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601) - CoT のツリー形拡張

---

## 次章の予告

ここで Part 4（単一エージェントパターン）は終わりだ。3つの核心パターンを学んだ：

- **Planning**：複雑なタスクをサブタスクに分解
- **Reflection**：出力品質を評価し、基準未満ならリトライ
- **Chain-of-Thought**：推論プロセスを可視化し、飛躍的なミスを減らす

でも、1つのエージェントでできることには限界がある。タスクが十分に複雑になると、複数のエージェントが協力する必要が出てくる。

それが Part 5 の内容だ。**マルチエージェント編成**。

次章では、まず編成の基礎から：単一エージェントでは足りないとき、複数のエージェントにどう分担させる？誰が誰に何をさせるか決める？失敗したらどうする？
