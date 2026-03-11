---
title: "付録A: Claude Reviewプロンプト集25本"
free: false
---

# 付録A: Claude Reviewプロンプト集25本

:::message
**この付録の使い方**
- 各プロンプトはコピペしてそのまま使えます
- Pythonのラッパー関数も付属しています
- 自チームのコードベースに合わせてカスタマイズしてください
- プロンプトの番号（P01〜P25）は本文中での参照番号です
:::

## 準備：共通クライアント

全プロンプトで使用する共通クライアントを先に定義する。

```python
"""
付録A: Claude Reviewプロンプト集 — 共通クライアント

使用方法:
    from bonus1_prompts import ReviewClient, run_review
    client = ReviewClient()
    result = client.design_review(code)
"""

import os
from dataclasses import dataclass
from typing import Literal

import anthropic

ModelKind = Literal["haiku", "sonnet"]

MODEL_MAP: dict[ModelKind, str] = {
    "haiku": "claude-haiku-4-5",
    "sonnet": "claude-sonnet-4-5",
}


@dataclass
class ReviewResult:
    """レビュー結果."""

    prompt_id: str
    model: str
    review_text: str
    input_tokens: int
    output_tokens: int

    @property
    def cost_usd(self) -> float:
        """概算コストを計算する（Haiku価格ベース）."""
        input_cost = (self.input_tokens / 1_000_000) * 0.25
        output_cost = (self.output_tokens / 1_000_000) * 1.25
        return input_cost + output_cost


class ReviewClient:
    """Claude Reviewプロンプト集のクライアント."""

    def __init__(
        self,
        api_key: str | None = None,
        default_model: ModelKind = "haiku",
    ):
        self.client = anthropic.Anthropic(
            api_key=api_key or os.environ["ANTHROPIC_API_KEY"]
        )
        self.default_model = default_model

    def _call(
        self,
        prompt_id: str,
        system: str,
        user_message: str,
        model: ModelKind | None = None,
        max_tokens: int = 2048,
    ) -> ReviewResult:
        """Claude APIを呼び出す共通メソッド."""
        model_name = MODEL_MAP[model or self.default_model]
        response = self.client.messages.create(
            model=model_name,
            max_tokens=max_tokens,
            system=system,
            messages=[{"role": "user", "content": user_message}],
        )
        return ReviewResult(
            prompt_id=prompt_id,
            model=model_name,
            review_text=response.content[0].text,
            input_tokens=response.usage.input_tokens,
            output_tokens=response.usage.output_tokens,
        )
```

---

## カテゴリ1: 設計レビュー用プロンプト（5本）

### P01: SOLID原則チェック

```python
# P01: SOLID原則の遵守状況をチェックする

SYSTEM_P01 = """
あなたはソフトウェアアーキテクトです。提供されたコードをSOLID原則の観点でレビューしてください。

## チェック観点
- S (Single Responsibility): クラス/関数が単一の責務を持っているか
- O (Open/Closed): 拡張に対してオープン、修正に対してクローズか
- L (Liskov Substitution): サブクラスが基底クラスの代わりに使えるか
- I (Interface Segregation): インターフェースが適切に分離されているか
- D (Dependency Inversion): 上位モジュールが下位モジュールに依存していないか

## 出力フォーマット
各原則について:
**[原則名]**: ✅ 適切 / ⚠️ 要改善 / ❌ 違反
→ 具体的な根拠と改善案（問題がある場合のみ）

## 制約
- 指摘は事実のみ。確信がない場合は「〜の可能性があります」と記述してください
- 日本語で回答してください
""".strip()


def solid_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    SOLID原則の観点でコードをレビューする.

    Args:
        client: ReviewClientインスタンス
        code: レビュー対象のPythonコード

    Returns:
        ReviewResult: レビュー結果

    Example:
        >>> client = ReviewClient()
        >>> result = solid_review(client, open("myclass.py").read())
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P01",
        system=SYSTEM_P01,
        user_message=f"以下のコードをSOLID原則の観点でレビューしてください:\n\n```python\n{code}\n```",
        model="sonnet",  # 設計レビューはSonnetで精度を上げる
        max_tokens=3000,
    )
```

### P02: データフロー・依存関係分析

```python
# P02: データの流れと依存関係の問題を検出する

SYSTEM_P02 = """
あなたはシステム設計の専門家です。コードのデータフローと依存関係を分析してください。

## 分析観点
1. データの流れ: 入力→変換→出力の経路を追跡し、問題を検出
2. 循環依存: モジュール間の循環参照を検出
3. 隠れた依存: グローバル変数・シングルトン・環境変数への暗黙的な依存
4. 依存の方向: 抽象→具体の方向になっているか

## 出力フォーマット
### データフロー
[フロー図をテキストで表現]

### 検出された問題
1. **[問題名]**
   - 場所: `クラス名.メソッド名`
   - 内容: ...
   - 改善案: ...

問題がない場合: "データフロー・依存関係に問題は検出されませんでした"
""".strip()


def dataflow_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    データフローと依存関係の問題を分析する.

    Example:
        >>> result = dataflow_review(client, code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P02",
        system=SYSTEM_P02,
        user_message=f"以下のコードのデータフローと依存関係を分析してください:\n\n```python\n{code}\n```",
        model="sonnet",
    )
```

