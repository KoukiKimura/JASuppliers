# JASuppliers ローカル実行手順

この手順書は、GitHubから最新ソースを取得し、各環境でローカル実行するための標準手順を定義する。

公式リポジトリ:

```text
https://github.com/KoukiKimura/JASuppliers.git
```

環境構築の詳細と現環境の導入状況は `Work/環境構築.md` を参照する。

## 1. 前提方針

- WSL2 / Docker Desktop は必須にしない。
- Windowsローカル環境では Laravel Herd + PostgreSQL Windows版を標準候補とする。
- DBはPostgreSQLを初期標準とする。
- Laravelの `.env` により、将来的に接続DBを切り替えられる構成にする。
- Mock確認は、相手側を含む各ローカル環境で構築・実行する方法を主とする。
- Web公開・クラウド・VPS等へのデプロイ方式は、Laravel実装後に別途検討する。
- `.env`、DBダンプ、APIキー、接続文字列、個人情報はGitにコミットしない。

## 2. 必要なソフトウェア

| 種別 | 推奨 |
|------|------|
| Git | Git for Windows |
| PHP / Composer | Laravel Herd for Windows |
| Node.js / npm | Node.js LTSまたは現行安定版 |
| DB | PostgreSQL 18.x Windows版 |
| エディタ | Visual Studio Code |

補足:

- Herdを使う場合、PHPとComposerを個別にインストールしなくてもよい。
- Node.jsはフロントエンドビルドに使用する。
- PostgreSQLはローカルDBとして使用する。

## 3. 初回取得

任意の作業フォルダで以下を実行する。

```powershell
git clone https://github.com/KoukiKimura/JASuppliers.git
cd JASuppliers
```

既に取得済みの場合は、最新化する。

```powershell
git pull --rebase origin main
```

## 4. PostgreSQLの準備

PostgreSQLをインストール後、開発用DBとユーザーを作成する。

例:

```sql
CREATE USER jasuppliers_user WITH PASSWORD 'change_me';
CREATE DATABASE jasuppliers_dev OWNER jasuppliers_user;
```

接続情報は `.env` にだけ記載する。

```env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=jasuppliers_dev
DB_USERNAME=jasuppliers_user
DB_PASSWORD=change_me
```

## 5. Laravelアプリ作成後の初回セットアップ

Laravel本体がリポジトリに追加された後は、以下を実行する。

```powershell
composer install
npm install
copy .env.example .env
php artisan key:generate
php artisan migrate --seed
```

Herdを使用する場合:

- HerdのSites管理にプロジェクトを登録する。
- HerdのサイトURLでアクセスする。

Herdを使わずPHP組み込みサーバーで確認する場合:

```powershell
php artisan serve --host=127.0.0.1 --port=8000
npm run dev
```

ブラウザで以下を開く。

```text
http://127.0.0.1:8000
```

## 6. 最新ソース反映手順

他環境に最新ソースを反映する場合は、対象環境で以下を実行する。

```powershell
git pull --rebase origin main
composer install
npm install
php artisan migrate --seed
php artisan optimize:clear
```

フロントエンドのビルドが必要な場合:

```powershell
npm run build
```

## 7. Mock確認方法

Mock確認は、GitHubから最新ソースを取得した各ローカル環境で実施する。

確認者は以下の流れで環境を作成する。

```powershell
git clone https://github.com/KoukiKimura/JASuppliers.git
cd JASuppliers
composer install
npm install
copy .env.example .env
php artisan key:generate
php artisan migrate --seed
php artisan serve --host=127.0.0.1 --port=8000
```

Laravel実装時には、相手環境で動作確認しやすいようにSeederで以下を用意する。

| 種別 | 内容 |
|------|------|
| ユーザー | 管理者、仕入担当者、経理担当者、店舗スタッフ |
| マスタ | 取引先、商品、カテゴリ、税区分、手数料率、掛率 |
| 業務データ | 委託搬入、販売実績、委託消化納入、買取発注、精算データ |

Mock確認は本番データではなく、必ずダミーデータで行う。
POS連携の想定データは `Work/サンプルデータ/POS取引明細_サンプル.csv` を使用する。
デプロイ方式は後日検討とし、この手順書ではローカル構築・実行を標準とする。

## 8. トラブルシュート

### PHPまたはComposerが見つからない

Herdを起動し、PHP/ComposerがPATHに追加されているか確認する。PowerShellを開き直してから再実行する。

```powershell
php -v
composer -V
```

### PostgreSQLに接続できない

PostgreSQLサービスが起動しているか確認し、`.env` の接続情報を見直す。

```powershell
php artisan migrate
```

### 依存関係が壊れた

以下を順に実行する。

```powershell
composer install
npm install
php artisan optimize:clear
```

### DBを初期化したい

開発環境のみで実行する。

```powershell
php artisan migrate:fresh --seed
```

## 9. Git運用ルール

- 通常作業は `main` の最新を取得してから開始する。
- 作業前に `git status` で未コミット差分を確認する。
- `.env`、DBダンプ、ログ、依存パッケージ、生成ビルド成果物はコミットしない。
- 仕様変更時は `AGENT.md` と関連資料を更新する。
