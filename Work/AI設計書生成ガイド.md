# AIによる設計書自動生成ガイド
## GitHub Copilot / OpenAI Codex で要件定義書から基本設計・詳細設計を生成する方法

**作成日**: 2026年5月8日  
**対象プロジェクト**: JASuppliers 仕入システム  
**対象AI**: GitHub Copilot (VS Code)、OpenAI Codex / ChatGPT API

---

## 1. 概要

### 1.1 AIを活用した設計書生成の全体フロー

```
┌─────────────────────────────────────────────────────────────────┐
│ Docs/要件定義/仕入システム_要件定義書.md                           │
│   ↓ AIへ読み込ませる（コンテキスト注入）                           │
├─────────────────────────────────────────────────────────────────┤
│ STEP 1: 基本設計書の生成                                          │
│  - システム構成図                                                 │
│  - 機能一覧・画面一覧                                             │
│  - データモデル（ER図相当）                                        │
│  - 業務フロー図（基本）                                            │
│   ↓                                                              │
├─────────────────────────────────────────────────────────────────┤
│ STEP 2: 詳細設計書の生成                                          │
│  - 画面設計書（入出力項目定義）                                    │
│  - テーブル定義書（DDL含む）                                       │
│  - API仕様書（エンドポイント定義）                                  │
│  - 処理フロー詳細                                                  │
│  - バッチ処理仕様                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 想定する利用シーン

| シーン | 推奨ツール | 特徴 |
|--------|-----------|------|
| VS Code 上でインタラクティブに生成 | GitHub Copilot (Agent Mode) | ファイル参照・自動生成・即時確認 |
| 大量ドキュメントの一括生成 | OpenAI API (Codex / GPT-4.1) | スクリプト化・自動化・大量処理 |
| 設計レビュー・品質確認 | GitHub Copilot Chat | 対話的な確認・修正指示 |

---

## 2. GitHub Copilot を使った方法（VS Code）

### 2.1 前提: プロジェクトのAI最適化設定

VS Code で GitHub Copilot を最大限活用するために、以下のファイルをプロジェクトに配置する。

#### ① `.github/copilot-instructions.md` の作成（リポジトリ全体への指示）

プロジェクトルート（`d:\Works\project\JASuppliers`）に `.github` フォルダを作成し、以下を配置する。

```markdown
# JASuppliers 仕入システム - Copilot 指示書

## プロジェクト概要
JAあぐりタウン等の産直市場・農産物直売所向け仕入管理システム。
委託・委託消化・買取の3形態の仕入を一元管理する。

## ドキュメント構造
- Docs/要件定義/ : 要件定義書（仕入システム_要件定義書.md 等）
- Docs/基本設計/ : 基本設計書を出力
- Docs/詳細設計/ : 詳細設計書を出力

## 設計書生成のルール
- 日本語で記述すること
- 表形式での項目定義を積極的に使用すること
- Mermaid記法でER図・フロー図を表現すること
- テーブル設計はLaravel Migration方針とPostgreSQL 18.xで実行可能な定義を含めること
- APIはRESTful設計とし、OpenAPI 3.0形式で記述すること
```

**配置コマンド（PowerShell）:**

```powershell
New-Item -ItemType Directory -Force -Path "d:\Works\project\JASuppliers\.github"
# copilot-instructions.md を上記内容で作成
```

#### ② パス別指示ファイル `.github/instructions/design.instructions.md` の作成

```markdown
---
applyTo: "Docs/**/*.md"
---

# 設計書作成時のルール
- 設計書は必ず「目的」「対象範囲」「前提条件」のセクションを含めること
- 機能IDは要件定義書のIDと対応させること（例: C-001, CC-001, B-001）
- 非機能要件を設計に反映させること
- セキュリティ（OWASP Top 10対策）を明記すること
```

---

### 2.2 Agent Mode を使った設計書生成手順

#### STEP 1: VS Code の Chat を Agent Mode で開く

1. VS Code の Chat パネルを開く（`Ctrl+Shift+P` → `Chat: Open Chat`）
2. モードを **「Agent」** に切り替える
3. モデルは **「Claude Sonnet 4.5」または「GPT-4.1」** を選択（推論精度が高い）

#### STEP 2: 要件定義書をコンテキストとして渡す

Copilot Chat で以下のように `#file` 参照を使って要件定義書を読み込ませる。

