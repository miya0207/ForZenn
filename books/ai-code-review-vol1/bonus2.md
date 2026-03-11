---
title: "付録B: AIコードレビュー 導入チェックリスト30項目"
free: false
---

# 付録B: AIコードレビュー 導入チェックリスト30項目

:::message
このチェックリストは「導入前」「導入中」「定着後」の3フェーズで使用します。
🔴 = 必須（スキップ不可）/ 🟡 = 推奨（理由があれば省略可能）
:::

本書のコード: https://github.com/miya0207/ForZenn

---

## フェーズ1: 導入前チェック（環境・方針の整備）

- [ ] 🔴 **APIキーの取得と権限設定が完了している**

  > Anthropic APIキーを取得し、GitHub Secretsに `ANTHROPIC_API_KEY` として登録してください。

  > ```bash
  > # GitHub CLI でSecretを設定
  > gh secret set ANTHROPIC_API_KEY --body "sk-ant-..."
  > ```

---

- [ ] 🔴 **月次コスト上限を設定している**

  > Anthropicのダッシュボードで月次利用上限を設定してください。初期は$10〜$20から始めることを推奨します。

  > ```python
  > # コスト見積もりスクリプト
  > def estimate_monthly_cost(
  >     prs_per_day: int,
  >     avg_diff_lines: int,
  >     days_per_month: int = 22,
  > ) -> float:
  >     tokens_per_pr = avg_diff_lines * 5  # 1行≒5トークン
  >     total_tokens = prs_per_day * tokens_per_pr * days_per_month
  >     cost_per_million = 0.80  # Haiku: $0.80/1M input tokens
  >     return (total_tokens / 1_000_000) * cost_per_million
  >
  > # 例: 1日5PR × 平均200行 × 22日
  > print(f"月次推定コスト: ${estimate_monthly_cost(5, 200):.2f}")
  > ```

---

- [ ] 🔴 **チームへの事前説明を実施している**

  > AIレビュー導入の目的・範囲・ルールをチームに共有してください。「AIが人間を置き換える」という誤解を防ぐことが最重要です。

  > **説明すべき3点:**
  > 1. AIはあくまで「第0レビュー」であり、最終判断は人間が行う
  > 2. AIの指摘は「提案」である（全て受け入れる必要はない）
  > 3. 最初の2週間はパイロット期間として、フィードバックを積極的に収集する

---

- [ ] 🔴 **レビューポリシーを文書化している**

  > どのPRにAIレビューを適用するか、AIの指摘をどう扱うかを明文化してください。

  > ```markdown
  > # AI Code Review ポリシー
  >
  > ## 対象PR
  > - 変更行数50行以上のPR
  > - mainブランチへのマージPR
  > - ※ dependabot、ドキュメントのみの変更は除外
  >
  > ## AIの指摘の扱い
  > - 🔴重大: マージ前に対応必須（または無視する理由をコメントで明記）
  > - 🟡警告: 対応を検討し、チームレビュアーが最終判断
  > - 🔵提案: 参考情報として扱い、必須ではない
  >
  > ## false positive について
  > - 誤った指摘は `[AI-FP]` タグをコメントに追加してCloseする
  > - 月次レトロスペクティブでパターンを集計し、プロンプトを改善する
  > ```

---

- [ ] 🟡 **既存のLintツール・静的解析ツールとの役割分担を整理している**

  > AIレビューは既存ツールの「補完」です。重複する指摘を避けるため、役割を分担してください。

  > | ツール | 担当領域 |
  > |-------|---------|
  > | flake8/ruff | PEP8、構文エラー |
  > | mypy | 型チェック |
  > | bandit | 既知のセキュリティパターン |
  > | **Claude AI** | 設計・文脈を要する問題 |

---

- [ ] 🟡 **テスト環境でパイロット実行を完了している**

  > 本番導入前に、過去のPR（クローズ済み）でAIレビューをテスト実行し、コメントの質とコストを確認してください。

  > ```bash
  > python scripts/ai_reviewer.py \
  >   --pr-url https://github.com/org/repo/pull/123 \
  >   --dry-run  # コメントは投稿せず結果を表示のみ
  > ```

---

## フェーズ2: 導入中チェック（最初の2〜4週間）

- [ ] 🔴 **AIのコメントにチームが反応しているか確認している**

  > 導入後1週間で、AIコメントの「対応率」を計測してください。

  > ```python
  > # メトリクス計算スクリプト
  > def calc_adoption_rate(ai_comments: list, resolved: list) -> float:
  >     if not ai_comments:
  >         return 0.0
  >     return len(resolved) / len(ai_comments)
  > # 目標: 2週間後に60%以上の対応率
  > ```

