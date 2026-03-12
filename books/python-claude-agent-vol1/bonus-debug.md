---
title: "付録B: AIエージェントデバッグガイド"
---

:::message
**この付録で学べること**
- ツール呼び出しのトレース方法とデバッグ戦略
- 無限ループ・コストスパイク・ハルシネーションの調査手順
- リプレイテストで再現性の高いデバッグ環境を構築する手法
:::

# 付録B: AIエージェントデバッグガイド

「エージェントが途中で止まった。なぜ？」

LLMエージェントのデバッグは通常のプログラムと違う。スタックトレースだけでは原因がわからない。Claudeが**なぜその判断をしたか**を追跡する必要がある。

---

## B.1 構造化ロギングのセットアップ

まず「見える化」が全ての基本だ。

```python
# debug/logger.py
import json
import logging
import time
from contextvars import ContextVar
from dataclasses import dataclass, field, asdict
from datetime import datetime
from pathlib import Path
from typing import Any


# リクエストごとのトレースIDを管理
trace_id_var: ContextVar[str] = ContextVar("trace_id", default="unknown")


@dataclass
class AgentTrace:
    """1回のエージェント実行の完全なトレース."""
    trace_id: str
    started_at: str
    keyword: str
    iterations: list[dict] = field(default_factory=list)
    tool_calls: list[dict] = field(default_factory=list)
    final_answer: str = ""
    total_tokens: int = 0
    total_cost_usd: float = 0.0
    error: str = ""
    elapsed_seconds: float = 0.0


class AgentLogger:
    """構造化ログを出力するエージェントロガー."""

    def __init__(self, log_dir: str = "out/logs"):
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)

        # JSON Lines形式でログを追記
        self.log_file = self.log_dir / f"agent_{datetime.now():%Y%m%d}.jsonl"

        # コンソール出力（開発時のみ）
        logging.basicConfig(
            level=logging.DEBUG,
            format="%(asctime)s [%(levelname)s] %(message)s"
        )
        self.logger = logging.getLogger("agent")

    def start_trace(self, keyword: str) -> AgentTrace:
        """新しいトレースを開始する."""
        import uuid
        trace_id = str(uuid.uuid4())[:8]
        trace_id_var.set(trace_id)

        trace = AgentTrace(
            trace_id=trace_id,
            started_at=datetime.now().isoformat(),
            keyword=keyword,
        )
        self.logger.info(f"[{trace_id}] トレース開始: keyword={keyword}")
        return trace

    def log_iteration(
        self,
        trace: AgentTrace,
        iteration: int,
        model_response: Any,
        elapsed: float,
    ):
        """各イテレーションの詳細を記録."""
        iteration_data = {
            "iteration": iteration,
            "stop_reason": model_response.stop_reason,
            "input_tokens": model_response.usage.input_tokens,
            "output_tokens": model_response.usage.output_tokens,
            "elapsed_ms": int(elapsed * 1000),
            "content_blocks": len(model_response.content),
        }
        trace.iterations.append(iteration_data)
        trace.total_tokens += (
            model_response.usage.input_tokens +
            model_response.usage.output_tokens
        )

        self.logger.debug(
            f"[{trace.trace_id}] イテレーション{iteration}: "
            f"stop_reason={model_response.stop_reason}, "
            f"tokens={iteration_data['output_tokens']}"
        )

    def log_tool_call(
        self,
        trace: AgentTrace,
        tool_name: str,
        inputs: dict,
        result: Any,
        elapsed: float,
    ):
        """ツール呼び出しの詳細を記録."""
        tool_data = {
            "tool": tool_name,
            "inputs": inputs,
            "result_preview": str(result)[:200],  # 長すぎる場合は切り詰め
            "elapsed_ms": int(elapsed * 1000),
            "success": "error" not in str(result).lower(),
        }
        trace.tool_calls.append(tool_data)

        self.logger.info(
            f"[{trace.trace_id}] ツール呼び出し: {tool_name} "
            f"({'成功' if tool_data['success'] else '失敗'}) "
            f"{elapsed*1000:.0f}ms"
        )

    def end_trace(self, trace: AgentTrace, final_answer: str, error: str = ""):
        """トレースを終了して保存."""
        trace.final_answer = final_answer[:500]  # 先頭500字
        trace.error = error
        trace.elapsed_seconds = (
            datetime.now() - datetime.fromisoformat(trace.started_at)
        ).total_seconds()

        # JSON Linesに追記
        with self.log_file.open("a", encoding="utf-8") as f:
            f.write(json.dumps(asdict(trace), ensure_ascii=False) + "\n")

        status = "✅" if not error else "❌"
        self.logger.info(
            f"[{trace.trace_id}] {status} 完了: "
            f"{trace.elapsed_seconds:.1f}秒, "
            f"{trace.total_tokens}tokens"
        )
```

---

## B.2 ツール呼び出しのトレース

