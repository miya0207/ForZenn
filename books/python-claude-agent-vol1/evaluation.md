---
title: "評価・テスト・モニタリング — 本番品質を守る仕組み"
---

:::message
**この章で学べること**
- 確率的なLLM出力をテストする「性質ベーステスト」の考え方
- Claude Haikuでコスト効率よく動く「LLM-as-Judge」評価フレームワーク
- 本番稼働中のエージェントを90%合格率SLAで守る品質ゲート実装
:::

# 第7章: 評価・テスト・モニタリング

「このエージェント、ちゃんと動いてますか？」

この問いに自信を持って答えられるか。それが本番運用と実験の境界線だ。

LLMの出力は**確率的**だ。同じプロンプトでも毎回わずかに変わる。`assert output == "expected"` という従来のユニットテストは機能しない。では、どうテストするのか。

答えは「**性質をテストする**」こと。

---

## 7.1 なぜ従来のテストが効かないのか

```python
# ❌ LLMには使えない従来のテスト
def test_generate_title():
    title = generate_title("Python入門")
    assert title == "Python完全入門ガイド2024"  # 毎回変わる
```

```python
# ✅ 性質ベーステスト
def test_generate_title():
    title = generate_title("Python入門")
    assert len(title) <= 60          # 文字数制約
    assert "Python" in title         # キーワード含有
    assert not title.startswith(" ") # 余分な空白なし
```

**性質（Property）** とは「この出力が必ず満たすべき条件」だ。内容は変わっても、性質は一定でなければならない。

---

## 7.2 性質ベーステストの実装

```python
# evaluation/property_tests.py
import pytest
from dataclasses import dataclass
from typing import Callable
import re


@dataclass
class ArticleProperties:
    """記事が満たすべき性質の定義."""
    min_length: int = 800
    max_length: int = 5000
    required_sections: list[str] = None
    forbidden_patterns: list[str] = None
    keyword_must_appear: str = ""

    def __post_init__(self):
        if self.required_sections is None:
            self.required_sections = []
        if self.forbidden_patterns is None:
            self.forbidden_patterns = []


def check_article_properties(
    content: str,
    props: ArticleProperties
) -> tuple[bool, list[str]]:
    """記事が性質を満たすか検証. (合格/不合格, エラーリスト) を返す."""
    errors = []

    # 文字数チェック
    length = len(content)
    if length < props.min_length:
        errors.append(f"文字数不足: {length} < {props.min_length}")
    if length > props.max_length:
        errors.append(f"文字数超過: {length} > {props.max_length}")

    # 必須セクションチェック
    for section in props.required_sections:
        if section not in content:
            errors.append(f"必須セクション欠如: '{section}'")

    # 禁止パターンチェック
    for pattern in props.forbidden_patterns:
        if re.search(pattern, content):
            errors.append(f"禁止パターン検出: '{pattern}'")

    # キーワード含有チェック
    if props.keyword_must_appear and props.keyword_must_appear not in content:
        errors.append(f"キーワード欠如: '{props.keyword_must_appear}'")

    return len(errors) == 0, errors
```

---

## 7.3 SLAベーステスト — 確率的な合格率で管理する

100%合格を要求すると、LLMはそれを達成できない。現実的なアプローチは「**N回中X%が合格すればOK**」というSLA（Service Level Agreement）を設定することだ。

