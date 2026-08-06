# dbt Core × dbt Projects on Snowflake 開発環境構築ガイド

対象読者: 新人エンジニア
目的: ローカルでは dbt Core を使って開発し、開発者ごとの Snowflake DB/スキーマに `snow dbt deploy` で UT（開発者単体テスト）環境をデプロイできるようにする

---

## 1. 全体像

```
[ローカルPC]
  dbt Core (dbt-snowflake) で models/tests を開発
  dbt run / dbt test をローカルから直接 Snowflake の "開発者専用スキーマ" に対して実行
        │
        │ git commit / push
        ▼
[GitHubリポジトリ]
        │
        │ 開発者が snow dbt deploy を実行 (手動 or Makefile)
        ▼
[Snowflake]
  dbt Projects on Snowflake の DBT PROJECT オブジェクトとして
  開発者ごとの DB/スキーマにデプロイ・実行 (snow dbt execute / EXECUTE DBT PROJECT)
  → UT環境として動作確認
        │
        │ PRマージ後、CI/CDで prod 用 DBT PROJECT オブジェクトへデプロイ
        ▼
[本番 DBT PROJECT オブジェクト] (Snowflake Tasks でスケジュール実行)
```

ポイント:
- **ローカル開発**: 通常の dbt Core ワークフロー（高速なイテレーション、IDE補完、dbt-checkpoint等が使える）
- **UT確認**: Snowflake上にコードを実体としてデプロイし、Snowflakeネイティブの実行環境（`EXECUTE DBT PROJECT`）で動かして最終確認
- **本番**: Snowflake Tasks や CI/CD（GitHub Actions等）から `snow dbt deploy` → `snow dbt execute` で本番実行

---

## 2. 前提ソフトウェア

| ツール | 用途 | バージョン目安 |
|---|---|---|
| pyenv | Pythonバージョン管理 | 最新 |
| Python | dbt / Snowflake CLI 実行基盤 | 3.11 系推奨（3.10以上必須） |
| uv | Python仮想環境・パッケージ管理 | 最新 |
| dbt-core | ローカル開発用dbt本体 | プロジェクトで固定（例: 1.10.x） |
| dbt-snowflake | Snowflakeアダプタ | dbt-coreと同系統バージョン |
| Snowflake CLI (`snow`) | `snow dbt deploy` / `snow dbt execute` 用 | 最新（3.21以上推奨、env.yml機能を使う場合） |
| git | バージョン管理 | 最新 |

---

## 3. ディレクトリ構成

リポジトリ全体（例: `analytics-dbt`）:

```
analytics-dbt/
├── .git/
├── .gitignore
├── README.md
├── pyproject.toml           # uvで管理するPython依存関係
├── uv.lock
├── Makefile                 # よく使うコマンドをラップ
├── .dbt/
│   └── profiles.yml.example # profiles.ymlのサンプル（実物は各自 ~/.dbt/ に置く）
├── dbt_project/              # ← dbt Projects on Snowflake にデプロイする単位
│   ├── dbt_project.yml
│   ├── profiles.yml          # dbt Projects on Snowflake用（デプロイ時に同梱される）
│   ├── env.yml                # 開発者ごとの環境変数定義（任意, CLI 3.21+）
│   ├── packages.yml
│   ├── package-lock.yml
│   ├── models/
│   │   ├── staging/
│   │   │   ├── _staging__sources.yml
│   │   │   └── stg_xxx.sql
│   │   ├── intermediate/
│   │   └── marts/
│   │       ├── _marts__models.yml
│   │       └── fct_xxx.sql
│   ├── seeds/
│   ├── macros/
│   ├── tests/
│   ├── snapshots/
│   └── dbt_packages/         # dbt deps 実行後に生成（.gitignore対象）
└── scripts/
    └── deploy_dev.sh          # 開発者が自分のUT環境へデプロイするスクリプト
```

**なぜ `dbt_project/` を切っているか**
`snow dbt deploy` はディレクトリ単位で Snowflake にアップロードします。リポジトリ直下に CI設定やドキュメントも置きたいので、dbtプロジェクト本体だけをサブディレクトリに分離し、`--source dbt_project` で指定できるようにしています。

**`profiles.yml` の役割の違いに注意**
- ローカル実行用: `~/.dbt/profiles.yml`（各自のホームディレクトリ、リポジトリには含めない）
- Snowflakeデプロイ用: `dbt_project/profiles.yml`（リポジトリにコミットし、デプロイ時にSnowflake側へコピーされる。database/role/schema/typeを必ず定義。パスワード等の秘密情報は書かない）

---

## 4. インストール手順

### 4-1. pyenvでPythonを入れる