**基本設計書の生成プロンプト（コピー用）:**

```
#file:Docs/要件定義/仕入システム_要件定義書.md

上記の要件定義書を読み込み、仕入システムの基本設計書を作成してください。

【出力内容】
1. システム全体構成図（Mermaid graph記法）
2. 機能一覧（委託・委託消化・買取の各モジュール）
3. 主要な画面一覧（画面ID・画面名・機能概要）
4. 主要なテーブル一覧（テーブルID・テーブル名・概要）
5. 主要な業務フロー（委託精算フローをMermaid sequenceDiagram記法で）

【出力先】
Docs/基本設計/仕入システム_基本設計書.md
```

**詳細設計書の生成プロンプト（コピー用）:**

```
#file:Docs/要件定義/仕入システム_要件定義書.md
#file:Docs/基本設計/仕入システム_基本設計書.md

上記の要件定義書・基本設計書をもとに、委託仕入モジュールの詳細設計書を作成してください。

【出力内容】
1. テーブル定義書（出荷者マスタ・商品マスタ・搬入ヘッダ・搬入明細・精算ヘッダ・精算明細）
   - カラム名（英語）、データ型、桁数、NULL可否、デフォルト値、備考
   - PRIMARY KEY・INDEX・FOREIGN KEY制約
   - Laravel Migration方針とPostgreSQL 18.xで実行可能なDDL
2. 委託精算バッチ処理の詳細フロー（Mermaid flowchart記法）
3. 出荷者向け精算明細API仕様（OpenAPI 3.0 YAML形式）

【出力先】
Docs/詳細設計/委託仕入_詳細設計書.md
```

#### STEP 3: 段階的に設計書を深める

1回のプロンプトで全部生成しようとせず、**モジュール単位・機能単位** で分割して生成する。

```
# 推奨する分割例

Phase 1: 基本設計書（全体）
  → Docs/基本設計/仕入システム_基本設計書.md

Phase 2: 詳細設計書（委託）
  → Docs/詳細設計/委託仕入_詳細設計書.md

Phase 3: 詳細設計書（委託消化）
  → Docs/詳細設計/委託消化仕入_詳細設計書.md

Phase 4: 詳細設計書（買取）
  → Docs/詳細設計/買取仕入_詳細設計書.md

Phase 5: 共通設計書（マスタ・POS連携・会計連携）
  → Docs/詳細設計/共通機能_詳細設計書.md
```

---

### 2.3 Prompt Files を使った繰り返し作業の自動化

`.github/prompts/` フォルダにプロンプトファイルを作成すると、繰り返し同じ指示を実行できる。

**例: `.github/prompts/generate-table-design.prompt.md`**

```markdown
---
mode: 'agent'
model: 'gpt-4.1'
tools: ['read_file', 'create_file', 'semantic_search']
description: 'テーブル定義書の自動生成'
---

# テーブル定義書生成プロンプト

#file:Docs/要件定義/仕入システム_要件定義書.md を読み込み、
指定されたモジュールのテーブル定義書を以下の形式で生成してください。

## 出力形式
各テーブルについて:
1. テーブル概要（目的・用途）
2. カラム定義表（カラム名・論理名・型・桁・NULL・PK/FK・説明）
3. インデックス定義
4. Laravel Migration方針 + PostgreSQL 18.x対応DDL
5. サンプルデータ（INSERT文 3件）

## 対象モジュール
{{module_name}}
```

**使い方（VS Code Chat）:**

```
/generate-table-design module_name=委託仕入
```

---

### 2.4 コンテキスト参照の効率的な使い方

| 参照方法 | 記法 | 用途 |
|----------|------|------|
| 特定ファイル参照 | `#file:Docs/要件定義/仕入システム_要件定義書.md` | 特定の設計書を渡す |
| フォルダ全体参照 | `#folder:Docs/要件定義` | フォルダ配下の全ファイル |
| ワークスペース検索 | `#codebase` | ワークスペース全体を意味的に検索 |
| シンボル参照 | `#委託精算計算` | 特定の関数・クラス・定義の参照 |
| Web参照 | `#fetch:https://...` | 外部URLの内容を取り込む |