```python
# evaluation/sla_tests.py
from dataclasses import dataclass, field
import statistics


@dataclass
class SLAResult:
    """SLAテストの結果."""
    passed: int = 0
    failed: int = 0
    total: int = 0
    pass_rate: float = 0.0
    errors: list[str] = field(default_factory=list)
    meets_sla: bool = False


def run_sla_test(
    generate_fn: Callable[[], str],
    check_fn: Callable[[str], tuple[bool, list[str]]],
    n_trials: int = 20,
    sla_pass_rate: float = 0.90,
) -> SLAResult:
    """
    generate_fn を n_trials 回実行し、check_fn で検証する。
    sla_pass_rate 以上合格すれば meets_sla=True。
    """
    result = SLAResult(total=n_trials)
    all_errors = []

    for i in range(n_trials):
        output = generate_fn()
        ok, errors = check_fn(output)
        if ok:
            result.passed += 1
        else:
            result.failed += 1
            all_errors.extend(errors[:2])  # 最初の2エラーのみ記録

    result.pass_rate = result.passed / n_trials
    result.meets_sla = result.pass_rate >= sla_pass_rate
    result.errors = list(set(all_errors))  # 重複除去
    return result


# テスト例
class TestStochasticBehavior:
    """確率的なLLM出力の性質テスト."""

    def test_article_generation_sla(self, article_generator):
        """記事生成が90%以上の確率で品質基準を満たすこと."""
        props = ArticleProperties(
            min_length=800,
            max_length=3000,
            required_sections=["## ", "### "],
            forbidden_patterns=[r"申し訳", r"できません"],
            keyword_must_appear="Python",
        )

        result = run_sla_test(
            generate_fn=lambda: article_generator.generate("Python入門"),
            check_fn=lambda content: check_article_properties(content, props),
            n_trials=20,
            sla_pass_rate=0.90,
        )

        assert result.meets_sla, (
            f"SLA未達: {result.pass_rate:.0%} < 90%\n"
            f"頻出エラー: {result.errors[:3]}"
        )

    def test_title_generation_sla(self, article_generator):
        """タイトル生成が100%の確率で文字数制約を満たすこと."""
        # タイトルは決定論的な制約なのでSLA=100%でも現実的
        result = run_sla_test(
            generate_fn=lambda: article_generator.generate_title("Python入門"),
            check_fn=lambda t: (
                len(t) <= 60 and len(t) >= 10,
                [f"タイトル長さ異常: {len(t)}文字"] if not (10 <= len(t) <= 60) else []
            ),
            n_trials=10,
            sla_pass_rate=1.0,
        )
        assert result.meets_sla, f"タイトルSLA未達: {result.errors}"
```

:::message alert
**なぜSLAを100%にしないのか**

LLMのサンプリング（temperature > 0）は本質的に確率的だ。稀に出力フォーマットが崩れることは避けられない。

「100%要求 → テスト失敗 → 人間が確認 → 問題なし」を繰り返すより、「90%要求 → 残り10%はリトライ/人間確認」という設計が現実的で持続可能だ。

notecreatorでは品質検査6項目の合格率を計測し、80%を下回ったらDiscord通知を送る仕組みにしている。
:::

---

## 7.4 LLM-as-Judge — AIがAIの出力を評価する

人間が毎回記事を採点するのは非現実的だ。Claude Haikuを「採点官」として使う。コストはHaikuなので1評価あたり約0.003円。

```python
# evaluation/llm_judge.py
import json
import re
from dataclasses import dataclass
import anthropic


@dataclass
class JudgementResult:
    """LLM評価結果 — 4軸評価."""
    readability: int      # 読みやすさ 1-10
    accuracy: int         # 情報の正確さ 1-10
    engagement: int       # 読者の引きつけ度 1-10
    compliance: int       # 指示への準拠度 1-10
    overall: float        # 総合スコア (加重平均)
    reasoning: str        # 判断根拠
    improvement: str      # 改善提案


JUDGE_PROMPT = """あなたは技術記事の品質評価専門家です。
以下の記事を4つの軸で評価してください。

## 評価対象記事
キーワード: {keyword}
本文:
{article}

## 評価基準
各軸を1〜10点で評価し、以下のJSON形式で回答してください:

  {{
    "readability": <1-10>,
    "accuracy": <1-10>,
    "engagement": <1-10>,
    "compliance": <1-10>,
    "reasoning": "<評価根拠を2文で>",
    "improvement": "<最重要改善点を1つ>"
  }}

評価基準:
- readability: 文章の流れ、段落構成、見出し構造
- accuracy: 技術的な正確さ、事実確認
- engagement: 読者を引きつけるフック、具体例
- compliance: キーワードの自然な使用、文字数制約
"""


class LLMJudge:
    """Claude Haikuを使った記事品質評価器."""

    WEIGHTS = {
        "readability": 0.3,
        "accuracy": 0.3,
        "engagement": 0.2,
        "compliance": 0.2,
    }

    def __init__(self, api_key: str):
        self.client = anthropic.Anthropic(api_key=api_key)

    def judge(self, keyword: str, article: str) -> JudgementResult:
        """記事を4軸で評価する."""
        prompt = JUDGE_PROMPT.format(
            keyword=keyword,
            article=article[:3000],  # コスト節約: 先頭3000字
        )

        response = self.client.messages.create(
            model="claude-haiku-4-5",  # 評価はHaikuで十分
            max_tokens=512,
            messages=[{"role": "user", "content": prompt}]
        )

        return self._parse_response(response.content[0].text)

    def _parse_response(self, text: str) -> JudgementResult:
        """JSONレスポンスをパースする."""
        # コードブロック内のJSONを抽出
        match = re.search(r'```json\s*(.*?)\s*```', text, re.DOTALL)
        if not match:
            # フォールバック: テキスト全体をJSONとして試みる
            match_plain = re.search(r'\{.*\}', text, re.DOTALL)
            json_str = match_plain.group() if match_plain else "{}"
        else:
            json_str = match.group(1)

        data = json.loads(json_str)

        r = data.get("readability", 5)
        a = data.get("accuracy", 5)
        e = data.get("engagement", 5)
        c = data.get("compliance", 5)

        overall = (
            r * self.WEIGHTS["readability"] +
            a * self.WEIGHTS["accuracy"] +
            e * self.WEIGHTS["engagement"] +
            c * self.WEIGHTS["compliance"]
        )

        return JudgementResult(
            readability=r,
            accuracy=a,
            engagement=e,
            compliance=c,
            overall=round(overall, 2),
            reasoning=data.get("reasoning", ""),
            improvement=data.get("improvement", ""),
        )

    def batch_judge(
        self,
        articles: list[dict],  # [{"keyword": str, "content": str}]
    ) -> list[JudgementResult]:
        """複数記事を一括評価する."""
        return [
            self.judge(a["keyword"], a["content"])
            for a in articles
        ]
```

