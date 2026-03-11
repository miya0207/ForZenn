---
title: "付録B: AI Agent Design Checklist（設計チェックリスト20項目）"
free: false
---

# 付録B: AI Agent Design Checklist（設計チェックリスト20項目）

本チェックリストは、AIエージェントを本番環境に投入する前に確認すべき20項目をまとめたものです。各項目には「なぜ重要か」「OK例」「NG例」を添えています。設計レビューや実装後のセルフチェックにご活用ください。

---

## 使い方

```
□ 設計フェーズ（要件定義 + アーキテクチャ設計）で使用
□ 実装フェーズ（ツール設計）で使用
□ テスト・本番投入前（信頼性・コスト）で使用
```

チェックが通らない項目は **実装を止めて設計に戻る** ことを強く推奨します。「動いてから直す」AIエージェントは、本番で数万円のAPI費用を溶かしてから問題が発覚することが多いためです。

---

## カテゴリ1: 要件定義（5項目）

*解くべき問題を正確に定義できているか*

---

- [ ] **エージェントが解く問題を「入力→処理→出力」の形式で1文で言語化できている**

  > **なぜ重要か**: AIエージェントの失敗の多くは「何をさせたいか」が曖昧なまま実装を始めることに起因する。問題が1文で言えない場合、LLMへのプロンプトも必然的に曖昧になり、ハルシネーションや意図しない行動を引き起こす。
  
  > **OK例**:
  > ```
  > 問題定義書（例: note記事生成エージェント）
  > ─────────────────────────────────────
  > 入力: ユーザーが指定したキーワード（例: "Python非同期処理"）
  > 処理: Web検索→情報収集→構成案生成→本文執筆→品質チェック
  > 出力: Markdown形式の記事（2000〜3000文字、コードブロック付き）
  > 成功条件: 可読性スコア80以上、コードが実行可能、誤情報なし
  > ```
  
  > **NG例**:
  > ```
  > 問題定義書（悪い例）
  > ─────────────────────
  > 「いい感じにコンテンツを自動生成するエージェント」
  > → 「いい感じ」の定義がない
  > → 成功/失敗の判定基準がない
  > → ツール選択の根拠が持てない
  > ```

---

- [ ] **「エージェントが必要な理由」を、スクリプト・RAG・プロンプトエンジニアリングと比較して説明できている**

  > **なぜ重要か**: エージェントは複雑性・コスト・デバッグ難易度が格段に高い。単純な処理ならば通常のPythonスクリプトやRAGのほうが高速・安価・信頼性が高い。過剰なエージェント化はシステムを不必要に複雑にする典型的なアンチパターンである。
  
  > **OK例**:
  > ```python
  > # 判断マトリクス（コード中のコメントとして記録する）
  > 
  > # ✅ エージェントを選ぶ条件:
  > #   - 処理ステップ数が実行時まで不明（動的計画）
  > #   - 外部ツール（API・DB・ブラウザ）の呼び出しが必要
  > #   - 中間結果に基づいて次のアクションを変える必要がある
  > 
  > # ❌ スクリプトで十分な条件:
  > #   - 処理ステップが固定（ETLパイプラインなど）
  > #   - 入力→出力の変換のみで判断不要
  > #   - レスポンスタイム < 1秒が求められる
  > ```
  
  > **NG例**:
  > ```python
  > # 「AIエージェントで書けば将来的に拡張できるから」という理由だけで
  > # 単純なCSV変換処理をエージェント化した例
  > 
  > # 実際のコスト:
  > #   スクリプト: 0円/回、0.01秒
  > #   エージェント: 約0.02ドル/回、3〜10秒
  > # → 月10万回処理なら約2,000ドルの無駄コスト
  > ```

---

