---
title: "付録A: Claude Agent Template（完全実装コード集）"
free: false
---

# 付録A: Claude Agent Template（完全実装コード集）

本付録では、本書で解説してきたClaude Agentの概念を、そのまま本番環境で使えるテンプレートとして提供します。4つのファイルはすべて独立して動作し、組み合わせることで完全なReActエージェントが構築できます。

---

## ファイル構成と依存関係

```
project/
├── agent.py          # メインのReActループ（tool_manager, memory, cost_trackerに依存）
├── tool_manager.py   # ツール登録・実行（依存なし）
├── memory.py         # 4層メモリ（依存なし）
└── cost_tracker.py   # コスト追跡（依存なし）
```

**インストール必須パッケージ:**

```bash
pip install anthropic>=0.25.0 tiktoken>=0.6.0
```

---

## agent.py — ReActループ本体

```python
"""
agent.py — Claude ReActエージェント 本体

【目的】
Anthropic Claude APIを使ったReAct（Reasoning + Acting）ループを実装する。
思考(Thought) → 行動(Action) → 観察(Observation) のサイクルを自動で回し、
ツール呼び出しが不要になった時点で最終回答を返す。

【使い方】
    from agent import ClaudeAgent
    from tool_manager import ToolManager

    manager = ToolManager()
    agent = ClaudeAgent(tool_manager=manager)
    result = agent.run("東京の現在時刻を教えて")
    print(result)

【本番での落とし穴】
    - max_iterations を設定しないと無限ループになる（デフォルト10回）
    - Claude がツール結果を無視して hallucination することがある
      → tool_result の content を必ず検証すること
    - stream=True にすると tool_use ブロックが分割されて届く
      → 本テンプレートは非ストリームで実装（本番移行時は要注意）
"""

import logging
import json
from typing import Any
from datetime import datetime

import anthropic

from tool_manager import ToolManager
from memory import AgentMemory
from cost_tracker import CostTracker

# ロガー設定（呼び出し元でハンドラを設定することを推奨）
logger = logging.getLogger(__name__)


class ClaudeAgent:
    """
    ReActパターンで動作するClaudeエージェント。

    Attributes:
        client: Anthropic APIクライアント
        model: 使用するモデル名
        tool_manager: ツール登録・実行を管理するオブジェクト
        memory: 会話履歴・要約を管理する4層メモリ
        cost_tracker: トークン消費とコストを追跡するオブジェクト
        max_iterations: ReActループの最大反復回数（無限ループ防止）
        system_prompt: エージェントの振る舞いを規定するシステムプロンプト
    """

    DEFAULT_SYSTEM_PROMPT = """あなたは有能なAIアシスタントです。
ユーザーの質問に答えるため、必要に応じてツールを使用してください。
ツールを使う場合は、結果を慎重に確認してから回答を組み立ててください。
不確かな情報は「確認できていない」と正直に伝えてください。"""

    def __init__(
        self,
        tool_manager: ToolManager | None = None,
        memory: AgentMemory | None = None,
        cost_tracker: CostTracker | None = None,
        model: str = "claude-opus-4-5",
        max_iterations: int = 10,
        system_prompt: str | None = None,
    ) -> None:
        self.client = anthropic.Anthropic()
        self.model = model
        self.tool_manager = tool_manager or ToolManager()
        self.memory = memory or AgentMemory()
        self.cost_tracker = cost_tracker or CostTracker()
        self.max_iterations = max_iterations
        self.system_prompt = system_prompt or self.DEFAULT_SYSTEM_PROMPT

    def run(self, user_input: str) -> str:
        """
        ユーザー入力を受け取り、ReActループを回して最終回答を返す。

        Args:
            user_input: ユーザーからの入力テキスト

        Returns:
            エージェントの最終回答テキスト

        Raises:
            RuntimeError: max_iterations を超えてもループが終了しない場合
        """
        logger.info(f"[Agent.run] START | input='{user_input[:50]}...'")

        # ワーキングメモリにユーザー入力を追加
        self.memory.add_message("user", user_input)
        messages = self.memory.get_context_window()

        for iteration in range(1, self.max_iterations + 1):
            logger.debug(f"[Agent.run] iteration={iteration}")

            try:
                response = self.client.messages.create(
                    model=self.model,
                    max_tokens=4096,
                    system=self.system_prompt,
                    tools=self.tool_manager.get_tool_schemas(),
                    messages=messages,
                )
            except anthropic.APIError as e:
                logger.error(f"[Agent.run] API error: {e}")
                raise

            # コスト記録
            self.cost_tracker.record(
                model=self.model,
                input_tokens=response.usage.input_tokens,
                output_tokens=response.usage.output_tokens,
                label=f"iteration_{iteration}",
            )

            # stop_reason が "end_turn" → ツール呼び出し不要、最終回答を返す
            if response.stop_reason == "end_turn":
                final_text = self._extract_text(response)
                self.memory.add_message("assistant", final_text)
                logger.info(f"[Agent.run] DONE in {iteration} iterations")
                return final_text

            # stop_reason が "tool_use" → ツールを実行して結果を返す
            if response.stop_reason == "tool_use":
                # assistantメッセージをそのまま追加（tool_use ブロック含む）
                messages.append({"role": "assistant", "content": response.content})

                # ツール結果をまとめて user メッセージとして追加
                tool_results = self._execute_tools(response.content)
                messages.append({"role": "user", "content": tool_results})
                continue

            # 想定外の stop_reason（安全弁）
            logger.warning(f"[Agent.run] Unexpected stop_reason: {response.stop_reason}")
            break

        raise RuntimeError(
            f"ReActループが {self.max_iterations} 回を超えました。"
            "max_iterations を増やすか、タスクを分割してください。"
        )

    def _extract_text(self, response: anthropic.types.Message) -> str:
        """レスポンスからテキストブロックを抽出する。"""
        texts = [
            block.text
            for block in response.content
            if hasattr(block, "text")
        ]
        return "\n".join(texts)

    def _execute_tools(
        self, content_blocks: list[Any]
    ) -> list[dict[str, Any]]:
        """
        tool_use ブロックをすべて実行し、tool_result リストを返す。

        Returns:
            Anthropic API が期待する tool_result 形式のリスト
        """
        results = []
        for block in content_blocks:
            if block.type != "tool_use":
                continue

            tool_name = block.name
            tool_input = block.input
            tool_use_id = block.id

            logger.info(f"[Agent] Calling tool: {tool_name} | input={tool_input}")

            try:
                output = self.tool_manager.execute(tool_name, tool_input)
                is_error = False
            except Exception as e:
                logger.error(f"[Agent] Tool '{tool_name}' failed: {e}")
                output = f"エラー: {str(e)}"
                is_error = True

            results.append({
                "type": "tool_result",
                "tool_use_id": tool_use_id,
                "content": str(output),
                "is_error": is_error,
            })

        return results


# ============================================================
# 動作確認
# ============================================================
if __name__ == "__main__":
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    )

    # サンプルツール: 現在時刻を返す
    manager = ToolManager()

    @manager.register(
        name="get_current_time",
        description="現在の日時をISO形式で返す",
        input_schema={"type": "object", "properties": {}, "required": []},
    )
    def get_current_time(**kwargs: Any) -> str:
        return datetime.now().isoformat()

    agent = ClaudeAgent(tool_manager=manager, model="claude-haiku-4-5")
    answer = agent.run("今何時ですか？")
    print(f"\n【最終回答】\n{answer}")
```

