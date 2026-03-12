---
title: "ツールシステム設計パターン：ファイル操作・Web検索・DB操作"
free: false
---

# ツールシステム設計パターン：ファイル操作・Web検索・DB操作

:::message
**この章で学べること**
- ツール設計3原則（単一責任・冪等性・型安全）を実際のコードで実装できる
- パストラバーサル攻撃対策を含む安全なファイルシステムツールを実装できる
- ToolExecutorによるエラー情報のClaude連携設計を理解できる
:::

---

## はじめに（なぜこれが重要か）

2023年にMeta AIが発表した論文「Toolformer: Language Models Can Teach Themselves to Use Tools」は、AIエージェント開発の世界を大きく変えました。Toolformerの核心的な発見は、「LLMは外部ツールの**使い方を自分で学習できる**」という事実です。

しかし、現場エンジニアにとってより重要な教訓は別にあります。

> 「ツールの設計品質が、エージェント全体の信頼性を決定する」

Toolformerの実験では、ツール自体の品質が低いと、モデルがどれだけ賢くてもエラーが伝播して最終出力が崩壊することが示されています。これは現場でも毎日起きている現象です。

:::message alert
**本番の落とし穴：ツールエラーで無限ループ**

Web検索ツールが404を返し続けた場合、エージェントはどうするか？
エラーハンドリングが甘いと「リトライ→404→リトライ→404...」の無限ループに入り、30分で$50のAPIコストが消える。

**ツール設計の良し悪しが、エージェントのコスト効率を決定する。**
:::

---

## 4.1 ツール設計の3原則：単一責任・冪等性・型安全

### ツール呼び出しのフロー

```
Claude → ツール呼び出し → 結果取得 → 次の判断 → ...
          ↑
     このツールが壊れると全体が崩れる
```

### 原則1：単一責任（Single Responsibility）

```python
# ❌ 悪い例：1つのツールに複数の責任を持たせる
def search_and_save_and_analyze(
    query: str,
    filename: str,
    analysis_type: str
) -> dict:
    """検索して保存して分析する（やりすぎ）"""
    results = requests.get(f"https://api.example.com/search?q={query}")
    with open(filename, 'w') as f:
        f.write(results.text)
    return {"saved": filename, "analysis": "..."}
    # 失敗したとき「どのステップが原因か」Claudeが判断できない

# ✅ 良い例：各ツールは1つのことだけを行う
def search_web(query: str, limit: int = 10) -> list[dict]:
    """Web検索のみを担当"""
    ...

def save_to_file(content: str, filepath: str) -> dict:
    """ファイル保存のみを担当"""
    ...

def analyze_content(content: str, analysis_type: str) -> dict:
    """分析のみを担当"""
    ...
```

### 原則2：冪等性（Idempotency）

同じ入力で何度呼ばれても、副作用が重複しないことが重要です。エージェントはリトライするからです。

```python
# ❌ 悪い例：呼ぶたびにファイルが増える
def save_result(content: str, filename: str) -> str:
    timestamp = datetime.now().timestamp()
    path = f"{filename}_{timestamp}.txt"  # 毎回新しいファイル
    with open(path, 'w') as f:
        f.write(content)
    return path

# ✅ 良い例：冪等性を保証する
def save_result(content: str, filename: str, overwrite: bool = True) -> dict:
    """同じfilenameで呼ばれたら上書き（冪等）"""
    path = Path(filename)
    path.parent.mkdir(parents=True, exist_ok=True)  # exist_ok=Trueが重要

    if path.exists() and not overwrite:
        return {"status": "skipped", "path": str(path), "reason": "already_exists"}

    path.write_text(content, encoding='utf-8')
    return {"status": "saved", "path": str(path), "size_bytes": len(content.encode())}
```

### 原則3：型安全（Type Safety）