**複数ファイルの同時参照例:**

```
#file:Docs/要件定義/仕入システム_要件定義書.md
#file:Docs/基本設計/仕入システム_基本設計書.md

これら2つのドキュメントを参照し、買取仕入モジュールの
画面設計書（画面ID・画面名・入出力項目・バリデーション定義）を
Docs/詳細設計/買取仕入_画面設計書.md に出力してください。
```

---

## 3. OpenAI Codex / ChatGPT API を使った方法

### 3.1 基本的なアプローチ（RAG: 検索拡張生成）

大量の設計書を一括生成する場合は、**OpenAI API（GPT-4.1 / o3系）** を使ってスクリプト化する。

#### アーキテクチャ概要

```
┌──────────────────────────────────┐
│  要件定義書（Markdown）            │
│  Docs/要件定義/*.md               │
└──────────┬───────────────────────┘
           │ ファイル読み込み
           ↓
┌──────────────────────────────────┐
│  Pythonスクリプト / Node.js       │
│  - ドキュメントをチャンクに分割     │
│  - systemプロンプトに埋め込む      │
│  - OpenAI API呼び出し             │
│  - レスポンスをMarkdownに書き出し  │
└──────────┬───────────────────────┘
           │
           ↓
┌──────────────────────────────────┐
│  Docs/基本設計/*.md               │
│  Docs/詳細設計/*.md               │
└──────────────────────────────────┘
```

#### Python サンプルスクリプト

```python
"""
JASuppliers 設計書自動生成スクリプト
要件定義書を読み込み、OpenAI APIで設計書を生成する

必要パッケージ: pip install openai
"""

import os
from pathlib import Path
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# 要件定義書を読み込む
def load_requirements(file_path: str) -> str:
    with open(file_path, "r", encoding="utf-8") as f:
        return f.read()

# 設計書を生成する
def generate_design_doc(
    requirements: str,
    doc_type: str,
    module_name: str,
    output_path: str
) -> str:

    # システムプロンプト（ロール・ルール定義）
    system_prompt = f"""
あなたはシステム設計の専門家です。
JASuppliers（農産物直売所向け仕入管理システム）の{doc_type}を作成します。

## 出力ルール
- 日本語で記述する
- Markdown形式で出力する
- テーブル定義にはLaravel Migration方針とPostgreSQL 18.x対応DDLを含める
- フロー図はMermaid記法（flowchart/sequenceDiagram）を使う
- 要件定義書の機能ID（C-001等）を必ず参照する
- セキュリティ要件（OWASP Top 10）を設計に反映する

## 要件定義書（全文）
<requirements>
{requirements}
</requirements>
"""

    # ユーザープロンプト（生成指示）
    user_prompt = f"""
{module_name}モジュールの{doc_type}を作成してください。

出力には以下を含めること:
1. 概要・目的・対象範囲
2. 機能一覧（要件定義書のIDと対応）
3. データモデル（ER図 / Mermaid erDiagram）
4. テーブル定義書（全カラム・DDL付き）
5. 主要な処理フロー（Mermaid sequenceDiagram）
6. バリデーション・エラー処理の定義
"""

    response = client.chat.completions.create(
        model="gpt-4.1",       # GPT-4.1 推奨（コンテキスト1Mトークン対応）
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt},
        ],
        max_tokens=8192,
        temperature=0.2,       # 設計書は低温度設定（一貫性重視）
    )

    content = response.choices[0].message.content

    # ファイルに書き出す
    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    with open(output_path, "w", encoding="utf-8") as f:
        f.write(content)

    print(f"✅ 生成完了: {output_path}")
    return content


# メイン処理
if __name__ == "__main__":
    # 要件定義書の読み込み
    req_path = r"d:\Works\project\JASuppliers\Docs\要件定義\仕入システム_要件定義書.md"
    requirements = load_requirements(req_path)

    # 生成タスク定義
    tasks = [
        {
            "doc_type": "基本設計書",
            "module_name": "仕入システム全体",
            "output_path": r"d:\Works\project\JASuppliers\Docs\基本設計\仕入システム_基本設計書.md"
        },
        {
            "doc_type": "詳細設計書",
            "module_name": "委託仕入モジュール",
            "output_path": r"d:\Works\project\JASuppliers\Docs\詳細設計\委託仕入_詳細設計書.md"
        },
        {
            "doc_type": "詳細設計書",
            "module_name": "委託消化仕入モジュール",
            "output_path": r"d:\Works\project\JASuppliers\Docs\詳細設計\委託消化仕入_詳細設計書.md"
        },
        {
            "doc_type": "詳細設計書",
            "module_name": "買取仕入モジュール",
            "output_path": r"d:\Works\project\JASuppliers\Docs\詳細設計\買取仕入_詳細設計書.md"
        },
    ]

    for task in tasks:
        generate_design_doc(
            requirements=requirements,
            doc_type=task["doc_type"],
            module_name=task["module_name"],
            output_path=task["output_path"]
        )
```