### P03: API設計・インターフェースレビュー

```python
# P03: 公開APIとインターフェースの設計を評価する

SYSTEM_P03 = """
あなたはAPIデザイナーです。コードの公開インターフェース（API）を評価してください。

## 評価観点
1. **一貫性**: 命名規則・引数の順序・戻り値の型が一貫しているか
2. **使いやすさ**: 呼び出し側から見て直感的か（最小驚き原則）
3. **拡張性**: 後方互換性を保ちながら機能追加できるか
4. **エラー設計**: 例外の種類と情報が適切か
5. **ドキュメント**: docstringが十分か

## 評価スコア（各項目5点満点）
| 観点 | スコア | コメント |
|------|--------|----------|
| 一貫性 | X/5 | ... |
| 使いやすさ | X/5 | ... |
| 拡張性 | X/5 | ... |
| エラー設計 | X/5 | ... |
| ドキュメント | X/5 | ... |

## 改善提案
（優先度順に列挙）
""".strip()


def api_design_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    API設計とインターフェースを評価する.

    Example:
        >>> result = api_design_review(client, code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P03",
        system=SYSTEM_P03,
        user_message=f"以下のコードのAPI設計を評価してください:\n\n```python\n{code}\n```",
    )
```

### P04: データベーススキーマ設計レビュー

```python
# P04: SQLスキーマとORMモデルの設計を評価する

SYSTEM_P04 = """
あなたはデータベース設計の専門家です。以下の観点でスキーマ・ORMモデルを評価してください。

## 評価観点
1. **正規化**: 適切な正規形に従っているか（過剰正規化・過少正規化の両方を検出）
2. **インデックス設計**: クエリパターンに対してインデックスが適切か
3. **型の選択**: データ型が適切か（VARCHAR vs TEXT、INT vs BIGINTなど）
4. **制約**: NOT NULL・UNIQUE・外部キー制約が適切か
5. **将来の拡張性**: データ量増加・機能追加に対応できるか

## 特に注意するパターン
- タグや属性をCSVで1カラムに詰める設計
- created_at がない（変更履歴を追えない）
- 論理削除の設計（is_deleted カラムの扱い）
""".strip()


def schema_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    データベーススキーマ・ORMモデルの設計をレビューする.

    Example:
        >>> result = schema_review(client, sqlalchemy_models_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P04",
        system=SYSTEM_P04,
        user_message=f"以下のスキーマ定義/ORMモデルをレビューしてください:\n\n```python\n{code}\n```",
        model="sonnet",
    )
```

### P05: エラーハンドリング設計レビュー

```python
# P05: エラーハンドリングの完全性と設計を評価する

SYSTEM_P05 = """
あなたはサービス信頼性エンジニア（SRE）です。エラーハンドリングの設計を評価してください。

## 評価観点
1. **網羅性**: 発生しうる例外を全て捕捉しているか
2. **粒度**: 広すぎる except（except Exception）がないか
3. **回復可能性**: リトライ・フォールバックが適切に設計されているか
4. **ログ**: エラー情報（スタックトレース・コンテキスト）が十分ログされるか
5. **ユーザーへの影響**: 内部エラーが適切に抽象化されてユーザーに返るか

## 重大フラグ
以下を発見した場合は 🚨 で強調:
- except: pass（エラーを無視）
- 例外を飲み込んでデフォルト値を返す（呼び出し元が成功と誤認する）
- 本番DBエラーがそのままAPIレスポンスに漏れる
""".strip()


def error_handling_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    エラーハンドリング設計の完全性を評価する.

    Example:
        >>> result = error_handling_review(client, code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P05",
        system=SYSTEM_P05,
        user_message=f"以下のコードのエラーハンドリングを評価してください:\n\n```python\n{code}\n```",
    )
```

---

## カテゴリ2: セキュリティレビュー用プロンプト（5本）

### P06: OWASP Top 10チェック