---

- [ ] 🔴 **false positive率を計測している（目標: 20%以下）**

  > AIが「問題あり」と指摘したが、実際には問題なかったケースを記録してください。

  > ```python
  > # false_positives.jsonl でログを蓄積
  > import json
  > from pathlib import Path
  >
  > def log_false_positive(
  >     pr_url: str,
  >     comment_id: str,
  >     category: str,
  >     reason: str,
  > ) -> None:
  >     entry = {
  >         "pr_url": pr_url,
  >         "comment_id": comment_id,
  >         "category": category,
  >         "reason": reason,
  >         "timestamp": datetime.now().isoformat(),
  >     }
  >     with open("false_positives.jsonl", "a") as f:
  >         f.write(json.dumps(entry, ensure_ascii=False) + "\n")
  > ```

---

- [ ] 🟡 **コメントが多すぎてノイズになっていないか確認している**

  > 1PRあたりのAIコメント数が10件を超えている場合、プロンプトを調整してください。

  > **調整方針:**
  > - 🔴重大のみをコメント投稿し、🟡🔵はサマリーとして1コメントにまとめる
  > - ファイル単位でフィルタリング（テストファイルはスキップ等）

---

- [ ] 🟡 **AI指摘に対する「異議申し立て」プロセスを設けている**

  > 開発者がAIの指摘に反論できる仕組みを作ってください。

  > ```
  > <!-- PRコメントのテンプレート -->
  > @ai-reviewer この指摘は [FP] です。
  > 理由: [理由を書く]
  > ```

---

- [ ] 🟡 **レビュー時間の変化を計測している**

  > AIレビュー導入による人間のレビュー時間の変化を計測してください。目標は20〜30%の削減です。

---

## フェーズ3: 定着後チェック（継続的改善）

- [ ] 🔴 **月次でプロンプトを改善している**

  > 蓄積したfalse positiveデータを元に、毎月プロンプトをレビューし改善してください。

  > ```python
  > def generate_improvement_prompt(fp_data: list[dict]) -> str:
  >     categories = [fp["category"] for fp in fp_data]
  >     top_fp = max(set(categories), key=categories.count)
  >
  >     return f"""
  > 過去1ヶ月のデータで、{top_fp}カテゴリのfalse positiveが{categories.count(top_fp)}件ありました。
  > 具体的な事例:
  > {chr(10).join(fp['reason'] for fp in fp_data if fp['category'] == top_fp[:3])}
  >
  > このカテゴリの指摘精度を上げるためのプロンプト改善案を3つ提案してください。
  > """
  > ```

---

- [ ] 🔴 **AIのモデルバージョンアップを定期的に評価している**

  > Anthropicが新モデルをリリースしたとき、同じプロンプトで品質比較を実施してください。

  > ```python
  > def compare_models(code_samples: list[str], models: list[str]) -> dict:
  >     """同じコードを複数モデルでレビューし比較"""
  >     results = {}
  >     for model in models:
  >         scores = []
  >         for code in code_samples:
  >             result = review_code(code, model=model)
  >             scores.append(result.get("overall_score", 0))
  >         results[model] = {
  >             "avg_score": sum(scores) / len(scores),
  >             "cost_per_review": estimate_cost(model, code_samples),
  >         }
  >     return results
  > ```

---

- [ ] 🟡 **AIが検出した問題の傾向を定期的にチームに共有している**

  > 「このチームでよく出る問題Top5」を月次でSlackやWikiに共有してください。繰り返し出る問題はチーム教育の機会です。

---

- [ ] 🟡 **AIレビューの「当たった事例」と「外した事例」をドキュメント化している**

  > 成功と失敗の両方を記録することで、チームのAIリテラシーが向上します。

---

- [ ] 🟡 **新メンバーのオンボーディングにAIレビューを活用している**

  > 新メンバーが提出したPRに対するAIレビューは、コーディング規約の学習機会として活用できます。

---

## チェックリストの使い方まとめ

```python
# 導入フェーズに応じた優先度
PHASE_1_MUST = ["APIキー", "コスト上限", "チーム説明", "ポリシー文書化"]
PHASE_2_MUST = ["対応率計測", "FP率計測"]
PHASE_3_MUST = ["月次プロンプト改善", "モデルバージョン評価"]

# 最低限これだけは最初にやる
MINIMUM_VIABLE = PHASE_1_MUST  # 4項目
```

---

*このチェックリストは https://github.com/miya0207/ForZenn で最新版を公開しています。*
