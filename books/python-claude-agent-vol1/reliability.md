---
title: "本番信頼性設計：リトライ・フォールバック・コスト管理"
free: false
---

# 本番信頼性設計：リトライ・フォールバック・コスト管理

:::message
**この章で学べること**
- 指数バックオフ+ジッターによるClaude API 429エラーのハンドリングを実装できる
- Sonnet→Haikuの自動フォールバックチェーンを実装できる
- 月次コスト予算アラートシステムを実装できる
:::

---

## はじめに（なぜこれが重要か）

「Claudeエージェントを本番に出したら、翌朝エラーログが1000行あった」——これはよくある話だ。

開発環境でのテストは1〜2回のAPI呼び出しで完結するが、本番では話が違う。1日100記事生成するnoteクリエイターなら、毎月3,000回以上のAPIコールが走る。その中で、レート制限、タイムアウト、モデルの廃止、コスト超過、APIのバグ——あらゆる障害が牙をむく。

Anthropicの公式ドキュメント「Usage Tiers and Rate Limits」を読めば、無料Tier（Tier 1）ではRPM（Requests Per Minute）が5、TPM（Tokens Per Minute）が20,000という制限があることがわかる。Tier 4でも1分間に4,000リクエスト、400万トークンという上限が存在する。どんなTierでも**レート制限は確実にヒットする**——これが前提だ。

この章では、notecreatorのコードをベースに、**本番で生き残るClaudeエージェント**の信頼性設計を体系的に学ぶ。

---

## 6.1 指数バックオフリトライの実装（Claude API 429対策）

### なぜ「ただのリトライ」では駄目なのか

```python
# ❌ アンチパターン：固定間隔リトライ
import anthropic

def bad_generate(client, prompt: str) -> str:
    for _ in range(3):
        try:
            response = client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=1024,
                messages=[{"role": "user", "content": prompt}],
            )
            return response.content[0].text
        except anthropic.RateLimitError:
            time.sleep(1)  # 1秒固定待機←問題！
    raise RuntimeError("3回失敗")
```

このコードの何が問題か？レート制限（HTTP 429）が発生するとき、APIサーバー側は**過負荷状態**にある。全クライアントが一斉に「1秒後」に再試行すると、サーバーへのリクエストが再び殺到し、さらなる429を引き起こす——「サンダリングハード問題」と呼ばれる現象だ。

指数バックオフ（Exponential Backoff）は、待機時間を指数関数的に増やすことでこの問題を回避する。さらに**ジッター（ランダムなゆらぎ）**を加えると、複数プロセスが同時に動いていても再同期を防げる。

```python
# ✅ 改善版：指数バックオフ + ジッター付きリトライ
import anthropic
import os
import random
import time
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional

from dotenv import load_dotenv

_ROOT = Path(__file__).resolve().parent
load_dotenv(_ROOT / ".env", override=False)

DEFAULT_MODEL = "claude-sonnet-4-6"
MAX_RETRIES = 3
RETRY_BASE_DELAY = 2.0


@dataclass
class GenerationResult:
    """API呼び出し結果（コスト追跡のためメタデータを含む）"""
    text: str
    model: str
    input_tokens: int
    output_tokens: int
    attempt_count: int
    total_wait_seconds: float


class ClaudeClient:
    def __init__(
        self,
        api_key: str,
        model: Optional[str] = None,
        model_kind: Optional[str] = None,
    ):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.model = model or self._model_from_env(model_kind)
        print(f"  ClaudeClient: model={self.model}")

    def _model_from_env(self, kind: Optional[str] = None) -> str:
        """環境変数からモデル名を取得"""
        if kind:
            v = os.getenv(f"CLAUDE_MODEL_{kind}")
            if v:
                return v
        return os.getenv("CLAUDE_MODEL", DEFAULT_MODEL)

    def generate(
        self,
        system_prompt: str,
        user_prompt: str,
        max_tokens: int = 4096,
    ) -> GenerationResult:
        """Claude APIでテキスト生成（指数バックオフリトライ付き）"""
        total_wait = 0.0

        for attempt in range(MAX_RETRIES):
            try:
                response = self.client.messages.create(
                    model=self.model,
                    max_tokens=max_tokens,
                    system=system_prompt,
                    messages=[{"role": "user", "content": user_prompt}],
                )
                return GenerationResult(
                    text=response.content[0].text,
                    model=self.model,
                    input_tokens=response.usage.input_tokens,
                    output_tokens=response.usage.output_tokens,
                    attempt_count=attempt + 1,
                    total_wait_seconds=total_wait,
                )

            except anthropic.RateLimitError as e:
                if attempt == MAX_RETRIES - 1:
                    raise  # 最終試行は例外をそのまま伝播

                # 指数バックオフ + ジッター（0.5〜1.5倍のランダム係数）
                base_delay = RETRY_BASE_DELAY * (2 ** attempt)
                jitter = random.uniform(0.5, 1.5)
                delay = base_delay * jitter
                total_wait += delay

                print(
                    f"  ⚠️  レート制限(429): {delay:.1f}秒待機 "
                    f"(試行 {attempt + 1}/{MAX_RETRIES})"
                )
                time.sleep(delay)

            except anthropic.NotFoundError:
                raise ValueError(
                    f"モデル '{self.model}' が見つかりません。"
                    f".envを確認: CLAUDE_MODEL=claude-sonnet-4-6"
                )

            except anthropic.APIStatusError as e:
                # 5xx系は一時的なサーバーエラーなので同様にリトライ
                if e.status_code >= 500 and attempt < MAX_RETRIES - 1:
                    delay = RETRY_BASE_DELAY * (2 ** attempt)
                    print(f"  ⚠️  サーバーエラー({e.status_code}): {delay:.1f}秒待機")
                    time.sleep(delay)
                    total_wait += delay
                else:
                    raise
```