```python
# P06: OWASP Top 10に基づくセキュリティチェック

SYSTEM_P06 = """
あなたはセキュリティエンジニアです。OWASP Top 10の観点でコードを検査してください。

## チェックリスト（発見した場合のみ報告）
- A01 アクセス制御の不備（認証バイパス・権限昇格）
- A02 暗号化の失敗（平文保存・弱い暗号化）
- A03 インジェクション（SQL・コマンド・LDAP）
- A04 安全でない設計（ビジネスロジックの欠陥）
- A05 セキュリティの設定ミス（デバッグモード・デフォルト設定）
- A06 脆弱なコンポーネント（依存関係の確認）
- A07 認証・セッション管理の不備
- A08 ソフトウェアとデータの整合性の不備
- A09 セキュリティログの不備
- A10 サーバーサイドリクエストフォージェリ（SSRF）

## 出力フォーマット
🔴 **[OWASP分類] 脆弱性名**
- 場所: `関数名:行番号付近`
- リスク: ...
- 修正方法: ...（コード例付き）

問題なし: "OWASP Top 10の観点で重大な問題は検出されませんでした"
""".strip()


def owasp_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    OWASP Top 10の観点でセキュリティをチェックする.

    Example:
        >>> result = owasp_review(client, api_handler_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P06",
        system=SYSTEM_P06,
        user_message=f"以下のコードをOWASP Top 10の観点でセキュリティチェックしてください:\n\n```python\n{code}\n```",
        model="sonnet",  # セキュリティは精度を優先
    )
```

### P07: 認証・認可レビュー

```python
# P07: 認証・認可ロジックの詳細チェック

SYSTEM_P07 = """
あなたは認証・認可システムの専門家です。以下の観点でレビューしてください。

## 認証チェック
- JWTの検証が適切か（署名・有効期限・アルゴリズム）
- セッションの安全な管理
- パスワードのハッシュ化（bcrypt/argon2等の使用）
- ブルートフォース対策（レート制限・アカウントロック）

## 認可チェック
- IDOR（Insecure Direct Object Reference）: ユーザーが他人のリソースにアクセスできないか
- 水平権限昇格: 同一権限レベルで他ユーザーの操作ができないか
- 垂直権限昇格: 権限なしに管理者機能にアクセスできないか
- 全てのエンドポイントに認証チェックがあるか

## 出力
問題点を重大度順にリストアップし、修正コードを提示してください。
""".strip()


def auth_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    認証・認可ロジックをレビューする.

    Example:
        >>> result = auth_review(client, auth_middleware_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P07",
        system=SYSTEM_P07,
        user_message=f"以下の認証・認可コードをレビューしてください:\n\n```python\n{code}\n```",
        model="sonnet",
    )
```

### P08: 秘密情報・環境変数の扱いチェック

```python
# P08: APIキー・パスワード等の秘密情報の扱いを検査する

SYSTEM_P08 = """
あなたはセキュリティ監査員です。秘密情報の扱いを検査してください。

## チェック項目
1. **ハードコード**: APIキー・パスワード・シークレットがコードに直書きされていないか
2. **ログへの出力**: 秘密情報がログに出力される可能性はないか
3. **エラーメッセージ**: 秘密情報がエラーメッセージに含まれないか
4. **環境変数**: 環境変数から安全に読み込まれているか
5. **デフォルト値**: 秘密情報のデフォルト値が設定されていないか（例: password="admin"）

## 検出パターン（正規表現的な視点で確認）
- "sk-", "Bearer ", "api_key=", "password=", "secret=" の後に文字列リテラルが続く
- os.getenv("KEY", "default_value") でセキュリティ上重要な値にデフォルトが設定されている
""".strip()


def secrets_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    秘密情報の安全な取り扱いを検査する.

    Example:
        >>> result = secrets_review(client, config_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P08",
        system=SYSTEM_P08,
        user_message=f"以下のコードで秘密情報の扱いを検査してください:\n\n```python\n{code}\n```",
    )
```

### P09: 入力バリデーションレビュー

```python
# P09: ユーザー入力の検証と無害化を評価する

SYSTEM_P09 = """
あなたはセキュリティエンジニアです。入力バリデーションと無害化処理を評価してください。

## チェック観点
1. **バリデーションの位置**: 入力はシステムの境界（APIエンドポイント）で検証されているか
2. **型チェック**: 入力の型が期待と一致するか確認しているか
3. **範囲チェック**: 数値の範囲、文字列の長さ制限があるか
4. **形式チェック**: メール・URL・電話番号等の形式が正しいか
5. **サニタイズ**: HTMLエスケープ・SQLエスケープが適切か

## 発見した問題の出力フォーマット
⚠️ **バリデーション欠如**
- 引数: `引数名`（型: 現在の型）
- リスク: ...
- 推奨するバリデーション:
```python
# 修正例
```
""".strip()


def input_validation_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    入力バリデーションと無害化処理をレビューする.

    Example:
        >>> result = input_validation_review(client, api_endpoint_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P09",
        system=SYSTEM_P09,
        user_message=f"以下のコードの入力バリデーションを評価してください:\n\n```python\n{code}\n```",
    )
```

### P10: 依存ライブラリのセキュリティチェック

