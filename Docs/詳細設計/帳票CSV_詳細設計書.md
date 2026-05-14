# 帳票CSV 詳細設計書

**プロジェクト名**: JASuppliers 仕入システム
**作成日**: 2026年5月14日
**最終更新**: 2026年5月14日
**バージョン**: 1.0
**対象要件**: 3.4、3.5、C-007、C-008、CC-010、B-009

---

## 1. 目的

PDF、Excel、仕訳CSV、全銀協形式CSVの出力仕様と出力履歴管理を定義する。

---

## 2. 対象出力

| 出力 | 形式 | 主な利用者 |
|------|------|------------|
| 委託精算明細書 | PDF | 経理担当者、出荷者確認用 |
| 消化精算書 | PDF | 経理担当者、サプライヤー確認用 |
| 仕入明細書 | PDF | 経理担当者 |
| 在庫一覧表 | Excel / PDF | 店舗スタッフ、仕入担当者 |
| 棚卸差異報告書 | PDF | 店舗スタッフ、管理者 |
| 仕訳データ | 汎用CSV | 経理担当者 |
| 振込依頼データ | 全銀協形式CSV | 経理担当者 |

---

## 3. テーブル定義

本書で定義する業務テーブルには、特記がない限り `created_at`、`updated_at`、`created_by`、`updated_by` を保持する。出力履歴は追記型で管理し、既存ファイルを上書きしない。

### 3.1 `document_files`

| カラム | 型 | NULL | 制約 | 説明 |
|--------|----|------|------|------|
| `id` | bigserial | NO | PK | ファイルID |
| `document_type` | varchar(32) | NO | index | 帳票種別 |
| `source_type` | varchar(32) | YES | index | 元データ種別 |
| `source_id` | bigint | YES |  | 元データID |
| `file_name` | varchar(255) | NO |  | ファイル名 |
| `file_path` | varchar(512) | NO |  | 保存パス |
| `mime_type` | varchar(128) | NO |  | MIME |
| `file_size` | bigint | NO |  | サイズ |
| `file_hash` | varchar(128) | NO |  | ハッシュ |
| `generated_by` | bigint | NO | FK `users.id` | 生成者 |
| `generated_at` | timestamp | NO |  | 生成日時 |
| `created_at` | timestamp | NO |  | 作成日時 |
| `updated_at` | timestamp | NO |  | 更新日時 |
| `created_by` | bigint | NO | FK `users.id` | 作成者 |
| `updated_by` | bigint | NO | FK `users.id` | 更新者 |

### 3.2 `accounting_exports`

| カラム | 型 | NULL | 制約 | 説明 |
|--------|----|------|------|------|
| `id` | bigserial | NO | PK | 仕訳出力ID |
| `export_no` | varchar(64) | NO | unique | 出力番号 |
| `target_month` | varchar(7) | NO | index | 対象年月 |
| `status` | varchar(32) | NO | index | `DRAFT` / `EXPORTED` / `CANCELLED` |
| `line_count` | integer | NO | default 0 | 行数 |
| `total_debit_amount` | integer | NO | default 0 | 借方合計 |
| `total_credit_amount` | integer | NO | default 0 | 貸方合計 |
| `document_file_id` | bigint | YES | FK | 出力ファイル |
| `exported_by` | bigint | YES | FK `users.id` | 出力者 |
| `exported_at` | timestamp | YES |  | 出力日時 |
| `created_by` | bigint | NO | FK `users.id` | 作成者 |
| `updated_by` | bigint | NO | FK `users.id` | 更新者 |
| `created_at` | timestamp | NO |  | 作成日時 |
| `updated_at` | timestamp | NO |  | 更新日時 |

### 3.3 `bank_transfer_exports`

