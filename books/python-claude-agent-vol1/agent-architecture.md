---
title: "エージェントアーキテクチャ大全：ReAct・PlanAct・Reflexion・ReWOO"
free: false
---

# エージェントアーキテクチャ大全：ReAct・PlanAct・Reflexion・ReWOO

:::message
**この章で学べること**
- ReAct・PlanAct・Reflexion・ReWOOの4パターンを実装し使い分けられる
- Anthropic tool_use（Function Calling）のスキーマ設計ベストプラクティスを習得できる
- タスク特性からアーキテクチャを選択する判断軸を持てる
:::

---

## はじめに（なぜこれが重要か）

「Claude にツールを渡したら、なぜか同じ検索を10回繰り返してループした」
「RAG を導入したが、関係ない文書を引いてくる」
「複数エージェントを動かしたら、デッドロックして全体が止まった」

これらはすべて、**アーキテクチャの選択を間違えた**ことで起きる本番トラブルです。

2022〜2023年にかけて発表された一連の論文──ReAct、Reflexion、CoALA──は単なる学術的な成果ではありません。「AIが道具を使って複雑な仕事をこなす」ための設計図であり、これを知らずにエージェントを組むのは、設計図なしで家を建てるようなものです。

本章では、研究論文の知識を**完全動作するコード**に落とし込みながら、notecreatorプロジェクト（AI記事工場）の実装を具体例として参照し、現場で即戦力になるアーキテクチャ知識を習得します。

---

## Claude Agent 全体アーキテクチャ

本書で構築するシステムの全体像：

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Agent System                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Architecture Layer                   │    │
│  │  ReAct │ PlanAct │ Reflexion │ ReWOO │ Multi-Agent  │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│              ┌───────────┼───────────┐                      │
│              ▼           ▼           ▼                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ Tool Layer  │ │Memory Layer │ │Reliability  │           │
│  │             │ │             │ │   Layer     │           │
│  │ file_tool   │ │ Working Mem │ │ Retry       │           │
│  │ web_search  │ │ Episodic M  │ │ Fallback    │           │
│  │ db_tool     │ │ Semantic M  │ │ Cost Track  │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               Evaluation Layer                       │    │
│  │   Property-Based Tests │ LLM-as-Judge │ Monitoring  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 8.1 ReAct変種の比較と実装：PlanAct・Reflexion・ReWOO

### なぜReActだけでは不十分か

ReAct論文（Yao et al., 2022）が提唱したThought→Action→Observationループは革命的でした。しかし実務では3つの限界があります。

| 限界 | 症状 | 対策アーキテクチャ |
|------|------|------------------|
| 計画なし実行 | ツールを場当たり的に呼び、無駄なAPI呼び出しが増える | **PlanAct** |
| 失敗から学ばない | 同じミスを繰り返す | **Reflexion** |
| 並列化できない | ステップが直列でスループットが低い | **ReWOO** |

notecreatorの`run_auto.py`が「トレンド収集→記事生成→SEO最適化→投稿」と明確にフェーズを分けているのは、まさにPlanActの思想です。一つのエージェントがその場で考えながら動くのではなく、**先にプランを立て、プランに従って実行する**。