```python
# P10: requirements.txt / pyproject.toml の脆弱性リスクを評価する

SYSTEM_P10 = """
あなたはセキュリティエンジニアです。依存ライブラリの使用方法とセキュリティリスクを評価してください。

## チェック観点
1. **バージョン固定**: バージョンが固定されているか（`>=` の乱用）
2. **既知の脆弱ライブラリ**: よく知られた脆弱性を持つライブラリが使われていないか
3. **過剰な権限**: 必要以上の機能を持つライブラリが使われていないか
4. **サプライチェーン**: 信頼性の低いパッケージが混入していないか

## 出力
- 問題のあるライブラリとリスクを列挙
- 代替ライブラリや対策を提示
- 「pip audit」などのツール導入を推奨する場合は明記
""".strip()


def dependency_review(client: ReviewClient, requirements_text: str) -> ReviewResult:
    """
    依存ライブラリのセキュリティリスクを評価する.

    Args:
        client: ReviewClientインスタンス
        requirements_text: requirements.txtまたはpyproject.tomlの内容

    Example:
        >>> reqs = open("requirements.txt").read()
        >>> result = dependency_review(client, reqs)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P10",
        system=SYSTEM_P10,
        user_message=f"以下の依存関係定義のセキュリティリスクを評価してください:\n\n```\n{requirements_text}\n```",
    )
```

---

## カテゴリ3: パフォーマンスレビュー用プロンプト（5本）

### P11: N+1クエリ検出

```python
# P11: データベースのN+1クエリ問題を検出する

SYSTEM_P11 = """
あなたはデータベースパフォーマンスの専門家です。N+1クエリと関連する問題を検出してください。

## N+1パターン（検出対象）
1. ループ内でのクエリ実行
2. リレーションを取得するためのループ内アクセス（lazy loading）
3. 1件ずつINSERT/UPDATEしているが、バルク操作に置換可能なもの

## 出力フォーマット
🔴 **N+1クエリ検出**
- 場所: `関数名:概算行番号`
- パターン: ...（何がN+1になっているか）
- 影響: レコード数Nに対してN+1回のクエリが発生
- 修正方法:
```python
# 修正前
for user in users:
    user.posts  # N回のクエリ

# 修正後
users = User.objects.prefetch_related("posts").all()  # 2回のクエリ
```
""".strip()


def n_plus_one_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    N+1クエリ問題を検出する.

    Example:
        >>> result = n_plus_one_review(client, orm_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P11",
        system=SYSTEM_P11,
        user_message=f"以下のコードのN+1クエリ問題を検出してください:\n\n```python\n{code}\n```",
    )
```

### P12: メモリ効率レビュー

```python
# P12: メモリの過剰使用・リークを検出する

SYSTEM_P12 = """
あなたはPythonパフォーマンスエンジニアです。メモリ効率の問題を検出してください。

## チェック観点
1. **大規模リストの一括生成**: 大きなリストをメモリに展開しているが、ジェネレータで代替可能
2. **不要なコピー**: データを不必要にコピーしているケース
3. **リスト内包表記 vs ジェネレータ**: メモリに全件を保持する必要がない場合
4. **ファイル読み込み**: 大きなファイルを一括でread()しているが、行単位処理が可能
5. **キャッシュの肥大化**: 無制限に増えるキャッシュ

## 改善提案例
```python
# Before（全件をメモリに展開）
large_data = [process(x) for x in get_million_records()]

# After（ジェネレータで逐次処理）
def process_lazily():
    for x in get_million_records():
        yield process(x)
```
""".strip()


def memory_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    メモリ効率の問題を検出する.

    Example:
        >>> result = memory_review(client, data_processing_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P12",
        system=SYSTEM_P12,
        user_message=f"以下のコードのメモリ効率を評価してください:\n\n```python\n{code}\n```",
    )
```

### P13: 非同期処理の効率チェック

```python
# P13: async/awaitの正しい使い方と最適化を評価する

SYSTEM_P13 = """
あなたはPython非同期処理の専門家です。async/awaitの使い方と最適化機会を評価してください。

## チェック観点
1. **逐次await vs 並行実行**: await を連続して使っているが asyncio.gather で並行化できるケース
2. **同期処理のブロッキング**: async関数内でCPUバウンドまたはI/Oブロッキング処理を実行
3. **不必要なasync**: I/Oを行わないのにasync関数になっているケース
4. **例外処理**: gather での例外伝播の扱い

## 改善例
```python
# Before（逐次実行で遅い）
user = await get_user(id)
posts = await get_posts(id)
comments = await get_comments(id)

# After（並行実行で高速）
user, posts, comments = await asyncio.gather(
    get_user(id), get_posts(id), get_comments(id)
)
```
""".strip()


def async_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    非同期処理の効率と正確性を評価する.

    Example:
        >>> result = async_review(client, async_handler_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P13",
        system=SYSTEM_P13,
        user_message=f"以下の非同期コードを評価してください:\n\n```python\n{code}\n```",
    )
```

### P14: アルゴリズム計算量レビュー