```python
# debug/tool_tracer.py

class TracedToolManager:
    """ツール呼び出しをトレースするラッパー."""

    def __init__(self, manager: "ToolManager", logger: AgentLogger):
        self.manager = manager
        self.logger = logger

    def execute(
        self,
        name: str,
        inputs: dict,
        trace: AgentTrace,
    ) -> Any:
        """ツールを実行してトレースに記録する."""
        start = time.time()

        self.logger.logger.debug(
            f"[{trace.trace_id}] ツール実行開始: {name}\n"
            f"  入力: {json.dumps(inputs, ensure_ascii=False)[:200]}"
        )

        result = self.manager.execute(name, inputs)
        elapsed = time.time() - start

        self.logger.log_tool_call(trace, name, inputs, result, elapsed)
        return result
```

---

## B.3 よくある障害パターンと対処法

### パターン1: 無限ループ

**症状**: エージェントが同じツールを繰り返し呼び出す。コストが急増する。

```
イテレーション1: fetch_trends(source="hackernews") → 成功
イテレーション2: fetch_trends(source="hackernews") → 成功 ← 同じ!
イテレーション3: fetch_trends(source="hackernews") → 成功 ← 同じ!
...（max_iterationsまで続く）
```

**原因**: ツールの結果がClaudeの期待する形式と違う。Claudeが「まだ必要な情報が得られていない」と判断して再実行する。

**診断**:
```python
def detect_loop(trace: AgentTrace, window: int = 3) -> bool:
    """直近N回のツール呼び出しが同じなら無限ループを検出."""
    if len(trace.tool_calls) < window:
        return False
    recent = trace.tool_calls[-window:]
    tools = [t["tool"] for t in recent]
    inputs = [str(t["inputs"]) for t in recent]
    return len(set(tools)) == 1 and len(set(inputs)) == 1
```

**対処**:
```python
# ❌ 問題のあるツール結果
{"data": [{"title": "foo"}, {"title": "bar"}]}  # 構造が複雑

# ✅ Claudeが理解しやすいフラット形式
{
    "keywords": ["foo", "bar", "baz"],
    "count": 3,
    "source": "hackernews",
    "message": "3件のキーワードを取得しました"
}
```

---

### パターン2: コストスパイク

**症状**: 1回の実行で想定外のコストが発生する。

**診断スクリプト**:
```python
# debug/cost_spike.py
import json
from pathlib import Path
from datetime import datetime, timedelta


def investigate_cost_spike(
    log_file: str = "out/logs/agent_today.jsonl",
    threshold_usd: float = 1.0,
):
    """コストスパイクのあったトレースを調査する."""
    spikes = []

    with open(log_file) as f:
        for line in f:
            trace = json.loads(line)
            if trace["total_cost_usd"] > threshold_usd:
                spikes.append(trace)

    if not spikes:
        print(f"コストスパイクなし (閾値: ${threshold_usd})")
        return

    print(f"\n⚠️ コストスパイク: {len(spikes)}件\n")
    for spike in sorted(spikes, key=lambda x: x["total_cost_usd"], reverse=True):
        print(f"trace_id: {spike['trace_id']}")
        print(f"  コスト: ${spike['total_cost_usd']:.4f}")
        print(f"  イテレーション数: {len(spike['iterations'])}")
        print(f"  ツール呼び出し: {len(spike['tool_calls'])}回")
        print(f"  主要ツール: {_count_tools(spike['tool_calls'])}")
        print(f"  最終エラー: {spike['error'] or 'なし'}")
        print()


def _count_tools(tool_calls: list[dict]) -> str:
    counts = {}
    for tc in tool_calls:
        counts[tc["tool"]] = counts.get(tc["tool"], 0) + 1
    return ", ".join(f"{k}×{v}" for k, v in counts.items())
```

**対処**: コストスパイクの原因トップ3と対策

| 原因 | 診断 | 対策 |
|------|------|------|
| ループによる多数イテレーション | `len(iterations) > 5` | max_iterations を5〜7に下げる |
| コンテキスト肥大化 | `input_tokens > 50000` | スライディングウィンドウ圧縮 |
| Sonnetを使うべきでない処理 | 評価/SEOでSonnet使用 | Haikuに切り替える |

---

### パターン3: ハルシネーション

**症状**: 記事に存在しないライブラリのメソッド、間違ったAPI仕様が含まれる。

**検出**:
```python
# debug/hallucination_detector.py
import re


PYTHON_HALLUCINATION_PATTERNS = [
    # 存在しないメソッドのパターン
    r'anthropic\.client\.',          # 正しくは anthropic.Anthropic()
    r'\.get_messages\(',             # 存在しないメソッド
    r'response\.text',               # 正しくは response.content[0].text
    # 旧APIパターン
    r'openai\.Completion\.create',   # 旧版API
    r'chatgpt\.',                    # 存在しない
]


def detect_hallucinations(content: str) -> list[str]:
    """コードブロック内のハルシネーションを検出する."""
    findings = []

    # コードブロックを抽出
    code_blocks = re.findall(r'```python(.*?)```', content, re.DOTALL)

    for code in code_blocks:
        for pattern in PYTHON_HALLUCINATION_PATTERNS:
            if re.search(pattern, code):
                findings.append(f"疑わしいパターン: `{pattern}` がコード中に見つかりました")

    return findings
```