---

## tool_manager.py — ツール登録・実行管理

```python
"""
tool_manager.py — Claude エージェント用ツール管理モジュール

【目的】
Pythonの関数をClaude APIが解釈できる「ツール」として登録・管理し、
エージェントからの呼び出しを一元的に処理する。

【使い方】
    from tool_manager import ToolManager

    manager = ToolManager()

    @manager.register(
        name="add",
        description="2つの整数を足し算する",
        input_schema={
            "type": "object",
            "properties": {
                "a": {"type": "integer"},
                "b": {"type": "integer"},
            },
            "required": ["a", "b"],
        },
    )
    def add(a: int, b: int, **kwargs) -> int:
        return a + b

    result = manager.execute("add", {"a": 3, "b": 5})
    print(result)  # 8

【設計メモ】
    input_schema は JSON Schema 形式。Claude が引数を生成する際のガイドになる。
    **kwargs を関数シグネチャに入れておくと、Claude が余分な引数を渡しても安全。
"""

import logging
from typing import Any, Callable

logger = logging.getLogger(__name__)


class ToolNotFoundError(Exception):
    """登録されていないツールを呼び出したときの例外。"""


class ToolManager:
    """
    ツールの登録・スキーマ生成・実行を担うクラス。

    Attributes:
        _tools: ツール名 → 関数のマッピング
        _schemas: Claude API に渡すツールスキーマのリスト
    """

    def __init__(self) -> None:
        self._tools: dict[str, Callable[..., Any]] = {}
        self._schemas: list[dict[str, Any]] = []

    def register(
        self,
        name: str,
        description: str,
        input_schema: dict[str, Any],
    ) -> Callable:
        """
        デコレータ形式でツールを登録する。

        Args:
            name: ツールの識別名（Claude が呼び出す際に使う）
            description: ツールの説明（Claude が判断に使う。詳しく書くほど精度が上がる）
            input_schema: 引数の JSON Schema

        Returns:
            デコレータ関数
        """
        def decorator(func: Callable) -> Callable:
            if name in self._tools:
                logger.warning(f"[ToolManager] Tool '{name}' is being overwritten.")
            self._tools[name] = func
            self._schemas.append({
                "name": name,
                "description": description,
                "input_schema": input_schema,
            })
            logger.debug(f"[ToolManager] Registered tool: '{name}'")
            return func
        return decorator

    def execute(self, name: str, tool_input: dict[str, Any]) -> Any:
        """
        ツールを名前で呼び出す。

        Args:
            name: ツール名
            tool_input: Claude が生成した引数 dict

        Returns:
            ツールの実行結果

        Raises:
            ToolNotFoundError: 未登録のツールを呼び出した場合
            Exception: ツール内部でエラーが発生した場合（そのまま再raise）
        """
        if name not in self._tools:
            raise ToolNotFoundError(
                f"Tool '{name}' is not registered. "
                f"Available: {list(self._tools.keys())}"
            )
        logger.info(f"[ToolManager] Executing '{name}' with {tool_input}")
        return self._tools[name](**tool_input)

    def get_tool_schemas(self) -> list[dict[str, Any]]:
        """Claude API の tools パラメータに渡すスキーマ一覧を返す。"""
        return list(self._schemas)

    def list_tools(self) -> list[str]:
        """登録済みツール名の一覧を返す。"""
        return list(self._tools.keys())


# ============================================================
# 動作確認
# ============================================================
if __name__ == "__main__":
    logging.basicConfig(level=logging.DEBUG)

    manager = ToolManager()

    @manager.register(
        name="multiply",
        description="2つの数値を掛け算する",
        input_schema={
            "type": "object",
            "properties": {
                "x": {"type": "number", "description": "1つ目の数値"},
                "y": {"type": "number", "description": "2つ目の数値"},
            },
            "required": ["x", "y"],
        },
    )
    def multiply(x: float, y: float, **kwargs: Any) -> float:
        return x * y

    print("登録ツール:", manager.list_tools())
    print("3 × 7 =", manager.execute("multiply", {"x": 3, "y": 7}))

    try:
        manager.execute("nonexistent", {})
    except ToolNotFoundError as e:
        print(f"期待通りのエラー: {e}")
```