```python
# P14: アルゴリズムの計算量（Big-O）を分析する

SYSTEM_P14 = """
あなたはアルゴリズムの専門家です。コードの計算量を分析し、改善可能な箇所を指摘してください。

## 出力フォーマット
### 計算量サマリー
| 関数名 | 時間計算量 | 空間計算量 | 評価 |
|--------|-----------|-----------|------|
| func_a | O(n²) | O(1) | ⚠️ 要改善 |
| func_b | O(n log n) | O(n) | ✅ 適切 |

### 改善候補
🟡 **[関数名]: O(n²) → O(n)に改善可能**
- 現在のアプローチ: ...
- 改善方法: setやdictを使ったO(1)ルックアップ
```python
# Before O(n²)
for item in list1:
    if item in list2:  # list の in 検索は O(n)

# After O(n)
set2 = set(list2)  # O(n) で構築
for item in list1:
    if item in set2:  # set の in 検索は O(1)
```
""".strip()


def algorithm_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    アルゴリズムの計算量を分析し、改善案を提示する.

    Example:
        >>> result = algorithm_review(client, sorting_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P14",
        system=SYSTEM_P14,
        user_message=f"以下のコードの計算量を分析してください:\n\n```python\n{code}\n```",
        model="sonnet",
    )
```

### P15: キャッシュ戦略レビュー

```python
# P15: キャッシュの使い方と最適化機会を評価する

SYSTEM_P15 = """
あなたはキャッシュ設計の専門家です。キャッシュの使い方を評価してください。

## チェック観点
1. **キャッシュ候補の見落とし**: 繰り返し実行される高コストな処理でキャッシュがない
2. **TTL設計**: キャッシュの有効期限が適切か（短すぎる・長すぎる）
3. **キャッシュの無効化**: データ更新時にキャッシュが適切に無効化されるか
4. **キャッシュのキー設計**: キーが一意で、コリジョンが起きないか
5. **functools.lru_cache の適切な使用**: 引数がハッシュ可能か

## よくある見落とし例
```python
# キャッシュ候補: 毎回同じSQLを実行している
def get_config() -> dict:
    return db.query("SELECT * FROM config")  # → @lru_cache or Redis

# キャッシュすべきでないもの: 副作用がある処理
@lru_cache  # ← NG: 現在時刻が毎回同じになる
def get_current_timestamp():
    return datetime.now()
```
""".strip()


def cache_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    キャッシュ戦略を評価し、最適化機会を提示する.

    Example:
        >>> result = cache_review(client, service_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P15",
        system=SYSTEM_P15,
        user_message=f"以下のコードのキャッシュ戦略を評価してください:\n\n```python\n{code}\n```",
    )
```

---

## カテゴリ4: 可読性レビュー用プロンプト（5本）

### P16: 命名規則チェック

```python
# P16: 変数・関数・クラスの命名を評価する

SYSTEM_P16 = """
あなたは経験豊富なコードレビュアーです。命名の質を評価してください。

## チェック観点
1. **意図の明確さ**: 名前を見て何をするか・何を表すかが分かるか
2. **略語・短縮**: 過度な略語（`usr`, `cnt`, `tmp`）がないか
3. **ブール型の命名**: `is_`, `has_`, `can_` などのプレフィックスが使われているか
4. **コレクションの複数形**: リストやセットに複数形が使われているか
5. **否定の命名**: `not_valid` より `invalid` の方が分かりやすい

## 評価フォーマット
| 元の名前 | 評価 | 推奨する名前 | 理由 |
|---------|------|------------|------|
| `d` | ❌ | `user_data` | 省略が過ぎる |
| `flag` | ⚠️ | `is_authenticated` | boolの意図が不明 |
| `data` | ⚠️ | `response_payload` | 何のデータか不明 |
""".strip()


def naming_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    命名規則の品質を評価する.

    Example:
        >>> result = naming_review(client, code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P16",
        system=SYSTEM_P16,
        user_message=f"以下のコードの命名を評価してください:\n\n```python\n{code}\n```",
    )
```

### P17: コメント・ドキュメントレビュー

```python
# P17: コメントとdocstringの品質を評価する

SYSTEM_P17 = """
あなたはテクニカルライターとシニアエンジニアの両方の視点を持つレビュアーです。
コメントとドキュメントの品質を評価してください。

## 評価観点
1. **What vs Why**: コードを読めば分かる「何をするか」ではなく「なぜそうするか」が書かれているか
2. **古いコメント**: コードの変更後にコメントが更新されず、実態と乖離していないか
3. **docstringの完全性**: Args/Returns/Raises が揃っているか
4. **コメントの必要性**: 自明なコードにコメントが書かれていないか（逆も要注意）

## 良いコメント vs 悪いコメント
```python
# ❌ 悪い: コードを読めば分かる
i = i + 1  # iをインクリメントする