```python
# ファイル: agents/react_variants.py
"""
ReAct 3変種の実装比較
- BaseReAct: 元論文の実装
- PlanAct: 先行計画型
- Reflexion: 自己反省型
- ReWOO: 並列化対応型
"""

import json
import time
import asyncio
from dataclasses import dataclass, field
from typing import Any, Callable, Optional
from enum import Enum

import anthropic


# ─────────────────────────────────────────────
# 共通基盤
# ─────────────────────────────────────────────

@dataclass
class ToolResult:
    tool_name: str
    result: Any
    error: Optional[str] = None
    elapsed_ms: float = 0.0

@dataclass
class AgentStep:
    thought: str
    action: str
    observation: str
    step_num: int

class AgentFinish(Exception):
    def __init__(self, answer: str):
        self.answer = answer


TOOLS: dict[str, Callable] = {}

def register_tool(name: str):
    """ツール登録デコレータ"""
    def decorator(fn: Callable):
        TOOLS[name] = fn
        return fn
    return decorator

@register_tool("search_web")
def search_web(query: str) -> str:
    """ダミー検索（実際はSerpAPI等を呼ぶ）"""
    time.sleep(0.1)  # API呼び出しを模倣
    return f"検索結果: '{query}' に関する情報: Python は2023年も最も人気の言語。"

@register_tool("calculate")
def calculate(expression: str) -> str:
    """安全な数式評価"""
    try:
        allowed = set('0123456789+-*/.() ')
        if not all(c in allowed for c in expression):
            return "エラー: 許可されていない文字が含まれています"
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"計算エラー: {e}"

@register_tool("write_file")
def write_file(path: str, content: str) -> str:
    """ファイル書き込み"""
    from pathlib import Path
    Path(path).write_text(content, encoding="utf-8")
    return f"ファイル '{path}' に書き込みました ({len(content)} 文字)"


# ─────────────────────────────────────────────
# 1. BaseReAct（元論文の実装）
# ─────────────────────────────────────────────

class BaseReActAgent:
    """
    ReAct: Synergizing Reasoning and Acting in Language Models (Yao et al., 2022)
    Thought → Action → Observation を繰り返すシンプルな実装
    """

    SYSTEM_PROMPT = """あなたは問題解決エージェントです。
以下の形式で思考・行動を繰り返してください：

Thought: [現状の分析と次のアクション計画]
Action: [ツール名]
Action Input: [JSON形式の引数]

ツールの結果が返ってきたら、それを踏まえて次の Thought から続けてください。
最終回答が出たら:
Final Answer: [回答]
"""

    def __init__(self, client: anthropic.Anthropic, model: str = "claude-opus-4-6"):
        self.client = client
        self.model = model
        self.steps: list[AgentStep] = []
        self.max_steps = 10  # ループ防止の上限

    def run(self, task: str) -> str:
        messages = [{"role": "user", "content": task}]

        for step_num in range(self.max_steps):
            response = self.client.messages.create(
                model=self.model,
                max_tokens=2048,
                system=self.SYSTEM_PROMPT,
                messages=messages,
            )

            text = response.content[0].text
            messages.append({"role": "assistant", "content": text})

            # Final Answer の検出
            if "Final Answer:" in text:
                return text.split("Final Answer:")[-1].strip()

            # Action の解析と実行
            observation = self._execute_action(text)
            messages.append({
                "role": "user",
                "content": f"Observation: {observation}"
            })

        return "最大ステップ数に達しました。タスクを分割してください。"

    def _execute_action(self, text: str) -> str:
        lines = text.split('\n')
        action = action_input = None

        for i, line in enumerate(lines):
            if line.startswith("Action:"):
                action = line.replace("Action:", "").strip()
            if line.startswith("Action Input:"):
                raw = line.replace("Action Input:", "").strip()
                try:
                    action_input = json.loads(raw)
                except json.JSONDecodeError:
                    return f"Action Input のJSON解析に失敗: {raw}"

        if not action:
            return "Actionが見つかりませんでした"

        if action not in TOOLS:
            return f"未知のツール: {action}。利用可能: {list(TOOLS.keys())}"

        try:
            result = TOOLS[action](**action_input) if action_input else TOOLS[action]()
            return str(result)
        except Exception as e:
            return f"ツール実行エラー ({action}): {e}"


# ─────────────────────────────────────────────
# 2. PlanAct（先行計画型）
# ─────────────────────────────────────────────

class PlanActAgent:
    """
    PlanAct: まず全体計画を立て、計画に従って実行する
    notecreatorの run_auto.py と同じ思想

    利点: ツール呼び出し数が少ない、デバッグしやすい
    欠点: 計画が間違っていると全体が崩れる
    """

    PLAN_PROMPT = """タスクを達成するための実行計画を、JSON配列で出力してください。

タスク: {task}

利用可能なツール: {tools}

以下の形式で出力:
{{
  "plan": [
    {{"step": 1, "tool": "ツール名", "args": {{}}, "description": "何をするか"}},
    ...
  ],
  "reasoning": "なぜこの計画にしたか"
}}"""

    def __init__(self, client: anthropic.Anthropic, model: str = "claude-opus-4-6"):
        self.client = client
        self.model = model

    def run(self, task: str) -> str:
        # フェーズ1: 計画立案
        print("  [PlanAct] フェーズ1: 計画立案中...")
        plan_data = self._create_plan(task)

        print(f"  [PlanAct] 計画: {plan_data.get('reasoning', '')}")

        # フェーズ2: 計画実行
        results = []
        for step in plan_data.get("plan", []):
            print(f"  [PlanAct] ステップ{step['step']}: {step['description']}")
            tool_name = step["tool"]
            args = step.get("args", {})

            if tool_name not in TOOLS:
                results.append(f"Step {step['step']}: ツール '{tool_name}' が存在しません")
                continue

            try:
                result = TOOLS[tool_name](**args)
                results.append(f"Step {step['step']} ({step['description']}): {result}")
            except Exception as e:
                results.append(f"Step {step['step']} エラー: {e}")

        # フェーズ3: 結果統合
        synthesis_prompt = f"""
タスク: {task}

実行結果:
{chr(10).join(results)}

上記の実行結果を統合して、タスクへの最終回答を作成してください。
"""
        final = self.client.messages.create(
            model=self.model,
            max_tokens=1024,
            messages=[{"role": "user", "content": synthesis_prompt}]
        )
        return final.content[0].text

    def _create_plan(self, task: str) -> dict:
        prompt = self.PLAN_PROMPT.format(
            task=task,
            tools=list(TOOLS.keys())
        )
        response = self.client.messages.create(
            model=self.model,
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        text = response.content[0].text
        if "```json" in text:
            text = text.split("```json")[1].split("```")[0]
        elif "```" in text:
            text = text.split("```")[1].split("```")[0]
        try:
            return json.loads(text.strip())
        except json.JSONDecodeError:
            return {"plan": [], "reasoning": "計画の解析に失敗"}