- [ ] **Human-in-the-Loop（人間の介入）が必要なポイントを明示している**

  > **なぜ重要か**: すべてを自動化すると、エージェントが誤った判断をした際の被害が拡大する。特に「外部への書き込み」「金銭的な処理」「ユーザーへの直接配信」を含むフローは、最低1か所の人間確認ポイントが必要である。
  
  > **OK例**:
  > ```python
  > from enum import Enum
  > from dataclasses import dataclass
  > 
  > class ReviewRequired(Enum):
  >     """人間レビューが必要なアクションを列挙"""
  >     PUBLISH_TO_PRODUCTION = "本番公開"
  >     SEND_EMAIL_TO_USER    = "ユーザーへのメール送信"
  >     DELETE_RECORDS        = "レコード削除"
  >     EXCEED_BUDGET_LIMIT   = "予算上限超過"
  > 
  > @dataclass
  > class AgentAction:
  >     action_type: str
  >     requires_human_review: bool
  >     review_reason: str | None = None
  > 
  >     def execute(self) -> None:
  >         if self.requires_human_review:
  >             # Slackへの承認依頼、メール通知などを実装
  >             raise HumanReviewRequired(self.review_reason)
  >         self._do_execute()
  > ```
  
  > **NG例**:
  > ```python
  > # 全自動で本番DBに書き込むエージェント（最悪のパターン）
  > def agent_loop(user_request: str) -> None:
  >     plan = llm.generate_plan(user_request)
  >     for step in plan.steps:
  >         step.execute()  # ←承認なしに本番DBへのDELETEも実行される
  > ```

---

- [ ] **エージェントの「失敗モード」を3パターン以上洗い出している**

  > **なぜ重要か**: AIエージェントは従来のソフトウェアと異なり、「決定論的に失敗する」のではなく「確率的・創造的に失敗する」。想定外の失敗モードへの備えがないと、本番でのインシデント対応が著しく困難になる。
  
  > **OK例**:
  > ```markdown
  > ## 失敗モードカタログ（note記事生成エージェントの例）
  > 
  > | # | 失敗モード | 発生確率 | 影響度 | 検知方法 | 対策 |
  > |---|-----------|---------|-------|---------|-----|
  > | 1 | LLMが架空の引用を生成 | 中 | 高 | 引用URLの疎通確認 | Web検索で一次情報を検証 |
  > | 2 | ツール呼び出しの無限ループ | 低 | 高 | ステップ数カウンター | max_steps=20で強制終了 |
  > | 3 | コンテキスト長超過 | 中 | 中 | トークン数監視 | チャンク分割処理 |
  > | 4 | 外部API(検索)のタイムアウト | 中 | 中 | タイムアウト検知 | キャッシュ+フォールバック |
  > | 5 | 有害コンテンツの生成 | 低 | 極高 | コンテンツフィルター | 出力の事後検証 |
  > ```
  
  > **NG例**:
  > ```markdown
  > ## リスク
  > - APIが落ちるかも → エラーハンドリングする
  > （具体性がなく、実装に繋がらない）
  > ```

---

- [ ] **非機能要件（レイテンシ・コスト・可用性）の目標値を数値で定義している**

  > **なぜ重要か**: 「速くて安いエージェント」という定性的な要件では、実装後に「これで十分か」を判断できない。本番投入前に数値目標がないと、パフォーマンスチューニングの優先度判断が主観的になる。
  
  > **OK例**:
  > ```python
  > # config/sla.py（SLA定義ファイル）
  > from dataclasses import dataclass
  > 
  > @dataclass(frozen=True)
  > class AgentSLA:
  >     """エージェントのSLA定義"""
  >     # レイテンシ
  >     p50_latency_sec: float = 8.0    # 中央値8秒以内
  >     p95_latency_sec: float = 30.0   # 95パーセンタイル30秒以内
  >     timeout_sec: float = 60.0       # タイムアウト60秒
  > 
  >     # コスト
  >     max_cost_per_request_usd: float = 0.05   # 1リクエスト5セント以内
  >     monthly_budget_usd: float = 500.0        # 月500ドル上限
  > 
  >     # 可用性
  >     success_rate_target: float = 0.95        # 成功率95%以上
  >     max_consecutive_failures: int = 3        # 連続失敗3回でアラート
  > 
  > AGENT_SLA = AgentSLA()
  > ```
  
  > **NG例**:
  > ```python
  > # 数値目標なし・要件なし
  > # 「できるだけ速く、できるだけ安く動かす」
  > # → 何をチューニングすれば「完成」かわからない
  > # → コスト爆発に気づくのが請求書が来た翌月になる
  > ```