---

## 7.5 実際の評価スコア分布

notecreatorで3ヶ月・90記事を評価した結果:

| 評価軸 | 平均 | 最低 | 最高 | 合格率(≥7) |
|--------|------|------|------|------------|
| readability | 7.8 | 5.0 | 9.5 | 89% |
| accuracy | 8.1 | 6.0 | 9.0 | 94% |
| engagement | 6.9 | 4.0 | 9.0 | 72% |
| compliance | 8.4 | 7.0 | 10.0 | 98% |
| **overall** | **7.8** | **5.5** | **9.2** | **87%** |

engagement（引きつけ度）が最も低い。フックとなる具体的な数字や失敗談を導入部に入れると改善する。

---

## 7.6 本番品質ゲート — 6項目チェック

```python
# automation/quality/inspector.py
from dataclasses import dataclass, field


@dataclass
class QualityReport:
    """品質検査レポート."""
    keyword: str
    passed: bool
    score: float
    checks: dict[str, bool] = field(default_factory=dict)
    warnings: list[str] = field(default_factory=list)
    llm_scores: dict[str, float] = field(default_factory=dict)


class QualityInspector:
    """6項目の品質ゲート."""

    MIN_LENGTH = 800
    MIN_LLM_SCORE = 6.5

    def inspect(
        self,
        keyword: str,
        title: str,
        content: str,
        llm_judge: LLMJudge | None = None,
    ) -> QualityReport:
        """記事品質を検査する."""
        checks = {}
        warnings = []

        # 1. 文字数チェック
        checks["length"] = len(content) >= self.MIN_LENGTH
        if not checks["length"]:
            warnings.append(f"文字数不足: {len(content)}字 (最低{self.MIN_LENGTH}字)")

        # 2. 見出し構造チェック
        h2_count = content.count("## ")
        checks["headings"] = h2_count >= 2
        if not checks["headings"]:
            warnings.append(f"見出し(##)が{h2_count}個 (最低2個必要)")

        # 3. コードブロックバランスチェック
        backtick_count = content.count("```")
        checks["code_blocks"] = backtick_count % 2 == 0
        if not checks["code_blocks"]:
            warnings.append(f"コードブロックが閉じていない (```が{backtick_count}個)")

        # 4. キーワード含有チェック
        checks["keyword"] = keyword.lower() in content.lower()
        if not checks["keyword"]:
            warnings.append(f"キーワード '{keyword}' が記事内に見つからない")

        # 5. タイトル長さチェック
        checks["title_length"] = 10 <= len(title) <= 60
        if not checks["title_length"]:
            warnings.append(f"タイトル長さ異常: {len(title)}字")

        # 6. LLM品質スコアチェック (オプション)
        llm_scores = {}
        if llm_judge:
            judgement = llm_judge.judge(keyword, content)
            llm_scores = {
                "readability": judgement.readability,
                "accuracy": judgement.accuracy,
                "engagement": judgement.engagement,
                "overall": judgement.overall,
            }
            checks["llm_quality"] = judgement.overall >= self.MIN_LLM_SCORE
            if not checks["llm_quality"]:
                warnings.append(
                    f"LLMスコア低下: {judgement.overall:.1f} < {self.MIN_LLM_SCORE}\n"
                    f"  改善提案: {judgement.improvement}"
                )

        # 総合判定: 全チェック合格
        passed = all(checks.values())
        score = sum(checks.values()) / len(checks)

        return QualityReport(
            keyword=keyword,
            passed=passed,
            score=score,
            checks=checks,
            warnings=warnings,
            llm_scores=llm_scores,
        )