---

## memory.py — 4層メモリ実装

```python
"""
memory.py — Claude エージェント用 4層メモリ実装

【目的】
長い会話でコンテキストウィンドウを溢れさせないために、
4層のメモリ階層で情報を管理する。

    Layer 1: ワーキングメモリ  — 直近N件の会話（最優先）
    Layer 2: エピソードメモリ  — 会話の要約（中期記憶）
    Layer 3: セマンティックメモリ — ユーザーの属性・好み（永続的事実）
    Layer 4: 手続きメモリ      — エージェントの行動パターン（スキル記録）

【使い方】
    from memory import AgentMemory

    mem = AgentMemory(working_memory_size=10)
    mem.add_message("user", "Pythonが好きです")
    mem.add_semantic("language_preference", "Python")
    messages = mem.get_context_window()

【本番での落とし穴】
    要約（エピソードメモリ）は Claude に別途リクエストが必要。
    本実装は要約をプレースホルダーとして保持する構造のみ提供する。
    実際の要約呼び出しは agent.py 側で実装すること。
"""

import logging
from collections import deque
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any

logger = logging.getLogger(__name__)


@dataclass
class Message:
    """会話の1ターンを表すデータクラス。"""
    role: str          # "user" | "assistant"
    content: str
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())


class AgentMemory:
    """
    4層メモリを統合管理するクラス。

    Args:
        working_memory_size: ワーキングメモリに保持する最大メッセージ件数
    """

    def __init__(self, working_memory_size: int = 20) -> None:
        # Layer 1: ワーキングメモリ（キューで古いものを自動削除）
        self._working: deque[Message] = deque(maxlen=working_memory_size)

        # Layer 2: エピソードメモリ（会話要約のリスト）
        self._episodes: list[str] = []

        # Layer 3: セマンティックメモリ（key-value で事実を保存）
        self._semantic: dict[str, Any] = {}

        # Layer 4: 手続きメモリ（成功/失敗したアクションのパターン）
        self._procedural: list[dict[str, Any]] = []

        logger.debug(
            f"[Memory] Initialized with working_memory_size={working_memory_size}"
        )

    # ----------------------------------------------------------------
    # Layer 1: ワーキングメモリ
    # ----------------------------------------------------------------
    def add_message(self, role: str, content: str) -> None:
        """会話メッセージをワーキングメモリに追加する。"""
        if role not in ("user", "assistant"):
            raise ValueError(f"role は 'user' か 'assistant' のみ許可: got '{role}'")
        msg = Message(role=role, content=content)
        self._working.append(msg)
        logger.debug(f"[Memory.L1] Added {role} message ({len(content)} chars)")

    def get_context_window(self) -> list[dict[str, str]]:
        """
        Claude API の messages パラメータ形式で会話履歴を返す。
        エピソードメモリがある場合は先頭に要約を挿入する。
        """
        messages: list[dict[str, str]] = []

        # エピソードメモリを要約として挿入
        if self._episodes:
            summary = "\n".join(f"- {ep}" for ep in self._episodes[-3:])
            messages.append({
                "role": "user",
                "content": f"【過去の会話要約】\n{summary}",
            })
            messages.append({
                "role": "assistant",
                "content": "了解しました。過去の文脈を踏まえて対応します。",
            })

        # ワーキングメモリを追加
        for msg in self._working:
            messages.append({"role": msg.role, "content": msg.content})

        return messages

    def working_memory_count(self) -> int:
        """現在のワーキングメモリのメッセージ件数を返す。"""
        return len(self._working)

    # ----------------------------------------------------------------
    # Layer 2: エピソードメモリ
    # ----------------------------------------------------------------
    def add_episode(self, summary: str) -> None:
        """会話要約をエピソードメモリに追加する。"""
        self._episodes.append(summary)
        logger.info(f"[Memory.L2] Episode added: '{summary[:60]}...'")

    def get_episodes(self) -> list[str]:
        """全エピソード要約を返す。"""
        return list(self._episodes)

    # ----------------------------------------------------------------
    # Layer 3: セマンティックメモリ
    # ----------------------------------------------------------------
    def add_semantic(self, key: str, value: Any) -> None:
        """ユーザーに関する事実・属性を保存する。"""
        self._semantic[key] = value
        logger.debug(f"[Memory.L3] Semantic set: {key}={value}")

    def get_semantic(self, key: str, default: Any = None) -> Any:
        """セマンティックメモリから値を取得する。"""
        return self._semantic.get(key, default)

    def get_all_semantic(self) -> dict[str, Any]:
        """全セマンティック情報を返す。"""
        return dict(self._semantic)

    # ----------------------------------------------------------------
    # Layer 4: 手続きメモリ
    # ----------------------------------------------------------------
    def record_action(
        self,
        tool_name: str,
        success: bool,
        context: str = "",
    ) -> None:
        """ツール呼び出しの成否をパターンとして記録する。"""
        record = {
            "tool": tool_name,
            "success": success,
            "context": context,
            "timestamp": datetime.now().isoformat(),
        }
        self._procedural.append(record)
        logger.debug(f"[Memory.L4] Action recorded: {record}")

    def get_success_rate(self, tool_name: str) -> float:
        """指定ツールの成功率を返す（0.0〜1.0）。"""
        records = [r for r in self._procedural if r["tool"] == tool_name]
        if not records:
            return 0.0
        return sum(1 for r in records if r["success"]) / len(records)

    def clear_working_memory(self) -> None:
        """ワーキングメモリをリセットする（新セッション開始時などに使用）。"""
        self._working.clear()
        logger.info("[Memory.L1] Working memory cleared.")


# ============================================================
# 動作確認
# ============================================================
if __name__ == "__main__":
    logging.basicConfig(level=logging.DEBUG)

    mem = AgentMemory(working_memory_size=5)

    # Layer 1
    mem.add_message("