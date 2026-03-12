---
title: "付録A: プロンプトテンプレートライブラリ"
free: false
---

:::message
**この付録で学べること**
- コピーして即使えるシステムプロンプト・few-shot・CoTテンプレート集
- バージョン管理できるYAML形式プロンプト設計パターン
- A/Bテストハーネスで最適なプロンプトを自動選択する手法
:::

# 付録A: プロンプトテンプレートライブラリ

プロンプトは「コード」だ。バージョン管理し、テストし、改善サイクルを回す必要がある。

この付録ではnotecreatorで実際に使っているプロンプトテンプレートを全公開する。

---

## A.1 システムプロンプトテンプレート集

### 技術記事ライター

```python
# prompts/article_writer.py

ARTICLE_WRITER = """あなたは技術系メディアの専門ライターです。

## ミッション
読者が「これを読んで実装できた」と感じる記事を書く。

## 執筆原則
1. **具体性**: 抽象論より動くコードを優先する
2. **構造化**: H2見出しで3〜5セクションに分ける
3. **検索意図**: 読者が「何を解決したくて検索したか」を意識する
4. **数値**: 「速い」より「3倍速い（比較環境: M2 Mac）」と書く

## 禁止事項
- 「ChatGPTと聞いてみました」系の薄いコンテンツ
- コードなしの概念解説のみ
- 「〜については様々な意見があります」系の逃げ
- 「申し訳ありませんが」「できません」

## 出力フォーマット
```markdown
# タイトル（キーワードを含む 40字以内）

## はじめに（問題提起・読者が抱える悩み）

## 解決策（コード付き実装例）

### 実装例1

### 実装例2

## 実践: {keyword}をプロジェクトに組み込む

## まとめ（3つのポイント）
```
"""

SEO_OPTIMIZER = """あなたはSEOコピーライターです。

## タスク
与えられた記事のタイトルとメタディスクリプションを最適化する。

## 制約
- タイトル: 32〜60文字
- メタディスクリプション: 80〜120文字
- キーワードを自然に1〜3回使用
- クリック率を高める表現（数字・疑問形・ベネフィット）

## 出力形式 (必ずJSON)
```json
{
  "title": "最適化後タイトル",
  "meta": "メタディスクリプション（120字以内）",
  "hashtags": ["#Python", "#AI", "#機械学習"],
  "slug": "url-friendly-slug"
}
```
"""

CODE_REVIEWER = """あなたはシニアPythonエンジニアです。

## レビュー観点
1. セキュリティ: SQLインジェクション、XSS、コマンドインジェクション
2. パフォーマンス: N+1クエリ、不要なループ、メモリリーク
3. 保守性: 命名規則、型ヒント、docstring
4. テスタビリティ: 副作用の分離、モック可能な設計

## 出力フォーマット
```markdown
## 🔴 重大な問題（要修正）
- [場所]: 問題の説明と修正例

## 🟡 改善提案（推奨）
- [場所]: 改善点と理由

## 🟢 良い点
- 良いコードパターンを明示的に褒める

## 修正済みコード
```python
# 修正例
```
```
"""

DATA_ANALYST = """あなたはデータアナリストです。

## 分析方針
- 相関と因果を混同しない
- サンプルサイズを必ず明示する
- 「有意差あり」ではなく効果量(Cohen's d等)で語る
- 外れ値の処理方針を説明する

## 出力フォーマット
```markdown
## 分析サマリー（3行以内）

## 方法論
- データ: N={n}, 期間: {period}
- 手法: {method}
- 外れ値処理: {outlier_handling}

## 結果
| 指標 | 値 | 信頼区間 |
|------|-----|---------|

## 解釈と注意点
```
"""
```

---

## A.2 Few-shotテンプレート — 例を見せて品質を引き出す