# ─────────────────────────────────────────────
# 3. Reflexion（自己反省型）
# ─────────────────────────────────────────────

@dataclass
class ReflexionMemory:
    """Reflexion の口頭強化学習メモリ"""
    task: str
    attempts: list[dict] = field(default_factory=list)
    reflections: list[str] = field(default_factory=list)

    def add_attempt(self, result: str, score: float, reflection: str):
        self.attempts.append({"result": result, "score": score})
        self.reflections.append(reflection)

    def get_context(self) -> str:
        if not self.reflections:
            return ""
        recent = self.reflections[-3:]  # 直近3回の反省
        return "過去の試行からの学び:\n" + "\n".join(f"- {r}" for r in recent)


class ReflexionAgent:
    """
    Reflexion: Language Agents with Verbal Reinforcement Learning (Shinn et al., 2023)

    失敗した試行を言語的に反省し、次の試行に活かす。
    評価関数が必要（スコアリング）。
    """

    REFLECT_PROMPT = """あなたはタスク実行の振り返りを行うエージェントです。

タスク: {task}
試行結果: {result}
成功基準: {criteria}

この試行の問題点を分析し、次回どう改善するか1〜3文で述べてください。
"""

    def __init__(
        self,
        client: anthropic.Anthropic,
        evaluator: Callable[[str, str], float],  # (task, result) -> 0.0~1.0
        model: str = "claude-opus-4-6",
        max_attempts: int = 3,
        success_threshold: float = 0.8,
    ):
        self.client = client
        self.model = model
        self.evaluator = evaluator
        self.max_attempts = max_attempts
        self.threshold = success_threshold

    def run(self, task: str, criteria: str = "") -> str:
        memory = ReflexionMemory(task=task)
        base_agent = BaseReActAgent(self.client, self.model)

        for attempt in range(self.max_attempts):
            print(f"  [Reflexion] 試行 {attempt + 1}/{self.max_attempts}")

            # 過去の反省をプロンプトに注入
            context = memory.get_context()
            full_task = f"{context}\n\n{task}" if context else task

            result = base_agent.run(full_task)
            score = self.evaluator(task, result)

            print(f"  [Reflexion] スコア: {score:.2f}")

            if score >= self.threshold:
                print("  [Reflexion] 成功基準を達成！")
                return result

            # 反省フェーズ
            reflection = self._reflect(task, result, criteria)
            memory.add_attempt(result, score, reflection)
            print(f"  [Reflexion] 反省: {reflection}")

        # 最高スコアの試行を返す
        best_idx = max(range(len(memory.attempts)),
                      key=lambda i: memory.attempts[i]["score"])
        return memory.attempts[best_idx]["result"]

    def _reflect(self, task: str, result: str, criteria: str) -> str:
        prompt = self.REFLECT_PROMPT.format(
            task=task, result=result[:500], criteria=criteria
        )
        resp = self.client.messages.create(
            model=self.model,
            max_tokens=256,
            messages=[{"role": "user", "content": prompt}]
        )
        return resp.content[0].text.strip()