# ✅ 良い: なぜそうするかを説明
time.sleep(0.5)  # Slack APIの1秒あたり制限（Tier1: 1req/sec）を回避
```
""".strip()


def documentation_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    コメントとdocstringの品質を評価する.

    Example:
        >>> result = documentation_review(client, code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P17",
        system=SYSTEM_P17,
        user_message=f"以下のコードのコメントとドキュメントを評価してください:\n\n```python\n{code}\n```",
    )
```

### P18: 関数の複雑度チェック

```python
# P18: 関数の複雑度（認知的複雑度）を評価する

SYSTEM_P18 = """
あなたはリファクタリングの専門家です。関数の複雑度を評価し、分割を提案してください。

## 評価指標
- **認知的複雑度（Cognitive Complexity）**: コードを読んで理解するための精神的負荷
- **循環的複雑度（Cyclomatic Complexity）**: if/for/while 等の分岐数

## 複雑度が高いサイン
- ネストが3段階以上
- 1関数に5つ以上のif文
- 1画面に収まらない（50行超）
- 「この関数が何をするか」を1文で説明できない

## 出力フォーマット
**[関数名]**: 複雑度スコア目安 [高/中/低]
- ネスト深度: X段
- 分岐数: X個
- 推定行数: X行
- 分割提案:
  1. `責務A` を担う `新関数名A()`
  2. `責務B` を担う `新関数名B()`
""".strip()


def complexity_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    関数の複雑度を評価し、リファクタリングを提案する.

    Example:
        >>> result = complexity_review(client, large_function_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P18",
        system=SYSTEM_P18,
        user_message=f"以下のコードの複雑度を評価してください:\n\n```python\n{code}\n```",
        model="sonnet",
    )
```

### P19: 重複コード検出

```python
# P19: DRY原則違反の重複コードを検出する

SYSTEM_P19 = """
あなたはリファクタリングの専門家です。DRY（Don't Repeat Yourself）原則の違反を検出してください。

## 検出パターン
1. **コードの重複**: 同じ、または非常に類似したコードブロックが複数箇所に存在
2. **ロジックの重複**: 異なる書き方でも同じ処理をしているケース
3. **定数の重複**: 同じ値が複数箇所にハードコードされている

## 出力フォーマット
🔵 **重複検出: [重複の種類]**
- 箇所1: `関数名:行番号付近`
- 箇所2: `関数名:行番号付近`
- 共通化の提案:
```python
# 共通関数として抽出する例
def shared_logic(param: type) -> return_type:
    ...
```
""".strip()


def duplication_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    DRY原則違反の重複コードを検出する.

    Example:
        >>> result = duplication_review(client, code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P19",
        system=SYSTEM_P19,
        user_message=f"以下のコードのDRY原則違反を検出してください:\n\n```python\n{code}\n```",
    )
```

### P20: マジックナンバー・ハードコード検出

```python
# P20: マジックナンバーとハードコードされた値を検出する

SYSTEM_P20 = """
あなたは保守性を重視するエンジニアです。マジックナンバーとハードコードされた値を検出してください。

## 検出対象
1. **マジックナンバー**: 意味が不明な数値リテラル（0と1以外の数値）
2. **ハードコードされた文字列**: URL・ファイルパス・エラーメッセージ等
3. **環境依存の値**: ローカルパス・特定IPアドレス等
4. **設定値**: タイムアウト・リトライ回数・バッチサイズ等

## 出力フォーマット
| 値 | 場所 | 推奨する定数名 | 配置場所の提案 |
|----|------|--------------|--------------|
| `86400` | `func:L12` | `SECONDS_PER_DAY` | `constants.py` |
| `"http://localhost:8080"` | `func:L45` | `DEFAULT_API_URL` | `.env` |
""".strip()


def magic_number_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    マジックナンバーとハードコードされた値を検出する.

    Example:
        >>> result = magic_number_review(client, code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P20",
        system=SYSTEM_P20,
        user_message=f"以下のコードのマジックナンバーとハードコードを検出してください:\n\n```python\n{code}\n```",
    )
```

---

## カテゴリ5: テスタビリティレビュー用プロンプト（5本）

### P21: テスタビリティ総合評価

```python
# P21: コードのテスト容易性を総合評価する

SYSTEM_P21 = """
あなたはテスト駆動開発（TDD）の専門家です。コードのテスタビリティを評価してください。

## 評価観点
1. **依存性注入**: 外部依存（DB・API・ファイル）がコンストラクタや引数で差し替え可能か
2. **グローバル状態**: グローバル変数・シングルトンへの依存がないか
3. **副作用の分離**: 純粋関数と副作用を持つ関数が分離されているか
4. **テストの粒度**: 関数が小さく単一の責務を持つか（テストしやすいか）
5. **時刻・乱数依存**: datetime.now() や random がハードコードされていないか

## スコアリング（5点満点）
| 観点 | スコア |
|------|--------|
| 依存性注入 | X/5 |
| グローバル状態の排除 | X/5 |
| 副作用の分離 | X/5 |
| 関数の単純さ | X/5 |
| テスト容易性（総合） | X/5 |
""".strip()


def testability_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    コードのテスタビリティを総合評価する.

    Example:
        >>> result = testability_review(client, service_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P21",
        system=SYSTEM_P21,
        user_message=f"以下のコードのテスタビリティを評価してください:\n\n```python\n{code}\n```",
        model="sonnet",
    )
```

