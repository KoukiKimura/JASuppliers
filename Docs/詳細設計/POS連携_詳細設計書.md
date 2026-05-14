# POS連携 詳細設計書

**プロジェクト名**: JASuppliers 仕入システム  
**作成日**: 2026年5月14日  
**最終更新**: 2026年5月14日  
**バージョン**: 1.0  
**対象要件**: 3.2、C-004、CC-005  

---

## 1. 目的

POSシステムから出力される想定CSVを取り込み、販売・返品・値下げ実績を仕入形態別の精算・仕入確定・在庫処理へ連携するための詳細仕様を定義する。

既存POSの実データ仕様は未入手のため、初期リリースでは `Work/サンプルデータ/POS取引明細_サンプル.csv` を基準とする。実POS仕様入手後は、取込アダプタ層で差分を吸収する。

---

## 2. 対象範囲

| 区分 | 対象 |
|------|------|
| 対象取込 | 販売、返品、値下げ販売 |
| 対象仕入形態 | 委託、委託消化、買取 |
| 取込方式 | 管理画面からのCSVアップロード |
| 文字コード | UTF-8 |
| 初期頻度 | 日次または任意タイミングでの手動取込 |
| 対象外 | POS実機API連携、POS別専用フォーマット、リアルタイム連携 |

---

## 3. CSVレイアウト

| カラム名 | 型 | 必須 | 桁・形式 | 説明 |
|----------|----|------|----------|------|
| `pos_transaction_id` | string | 必須 | 64以内 | POS側の取引明細ID |
| `transaction_type` | enum | 必須 | `SALE` / `RETURN` / `MARKDOWN` | 取引種別 |
| `store_code` | string | 必須 | 32以内 | 店舗コード |
| `register_no` | string | 必須 | 32以内 | レジ番号 |
| `receipt_no` | string | 必須 | 64以内 | レシート番号 |
| `line_no` | integer | 必須 | 1以上 | レシート内明細行番号 |
| `business_date` | date | 必須 | `YYYY-MM-DD` | 営業日 |
| `sold_at` | datetime | 必須 | ISO 8601 | 販売日時 |
| `barcode` | string | 任意 | 64以内 | JANまたは独自バーコード |
| `product_code` | string | 必須 | 64以内 | 商品コード |
| `product_name` | string | 必須 | 255以内 | 商品名 |
| `supplier_code` | string | 任意 | 64以内 | 出荷者・仕入先コード |
| `procurement_type` | enum | 必須 | `CONSIGNMENT` / `CONSUMPTION` / `PURCHASE` | 仕入形態 |
| `category_code` | string | 任意 | 64以内 | 商品カテゴリコード |
| `quantity` | decimal | 必須 | 12,3 | 数量。返品は負数 |
| `unit_price` | integer | 必須 | 0以上 | 売価単価（税込） |
| `gross_amount` | integer | 必須 | 正負可 | 値引前税込金額 |
| `discount_amount` | integer | 必須 | 0以上 | 値下げ・値引額 |
| `tax_rate` | decimal | 必須 | 8 または 10 | 適用税率 |
| `tax_amount` | integer | 必須 | 正負可 | 消費税額 |
| `net_amount` | integer | 必須 | 正負可 | 値引後税込金額 |
| `original_pos_transaction_id` | string | 任意 | 64以内 | 返品・訂正元取引ID |

---

## 4. テーブル定義

### 4.1 `pos_import_batches`

| カラム | 型 | NULL | 制約 | 説明 |
|--------|----|------|------|------|
| `id` | bigserial | NO | PK | 取込バッチID |
| `store_id` | bigint | NO | FK `stores.id` | 店舗 |
| `file_name` | varchar(255) | NO |  | 取込ファイル名 |
| `file_hash` | varchar(128) | NO | unique | ファイルハッシュ |
| `business_date_from` | date | YES |  | 対象営業日From |
| `business_date_to` | date | YES |  | 対象営業日To |
| `status` | varchar(32) | NO | index | `UPLOADED` / `PROCESSING` / `COMPLETED` / `COMPLETED_WITH_ERROR` / `FAILED` |
| `total_count` | integer | NO | default 0 | 総件数 |
| `success_count` | integer | NO | default 0 | 成功件数 |
| `error_count` | integer | NO | default 0 | エラー件数 |
| `started_at` | timestamp | YES |  | 開始日時 |
| `finished_at` | timestamp | YES |  | 終了日時 |
| `created_by` | bigint | NO | FK `users.id` | 実行者 |
| `updated_by` | bigint | NO | FK `users.id` | 更新者 |
| `created_at` | timestamp | NO |  | 作成日時 |
| `updated_at` | timestamp | NO |  | 更新日時 |