```bash
# pyenv未導入の場合
curl https://pyenv.run | bash
# ~/.bashrc 等にPATH設定を追加した後、シェルを再起動

pyenv install 3.11.9
pyenv local 3.11.9   # リポジトリ直下で実行し .python-version を作成
python --version     # 3.11.9 になっていることを確認
```

### 4-2. uvのインストール

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv --version
```

### 4-3. Python仮想環境とdbtのインストール（uv経由）

リポジトリ直下で:

```bash
uv venv --python 3.11.9      # pyenvで入れた3.11.9を使う
source .venv/bin/activate

uv pip install dbt-core==1.10.* dbt-snowflake==1.10.*
dbt --version
```

`pyproject.toml` に固定しておくと再現性が上がります:

```toml
[project]
name = "analytics-dbt"
version = "0.1.0"
requires-python = ">=3.11,<3.12"
dependencies = [
    "dbt-core==1.10.*",
    "dbt-snowflake==1.10.*",
]
```

```bash
uv sync
```

### 4-4. Snowflake CLI (`snow`) のインストール

同じ仮想環境に入れてもよいですが、他プロジェクトとの競合を避けるため `uv tool` か `pipx` での分離インストールを推奨します。

```bash
# 推奨: uv tool で分離インストール
uv tool install snowflake-cli

# または pipx
pipx install snowflake-cli

snow --version
```

> 旧パッケージ名 `snowflake-cli-labs` は廃止されています。必ず `snowflake-cli` を使ってください。

### 4-5. Snowflake接続設定（snow CLI）

```bash
snow connection add
```

対話形式で以下を入力します。

- Connection name: `myconn`（各自わかりやすい名前）
- Account: 組織のSnowflakeアカウント識別子
- User: 自分のSnowflakeユーザー名
- 認証方式: 鍵ペア認証 or SSO（社内ポリシーに従う）
- Role / Warehouse / Database / Schema: 開発用のデフォルト値

確認:

```bash
snow connection test -c myconn
snow connection set-default myconn
```

---

## 5. Snowflake側の事前準備（開発者ごとの環境）

`snow dbt deploy` でデプロイする際、**profiles.ymlで指定するスキーマは事前にSnowflake上に存在している必要があります**（dbt Coreのようにdbtが自動作成してくれるわけではありません）。

管理者・チームリードが以下を実行して、開発者ごとのスキーマを用意します。

```sql
-- 開発者ごとに用意する例（Ikeda用）
CREATE SCHEMA IF NOT EXISTS ANALYTICS_DEV.DBT_IKEDA;
GRANT ALL ON SCHEMA ANALYTICS_DEV.DBT_IKEDA TO ROLE DBT_DEV_ROLE;
```

命名規則の例: `<DEV_DB>.DBT_<開発者名>`

---

## 6. dbtプロジェクト設定ファイル

### 6-1. `dbt_project/dbt_project.yml`（抜粋）

```yaml
name: 'analytics'
version: '1.0.0'
profile: 'analytics'   # profiles.yml内のプロファイル名と一致させる

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

models:
  analytics:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

### 6-2. `dbt_project/profiles.yml`（Snowflakeデプロイ用・リポジトリにコミット）

```yaml
analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: 'not needed'   # dbt Projects on Snowflake実行時は現在のセッションを使うため不要
      user: 'not needed'
      role: DBT_DEV_ROLE
      warehouse: DBT_DEV_WH
      database: ANALYTICS_DEV
      schema: DBT_IKEDA        # ← 開発者ごとに書き換える、または env.yml で変数化
      threads: 4
    prod:
      type: snowflake
      account: 'not needed'
      user: 'not needed'
      role: DBT_PROD_ROLE
      warehouse: DBT_PROD_WH
      database: ANALYTICS_PROD
      schema: CORE
      threads: 8
```

開発者ごとに `schema` を書き換える運用は事故りやすいため、`env.yml` でスキーマ名を変数化しておくとより安全です（6-3参照）。

### 6-3. `dbt_project/env.yml`（任意・Snowflake CLI 3.21以降）

```yaml
env:
  DBT_SCHEMA: DBT_IKEDA
```

`profiles.yml` 側で `schema: "{{ env_var('DBT_SCHEMA') }}"` のように参照すると、`snow dbt deploy` の `--env-file-dir` / `--default-env` で開発者ごとに切り替えられます。

### 6-4. ローカル実行用 `~/.dbt/profiles.yml`（各自のホームディレクトリ、Git管理外）

```yaml
analytics:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: xy12345.ap-northeast-1
      user: "{{ env_var('DBT_SNOWFLAKE_USER') }}"
      authenticator: externalbrowser   # またはprivate_key_path等、社内ポリシーに従う
      role: DBT_DEV_ROLE
      warehouse: DBT_DEV_WH
      database: ANALYTICS_DEV
      schema: DBT_IKEDA
      threads: 4
```

---