### 3.2 コンテキストウィンドウの管理（重要）

| モデル | コンテキストウィンドウ | 最大出力 | 特徴 |
|--------|---------------------|---------|------|
| GPT-4.1 | **1,000,000 トークン** | 32,768 | 大規模ドキュメントに最適 |
| GPT-4.1 mini | 1,000,000 トークン | 16,384 | コスト効率が高い |
| o3 | 200,000 トークン | 100,000 | 複雑な推論・設計判断に最適 |
| o4-mini | 200,000 トークン | 100,000 | 推論モデルのコスト削減版 |

**トークン数の目安:**
- 日本語1文字 ≒ 1〜2トークン
- A4 1ページ相当（約800文字） ≒ 800〜1,600トークン
- 要件定義書全体（約10,000文字） ≒ 10,000〜20,000トークン
- **GPT-4.1 なら要件定義書全体を余裕で渡せる**

---

### 3.3 プロンプト設計のベストプラクティス（OpenAI API）

#### 効果的なプロンプト構造（Markdown + XML タグ）

```
# Identity（AIの役割定義）
あなたはシステムアーキテクトです。

# Instructions（指示・ルール）
- Markdown形式で出力する
- Mermaid記法でER図を作成する
- ...

# Context（コンテキスト情報）
<requirements_document>
（要件定義書の内容をここに挿入）
</requirements_document>

# Task（具体的なタスク）
上記の要件定義書をもとに、委託仕入モジュールのテーブル定義書を作成してください。
```

#### 推論モデル（o3 / o4-mini）を使う場合の注意

- 詳細な手順指示は不要（**ゴールだけを明確に伝える**）
- 「なぜ」「何を」を明示する（「どうやって」は不要）

```python
# 推論モデル向けプロンプト例
user_prompt = """
要件定義書に基づいて、仕入システムのデータベース設計を行ってください。

目標: 委託・委託消化・買取の3形態を統一的に管理できるテーブル構造
制約: PostgreSQL 18.x、Laravel Migration、正規化は第3正規形以上、トレーサビリティの確保が必須
出力: テーブル定義書（Markdown）＋ ERDiagram（Mermaid）＋ DDL
"""

response = client.chat.completions.create(
    model="o3",
    reasoning_effort="high",  # 複雑な設計には high を指定
    messages=[
        {"role": "developer", "content": system_prompt},
        {"role": "user", "content": user_prompt},
    ]
)
```

---

## 4. 効率的なワークフロー（推奨手順）

### 4.1 推奨ワークフロー全体図

```
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 0: 準備（30分）                                            │
│  1. .github/copilot-instructions.md を作成                      │
│  2. .github/instructions/design.instructions.md を作成          │
│  3. AGENT.md にプロジェクト概要を記載                             │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 1: 基本設計書の生成（GitHub Copilot Agent Mode）           │
│  プロンプト: 要件定義書 → 基本設計書                              │
│  確認: システム構成・機能一覧・画面一覧・テーブル一覧が揃っているか │
│  出力: Docs/基本設計/仕入システム_基本設計書.md                    │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 2: 詳細設計書の生成（モジュール単位）                        │
│  入力: 要件定義書 + 基本設計書                                    │
│  モジュール別に順次生成:                                          │
│   a. 委託仕入_詳細設計書.md                                       │
│   b. 委託消化仕入_詳細設計書.md                                   │
│   c. 買取仕入_詳細設計書.md                                       │
│   d. 共通機能_詳細設計書.md（マスタ・POS連携・会計連携）           │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 3: レビュー・品質確認（GitHub Copilot Chat）               │
│  プロンプト例:                                                    │
│  「#file:詳細設計書.md の整合性を確認し、要件定義書との不整合を    │
│    リストアップしてください」                                      │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│ PHASE 4: コード生成への展開                                       │
│  入力: 詳細設計書（テーブル定義・API仕様）                        │
│  出力: DDL、APIスタブ、型定義、テストコード                       │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 プロンプトのフェーズ別テンプレート

#### 基本設計書生成プロンプト（Phase 1 用）

```
#file:Docs/要件定義/仕入システム_要件定義書.md