:::message alert
**ポイント: GenerationResult の設計**

テキストだけでなく、トークン数・試行回数・待機時間を記録しておくことで、後述するコスト管理が容易になる。

エージェントの全APIコールでこの情報を記録する習慣をつけると、月末の請求書に驚かなくなる。
:::

---

## 6.2 フォールバック戦略：Sonnet→Haiku自動降格

### モデルヒエラルキーを活用する

| モデル | 用途 | コスト（入力/出力 per MTok） | 特徴 |
|--------|------|--------------------------|------|
| claude-opus-4 | 最高品質 | $15/$75 | 複雑な推論 |
| claude-sonnet-4-6 | バランス | $3/$15 | 本番デフォルト |
| claude-haiku-4-5 | 高速・安価 | $0.8/$4 | フィルタリング向け |

```python
# フォールバックチェーン定義
FALLBACK_CHAIN = [
    "claude-sonnet-4-6",   # プライマリ
    "claude-haiku-4-5",    # フォールバック1
]

MAX_RETRIES = 3
RETRY_BASE_DELAY = 2.0


@dataclass
class GenerationResult:
    text: str
    model: str
    input_tokens: int
    output_tokens: int
    attempt_count: int
    fallback_used: bool  # フォールバックが発動したかどうか


class ResilientClaudeClient:
    """フォールバック付きClaudeクライアント"""

    def __init__(self, api_key: str):
        self.client = anthropic.Anthropic(api_key=api_key)
        self._fallback_chain = FALLBACK_CHAIN.copy()

    def generate(
        self,
        system_prompt: str,
        user_prompt: str,
        max_tokens: int = 4096,
        require_quality: bool = False,  # Trueの場合フォールバックしない
    ) -> GenerationResult:
        """
        フォールバックチェーン付き生成。
        require_quality=Trueなら品質優先でフォールバックしない。
        """
        models_to_try = (
            self._fallback_chain[:1]  # プライマリのみ
            if require_quality
            else self._fallback_chain
        )

        last_exception = None

        for model_idx, model in enumerate(models_to_try):
            fallback_used = model_idx > 0
            if fallback_used:
                print(f"  🔄 フォールバック発動: {models_to_try[model_idx - 1]} → {model}")

            for attempt in range(MAX_RETRIES):
                try:
                    response = self.client.messages.create(
                        model=model,
                        max_tokens=max_tokens,
                        system=system_prompt,
                        messages=[{"role": "user", "content": user_prompt}],
                    )
                    return GenerationResult(
                        text=response.content[0].text,
                        model=model,
                        input_tokens=response.usage.input_tokens,
                        output_tokens=response.usage.output_tokens,
                        attempt_count=attempt + 1,
                        fallback_used=fallback_used,
                    )

                except anthropic.RateLimitError as e:
                    if attempt == MAX_RETRIES - 1:
                        print(f"  ❌ {model} でのリトライ上限到達 → 次モデルへ")
                        last_exception = e
                        break

                    delay = RETRY_BASE_DELAY * (2 ** attempt) * random.uniform(0.5, 1.5)
                    print(f"  ⚠️  {model} レート制限: {delay:.1f}秒待機")
                    time.sleep(delay)

                except anthropic.APIStatusError as e:
                    if e.status_code >= 500 and attempt < MAX_RETRIES - 1:
                        delay = RETRY_BASE_DELAY * (2 ** attempt)
                        time.sleep(delay)
                    else:
                        last_exception = e
                        break

        # 全モデルで失敗
        raise RuntimeError(
            f"全フォールバックが失敗しました。最後のエラー: {last_exception}"
        )
```