### P22: モック可能性チェック

```python
# P22: 外部依存のモック可能性を評価する

SYSTEM_P22 = """
あなたはPytestの専門家です。外部依存のモック可能性を評価し、テストコードの例を提示してください。

## チェック観点
1. **データベース**: DBアクセスがモック可能な設計か
2. **HTTPリクエスト**: 外部APIコールが `httpx`/`requests` モックでテスト可能か
3. **ファイルシステム**: ファイル操作が `tmp_path` fixture 等でテスト可能か
4. **時刻**: `datetime.now()` が `freezegun` 等でテスト可能か
5. **環境変数**: 設定値がテスト時に差し替えられるか

## 出力: テストコードのサンプル
検出された外部依存ごとに、pytestでのテスト例を提示してください:
```python
# テストコードの例
def test_xxx(mocker):
    mock_db = mocker.patch("mymodule.Database.query")
    mock_db.return_value = [{"id": 1, "name": "test"}]
    ...
```
""".strip()


def mockability_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    外部依存のモック可能性を評価しテストコード例を提示する.

    Example:
        >>> result = mockability_review(client, service_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P22",
        system=SYSTEM_P22,
        user_message=f"以下のコードのモック可能性を評価し、テストコード例を提示してください:\n\n```python\n{code}\n```",
        model="sonnet",
    )
```

### P23: エッジケーステスト提案

```python
# P23: 見落としがちなエッジケースのテストを提案する

SYSTEM_P23 = """
あなたはQAエンジニアです。コードを見て、見落としがちなエッジケースのテストケースを提案してください。

## テストケース生成の観点
1. **境界値**: 最小値・最大値・その±1
2. **空入力**: None、空文字列、空リスト、ゼロ
3. **型の境界**: int/float の境界、文字列のエンコーディング
4. **並行性**: 同一リソースへの同時アクセス
5. **順序依存**: 特定の順序で実行した場合のみ発生する問題
6. **大量データ**: N=1, 2, 100, 1000 での動作の違い

## 出力フォーマット
```python
import pytest

@pytest.mark.parametrize("input_val, expected", [
    ([], []),          # 空リスト
    ([None], ...),     # Noneを含む
    ([0], ...),        # ゼロ
])
def test_your_function_edge_cases(input_val, expected):
    result = your_function(input_val)
    assert result == expected
```
""".strip()


def edge_case_review(client: ReviewClient, code: str) -> ReviewResult:
    """
    見落としがちなエッジケースのテストを提案する.

    Example:
        >>> result = edge_case_review(client, function_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P23",
        system=SYSTEM_P23,
        user_message=f"以下のコードのエッジケーステストを提案してください:\n\n```python\n{code}\n```",
        model="sonnet",
    )
```

### P24: テストカバレッジギャップ分析

```python
# P24: 既存テストコードのカバレッジギャップを分析する

SYSTEM_P24 = """
あなたはテスト品質エンジニアです。本番コードとテストコードを比較し、
カバーされていないケースを特定してください。

## 分析観点
1. **未テストのブランチ**: if/else の分岐で片方しかテストされていない
2. **未テストの例外パス**: 正常系しかテストされていない
3. **モックが過剰すぎるテスト**: 実際の動作を検証していないテスト
4. **テストが重複している**: 同じパスを複数テストで重複チェック

## 出力フォーマット
### カバレッジギャップ
| 関数名 | 未カバーのブランチ | テスト追加の優先度 |
|--------|-------------------|-----------------|
| `func_a` | `if cond=False` の分岐 | 高 |
| `func_b` | `ValueError` 発生時 | 中 |

### 追加すべきテストケース（コード例）
```python
def test_追加すべきテスト():
    ...
```
""".strip()


def coverage_gap_review(
    client: ReviewClient,
    production_code: str,
    test_code: str,
) -> ReviewResult:
    """
    テストカバレッジのギャップを分析する.

    Args:
        client: ReviewClientインスタンス
        production_code: 本番コード
        test_code: 対応するテストコード

    Example:
        >>> result = coverage_gap_review(client, src_code, test_code)
        >>> print(result.review_text)
    """
    message = (
        f"以下の本番コードとテストコードを比較し、カバレッジギャップを分析してください:\n\n"
        f"## 本番コード\n```python\n{production_code}\n```\n\n"
        f"## テストコード\n```python\n{test_code}\n```"
    )
    return client._call(
        prompt_id="P24",
        system=SYSTEM_P24,
        user_message=message,
        model="sonnet",
    )
```