```

---

## 7.7 モニタリング — 本番稼働中の品質追跡

```python
# evaluation/monitor.py
import sqlite3
from datetime import datetime
from pathlib import Path


class QualityMonitor:
    """品質スコアをSQLiteに記録・分析する."""

    def __init__(self, db_path: str = "out/quality.db"):
        self.db_path = Path(db_path)
        self._init_db()

    def _init_db(self):
        self.db_path.parent.mkdir(parents=True, exist_ok=True)
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS quality_logs (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    created_at TEXT NOT NULL,
                    keyword TEXT NOT NULL,
                    passed INTEGER NOT NULL,
                    score REAL NOT NULL,
                    length INTEGER NOT NULL,
                    llm_overall REAL,
                    warnings TEXT
                )
            """)

    def record(self, keyword: str, report: "QualityReport", length: int):
        """品質レポートを記録する."""
        with sqlite3.connect(self.db_path) as conn:
            conn.execute("""
                INSERT INTO quality_logs
                (created_at, keyword, passed, score, length, llm_overall, warnings)
                VALUES (?, ?, ?, ?, ?, ?, ?)
            """, (
                datetime.now().isoformat(),
                keyword,
                int(report.passed),
                report.score,
                length,
                report.llm_scores.get("overall"),
                "\n".join(report.warnings),
            ))

    def get_recent_pass_rate(self, days: int = 7) -> float:
        """直近N日の合格率を返す."""
        with sqlite3.connect(self.db_path) as conn:
            row = conn.execute("""
                SELECT
                    COUNT(*) as total,
                    SUM(passed) as passed
                FROM quality_logs
                WHERE created_at >= datetime('now', ?)
            """, (f"-{days} days",)).fetchone()

        total, passed = row
        return (passed / total) if total > 0 else 0.0

    def alert_if_degraded(self, threshold: float = 0.80) -> bool:
        """合格率が閾値を下回ったらTrueを返す（アラート用）."""
        rate = self.get_recent_pass_rate(days=7)
        if rate < threshold:
            print(f"⚠️ 品質劣化アラート: 直近7日の合格率 {rate:.0%} < {threshold:.0%}")
            return True
        return False
```

---

## 7.8 アンチパターン

| アンチパターン | 問題点 | 正しいアプローチ |
|--------------|--------|----------------|
| 完全一致テスト | LLMは毎回違う出力 | 性質ベーステスト |
| テストなしで本番投入 | 品質劣化に気づかない | SLAテスト + モニタリング |
| 人間が全件レビュー | スケールしない | LLM-as-Judge + 人間はサンプル確認 |
| スコアのみ見る | フォーマット崩れを見逃す | 性質テスト + スコアの両方 |
| 1回だけテスト | 確率的なため | N=20以上で合格率を計測 |

---

## 7.9 実務Tips

**CI/CDに組み込む:**
```yaml
# .github/workflows/quality.yml
- name: Run quality SLA tests
  run: python -m pytest evaluation/ -v --tb=short
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**週次品質レポートをDiscordに送る:**
```python
monitor = QualityMonitor()
rate = monitor.get_recent_pass_rate(days=7)
discord.send(f"📊 週次品質レポート: 合格率 {rate:.0%}")
```

**スコアが下がったらプロンプトを見直すサイクル:**
1. モニタリングで合格率低下を検知
2. 不合格記事の `warnings` を集計
3. 頻出するエラーパターンをプロンプトに反映
4. SLAテストで改善確認

---

## まとめ

| ポイント | 内容 |
|---------|------|
| 性質テスト | 「何が出るか」でなく「どんな性質を持つか」をテスト |
| SLA設計 | N=20回の合格率≥90%を目標に設定 |
| LLM-as-Judge | Haikuで自動評価、人間はサンプル確認に集中 |

次の章では、これらの品質管理の仕組みを含む**完全な本番エージェントテンプレート**を構築する。

→ 次章: [本番エージェント完全テンプレート](./production-template)