---

## カテゴリ2: アーキテクチャ設計（5項目）

*エージェント・ツール・メモリの構造が適切か*

---

- [ ] **エージェントのタイプ（ReAct / Plan-and-Execute / Multi-Agent）を明示的に選択し、その理由を記録している**

  > **なぜ重要か**: エージェントアーキテクチャの選択は、レイテンシ・コスト・信頼性のトレードオフに直結する。「とりあえずReAct」という選択は、複雑なタスクでは推論の発散を招き、「とりあえずMulti-Agent」は通信オーバーヘッドで単純タスクに過剰な複雑性をもたらす。
  
  > **OK例**:
  > ```python
  > # architecture_decision.py
  > """
  > アーキテクチャ決定記録 (ADR: Architecture Decision Record)
  > 
  > 決定: Plan-and-Execute パターンを採用
  > 
  > 理由:
  >   - タスク（記事生成）のステップ数は事前に予測可能
  >   - 各ステップが独立しており、並列実行が可能
  >   - ReActは自由度が高すぎてQA検証が困難
  > 
  > トレードオフ:
  >   + 各ステップをユニットテストしやすい
  >   + コスト予測が立てやすい（ステップ数 × 平均コスト）
  >   - 計画フェーズで失敗すると全体が崩れる
  >   - 動的な計画変更が苦手
  > 
  > 代替案と却下理由:
  >   - ReAct: ステップ数が不定でコスト予測困難
  >   - Multi-Agent: 現時点でオーバーエンジニアリング
  > """
  > ```
  
  > **NG例**:
  > ```python
  > # 理由なき選択
  > # 「LangChainのAgentExecutorがデフォルトでReActだったから」
  > # という理由だけで採用し、後からコスト爆発に気づく
  > ```

---

- [ ] **メモリの種類（短期・長期・エピソード記憶）を設計図に書き分けている**

  > **なぜ重要か**: メモリ設計はエージェントの「賢さ」と「コスト」の最大の決定要因である。すべてをコンテキストに詰め込む設計は、トークンコストを指数関数的に増加させ、コンテキスト長の上限で突然クラッシュする。
  
  > **OK例**:
  > ```python
  > from abc import ABC, abstractmethod
  > from typing import Any
  > import json
  > import redis
  > 
  > class ShortTermMemory:
  >     """現在の会話・タスクセッション内のメモリ（コンテキストウィンドウ）"""
  >     def __init__(self, max_tokens: int = 4096):
  >         self._history: list[dict] = []
  >         self.max_tokens = max_tokens
  > 
  >     def add(self, role: str, content: str) -> None:
  >         self._history.append({"role": role, "content": content})
  >         self._trim_to_limit()
  > 
  >     def _trim_to_limit(self) -> None:
  >         """古いメッセージを削除してトークン上限を守る"""
  >         # 簡易実装: 文字数ベースのトリム
  >         while sum(len(m["content"]) for m in self._history) > self.max_tokens * 4:
  >             self._history.pop(0)
  > 
  > class LongTermMemory:
  >     """ユーザー設定・過去の成果物（Vector DB / Redis に永続化）"""
  >     def __init__(self, redis_client: redis.Redis):
  >         self._redis = redis_client
  > 
  >     def save(self, key: str, value: Any, ttl_days: int = 30) -> None:
  >         self._redis.setex(
  >             name=f"agent:memory:{key}",
  >             time=ttl_days * 86400,
  >             value=json.dumps(value, ensure_ascii=False)
  >         )
  > 
  >     def load(self, key: str) -> Any | None:
  >         raw = self._redis.get(f"agent:memory:{key}")
  >         return json.loads(raw) if raw else None
  > ```
  
  > **NG例**:
  > ```python
  > # 全履歴をコンテキストに詰め込む（最悪パターン）
  > def build_prompt(history: list[str]) -> str:
  >     # 100回の会話履歴をすべて渡す
  >     # → GPT-4oで1回のリクエストが$0.50を超えることがある
  >     return "\n".join(history) + "\n\nNext action:"
  > ```

