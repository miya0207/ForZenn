---
title: "デモ：30分で動かすClaude Agent"
free: true
---

# デモ：30分で動かすClaude Agent

:::message
**この章で学べること**
- Claude APIを使ったReActエージェントの最小実装を動かせる
- ツール呼び出し（tool_use）の基本的な流れを体感できる
- 「次に何を学ぶべきか」の全体像をつかめる
:::

まず動かしてみよう。理論は後でいい。

このデモを動かすと、Claudeが**自分でツールを選び、実行し、結果を元に考え、最終回答を出す**という流れを目で見ることができる。

---

## セットアップ（5分）

```bash
# 1. 仮想環境を作成
python -m venv venv
source venv/bin/activate  # Windowsは: venv\Scripts\activate

# 2. 依存パッケージをインストール
pip install anthropic

# 3. APIキーを設定
export ANTHROPIC_API_KEY="sk-ant-..."  # Windowsは: set ANTHROPIC_API_KEY=sk-ant-...
```

APIキーはAnthropicのコンソール（https://console.anthropic.com）から取得できる。

---

## 完全なデモコード（agent.py）

以下のコードをそのままコピーして `agent.py` として保存してほしい。

```python
"""
agent.py — 最小のClaude ReActエージェント

動作確認用のデモ。実際のAPIを呼び出してツール使用のループを体感する。
"""

import json
import anthropic

# ─────────────────────────────────────────────
# ツール定義（Claudeに「何ができるか」を教える）
# ─────────────────────────────────────────────

TOOLS = [
    {
        "name": "get_weather",
        "description": "指定した都市の現在の天気情報を取得する。気温・天気・湿度を返す。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "天気を調べる都市名（例: Tokyo, Osaka, Sapporo）"
                }
            },
            "required": ["city"]
        }
    },
    {
        "name": "search_restaurants",
        "description": "天気と場所に基づいてランチ候補を検索する。屋外/屋内の席の有無も返す。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "都市名"},
                "weather": {"type": "string", "description": "現在の天気（例: 晴れ、雨、曇り）"},
                "meal_type": {"type": "string", "description": "食事タイプ（lunch/dinner）"}
            },
            "required": ["city", "weather", "meal_type"]
        }
    }
]

# ─────────────────────────────────────────────
# ツール実行（本番では実際のAPIを叩く）
# ─────────────────────────────────────────────

def execute_tool(name: str, inputs: dict) -> str:
    """ツールを実行して結果を文字列で返す"""

    if name == "get_weather":
        # 本番: requests.get("https://api.openweathermap.org/...")
        result = {
            "city": inputs["city"],
            "temperature_celsius": 22,
            "condition": "晴れ",
            "humidity_percent": 58,
            "feels_like": "快適"
        }
        return json.dumps(result, ensure_ascii=False)

    elif name == "search_restaurants":
        # 本番: requests.get("https://api.hotpepper.jp/...")
        result = {
            "recommendations": [
                {
                    "name": "テラス和食 銀座",
                    "type": "和食",
                    "seating": "テラス席あり（晴天向き）",
                    "price_range": "¥1,200〜1,800",
                    "rating": 4.5,
                    "reason": "晴天のランチにぴったり。テラス席が気持ちいい"
                },
                {
                    "name": "路地裏イタリアン Trattoria",
                    "type": "イタリアン",
                    "seating": "室内席のみ",
                    "price_range": "¥900〜1,500",
                    "rating": 4.2,
                    "reason": "コスパ最高のランチセット。女性に人気"
                }
            ]
        }
        return json.dumps(result, ensure_ascii=False)

    return json.dumps({"error": f"Unknown tool: {name}"})


# ─────────────────────────────────────────────
# ReActループ（エージェントの核心）
# ─────────────────────────────────────────────

def run_agent(user_query: str, max_iterations: int = 5) -> str:
    """
    ReActループを実行する。

    Thought（推論） → Action（ツール呼び出し） → Observation（結果確認）
    このサイクルをClaude自身が繰り返し、最終回答を出す。
    """
    client = anthropic.Anthropic()
    messages = [{"role": "user", "content": user_query}]

    print(f"\n{'='*55}")
    print(f"🎯 タスク: {user_query}")
    print(f"{'='*55}")

    for step in range(1, max_iterations + 1):
        print(f"\n[ステップ {step}] Claudeが考えています...")

        # LLMに次のアクションを推論させる
        response = client.messages.create(
            model="claude-haiku-4-5",   # デモなのでHaikuで十分
            max_tokens=1024,
            tools=TOOLS,
            messages=messages,
            system=(
                "あなたは与えられたツールを使って問題を解決するAIエージェントです。"
                "必要な情報が揃ったら、ツールを使わずに最終回答を日本語で述べてください。"
            )
        )

        # 停止理由を確認
        if response.stop_reason == "end_turn":
            # Claudeが「これ以上ツールは不要」と判断した
            final_answer = next(
                (block.text for block in response.content if hasattr(block, "text")),
                "（回答なし）"
            )
            print(f"\n✅ 最終回答:\n{final_answer}")
            return final_answer

        if response.stop_reason == "tool_use":
            # Claudeがツールを呼び出した
            tool_calls = [b for b in response.content if b.type == "tool_use"]

            # アシスタントの応答を履歴に追加
            messages.append({"role": "assistant", "content": response.content})

            # ツールを実行してObservationを収集
            tool_results = []
            for tool_call in tool_calls:
                print(f"  🔧 ツール呼び出し: {tool_call.name}")
                print(f"     入力: {json.dumps(tool_call.input, ensure_ascii=False)}")

                result = execute_tool(tool_call.name, tool_call.input)
                print(f"     結果: {result[:100]}...")

                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": result
                })

            # ツール結果を履歴に追加してループ継続
            messages.append({"role": "user", "content": tool_results})

        else:
            # 予期しない停止理由
            print(f"  ⚠️  予期しない停止理由: {response.stop_reason}")
            break

    return "最大ステップ数に達しました"


# ─────────────────────────────────────────────
# エントリーポイント
# ─────────────────────────────────────────────

if __name__ == "__main__":
    result = run_agent("東京の今日の天気を調べて、それに合ったランチを2つ提案してください")

    print(f"\n{'='*55}")
    print("実行完了")
```