```python
from typing import Literal
from pydantic import BaseModel, Field, validator

class SearchParams(BaseModel):
    query: str = Field(..., min_length=1, max_length=500)
    limit: int = Field(default=10, ge=1, le=100)
    source: Literal["hackernews", "reddit", "qiita"] = "hackernews"

    @validator('query')
    def sanitize_query(cls, v):
        # SQLインジェクション対策の一例
        forbidden = ["DROP", "DELETE", "INSERT", "--"]
        for word in forbidden:
            if word.upper() in v.upper():
                raise ValueError(f"危険なキーワードが含まれています: {word}")
        return v.strip()

def search_tool(params: dict) -> dict:
    """型安全なツールの入り口"""
    try:
        validated = SearchParams(**params)
    except ValueError as e:
        # 型エラーは明確なメッセージで返す（Claudeが理解できるように）
        return {"error": f"パラメータエラー: {str(e)}", "valid_params": SearchParams.schema()}

    return _do_search(validated)
```

---

## 4.2 ファイルシステムツールの実装と権限制御

ファイル操作ツールは「最も危険なツール」の1つです。エージェントが `../../etc/passwd` を読もうとしても防がなければなりません。

```python
# tools/file_tool.py
"""安全なファイル操作ツール - パストラバーサル攻撃を防ぐ"""

import hashlib
import json
import os
from pathlib import Path
from typing import Optional


# 許可するルートディレクトリ（環境変数で設定）
ALLOWED_ROOT = Path(os.environ.get("AGENT_WORKSPACE", "/tmp/agent_workspace")).resolve()


def _safe_path(relative_path: str) -> Path:
    """
    パストラバーサル攻撃を防ぐ安全なパス解決

    例）relative_path = "../../etc/passwd" → エラー
    例）relative_path = "output/result.txt" → OK
    """
    # 絶対パスへの変換
    target = (ALLOWED_ROOT / relative_path).resolve()

    # ALLOWED_ROOT の外に出ていないか確認
    try:
        target.relative_to(ALLOWED_ROOT)
    except ValueError:
        raise PermissionError(
            f"アクセス拒否: '{relative_path}' はワークスペース外を指しています。"
            f"許可されたディレクトリ: {ALLOWED_ROOT}"
        )

    return target


def read_file(relative_path: str, encoding: str = "utf-8") -> dict:
    """
    ワークスペース内のファイルを読み込む

    Returns:
        {"content": str, "size_bytes": int, "checksum": str}
    """
    try:
        path = _safe_path(relative_path)

        if not path.exists():
            return {"error": f"ファイルが見つかりません: {relative_path}"}

        if not path.is_file():
            return {"error": f"ディレクトリを読み込もうとしました: {relative_path}"}

        # ファイルサイズ制限（10MB）
        size = path.stat().st_size
        if size > 10 * 1024 * 1024:
            return {"error": f"ファイルが大きすぎます: {size:,} bytes (上限: 10MB)"}

        content = path.read_text(encoding=encoding)
        checksum = hashlib.sha256(content.encode()).hexdigest()[:8]

        return {
            "content": content,
            "size_bytes": size,
            "checksum": checksum,
            "path": str(path.relative_to(ALLOWED_ROOT))
        }

    except PermissionError as e:
        return {"error": str(e)}
    except UnicodeDecodeError:
        return {"error": f"バイナリファイルは読み込めません: {relative_path}"}


def write_file(relative_path: str, content: str, mode: str = "overwrite") -> dict:
    """
    ワークスペース内にファイルを書き込む

    Args:
        mode: "overwrite" | "append" | "create_new"（既存ファイルがあればエラー）
    """
    try:
        path = _safe_path(relative_path)
        path.parent.mkdir(parents=True, exist_ok=True)

        if mode == "create_new" and path.exists():
            return {"error": f"ファイルが既に存在します: {relative_path}"}

        if mode == "append":
            path.open("a", encoding="utf-8").write(content)
        else:
            path.write_text(content, encoding="utf-8")

        return {
            "status": "success",
            "path": str(path.relative_to(ALLOWED_ROOT)),
            "size_bytes": len(content.encode("utf-8")),
            "mode": mode
        }

    except PermissionError as e:
        return {"error": str(e)}


def list_files(relative_dir: str = ".", pattern: str = "*") -> dict:
    """ワークスペース内のファイル一覧を取得"""
    try:
        dir_path = _safe_path(relative_dir)

        if not dir_path.is_dir():
            return {"error": f"ディレクトリではありません: {relative_dir}"}

        files = []
        for f in dir_path.glob(pattern):
            if f.is_file():
                stat = f.stat()
                files.append({
                    "name": f.name,
                    "path": str(f.relative_to(ALLOWED_ROOT)),
                    "size_bytes": stat.st_size,
                    "modified": stat.st_mtime
                })

        return {"files": files, "count": len(files), "directory": relative_dir}

    except PermissionError as e:
        return {"error": str(e)}


# Claude用ツール定義
FILE_TOOLS = [
    {
        "name": "read_file",
        "description": "ワークスペース内のテキストファイルを読み込む",
        "input_schema": {
            "type": "object",
            "properties": {
                "relative_path": {
                    "type": "string",
                    "description": "ワークスペースからの相対パス (例: 'output/result.txt')"
                }
            },
            "required": ["relative_path"]
        }
    },
    {
        "name": "write_file",
        "description": "ワークスペース内にテキストファイルを書き込む",
        "input_schema": {
            "type": "object",
            "properties": {
                "relative_path": {"type": "string"},
                "content": {"type": "string"},
                "mode": {
                    "type": "string",
                    "enum": ["overwrite", "append", "create_new"],
                    "default": "overwrite"
                }
            },
            "required": ["relative_path", "content"]
        }
    }
]
```