---

- [ ] **ツールの依存関係グラフを作成し、循環依存がないことを確認している**

  > **なぜ重要か**: ToolAがToolBを呼び、ToolBがToolAを呼ぶ循環依存は、エージェントの無限ループを引き起こす。LLMはこのような循環を自力で検出できないため、設計段階での可視化が必須である。
  
  > **OK例**:
  > ```python
  > # tool_dependency_checker.py
  > import networkx as nx
  > from typing import NamedTuple
  > 
  > class ToolDependency(NamedTuple):
  >     from_tool: str
  >     to_tool: str
  > 
  > def validate_tool_dependencies(deps: list[ToolDependency]) -> None:
  >     """
  >     ツール依存グラフを検証し、循環依存があれば例外を発生させる
  >     
  >     使用例:
  >     deps = [
  >         ToolDependency("web_search", "content_parser"),
  >         ToolDependency("content_parser", "summarizer"),
  >         ToolDependency("summarizer", "article_writer"),
  >     ]
  >     validate_tool_dependencies(deps)  # OK: 有向非巡回グラフ
  >     """
  >     G = nx.DiGraph()
  >     for dep in deps:
  >         G.add_edge(dep.from_tool, dep.to_tool)
  > 
  >     cycles = list(nx.simple_cycles(G))
  >     if cycles:
  >         raise ValueError(
  >             f"循環依存を検出しました: {cycles}\n"
  >             "設計を見直してください。"
  >         )
  >     print("✅ ツール依存グラフ: 循環依存なし")
  >     print(f"   トポロジカル順序: {list(nx.topological_sort(G))}")
  > ```
  
  > **NG例**:
  > ```
  > 依存関係を文書化せずに実装 →
  > 本番で「なぜかエージェントが止まらない」インシデントが発生し、
  > デバッグに数時間かかる
  > ```

---

- [ ] **プロンプトをコードから分離し、バージョン管理している**

  > **なぜ重要か**: プロンプトはAIエージェントの「ロジック」そのものである。コード中にハードコードされたプロンプトは、A/Bテスト・ロールバック・チームでの共同編集を困難にする。プロンプトの変更がエージェント挙動に与える影響は、通常のコード変更より大きい場合がある。
  
  > **OK例**:
  > ```yaml
  > # prompts/article_writer_v2.yaml
  > version: "2.1.0"
  > updated_at: "2024-01-15"
  > author: "yamada"
  > changelog: "コードブロックの生成指示を追加"
  > 
  > system: |
  >   あなたは技術記事の専門ライターです。
  >   以下の制約を必ず守ってください:
  >   - 文字数: {min_chars}〜{max_chars}文字
  >   - コードブロックを最低{min_code_blocks}個含める
  >   - 読者レベル: {reader_level}
  > 
  > user_template: |
  >   以下のアウトラインに基づいて記事を執筆してください。
  >   
  >   アウトライン:
  >   {outline}
  >   
  >   参考情報:
  >   {references}
  > ```
  > ```python
  > # プロンプトをロードして使う
  > import yaml
  > from pathlib import Path
  > 
  > def load_prompt(name: str, version: str | None = None) -> dict:
  >     path = Path(f"prompts/{name}.yaml")
  >     prompt = yaml.safe_load(path.read_text(encoding="utf-8"))
  >     if version and prompt["version"] != version:
  >         raise ValueError(f"期待バージョン {version} != 実際 {prompt['version']}")
  >     return prompt
  > ```
  
  > **NG例**:
  > ```python
  > def write_article(outline: str) -> str:
  >     # コード中にベタ書き（レビューもA/Bテストも不可能）
  >     prompt = f"""あなたは記事ライターです。以下を書いてください: {outline}"""
  >     return llm.invoke(prompt)
  > ```

---

- [ ] **エージェントの状態遷移図（State Machine）を設計書に含めている**

  > **なぜ重要か**: エージェントの動作は「状態」の遷移として捉えることで、テスト可能・予測可能なシステムになる。状態遷移を明示しないと、「エージェントが今どのフェーズにいるか」をデバッグ中に追うことが極めて困難