### 4.2 `pos_sales_lines`

| カラム | 型 | NULL | 制約 | 説明 |
|--------|----|------|------|------|
| `id` | bigserial | NO | PK | POS明細ID |
| `pos_import_batch_id` | bigint | NO | FK | 取込バッチ |
| `store_id` | bigint | NO | FK | 店舗 |
| `product_id` | bigint | YES | FK | 商品。未照合時はNULL |
| `business_partner_id` | bigint | YES | FK | 出荷者・仕入先。未照合時はNULL |
| `pos_transaction_id` | varchar(64) | NO | unique pair | POS取引ID |
| `line_no` | integer | NO | unique pair | 明細行番号 |
| `transaction_type` | varchar(16) | NO | index | 取引種別 |
| `procurement_type` | varchar(16) | NO | index | 仕入形態 |
| `business_date` | date | NO | index | 営業日 |
| `sold_at` | timestamp | NO |  | 販売日時 |
| `barcode` | varchar(64) | YES | index | バーコード |
| `product_code` | varchar(64) | NO | index | 商品コード |
| `product_name` | varchar(255) | NO |  | 商品名 |
| `supplier_code` | varchar(64) | YES | index | 取引先コード |
| `quantity` | numeric(12,3) | NO |  | 数量 |
| `unit_price` | integer | NO |  | 売価単価 |
| `gross_amount` | integer | NO |  | 値引前税込金額 |
| `discount_amount` | integer | NO |  | 値下げ額 |
| `tax_rate` | numeric(5,2) | NO |  | 税率 |
| `tax_amount` | integer | NO |  | 税額 |
| `net_amount` | integer | NO |  | 値引後税込金額 |
| `original_pos_transaction_id` | varchar(64) | YES | index | 元取引ID |
| `is_closed` | boolean | NO | default false | 締め済みフラグ |
| `created_by` | bigint | NO | FK | 作成者 |
| `updated_by` | bigint | NO | FK | 更新者 |
| `created_at` | timestamp | NO |  | 作成日時 |
| `updated_at` | timestamp | NO |  | 更新日時 |

**一意制約**: `unique(pos_transaction_id, line_no)`  
**主なindex**: `(business_date, procurement_type)`, `(product_id, business_partner_id)`, `(store_id, business_date)`

### 4.3 `pos_import_errors`

| カラム | 型 | NULL | 制約 | 説明 |
|--------|----|------|------|------|
| `id` | bigserial | NO | PK | エラーID |
| `pos_import_batch_id` | bigint | NO | FK | 取込バッチ |
| `row_no` | integer | NO |  | CSV行番号 |
| `error_code` | varchar(64) | NO | index | エラーコード |
| `error_message` | text | NO |  | エラー内容 |
| `raw_payload` | jsonb | NO |  | 取込行のJSON |
| `status` | varchar(32) | NO | index | `OPEN` / `RESOLVED` / `IGNORED` |
| `resolved_by` | bigint | YES | FK `users.id` | 解決者 |
| `resolved_at` | timestamp | YES |  | 解決日時 |
| `created_by` | bigint | NO | FK | 作成者 |
| `updated_by` | bigint | NO | FK | 更新者 |
| `created_at` | timestamp | NO |  | 作成日時 |
| `updated_at` | timestamp | NO |  | 更新日時 |

---

## 5. 取込処理仕様

### 5.1 処理順序