## 7. 日々の開発フロー

### 7-1. 初回セットアップ

```bash
git clone <repo-url>
cd analytics-dbt
pyenv local 3.11.9
uv sync
source .venv/bin/activate

cd dbt_project
dbt deps          # packages.ymlの依存パッケージを取得
dbt debug          # ~/.dbt/profiles.ymlの接続確認
```

### 7-2. ローカルでの通常開発（高速イテレーション）

```bash
# モデルを編集した後
dbt run --select stg_xxx
dbt test --select stg_xxx
dbt build           # run+testをまとめて実行したい場合
```

ローカルではdbt Coreがそのまま自分のSnowflakeセッションから開発者スキーマに対して実行するので、通常のdbt開発と同じ感覚で高速に確認できます。

### 7-3. UT環境（Snowflakeネイティブ実行）への確認デプロイ

ローカルでの確認が済んだら、Snowflake上のdbt Projectオブジェクトとしてもデプロイして最終確認します。

```bash
# リポジトリルートから
snow dbt deploy analytics_ikeda_dev \
  --source dbt_project \
  --profiles-dir dbt_project \
  --default-target dev
```

- `analytics_ikeda_dev` は開発者ごとに一意な DBT PROJECT オブジェクト名（命名規則をチームで決めておく）
- 初回は新規作成、2回目以降は新しいバージョンとして更新されます
- `--force` は基本使わない（実行履歴が消えるため。使うのは意図的に作り直したい時のみ）

デプロイ後、Snowflake上で実行:

```bash
snow dbt execute analytics_ikeda_dev run
snow dbt execute analytics_ikeda_dev test
```

または Snowsight 上の Projects › dbt Projects からも実行・ログ確認が可能です。

### 7-4. `scripts/deploy_dev.sh` の例（毎回のコマンドを短縮）

```bash
#!/usr/bin/env bash
set -euo pipefail

DEV_NAME="${1:?使い方: deploy_dev.sh <あなたの名前 例: ikeda>}"
PROJECT_NAME="analytics_${DEV_NAME}_dev"

snow dbt deploy "${PROJECT_NAME}" \
  --source dbt_project \
  --profiles-dir dbt_project \
  --default-target dev

snow dbt execute "${PROJECT_NAME}" run
snow dbt execute "${PROJECT_NAME}" test
```

```bash
chmod +x scripts/deploy_dev.sh
./scripts/deploy_dev.sh ikeda
```

### 7-5. PRとレビュー

1. featureブランチを切って作業
2. ローカルで `dbt build` が通ることを確認
3. UT環境へ `snow dbt deploy` してSnowflakeネイティブ実行でも確認
4. PR作成 → レビュー
5. マージ後、CI/CD（GitHub Actions等）が `--default-target prod` で本番用 DBT PROJECT オブジェクトへ自動デプロイ

---

## 8. よくあるつまずきポイント

| 症状 | 原因 | 対処 |
|---|---|---|
| `snow dbt deploy` が「スキーマが存在しない」エラー | dbt Projects on Snowflakeはローカルdbt Coreと違いスキーマを自動作成しない | 事前に管理者にスキーマ作成を依頼（5章参照） |
| ローカルでは動くがUT環境で失敗 | `dbt_project/profiles.yml` と `~/.dbt/profiles.yml` の設定差異（role/warehouse等） | 2つのprofiles.ymlの内容を突き合わせる |
| `dbt deps` の結果が反映されない | `dbt_packages/` がデプロイに含まれていない/`--source`指定ミス | デプロイ前に必ず `dbt deps` を実行してから `snow dbt deploy` |
| 誤って `--force` で実行履歴が消えた | `--force` は `CREATE OR REPLACE DBT PROJECT` 相当 | 通常運用では使わない。バージョン更新は `--force` なしで十分 |
| `pyenv` のPythonをuvが拾わない | PATHの優先順位、または `uv venv --python` の指定漏れ | `uv venv --python ~/.pyenv/versions/3.11.9/bin/python` のようにフルパス指定 |

---

## 9. 参考リンク

- dbt Projects on Snowflake概要: https://docs.snowflake.com/en/user-guide/data-engineering/dbt-projects-on-snowflake
- snow dbt deployコマンドリファレンス: https://docs.snowflake.com/en/developer-guide/snowflake-cli/command-reference/dbt-commands/deploy
- スキーマ生成のカスタマイズ: https://docs.snowflake.com/en/user-guide/data-engineering/dbt-projects-on-snowflake-schema-customization
- Snowflake CLIインストール手順: https://docs.snowflake.com/en/developer-guide/snowflake-cli/installation/installation
- ベストプラクティス（env.ymlによる開発者ごとの設定管理など）: https://docs.snowflake.com/en/user-guide/data-engineering/dbt-projects-on-snowflake-best-practices