---

## 実行してみよう

```bash
python agent.py
```

こんな出力が表示されるはずだ：

```
=======================================================
🎯 タスク: 東京の今日の天気を調べて、それに合ったランチを2つ提案してください
=======================================================

[ステップ 1] Claudeが考えています...
  🔧 ツール呼び出し: get_weather
     入力: {"city": "Tokyo"}
     結果: {"city": "Tokyo", "temperature_celsius": 22, "condition": "晴れ"...

[ステップ 2] Claudeが考えています...
  🔧 ツール呼び出し: search_restaurants
     入力: {"city": "Tokyo", "weather": "晴れ", "meal_type": "lunch"}
     結果: {"recommendations": [{"name": "テラス和食 銀座"...

[ステップ 3] Claudeが考えています...

✅ 最終回答:
東京は現在22度の快適な晴天です。テラス席で気持ちよくランチを楽しめる絶好の日ですね。

おすすめの2店舗をご紹介します：

1. **テラス和食 銀座**（★4.5）
   晴天のテラス席でゆっくり和食ランチを楽しめます。¥1,200〜1,800。

2. **路地裏イタリアン Trattoria**（★4.2）
   コスパ抜群のランチセットが人気。¥900〜1,500で楽しめます。

=======================================================
実行完了
```

---

## 何が起きたのか

```
ユーザーの質問
      │
      ▼
 [ステップ1] Claudeが推論
    「天気情報が必要 → get_weather を呼ぼう」
      │
      ▼ ツール実行
 [ステップ2] 天気情報を受け取ってClaudeが推論
    「晴れとわかった → search_restaurants を呼ぼう」
      │
      ▼ ツール実行
 [ステップ3] レストラン情報を受け取ってClaudeが推論
    「全情報が揃った → 最終回答を出そう」
      │
      ▼
   最終回答
```

これが**ReActループ**だ。Reasoning（推論）とActing（行動）を交互に繰り返す。

単純なAPIコールとの決定的な違いは、**Claudeが自分で計画を立てて実行する**点だ。

---

## このデモには何が足りないか

このデモは意図的にシンプルに作った。本番で使えるエージェントには、これが必要になる：

| 欠けているもの | 本番での問題 | 本書で学ぶ章 |
|--------------|------------|------------|
| エラーハンドリング | ツールが失敗したとき即クラッシュ | ツールシステム設計パターン |
| リトライ処理 | APIが429を返すと止まる | 本番信頼性設計 |
| コスト管理 | いくら使ったか不明 | コスト最適化戦略 |
| メモリ | セッションをまたいで学習できない | メモリシステムアーキテクチャ |
| 品質検査 | 出力が正しいか確認できない | 評価・テスト・モニタリング |
| ロギング | 何が起きたか後から追えない | AIエージェントデバッグガイド |

本書の残りのチャプターは、この表の「本番での問題」をひとつずつ解決する。

---

:::message
**次のステップ**

次のチャプター「なぜエージェントが必要か」では、このデモを理論的に理解する。

ReAct論文（Yao et al. 2022）が示した実験結果と、単純なAPIコールがなぜ本番で機能しないかを解説する。
:::