---

## 4.3 Web検索ツール：スクレイピングvsAPI、レート制限対策

### スクレイピングvsAPIの選択基準

| 観点 | スクレイピング | API |
|------|--------------|-----|
| コスト | 無料（サーバー負荷） | 有料が多い |
| 安定性 | HTML構造変化で壊れる | 比較的安定 |
| レート制限 | 非公式で不明確 | 明示的 |
| **適用場面** | プロトタイプ・内部ツール | 本番サービス |

notecreatorはHackerNews・Reddit・QiitaのAPIを使用しています。これらはすべて**公式API**なので法的にクリアです。

```python
# automation/trends/collector.py

import json
import time
from functools import wraps
from typing import Optional

import requests

REQUEST_TIMEOUT = 15
USER_AGENT = "notecreator-trend-bot/1.0"

# ===== レート制限デコレータ =====

def rate_limited(calls_per_second: float = 1.0):
    """レート制限を適用するデコレータ"""
    min_interval = 1.0 / calls_per_second
    last_called = {}

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            func_name = func.__name__

            if func_name in last_called:
                elapsed = now - last_called[func_name]
                if elapsed < min_interval:
                    sleep_time = min_interval - elapsed
                    print(f"  レート制限: {sleep_time:.2f}秒待機 ({func_name})")
                    time.sleep(sleep_time)

            last_called[func_name] = time.time()
            return func(*args, **kwargs)
        return wrapper
    return decorator


# ===== データソース別フェッチ関数 =====

@rate_limited(calls_per_second=0.5)  # 2秒に1回まで
def fetch_hackernews(limit: int = 30) -> list[str]:
    """HackerNews トップ記事のタイトルを取得"""
    try:
        resp = requests.get(
            "https://hacker-news.firebaseio.com/v0/topstories.json",
            timeout=REQUEST_TIMEOUT,
        )
        resp.raise_for_status()
        story_ids = resp.json()[:limit]

        titles = []
        for sid in story_ids:
            r = requests.get(
                f"https://hacker-news.firebaseio.com/v0/item/{sid}.json",
                timeout=REQUEST_TIMEOUT,
            )
            if r.ok:
                item = r.json()
                if item and item.get("title"):
                    titles.append(item["title"])
            time.sleep(0.1)

        print(f"  HackerNews: {len(titles)}件取得")
        return titles
    except Exception as e:
        print(f"  HackerNews取得エラー: {e}")
        return []


@rate_limited(calls_per_second=0.5)
def fetch_reddit(subreddit: str = "technology", limit: int = 25) -> list[str]:
    """Reddit ホット記事のタイトルを取得"""
    try:
        resp = requests.get(
            f"https://www.reddit.com/r/{subreddit}/hot.json?limit={limit}",
            headers={"User-Agent": USER_AGENT},
            timeout=REQUEST_TIMEOUT,
        )
        resp.raise_for_status()
        data = resp.json()
        titles = [
            post["data"]["title"]
            for post in data.get("data", {}).get("children", [])
            if post.get("data", {}).get("title")
        ]
        print(f"  Reddit r/{subreddit}: {len(titles)}件取得")
        return titles
    except Exception as e:
        print(f"  Reddit取得エラー: {e}")
        return []


@rate_limited(calls_per_second=1.0)
def fetch_qiita(limit: int = 20) -> list[dict]:
    """Qiita トレンド記事のタイトル＋タグを取得"""
    try:
        resp = requests.get(
            "https://qiita.com/api/v2/items",
            params={
                "per_page": limit,
                "query": "tag:AI OR tag:Docker OR tag:Linux OR tag:初心者"
            },
            timeout=REQUEST_TIMEOUT,
        )
        resp.raise_for_status()
        items = resp.json()
        results = [
            {
                "title": item["title"],
                "tags": [t["name"] for t in item.get("tags", [])],
                "likes": item.get("likes_count", 0)
            }
            for item in items
            if item.get("title")
        ]
        print(f"  Qiita: {len(results)}件取得")
        return results
    except Exception as e:
        print(f"  Qiita取得エラー: {e}")
        return []
```