# ─────────────────────────────────────────────
# 4. ReWOO（並列化対応型）
# ─────────────────────────────────────────────

class ReWOOAgent:
    """
    ReWOO: Decoupling Reasoning from Observations for Efficient Augmented Language Models

    計画フェーズでツール呼び出しを全て決定し、並列実行する。
    依存関係がないツール呼び出しを非同期で並列化。
    """

    def run(self, task: str) -> str:
        return asyncio.run(self._async_run(task))

    async def _async_run(self, task: str) -> str:
        # ツール呼び出しを並列実行
        tasks = [
            asyncio.create_task(self._async_tool("search_web", {"query": task})),
            asyncio.create_task(self._async_tool("calculate", {"expression": "42 * 2"})),
        ]
        results = await asyncio.gather(*tasks)
        return f"並列実行結果: {results}"

    async def _async_tool(self, name: str, args: dict) -> str:
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(None, lambda: TOOLS[name](**args))


# ─────────────────────────────────────────────
# 使い分け判断関数
# ─────────────────────────────────────────────

def select_agent_architecture(
    task: str,
    requires_retry: bool = False,
    has_parallel_tools: bool = False,
    step_count_estimate: int = 5,
) -> str:
    """
    タスク特性からエージェントアーキテクチャを選択する

    Returns:
        "BaseReAct" | "PlanAct" | "Reflexion" | "ReWOO"
    """
    if has_parallel_tools and step_count_estimate > 3:
        return "ReWOO"  # 並列化でスループット向上
    if requires_retry:
        return "Reflexion"  # 品質が重要でやり直しOK
    if step_count_estimate > 5:
        return "PlanAct"  # 複雑なタスクは先に計画
    return "BaseReAct"  # シンプルなタスク


if __name__ == "__main__":
    # 動作確認（APIキー不要のツールのみ使用）
    print("ツール単体テスト:")
    print(calculate(expression="100 * 3.14"))
    print(search_web(query="Python 2024"))

    arch = select_agent_architecture(
        task="記事を5本書く",
        step_count_estimate=15,
    )
    print(f"\n推奨アーキテクチャ: {arch}")
```

---

## 8.2 Tool use（Function Calling）完全版：スキーマ設計からエラー伝達まで

### Anthropic の tool_use と従来のテキスト解析の違い

多くの「エージェント入門」コードはテキストをパースしてツールを呼び出します。しかし本番ではこれが崩壊します。理由は**LLMの出力フォーマットは保証されない**から。

Anthropic の tool_use（Function Calling）は、Claude がJSON以外の形式で引数を返すことを**構造的に防ぎます**。

```python
# ファイル: agents/tool_use_complete.py

import json
import time
import traceback
from typing import Any, Optional
from dataclasses import dataclass
import anthropic


# ─────────────────────────────────────────────
# スキーマ設計：良い例 vs 悪い例
# ─────────────────────────────────────────────

# ❌ 悪いスキーマ設計（説明が曖昧、型が緩い）
BAD_SCHEMA = {
    "name": "get_data",
    "description": "データを取得する",  # 何のデータ？どんな形式？
    "input_schema": {
        "type": "object",
        "properties": {
            "input": {"type": "string"},  # 何を入れる？
        }
    }
}

# ✅ 良いスキーマ設計（Claude が迷わない詳細な説明）
GOOD_SCHEMAS = [
    {
        "name": "search_articles",
        "description": (
            "キーワードでnote記事を検索する。"
            "検索結果は最大 limit 件のJSONリストで返る。"
            "存在しない場合は空リスト。"
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "keyword": {
                    "type": "string",
                    "description": "検索キーワード（日本語可）。例: 'Python 非同期処理'"
                },
                "limit": {
                    "type": "integer",
                    "description": "取得件数（1〜20）。デフォルト5",
                    "minimum": 1,
                    "maximum": 20,
                    "default": 5
                }
            },
            "required": ["keyword"]
        }
    },
    {
        "name": "create_article",
        "description": (
            "note記事の下書きを作成してファイルに保存する。"
            "成功時はファイルパスを返す。"
            "失敗時は error フィールドにエラー理由を含む JSON を返す。"
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "title": {
                    "type": "string",
                    "description": "記事タイトル（80文字以内推奨）",
                    "maxLength": 100
                },
                "content": {
                    "type": "string",
                    "description": "記事本文（Markdown形式）"
                },
                "category": {
                    "type": "string",
                    "enum": ["tech", "business", "life", "culture"],
                    "description": "カテゴリ"
                }
            },
            "required": ["title", "content", "category"]
        }
    }
]