| カラム | 型 | NULL | 制約 | 説明 |
|--------|----|------|------|------|
| `id` | bigserial | NO | PK | 振込出力ID |
| `payment_batch_id` | bigint | NO | FK `payment_batches.id` | 支払バッチ |
| `export_no` | varchar(64) | NO | unique | 出力番号 |
| `format_type` | varchar(32) | NO |  | `ZENGINKYO_CSV` |
| `line_count` | integer | NO | default 0 | 明細件数 |
| `total_amount` | integer | NO | default 0 | 振込合計 |
| `document_file_id` | bigint | YES | FK | 出力ファイル |
| `exported_by` | bigint | YES | FK `users.id` | 出力者 |
| `exported_at` | timestamp | YES |  | 出力日時 |
| `created_by` | bigint | NO | FK `users.id` | 作成者 |
| `updated_by` | bigint | NO | FK `users.id` | 更新者 |
| `created_at` | timestamp | NO |  | 作成日時 |
| `updated_at` | timestamp | NO |  | 更新日時 |

---

## 4. 出力項目

### 4.1 仕訳CSV

| 項目 | 内容 |
|------|------|
| 伝票日付 | 精算締め日または支払確定日 |
| 借方科目 | `account_mapping_rules.debit_account_code` |
| 貸方科目 | `account_mapping_rules.credit_account_code` |
| 税区分 | `account_mapping_rules.tax_code` |
| 金額 | 税込金額 |
| 摘要 | 仕入形態、取引先名、対象年月 |

### 4.2 全銀協形式CSV

| 項目 | 内容 |
|------|------|
| 振込指定日 | `payment_batches.payment_on` |
| 金融機関コード | 支払先口座 |
| 支店コード | 支払先口座 |
| 預金種目 | 普通・当座 |
| 口座番号 | 支払先口座 |
| 受取人名 | 口座名義 |
| 振込金額 | `payment_lines.payment_amount` |

---

## 5. API仕様

| メソッド | URI | 処理 | 権限 |
|----------|-----|------|------|
| GET | `/api/reports` | 帳票一覧 | 仕入担当者、経理担当者、管理者 |
| POST | `/api/reports/{type}/generate` | 帳票生成 | 仕入担当者、経理担当者、管理者 |
| GET | `/api/documents/{id}/download` | ファイル取得 | 権限保持者 |
| POST | `/api/accounting-exports` | 仕訳CSV出力 | 経理担当者、管理者 |
| POST | `/api/bank-transfer-exports` | 全銀協形式CSV出力 | 経理担当者、管理者 |

---

## 6. 画面仕様

| 画面ID | 画面名 | 入力・表示項目 |
|--------|--------|----------------|
| RPT-001 | 帳票出力 | 帳票種別、対象年月、取引先、出力形式、出力履歴 |
| ACC-001 | 会計出力 | 対象年月、仕訳件数、借方合計、貸方合計、出力状態 |
| PAY-001 | 支払管理 | 支払バッチ、支払件数、支払合計、全銀協形式CSV出力 |

---

## 7. 出力処理

1. 対象データを検索する。
2. 出力権限を確認する。
3. 出力行を生成する。
4. 合計金額と件数を検証する。
5. ファイルを保存する。
6. `document_files` と出力履歴を登録する。
7. 監査ログを記録する。

---

## 8. バリデーション

| 対象 | ルール |
|------|--------|
| 仕訳CSV | 借方合計と貸方合計が一致すること |
| 全銀協形式CSV | 支払確定済みバッチのみ出力可能 |
| 帳票 | 対象データが存在すること |
| 再出力 | 履歴を新規作成し、既存ファイルを上書きしない |
| ファイル | 出力ファイルハッシュを保持する |

---

## 9. テスト観点

- 各帳票が対象条件で出力されること。
- 仕訳CSVの貸借合計が一致すること。
- 全銀協形式CSVに口座情報と支払額が出力されること。
- 再出力で履歴が追加されること。
- 権限のない利用者が出力できないこと。

---

*作成: JASuppliers プロジェクト*