---

## 6.3 コスト計算の自動化（トークン数追跡とアラート）

### 「気づいたら$500請求」を防ぐ

```python
# automation/common/cost_tracker.py
"""APIコスト追跡・アラートシステム"""

import json
import os
import threading
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from pathlib import Path
from typing import Optional

# モデル別コスト（USD per 1,000,000 tokens）
MODEL_COSTS = {
    "claude-opus-4":       {"input": 15.0,  "output": 75.0},
    "claude-sonnet-4-6":   {"input": 3.0,   "output": 15.0},
    "claude-haiku-4-5":    {"input": 0.8,   "output": 4.0},
    "claude-sonnet-4-6":   {"input": 3.0,   "output": 15.0},
    "claude-haiku-4-5-20251001": {"input": 0.8, "output": 4.0},
}

MONTHLY_BUDGET_USD = float(os.getenv("CLAUDE_MONTHLY_BUDGET_USD", "50.0"))
ALERT_THRESHOLD_RATIO = 0.8  # 予算の80%でアラート


@dataclass
class TokenUsageRecord:
    """1回のAPI呼び出しの使用記録"""
    timestamp: str
    model: str
    input_tokens: int
    output_tokens: int
    cost_usd: float
    task_name: str = "unknown"


@dataclass
class MonthlySummary:
    """月次サマリー"""
    month: str  # "2025-07"
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    total_cost_usd: float = 0.0
    call_count: int = 0
    records: list = field(default_factory=list)


def calculate_cost(model: str, input_tokens: int, output_tokens: int) -> float:
    """トークン数からコストを計算（USD）"""
    costs = None
    for model_key, model_costs in MODEL_COSTS.items():
        if model.startswith(model_key) or model_key.startswith(model):
            costs = model_costs
            break

    if costs is None:
        print(f"  ⚠️  未知のモデル '{model}' → Sonnet価格で計算")
        costs = MODEL_COSTS["claude-sonnet-4-6"]

    input_cost = (input_tokens / 1_000_000) * costs["input"]
    output_cost = (output_tokens / 1_000_000) * costs["output"]
    return input_cost + output_cost


class CostTracker:
    """スレッドセーフなコスト追跡クラス"""

    def __init__(self, storage_path: Optional[Path] = None):
        self._storage_path = storage_path or Path("cost_log.json")
        self._lock = threading.Lock()
        self._summary = self._load_or_create_summary()

    def _current_month(self) -> str:
        return datetime.now(timezone.utc).strftime("%Y-%m")

    def _load_or_create_summary(self) -> MonthlySummary:
        month = self._current_month()
        if self._storage_path.exists():
            with open(self._storage_path) as f:
                data = json.load(f)
            if data.get("month") == month:
                summary = MonthlySummary(**{
                    k: v for k, v in data.items() if k != "records"
                })
                summary.records = data.get("records", [])
                return summary
        return MonthlySummary(month=month)

    def _save(self):
        """ロック内で呼ぶこと"""
        data = asdict(self._summary)
        with open(self._storage_path, "w") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def record(
        self,
        model: str,
        input_tokens: int,
        output_tokens: int,
        task_name: str = "unknown"
    ) -> float:
        """APIコールを記録して今月の累積コストを返す（USD）"""
        cost = calculate_cost(model, input_tokens, output_tokens)

        with self._lock:
            self._summary.total_input_tokens += input_tokens
            self._summary.total_output_tokens += output_tokens
            self._summary.total_cost_usd += cost
            self._summary.call_count += 1
            self._summary.records.append(asdict(TokenUsageRecord(
                timestamp=datetime.now(timezone.utc).isoformat(),
                model=model,
                input_tokens=input_tokens,
                output_tokens=output_tokens,
                cost_usd=cost,
                task_name=task_name
            )))
            self._save()

            # アラートチェック
            if self._summary.total_cost_usd >= MONTHLY_BUDGET_USD * ALERT_THRESHOLD_RATIO:
                self._alert(self._summary.total_cost_usd)

        return self._summary.total_cost_usd

    def _alert(self, current_cost: float) -> None:
        """予算アラート（本番ではSlack通知等に変更）"""
        ratio = current_cost / MONTHLY_BUDGET_USD
        print(
            f"\n🚨 コストアラート！ 今月の使用量: ${current_cost:.2f} "
            f"({ratio:.0%} of ${MONTHLY_BUDGET_USD:.0f} 予算)\n"
        )

    def get_summary(self) -> dict:
        """現在の月次サマリーを取得"""
        with self._lock:
            return {
                "month": self._summary.month,
                "total_cost_usd": self._summary.total_cost_usd,
                "total_cost_jpy": self._summary.total_cost_usd * 150,
                "call_count": self._summary.call_count,
                "budget_remaining_usd": MONTHLY_BUDGET_USD - self._summary.total_cost_usd,
                "budget_used_ratio": self._summary.total_cost_usd / MONTHLY_BUDGET_USD
            }


# 動作確認
if __name__ == "__main__":
    api_key = os.getenv("ANTHROPIC_API_KEY")
    client = ClaudeClient(api_key=api_key, model="claude-haiku-4-5")
    tracker = CostTracker()

    result = client.generate(
        system_prompt="あなたは技術ライターです。",
        user_prompt="Pythonのリトライ処理について1文で説明してください。",
        max_tokens=100,
    )
    print(f"生成テキスト: {result.text}")
    print(f"入力トークン: {result.input_tokens}, 出力: {result.output_tokens}")
    print(f"試行回数: {result.attempt_count}, 待機時間: {result.total_wait_seconds:.1f}秒")

    # コストを記録
    total = tracker.record(
        model=result.model,
        input_tokens=result.input_tokens,
        output_tokens=result.output_tokens,
        task_name="demo"
    )
    print(f"今月の累積コスト: ${total:.4f}")
    print(tracker.get_summary())
```