上記の要件定義書を精読し、JASuppliers 仕入システムの基本設計書を作成してください。

## 作成内容
### 1. システム全体構成
- アーキテクチャ概要（3層構造: フロントエンド/バックエンド/DB）
- 外部システム連携（POS/会計/銀行振込）

### 2. 機能一覧
- 要件定義書の機能IDと対応付けた機能一覧表
- 委託(C-xxx)・委託消化(CC-xxx)・買取(B-xxx) の全機能

### 3. 画面一覧
- 画面ID・画面名・主要機能・対応する要件定義ID

### 4. テーブル一覧
- テーブル論理名・物理名・主要カラム・仕入形態との対応

### 5. 業務フロー（主要3本）
- 委託精算フロー（Mermaid sequenceDiagram）
- 委託消化仕入確定フロー（Mermaid sequenceDiagram）
- 買取入荷フロー（Mermaid sequenceDiagram）

## 出力先
Docs/基本設計/仕入システム_基本設計書.md に出力してください。
```

#### 詳細設計書生成プロンプト（Phase 2 用・委託仕入例）

```
#file:Docs/要件定義/仕入システム_要件定義書.md
#file:Docs/基本設計/仕入システム_基本設計書.md

上記2つのドキュメントを読み込み、委託仕入モジュールの詳細設計書を作成してください。

## 作成内容

### 1. テーブル定義書
以下のテーブルをそれぞれ定義（カラム定義表 + Laravel Migration方針 + PostgreSQL 18.x DDL）:
- t_supplier（出荷者マスタ）
- t_product（商品マスタ）
- t_receive_header（搬入ヘッダ）
- t_receive_detail（搬入明細）
- t_settlement_header（精算ヘッダ）
- t_settlement_detail（精算明細）
- m_fee_rate（手数料率マスタ）

### 2. ER図
上記テーブルの関連図（Mermaid erDiagram記法）

### 3. 処理フロー詳細
- 月次精算バッチの詳細フロー（Mermaid flowchart LR）
- 精算金額計算ロジック（擬似コード含む）

### 4. API仕様（主要5本）
- GET /api/suppliers - 出荷者一覧取得
- POST /api/receives - 搬入登録
- GET /api/settlements/{year}/{month} - 精算一覧取得
- POST /api/settlements/calculate - 精算計算実行
- GET /api/settlements/{id}/export - 精算明細PDF出力

### 5. バリデーション定義
各APIのリクエスト・レスポンスのバリデーションルール

## 出力先
Docs/詳細設計/委託仕入_詳細設計書.md に出力してください。
```

---

## 5. 注意事項（重要）

### 5.1 コンテキストウィンドウと精度のトレードオフ

| リスク | 内容 | 対策 |
|--------|------|------|
| **コンテキスト汚染** | 長い会話の中で過去の指示が干渉する | 設計フェーズごとに**新しいチャットセッションを開始**する |
| **ハルシネーション** | AIが存在しない機能・テーブルを生成する | 要件定義書の機能IDを必ず参照させる。生成後に人間がレビューする |
| **整合性の欠如** | 複数回生成した設計書間で矛盾が生じる | 基本設計書を最初に確定させ、詳細設計は必ず基本設計を参照させる |
| **過剰生成** | 指示していない機能が追加される | プロンプトに「要件定義書に記載された機能のみ設計すること」を明記 |

### 5.2 生成AIに渡してはいけない情報

```
⚠️ 以下の情報は生成AIに渡さないこと（セキュリティ・個人情報保護）