---

## 実務Tips

**ツールのエラーメッセージは詳細にClaudeに伝える**

ツールが失敗した場合、`{"error": "ファイルが見つかりません: /path/to/file"}` のように具体的なエラー情報をClaude側に返すと、Claudeが自動的に修正戦略を考えます。単純な `{"error": "failed"}` では Claudeが次の行動を決定できません。

**レート制限デコレータで外部APIへの負荷を制御**

`@rate_limited(calls_per_second=2)` のようなデコレータを用意し、すべての外部API呼び出しに適用することで、503エラーや IP ブロックを防げます。

---

## アンチパターン

**❌ ツールに複数の責任を持たせる**

`search_and_summarize()` のように「検索してサマリーも返す」ツールは再利用性が下がり、テストも困難になります。`search()` と `summarize()` を分けることで、エージェントが柔軟に組み合わせられます。

**❌ ファイルパスを検証せずに使用**

ユーザー（またはLLM）から受け取ったパスをそのまま `open()` に渡すとパストラバーサル攻撃（`../../etc/passwd`）の危険があります。`ALLOWED_ROOT` によるパス制限を必ず実装してください。

---

## まとめ

| 学習内容 | 実装できるもの |
|---------|--------------|
| 3原則 | 単一責任・冪等性・型安全のツール設計 |
| セキュリティ | パストラバーサル対策・レート制限・タイムアウト設計 |
| エラー伝達 | ToolExecutorがエラーをClaude側に返す設計パターン |
| ツールチェーン | notecreatorの「トレンド収集→記事生成」の連鎖パターン |

:::message
**次のステップ**

次の章「メモリシステムアーキテクチャ」では、エージェントが「覚える」仕組みを実装します。CoALA論文の4層メモリモデルをSQLiteで実装し、会話履歴の圧縮で80%コスト削減を実現します。
:::