---

## 実務Tips

**Anthropic のUsage Tierを把握する**

APIのレート制限はTierによって大きく異なります（Tier1: 50RPM vs Tier4: 4000RPM）。本番ワークロードに応じてTierアップグレードを検討し、上限に近づいたら `x-ratelimit-remaining-requests` ヘッダーでプロアクティブに制御してください。

**コスト追跡は最初から設計に組み込む**

「後でコスト監視を追加しよう」は失敗します。ClaudeClientを実装するタイミングで `response.usage.input_tokens` を必ず記録する仕組みを組み込んでください。

---

## アンチパターン

**❌ 固定間隔リトライ（サンダリングハード）**

`time.sleep(60)` のような固定待機でリトライすると、すべてのクライアントが同じタイミングでリトライして再びサーバーを圧迫します。`2^attempt + random.uniform(0, 1)` の指数バックオフ+ジッターが必須です。

**❌ フォールバック先のモデルで品質劣化を無視**

Sonnet→Haikuのフォールバック時に品質が著しく下がるケースがあります。フォールバック発生時はDiscordやログで警告を出し、人間が確認できる仕組みを設けてください。

---

## まとめ

| 学習内容 | 実装できるもの |
|---------|--------------|
| リトライ戦略 | 指数バックオフ+ジッターでサンダリングハードを防ぐ |
| フォールバック | Sonnet→Haikuの自動降格でサービス継続性を確保 |
| コスト管理 | CostTrackerによるトークン追跡と月次予算アラート |

:::message
**次のステップ**

次の章「評価・テスト・モニタリング」では、このシステム全体の品質をどう「測って・改善し続けるか」を解説します。プロパティベーステスト・LLM-as-Judge評価フレームワークを実装します。
:::