**対処**: プロンプトに「使用するライブラリのバージョンを明記し、実在するAPIのみを使うこと」を追加する。また、検出した場合は品質ゲートで不合格にする。

---

## B.4 リプレイテスト — 「あの失敗」を再現する

本番で起きた問題をローカルで再現するための仕組みだ。

```python
# debug/replay.py
import json
from pathlib import Path


class TraceReplayer:
    """記録したトレースを再生して問題を再現する."""

    def __init__(self, log_file: str):
        self.traces = []
        with open(log_file) as f:
            for line in f:
                self.traces.append(json.loads(line))

    def find_failed(self) -> list[dict]:
        """エラーが発生したトレースを返す."""
        return [t for t in self.traces if t["error"]]

    def replay_tool_calls(
        self,
        trace_id: str,
        tool_executor: "ToolManager",
    ) -> list[dict]:
        """
        特定トレースのツール呼び出しを再実行する.

        本番で失敗したツール呼び出しをローカルで再現できる。
        """
        trace = next(t for t in self.traces if t["trace_id"] == trace_id)
        results = []

        for tool_call in trace["tool_calls"]:
            print(f"\n再実行: {tool_call['tool']}")
            print(f"  入力: {tool_call['inputs']}")

            result = tool_executor.execute(
                tool_call["tool"],
                tool_call["inputs"],
            )
            results.append({
                "tool": tool_call["tool"],
                "original_result": tool_call["result_preview"],
                "replayed_result": str(result)[:200],
                "match": str(result)[:200] == tool_call["result_preview"],
            })
            print(f"  結果一致: {results[-1]['match']}")

        return results


# 使用例
def debug_failed_run(trace_id: str):
    replayer = TraceReplayer("out/logs/agent_20260312.jsonl")

    print(f"=== トレース {trace_id} の再生 ===")
    failed = replayer.find_failed()
    print(f"失敗したトレース: {len(failed)}件")

    from tools import manager as tool_manager
    results = replayer.replay_tool_calls(trace_id, tool_manager)

    mismatches = [r for r in results if not r["match"]]
    if mismatches:
        print(f"\n⚠️ 結果が変わったツール: {len(mismatches)}件")
        for m in mismatches:
            print(f"  {m['tool']}: 本番とローカルで結果が異なる")
    else:
        print("\n✅ すべてのツール結果が一致 → ツール側の問題ではない")
```

---

## B.5 デバッグチェックリスト

エージェントの問題を調査する際の手順:

```
1. ログを確認する
   □ out/logs/agent_YYYYMMDD.jsonl を開く
   □ error フィールドが空でないトレースを探す
   □ iterations 数が多い (>5) トレースをピックアップ

2. ツール呼び出しを確認する
   □ tool_calls の success フィールドを確認
   □ 失敗したツールの inputs を確認
   □ 同じツールが繰り返されていないか確認

3. コストを確認する
   □ total_cost_usd が高いトレースをピックアップ
   □ 高コストの原因 (ループ/大きなコンテキスト/Sonnet誤用) を特定

4. リプレイテストを実行する
   □ TraceReplayer で問題のトレースを再生
   □ ツール結果が変わっていないか確認

5. プロンプトを修正する
   □ Claudeが誤った判断をした場合、システムプロンプトに制約を追加
   □ 修正後にSLAテストで品質を確認
```

---

## B.6 デバッグ用コマンド集

```bash
# 今日のログを確認
tail -f out/logs/agent_$(date +%Y%m%d).jsonl | python -m json.tool

# コストスパイクを調査
python -c "
from debug.cost_spike import investigate_cost_spike
investigate_cost_spike('out/logs/agent_today.jsonl', threshold_usd=0.5)
"

# 失敗トレースを一覧表示
python -c "
import json
with open('out/logs/agent_today.jsonl') as f:
    for line in f:
        t = json.loads(line)
        if t['error']:
            print(f\"{t['trace_id']}: {t['error'][:100]}\")
"

# 特定トレースの詳細確認
python -c "
import json
target = 'abc12345'
with open('out/logs/agent_today.jsonl') as f:
    for line in f:
        t = json.loads(line)
        if t['trace_id'] == target:
            print(json.dumps(t, indent=2, ensure_ascii=False))
"
```

---

## まとめ

| 問題 | 診断ツール | 対処 |
|------|-----------|------|
| 無限ループ | `detect_loop()` | max_iterations削減 + ツール出力改善 |
| コストスパイク | `investigate_cost_spike()` | モデル切替 + コンテキスト圧縮 |
| ハルシネーション | `detect_hallucinations()` | 品質ゲート強化 + プロンプト改善 |
| 再現困難な問題 | `TraceReplayer` | リプレイテストで再現 |

→ 最後の付録: [付録C: コスト最適化完全ガイド](./bonus-cost)