```python
# prompts/few_shot.py

def build_few_shot_prompt(
    task_description: str,
    examples: list[dict],  # [{"input": str, "output": str}]
    new_input: str,
) -> str:
    """
    Few-shotプロンプトを構築する.

    Args:
        task_description: タスクの説明
        examples: 入出力例のリスト（2〜5例が最適）
        new_input: 実際の入力

    Returns:
        構造化されたfew-shotプロンプト
    """
    prompt = f"{task_description}\n\n"
    prompt += "以下の例を参考にしてください:\n\n"

    for i, example in enumerate(examples, 1):
        prompt += f"### 例{i}\n"
        prompt += f"入力: {example['input']}\n"
        prompt += f"出力: {example['output']}\n\n"

    prompt += f"### 本番\n"
    prompt += f"入力: {new_input}\n"
    prompt += "出力: "

    return prompt


# 実際の使用例: タイトル生成
TITLE_EXAMPLES = [
    {
        "input": "Python asyncio 基礎",
        "output": "Python asyncioで非同期処理をマスターする実践ガイド【コード付き】"
    },
    {
        "input": "FastAPI 認証 JWT",
        "output": "FastAPI + JWT認証を30分で実装する完全チュートリアル2026"
    },
    {
        "input": "SQLite パフォーマンス チューニング",
        "output": "SQLiteを10倍速くする: WALモード・インデックス・VACUUMの実践最適化"
    },
]

def generate_seo_title(keyword: str) -> str:
    return build_few_shot_prompt(
        task_description="以下のキーワードから、SEO効果の高い技術記事タイトルを1つ生成してください。",
        examples=TITLE_EXAMPLES,
        new_input=keyword,
    )
```

---

## A.3 Chain-of-Thoughtテンプレート

```python
# prompts/chain_of_thought.py

COT_ARTICLE_ANALYSIS = """以下の手順で記事を分析してください:

**ステップ1: 読者の検索意図を特定する**
キーワード「{keyword}」で検索する人が抱える問題は何か?
→

**ステップ2: 競合記事の弱点を仮定する**
よくある記事が解決できていない点は何か?
→

**ステップ3: 差別化ポイントを決める**
この記事が提供する独自の価値は何か?
→

**ステップ4: 記事構成を設計する**
上記を踏まえた最適な見出し構成:
→

**ステップ5: 記事を執筆する**
設計に従って1200字以上の記事を書く:
→
"""

# 使用例
def analyze_and_write(keyword: str) -> str:
    """Chain-of-Thoughtで品質の高い記事生成."""
    return COT_ARTICLE_ANALYSIS.format(keyword=keyword)
```

---

## A.4 構造化出力テンプレート — XMLで確実にパース

```python
# prompts/structured_output.py
import re
from dataclasses import dataclass


STRUCTURED_ARTICLE_PROMPT = """記事を以下のXML形式で出力してください:

<article>
  <title>記事タイトル（40字以内）</title>
  <meta>メタディスクリプション（120字以内）</meta>
  <keywords>
    <keyword>メインキーワード</keyword>
    <keyword>サブキーワード1</keyword>
    <keyword>サブキーワード2</keyword>
  </keywords>
  <content>
記事本文（マークダウン形式、1200字以上）
  </content>
  <reading_time>推定読了時間（分）</reading_time>
</article>

キーワード: {keyword}
"""


@dataclass
class StructuredArticle:
    title: str
    meta: str
    keywords: list[str]
    content: str
    reading_time: int


def parse_structured_article(xml_text: str) -> StructuredArticle:
    """XMLレスポンスをパースする."""

    def extract(tag: str) -> str:
        match = re.search(
            rf'<{tag}>(.*?)</{tag}>',
            xml_text,
            re.DOTALL
        )
        return match.group(1).strip() if match else ""

    keywords = re.findall(r'<keyword>(.*?)</keyword>', xml_text)

    reading_time_str = extract("reading_time")
    try:
        reading_time = int(reading_time_str)
    except ValueError:
        reading_time = 5  # デフォルト

    return StructuredArticle(
        title=extract("title"),
        meta=extract("meta"),
        keywords=keywords,
        content=extract("content"),
        reading_time=reading_time,
    )
```

---

## A.5 プロンプトのバージョン管理 — YAML形式

プロンプトをYAMLファイルで管理することで、コードを変更せずにプロンプトを改善できる。

```yaml
# prompts/article_writer_v2.yml
name: article_writer
version: "2.1.0"
model: claude-sonnet-4-6
created_at: "2026-03-01"
changelog:
  - "2.1.0: 禁止事項に薄いAI記事パターンを追加"
  - "2.0.0: XML構造化出力に変更"
  - "1.0.0: 初版"

system: |
  あなたは技術系メディアの専門ライターです。

  ## ミッション
  読者が「これを読んで実装できた」と感じる記事を書く。

  [以下省略]

parameters:
  max_tokens: 4096
  temperature: 0.7

quality_checks:
  min_length: 1200
  required_sections:
    - "## はじめに"
    - "## まとめ"
  forbidden_patterns:
    - "申し訳ありません"
    - "ChatGPTによると"
```