- 実際の顧客データ・生産者の個人情報
- データベースの接続文字列・パスワード
- APIキー・認証トークン
- 実環境のIPアドレス・サーバー構成
- 未公開の営業情報・取引先情報
```

> ⚠️ **GitHub Copilot / OpenAI API** に渡したデータは、設定によってはAIの学習に使用される可能性がある。  
> 企業の機密情報・個人情報は **Business/Enterprise プラン** を利用するか、`telemetry.telemetryLevel: off` を設定してデータ送信を制限すること。

### 5.3 生成結果の品質確認チェックリスト

設計書が生成されたら、以下を必ずレビューすること。

**基本設計書**
- [ ] 要件定義書の全機能IDが設計書に反映されているか
- [ ] 3つの仕入形態（委託・委託消化・買取）がすべて設計されているか
- [ ] 外部システム連携（POS・会計・銀行）が設計に含まれているか
- [ ] 非機能要件（パフォーマンス・可用性）が設計に反映されているか

**詳細設計書**
- [ ] テーブルの主キー・外部キーが正しく設定されているか
- [ ] 消費税区分（軽減税率8%・標準税率10%）がテーブル設計に含まれているか
- [ ] 監査ログ（操作ユーザー・日時）が設計されているか
- [ ] バリデーション（必須・桁数・形式）が定義されているか
- [ ] DDLまたはMigrationが実際に実行可能な構文か確認する（PostgreSQL / Laravel Migrationで検証）

### 5.4 Copilot と Codex の使い分け

| 用途 | 推奨 | 理由 |
|------|------|------|
| インタラクティブな設計作業 | **GitHub Copilot Agent Mode** | リアルタイム確認・修正が容易 |
| 設計書の一括生成・自動化 | **OpenAI API (GPT-4.1)** | スクリプト化・バッチ処理に適す |
| 複雑な設計判断（アーキテクチャ選択など） | **OpenAI API (o3)** | 高度な推論が必要な場合 |
| 日常的なコーディング支援 | **GitHub Copilot（補完）** | VS Code上でのリアルタイム補完 |
| コードレビュー・品質確認 | **GitHub Copilot Chat** | 既存コードへの質問・レビュー |

---

## 6. プロジェクト向け AGENT.md の記述例

プロジェクトルートの `AGENT.md` に以下を記述することで、すべてのAIエージェントがプロジェクト概要を自動的に参照する。

```markdown
# JASuppliers 仕入システム

## プロジェクト概要
産直市場・農産物直売所向けの仕入管理システム。
委託・委託消化・買取の3形態の仕入を一元管理する。

## ドキュメント構造
- Docs/要件定義/ : 要件定義書
- Docs/基本設計/ : 基本設計書（システム構成・機能一覧・ER図）
- Docs/詳細設計/ : 詳細設計書（テーブル定義・API仕様・処理フロー）
- Work/          : 調査結果・作業メモ

## 設計上の重要な前提
- 仕入形態は「委託」「委託消化」「買取」の3種類
- POS連携（販売実績の日次取込）が必須
- 消費税は軽減税率(8%)と標準税率(10%)の区分管理が必要
- 精算は月次（生産者・出荷者への振込）
- 操作ログは7年間保管（法令要件）

## 使用技術スタック（想定）
- バックエンド: PHP 8.5系 / Laravel 13.x
- フロントエンド: Laravel標準構成を前提に、Blade中心またはVue.js/React採用をPhase 1で確定
- DB: PostgreSQL 18.x
- API: RESTful / OpenAPI 3.0
```

---

## 7. 参考リンク

| リソース | URL |
|---------|-----|
| GitHub Copilot カスタム指示（公式） | https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot |
| VS Code AI カスタマイズ | https://code.visualstudio.com/docs/copilot/copilot-customization |
| VS Code AI ベストプラクティス | https://code.visualstudio.com/docs/copilot/prompt-crafting |
| OpenAI プロンプトエンジニアリング | https://developers.openai.com/api/docs/guides/prompt-engineering |
| OpenAI モデル一覧 | https://developers.openai.com/api/docs/models |

---

*作成: JASuppliers プロジェクト チーム*  
*最終更新: 2026年5月15日*