1. CSVファイルをアップロードする。
2. ファイル拡張子、文字コード、ヘッダ行を検証する。
3. `file_hash` を算出し、同一ファイルの重複取込を警告する。
4. `pos_import_batches` を `PROCESSING` で登録する。
5. 1行ずつ型・必須・桁・コード値を検証する。
6. 店舗、商品、バーコード、取引先を照合する。
7. エラー行は `pos_import_errors` へ登録する。
8. 正常行は `pos_sales_lines` へ登録または更新する。
9. バッチ件数を更新し、完了状態へ変更する。

### 5.2 再取込ルール

| 条件 | 処理 |
|------|------|
| 同一 `pos_transaction_id` + `line_no` が未締め | 既存明細を更新する |
| 同一 `pos_transaction_id` + `line_no` が締め済み | 更新せずエラーとする |
| 締め済み期間の訂正 | 訂正取引または翌月調整として登録する |
| ファイルハッシュが同一 | 警告を表示し、担当者の明示操作がある場合のみ取込する |

### 5.3 エラーコード

| コード | 内容 | 補正方法 |
|--------|------|----------|
| `REQUIRED_MISSING` | 必須項目なし | CSV修正後に再取込 |
| `INVALID_TYPE` | 型不正 | CSV修正後に再取込 |
| `UNKNOWN_STORE` | 店舗未登録 | 店舗マスタ登録 |
| `UNKNOWN_PRODUCT` | 商品未登録 | 商品マスタ登録または商品紐付け |
| `UNKNOWN_PARTNER` | 取引先未登録 | 取引先マスタ登録 |
| `UNKNOWN_BARCODE` | バーコード未登録 | 商品バーコード登録 |
| `CLOSED_PERIOD` | 締め済み期間 | 訂正取引で対応 |
| `DUPLICATE_CLOSED_LINE` | 締め済み明細への重複 | 取込不可 |

---

## 6. API仕様

| メソッド | URI | 処理 | 権限 |
|----------|-----|------|------|
| GET | `/api/pos/import-batches` | 取込履歴一覧 | 仕入担当者、経理担当者、管理者 |
| POST | `/api/pos/import-batches` | CSVアップロード・取込開始 | 仕入担当者、管理者 |
| GET | `/api/pos/import-batches/{id}` | 取込結果詳細 | 仕入担当者、経理担当者、管理者 |
| GET | `/api/pos/import-errors` | 取込エラー一覧 | 仕入担当者、管理者 |
| PATCH | `/api/pos/import-errors/{id}/resolve` | エラー補正済みに変更 | 仕入担当者、管理者 |
| POST | `/api/pos/import-batches/{id}/retry-errors` | エラー行再処理 | 仕入担当者、管理者 |

---

## 7. 画面仕様

| 画面ID | 画面名 | 主な項目 |
|--------|--------|----------|
| POS-001 | POS取込 | ファイル選択、取込実行、取込履歴、件数、ステータス、エラー件数 |
| POS-002 | POS取込エラー補正 | 行番号、エラーコード、商品候補、取引先候補、補正状態 |

---

## 8. バリデーション

| 対象 | ルール |
|------|--------|
| ファイル | CSV拡張子、UTF-8、ヘッダ行あり |
| 一意キー | `pos_transaction_id` + `line_no` が必須 |
| 取引種別 | `SALE` / `RETURN` / `MARKDOWN` のみ |
| 仕入形態 | `CONSIGNMENT` / `CONSUMPTION` / `PURCHASE` のみ |
| 数量 | 販売・値下げは正数、返品は負数 |
| 金額 | `net_amount = gross_amount - discount_amount` を基本とする |
| 税率 | 8または10 |
| 締め済み明細 | 直接更新不可 |

---

## 9. テスト観点

- 正常CSVを取込できること。
- 販売、返品、値下げがそれぞれ登録されること。
- `CONSIGNMENT` / `CONSUMPTION` / `PURCHASE` が識別されること。
- 商品未登録行がエラーとして保留されること。
- 同一一意キーの未締め再取込が更新扱いになること。
- 締め済み明細の再取込がエラーになること。
- 取込履歴と監査ログが記録されること。

---

*作成: JASuppliers プロジェクト*