```python
# prompts/loader.py
import yaml
from pathlib import Path


class PromptLoader:
    """YAMLプロンプトを読み込むローダー."""

    def __init__(self, prompts_dir: str = "prompts"):
        self.dir = Path(prompts_dir)
        self._cache: dict[str, dict] = {}

    def load(self, name: str, version: str = "latest") -> dict:
        """プロンプトを名前で読み込む."""
        if name in self._cache:
            return self._cache[name]

        # 最新バージョンを自動選択
        files = sorted(self.dir.glob(f"{name}_v*.yml"), reverse=True)
        if not files:
            raise FileNotFoundError(f"プロンプトが見つかりません: {name}")

        data = yaml.safe_load(files[0].read_text())
        self._cache[name] = data
        return data

    def get_system(self, name: str) -> str:
        """システムプロンプトのみ取得."""
        return self.load(name)["system"]

    def get_params(self, name: str) -> dict:
        """API呼び出しパラメータを取得."""
        data = self.load(name)
        return data.get("parameters", {})
```

---

## A.6 A/Bテストハーネス

```python
# prompts/ab_test.py
import random
from dataclasses import dataclass, field
from typing import Callable


@dataclass
class ABVariant:
    name: str
    prompt: str
    weight: float = 0.5  # 表示確率


@dataclass
class ABTestResult:
    variant_a_scores: list[float] = field(default_factory=list)
    variant_b_scores: list[float] = field(default_factory=list)

    @property
    def winner(self) -> str:
        a_avg = sum(self.variant_a_scores) / max(len(self.variant_a_scores), 1)
        b_avg = sum(self.variant_b_scores) / max(len(self.variant_b_scores), 1)
        return "A" if a_avg >= b_avg else "B"

    @property
    def summary(self) -> str:
        a_avg = sum(self.variant_a_scores) / max(len(self.variant_a_scores), 1)
        b_avg = sum(self.variant_b_scores) / max(len(self.variant_b_scores), 1)
        return (
            f"A: {a_avg:.2f} (n={len(self.variant_a_scores)}) vs "
            f"B: {b_avg:.2f} (n={len(self.variant_b_scores)}) "
            f"→ Winner: {self.winner}"
        )


class PromptABTest:
    """プロンプトのA/Bテスト実行器."""

    def __init__(
        self,
        variant_a: ABVariant,
        variant_b: ABVariant,
        score_fn: Callable[[str], float],  # 出力を受け取ってスコアを返す
    ):
        self.variant_a = variant_a
        self.variant_b = variant_b
        self.score_fn = score_fn
        self.result = ABTestResult()

    def select_variant(self) -> ABVariant:
        """重み付きランダムでバリアントを選択."""
        total = self.variant_a.weight + self.variant_b.weight
        rand = random.random() * total
        return self.variant_a if rand < self.variant_a.weight else self.variant_b

    def record(self, variant_name: str, output: str):
        """出力結果を記録."""
        score = self.score_fn(output)
        if variant_name == self.variant_a.name:
            self.result.variant_a_scores.append(score)
        else:
            self.result.variant_b_scores.append(score)

    def get_result(self) -> ABTestResult:
        return self.result


# 使用例
def run_ab_test_example():
    variant_a = ABVariant(
        name="short_system",
        prompt="技術記事を書いてください。",
    )
    variant_b = ABVariant(
        name="detailed_system",
        prompt=ARTICLE_WRITER,  # 詳細なシステムプロンプト
    )

    # LLMスコアを評価関数として使用
    judge = LLMJudge(api_key="your-key")

    ab_test = PromptABTest(
        variant_a=variant_a,
        variant_b=variant_b,
        score_fn=lambda output: judge.judge("Python", output).overall,
    )

    # 20回テスト実行
    for _ in range(20):
        variant = ab_test.select_variant()
        # ... generate output using variant.prompt ...
        output = "生成された記事テキスト"  # 実際はAPI呼び出し
        ab_test.record(variant.name, output)

    print(ab_test.get_result().summary)
    # → A: 6.8 (n=10) vs B: 8.2 (n=10) → Winner: B
```

---

## まとめ

| テンプレート | 用途 | ファイル |
|------------|------|---------|
| ARTICLE_WRITER | 技術記事生成 | `prompts/article_writer.py` |
| SEO_OPTIMIZER | タイトル/meta最適化 | `prompts/article_writer.py` |
| Few-shot builder | 例示ベース品質向上 | `prompts/few_shot.py` |
| CoT template | 段階的推論 | `prompts/chain_of_thought.py` |
| XML structured | 確実なパース | `prompts/structured_output.py` |
| YAML versioning | プロンプト管理 | `prompts/loader.py` |
| A/B harness | 最適化 | `prompts/ab_test.py` |

→ 次の付録: [付録B: AIエージェントデバッグガイド](./bonus-debug)