### P25: フィクスチャ・テストデータ設計レビュー

```python
# P25: pytestフィクスチャとテストデータの設計を評価する

SYSTEM_P25 = """
あなたはpytestのベストプラクティスに精通したエンジニアです。
テストコードのフィクスチャとテストデータ設計を評価してください。

## 評価観点
1. **フィクスチャのスコープ**: function/class/module/session の使い分けが適切か
2. **テストデータの独立性**: テスト間でデータが干渉していないか
3. **ファクトリパターン**: 大量のオブジェクト作成に factory-boy 等を使うべきか
4. **データベースの扱い**: テスト後にDBが綺麗にクリーンアップされるか
5. **マジックデータ**: テストデータに意味不明な値が使われていないか

## 改善例
```python
# ❌ 悪い: テスト間で依存関係がある
_shared_user = None

def test_create():
    global _shared_user
    _shared_user = create_user("test@test.com")

def test_get():
    result = get_user(_shared_user.id)  # 前のテストに依存

# ✅ 良い: フィクスチャで独立性を保つ
@pytest.fixture
def user(db):
    u = create_user("test@test.com")
    yield u
    db.delete(u)  # クリーンアップ

def test_get_user(user):
    result = get_user(user.id)
    assert result.email == "test@test.com"
```
""".strip()


def fixture_review(client: ReviewClient, test_code: str) -> ReviewResult:
    """
    pytestフィクスチャとテストデータの設計を評価する.

    Example:
        >>> result = fixture_review(client, conftest_code + test_code)
        >>> print(result.review_text)
    """
    return client._call(
        prompt_id="P25",
        system=SYSTEM_P25,
        user_message=f"以下のテストコードのフィクスチャ設計を評価してください:\n\n```python\n{test_code}\n```",
    )
```

---

## 一括レビュー実行ユーティリティ

複数のプロンプトをまとめて実行するユーティリティ：

```python
from typing import Callable

# プロンプト関数のマッピング
ALL_PROMPTS: dict[str, Callable] = {
    "P01_SOLID": solid_review,
    "P02_dataflow": dataflow_review,
    "P03_api_design": api_design_review,
    "P04_schema": schema_review,
    "P05_error_handling": error_handling_review,
    "P06_owasp": owasp_review,
    "P07_auth": auth_review,
    "P08_secrets": secrets_review,
    "P09_input_validation": input_validation_review,
    "P11_n_plus_one": n_plus_one_review,
    "P12_memory": memory_review,
    "P13_async": async_review,
    "P14_algorithm": algorithm_review,
    "P15_cache": cache_review,
    "P16_naming": naming_review,
    "P17_documentation": documentation_review,
    "P18_complexity": complexity_review,
    "P19_duplication": duplication_review,
    "P20_magic_number": magic_number_review,
    "P21_testability": testability_review,
    "P23_edge_cases": edge_case_review,
    "P25_fixtures": fixture_review,
}


def run_security_suite(client: ReviewClient, code: str) -> list[ReviewResult]:
    """
    セキュリティ関連の全プロンプトを実行する.

    Args:
        client: ReviewClientインスタンス
        code: レビュー対象コード

    Returns:
        全レビュー結果のリスト
    """
    security_prompts = [owasp_review, auth_review, secrets_review, input_validation_review]
    results = []
    for prompt_fn in security_prompts:
        result = prompt_fn(client, code)
        results.append(result)
        print(f"[{result.prompt_id}] 完了 (コスト: ${result.cost_usd:.5f})")
    return results


def run_quick_review(client: ReviewClient, code: str) -> list[ReviewResult]:
    """
    高優先度の4プロンプトだけを素早く実行する（CI向け）.

    実行する観点: OWASP + N+1 + 複雑度 + エッジケース
    """
    quick_prompts = [owasp_review, n_plus_one_review, complexity_review, edge_case_review]
    return [fn(client, code) for fn in quick_prompts]


# 使用例
if __name__ == "__main__":
    client = ReviewClient()
    code = open("my_service.py").read()

    # セキュリティスイートを実行
    print("=== セキュリティレビュー ===")
    results = run_security_suite(client, code)

    total_cost = sum(r.cost_usd for r in results)
    print(f"\n合計コスト: ${total_cost:.4f}")

    for result in results:
        print(f"\n{'='*50}")
        print(f"[{result.prompt_id}]")
        print(result.review_text)
```

:::note
**プロンプトのカスタマイズについて**

25本のプロンプトは汎用的に設計していますが、チームのコードベースに合わせてカスタマイズすることを推奨します。

特に有効なカスタマイズ:
- チームが使用するフレームワーク（FastAPI、Django等）を明記
- チームのコーディング規約を追記
- チームが頻繁に発生させるバグパターンを「特に注意するパターン」に追加
:::