# ─────────────────────────────────────────────
# Tool Use ループの完全実装
# ─────────────────────────────────────────────

class ToolUseAgent:
    """
    Anthropic tool_use を使った完全な実装。

    重要な設計原則:
    1. ツールエラーは例外でなくJSONで返す（Claudeが理解できる）
    2. tool_use ブロックは全て処理してから次のターンに進む
    3. stop_reason が "tool_use" でないなら終了
    """

    def __init__(self, client: anthropic.Anthropic, model: str = "claude-opus-4-6"):
        self.client = client
        self.model = model
        self.call_log: list[dict] = []

    def run(self, user_message: str, max_turns: int = 10) -> str:
        messages = [{"role": "user", "content": user_message}]

        for turn in range(max_turns):
            response = self.client.messages.create(
                model=self.model,
                max_tokens=4096,
                tools=GOOD_SCHEMAS,
                messages=messages,
            )

            messages.append({
                "role": "assistant",
                "content": response.content
            })

            # ツール使用がなければ終了
            if response.stop_reason != "tool_use":
                return "".join(
                    block.text
                    for block in response.content
                    if hasattr(block, "text")
                )

            # ツール呼び出しを全て実行
            tool_results = []
            for block in response.content:
                if block.type != "tool_use":
                    continue

                print(f"  🔧 ツール呼び出し: {block.name}")

                start = time.time()
                try:
                    # ダミー実装（実際はTOOL_REGISTRYから関数を取得）
                    result = {"status": "ok", "tool": block.name, "input": block.input}
                except Exception as e:
                    result = {
                        "error": "TOOL_EXECUTION_FAILED",
                        "message": str(e),
                    }

                elapsed = (time.time() - start) * 1000
                self.call_log.append({
                    "tool": block.name,
                    "input": block.input,
                    "result": result,
                    "elapsed_ms": elapsed
                })

                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": json.dumps(result, ensure_ascii=False),
                })

            messages.append({
                "role": "user",
                "content": tool_results,
            })

        return "最大ターン数に達しました"
```

:::message alert
**なぜツールエラーを例外でなくJSONで返すか**

Pythonの例外をそのまま投げると、Claudeはそのエラーを「見られない」。
ツール実行結果は必ず `tool_result` メッセージとして返す必要がある。

```python
# ❌ アンチパターン：例外を投げる
def my_tool(path: str) -> str:
    if not os.path.exists(path):
        raise FileNotFoundError(path)  # Claudeには伝わらない

# ✅ 正しい：エラーをJSONで返す
def my_tool(path: str) -> dict:
    if not os.path.exists(path):
        return {"error": "FILE_NOT_FOUND", "path": path, "suggestion": "別のパスを試してください"}
```
:::

---

## まとめ

### 各パターンの使い分けガイド

| パターン | 適用場面 | 強み | 弱み |
|---------|---------|------|------|
| **ReAct** | 汎用タスク・対話的処理 | シンプル・デバッグしやすい | 並列化不可 |
| **PlanAct** | 長期タスク・段階的処理 | 計画の見直しが可能 | プランが変わりにくい |
| **Reflexion** | 品質が重要なタスク | 自己改善ループ | コスト高 |
| **ReWOO** | 並列化可能な多数ツール | 高スループット | 実装複雑 |

### ツール設計の黄金律

```
1. 単一責任 — 1ツール1機能
2. 冪等性   — 同じ入力は常に同じ結果
3. 型安全   — input_schema を必ず定義
4. エラー明示 — エラーは構造化JSONで返す（例外を投げない）
5. ログ記録  — 全呼び出しをcall_logに記録
```

### notecreatorでの活用

notecreatorの8ステージパイプラインは本章のパターンを組み合わせています：

- **トレンド収集** → PlanActの事前計画フェーズ
- **記事生成** → ReAct（品質検査ループ）
- **SEO最適化** → 単一責任ツール
- **Discord承認** → Human-in-the-loop（Reflexionの人間版）

:::message
**次のステップ**

次の章「Claude API完全実装ガイド」では、このアーキテクチャを支えるAPI実装の詳細——ストリーミング・ツール使用・コンテキスト管理——を本番品質で実装します。
:::
