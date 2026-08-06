# dbt on Snowflake Project リリース方法 設計書

## 概要

CodeBuild で dbt on Snowflake project のテストを行い、SEED_DB を clone してから `snow dbt deploy` で dbt を実行する CI 構成において、テスト通過後の各環境へのリリース方法を整理する。

## 環境構成

```
[自社 Snowflake アカウント]
  ├─ DEV環境: 開発者が個別に動かす
  ├─ CI環境:  CodeBuild → SEED_DB clone → テスト
  └─ IT環境:  結合テスト (← まずここへのリリース)

[お客さん Snowflake アカウント]
  └─ PROD環境: 本番 (← ゆくゆくはここ)
```

ポイントは Snowflake アカウントが別であること。同一アカウント内の移動とは話が違う。

## リリース方法の選択肢

### 選択肢1: Git ブランチベース (推奨)

各環境が同じ Git リポジトリの異なるブランチを参照する方式。dbt on Snowflake project で最も自然。

```
[Git Repository]
   ├─ feature/* → 開発者ローカル
   ├─ develop   → CI (CodeBuild テスト)
   ├─ release   → IT環境 (結合テスト)
   └─ main      → 本番 (お客さん環境)

[自社 Snowflake]
  CI用 project  → @git_repo/branches/develop/
  IT用 project  → @git_repo/branches/release/

[お客さん Snowflake]
  本番 project  → @git_repo/branches/main/
```

#### リリース手順

1. 開発者: feature → develop へマージ (PR)
2. CodeBuild: develop ブランチで CI テスト (SEED_DB clone → build)
3. CI 通過後: develop → release へマージ
4. IT 環境: `snow dbt deploy` で release ブランチを反映
5. 結合テスト通過後: release → main へマージ
6. 本番: お客さん環境で main ブランチを反映

#### IT 環境へのデプロイ (自社)

```bash
# IT環境のdbt projectを更新
snow sql -c it_env -q "
  ALTER GIT REPOSITORY IT_DBT_DB.INTEGRATIONS.DBT_REPO FETCH;
  
  CREATE OR REPLACE DBT PROJECT IT_DBT_DB.DEPLOYMENTS.DBT_PROJ_IT
    FROM '@IT_DBT_DB.INTEGRATIONS.DBT_REPO/branches/release/dbt_project/'
    EXTERNAL_ACCESS_INTEGRATIONS = (dbt_packages_eai);
"

# IT環境でビルド実行
snow sql -c it_env -q "
  EXECUTE DBT PROJECT IT_DBT_DB.DEPLOYMENTS.DBT_PROJ_IT
    ARGS='build --target it';
"
```

#### 本番へのデプロイ (お客さん環境)

```bash
# お客さん環境のdbt projectを更新
snow sql -c prod_env -q "
  ALTER GIT REPOSITORY PROD_DBT_DB.INTEGRATIONS.DBT_REPO FETCH;
  
  CREATE OR REPLACE DBT PROJECT PROD_DBT_DB.DEPLOYMENTS.DBT_PROJ_PROD
    FROM '@PROD_DBT_DB.INTEGRATIONS.DBT_REPO/branches/main/dbt_project/'
    EXTERNAL_ACCESS_INTEGRATIONS = (dbt_packages_eai);
"
```

#### メリット

- Git が唯一の真実 (Single Source of Truth)
- ブランチ間の差分が明確 (PR diff で何が変わるか一目瞭然)
- dbt on Snowflake project の標準的な使い方と一致
- ロールバックは Git revert → 再デプロイで完結

#### デメリット

- お客さん環境にも Git Repository 連携を設定する必要がある
- お客さんの Snowflake から Git サーバーへの通信経路確保が必要

### 選択肢2: Snowflake Stage 経由 (アーティファクト方式)

CI で検証済みの dbt プロジェクトをファイルとしてパッケージし、Stage に置いてデプロイする方式。

```
[CodeBuild CI 通過]
   ↓
[dbt project を zip/tar]
   ↓
[S3 にアップロード]
   ↓ (自社IT環境)
[Snowflake External Stage → CREATE DBT PROJECT FROM @stage]
   ↓ (お客さん本番)
[ファイルを受け渡し → お客さんの Internal Stage]
```

#### IT 環境へのデプロイ

```bash
# CIで検証済みのプロジェクトを zip
cd dbt_project && zip -r ../dbt-release-v1.2.0.zip . && cd ..

# S3 にアップロード
aws s3 cp dbt-release-v1.2.0.zip s3://dbt-releases/v1.2.0/

# Snowflake External Stage 経由でデプロイ
snow sql -c it_env -q "
  CREATE OR REPLACE DBT PROJECT IT_DBT_DB.DEPLOYMENTS.DBT_PROJ_IT
    FROM '@IT_DBT_DB.INTEGRATIONS.RELEASE_STAGE/v1.2.0/'
    EXTERNAL_ACCESS_INTEGRATIONS = (dbt_packages_eai);
"
```

#### 本番 (お客さん環境) へのデプロイ

```bash
# お客さんに zip を渡す (メール、ファイル共有、S3共有等)
# お客さん側で Internal Stage にアップロード

# お客さん環境
PUT file:///tmp/dbt-release-v1.2.0.zip @PROD_DBT_DB.INTEGRATIONS.RELEASE_STAGE/v1.2.0/;

CREATE OR REPLACE DBT PROJECT PROD_DBT_DB.DEPLOYMENTS.DBT_PROJ_PROD
  FROM '@PROD_DBT_DB.INTEGRATIONS.RELEASE_STAGE/v1.2.0/'
  EXTERNAL_ACCESS_INTEGRATIONS = (dbt_packages_eai);
```

#### メリット

- お客さん環境に Git 連携が不要 (ネットワーク制約が厳しい場合に有効)
- 「何をデプロイしたか」がファイルとして残る
- お客さんがファイルを受け取って自分でデプロイできる (引き渡し運用)

#### デメリット

- ファイル受け渡しの運用が発生
- Git とデプロイ先の乖離が起きやすい (差分管理が自前)
- `dbt deps` で外部パッケージを取得する場合、お客さん環境から外部通信が必要

### 選択肢3: Snowflake CLI (snow dbt) で直接デプロイ

CI で検証済みのリポジトリを、`snow dbt deploy` で直接各環境にデプロイする方式。

```
[CodeBuild CI 通過]
   ↓
[snow dbt deploy --database IT_DBT_DB ...]  → IT環境
   ↓ (結合テスト通過後)
[snow dbt deploy --database PROD_DBT_DB ...]  → お客さん環境
```

```bash
# IT環境へデプロイ
snow dbt deploy dbt_proj_it \
  --connection it_env \
  --database IT_DBT_DB \
  --schema DEPLOYMENTS \
  --source-path ./dbt_project

# 本番 (お客さん環境) へデプロイ
snow dbt deploy dbt_proj_prod \
  --connection prod_env \
  --database PROD_DBT_DB \
  --schema DEPLOYMENTS \
  --source-path ./dbt_project
```

#### メリット

- コマンド1つでデプロイ
- ローカルのソースディレクトリから直接
- Git 連携なしでも動く

#### デメリット

- デプロイ元マシンからお客さん Snowflake への接続が必要
- お客さんの認証情報を自社で扱う必要がある

### 選択肢4: お客さん環境の CodeBuild/CI で自動デプロイ

お客さん側にも CI 環境を作り、main マージをトリガーに自動デプロイする方式。

```
[自社]
  develop → CI通過 → release → IT環境テスト通過
       ↓
  release → main へマージ
       ↓
[お客さん]
  main ブランチ更新検知 → お客さんのCodeBuild → snow dbt deploy
```

#### メリット

- お客さん環境の認証情報を自社で持たなくてよい
- デプロイの責任がお客さん側に移る
- 完全自動化

#### デメリット

- お客さん環境に CI 基盤を作る必要がある
- 初期構築コスト高い

## 選択肢比較

| 観点 | ①Git ブランチ | ②Stage経由 | ③snow CLI直接 | ④お客さんCI |
|---|---|---|---|---|
| IT環境 (自社) へのリリース | ◎ | ○ | ◎ | - |
| 本番 (お客さん) へのリリース | ○ | ◎ | △ | ◎ |
| お客さんGit連携の要否 | 必要 | 不要 | 不要 | 必要 |
| お客さん認証情報の保持 | 不要 | 不要 | 必要 | 不要 |
| ロールバック | Git revert | 旧zip再デプロイ | Git checkout | Git revert |
| 監査追跡 | Git log | ファイル版数 | Git log | Git log |
| 構築の手軽さ | ○ | ○ | ◎ | △ |
| 長期運用の安定性 | ◎ | ○ | △ | ◎ |

## 推奨: 段階的に進める

### Phase 1 (今): 自社IT環境 → 選択肢① + ③ のハイブリッド

```
[Git]
  develop → CodeBuild CI (SEED_DB clone → テスト)
       ↓ CI通過
  develop → release へマージ
       ↓
[自社IT環境]
  snow dbt deploy で直接デプロイ (選択肢③)
  または
  Git Repository 連携で release ブランチを参照 (選択肢①)
```

自社環境なので認証情報もネットワークも自由。最もシンプルな `snow dbt deploy` で始めるのが早い。

### Phase 2 (将来): お客さん本番 → 選択肢① or ②

お客さんの環境制約によって決まる。

```
Q: お客さんの Snowflake から Git サーバーへの通信は可能?

  YES → 選択肢① (Git ブランチベース)
         お客さん環境に Git Repository 連携を設定
         main マージ → お客さんが FETCH → deploy

  NO  → 選択肢② (Stage 経由)
         検証済み zip をお客さんに渡す
         お客さんが PUT → CREATE DBT PROJECT
```

## Git ブランチ運用

```
feature/xxx ──PR──→ develop ──PR──→ release ──PR──→ main
                       ↓              ↓               ↓
                    CI (CodeBuild)  IT (結合テスト)   PROD (本番)
```

## profiles.yml (全環境分)

```yaml
snowflake:
  target: dev
  outputs:
    dev:
      type: snowflake
      database: DEV_DB
      schema: "{{ env_var('USER', 'dev') }}"
      warehouse: DEV_WH
      threads: 4
      query_tag: dbt_dev
    
    ci:
      type: snowflake
      database: "{{ var('target_database') }}"
      schema: marts
      warehouse: CI_WH
      threads: 4
      query_tag: "dbt_ci_{{ var('pr_number', 'unknown') }}"
    
    it:
      type: snowflake
      database: IT_DB
      schema: marts
      warehouse: IT_WH
      threads: 8
      query_tag: dbt_it
    
    prod:
      type: snowflake
      database: PROD_DB
      schema: marts
      warehouse: PROD_WH
      threads: 8
      query_tag: dbt_prod
```

## IT環境デプロイスクリプト (deploy_it.sh)

```bash
#!/usr/bin/env bash
# =============================================================================
# IT環境 (自社結合テスト) へのデプロイ
# releaseブランチのCIが通過した後に手動 or 自動で実行
# =============================================================================
set -euo pipefail

echo "===================================="
echo "Deploying to IT environment"
echo "===================================="

# Secrets Manager から鍵取得
KEY_FILE=$(mktemp /tmp/sf_key.XXXXXX.p8)
trap "shred -u ${KEY_FILE} 2>/dev/null || rm -f ${KEY_FILE}" EXIT

aws secretsmanager get-secret-value \
  --secret-id snowflake/dbt-it \
  --query SecretString --output text \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['private_key'])" \
  > "${KEY_FILE}"
chmod 600 "${KEY_FILE}"

# 接続設定
export SNOWFLAKE_CONNECTIONS_IT_ACCOUNT="${SNOWFLAKE_ACCOUNT}"
export SNOWFLAKE_CONNECTIONS_IT_USER="${SNOWFLAKE_USER}"
export SNOWFLAKE_CONNECTIONS_IT_PRIVATE_KEY_FILE="${KEY_FILE}"
export SNOWFLAKE_CONNECTIONS_IT_ROLE="DBT_IT_ROLE"
export SNOWFLAKE_CONNECTIONS_IT_WAREHOUSE="IT_WH"
export SNOWFLAKE_CONNECTIONS_IT_AUTHENTICATOR="SNOWFLAKE_JWT"

CONN="it"

# 1. dbt project をデプロイ (ソースディレクトリから直接)
echo "[1/3] Deploying dbt project..."
snow dbt deploy dbt_proj_it \
  -c "$CONN" \
  --database IT_DBT_DB \
  --schema DEPLOYMENTS \
  --source-path ./dbt_project \
  --external-access-integrations "dbt_packages_eai"

# 2. seed実行 (IT用テストデータ投入)
echo "[2/3] Running seeds and build..."
snow sql -c "$CONN" -q "
  EXECUTE DBT PROJECT IT_DBT_DB.DEPLOYMENTS.DBT_PROJ_IT
    ARGS='seed --target it';
"

# 3. build実行
echo "[3/3] Running build..."
snow sql -c "$CONN" -q "
  EXECUTE DBT PROJECT IT_DBT_DB.DEPLOYMENTS.DBT_PROJ_IT
    ARGS='build --target it --fail-fast';
"

echo "===================================="
echo "✅ IT deployment complete"
echo "===================================="
```

## CodeBuild buildspec (release ブランチ CI + IT デプロイ)

```yaml
version: 0.2

env:
  secrets-manager:
    SNOWFLAKE_ACCOUNT: snowflake/dbt-it:account
    SNOWFLAKE_USER: snowflake/dbt-it:user
    SNOWFLAKE_PRIVATE_KEY: snowflake/dbt-it:private_key

phases:
  install:
    runtime-versions:
      python: 3.11
    commands:
      - pip install --quiet snowflake-cli==3.5.0
  
  build:
    commands:
      - chmod +x scripts/deploy_it.sh
      - ./scripts/deploy_it.sh
```

## Phase 2: お客さん本番へのリリース (将来)

### パターンA: お客さんが Git 連携可能な場合

```
[自社]
  release テスト通過 → main へマージ

[お客さん]
  Snowflake Git Repository が main を参照
  運用チームが手動で:
    ALTER GIT REPOSITORY prod_repo FETCH;
    CREATE OR REPLACE DBT PROJECT prod_proj
      FROM '@prod_repo/branches/main/dbt_project/';
  
  または Task で自動:
    定期的に FETCH → バージョン確認 → deploy
```

### パターンB: Git 連携できない場合 (閉域ネットワーク等)

```
[自社]
  release テスト通過 → main へマージ
       ↓
  リリース zip 作成 (buildspec で自動)
       ↓
  S3 に配置 or お客さんに送付

[お客さん]
  受領した zip を Internal Stage にアップ
  運用チームが:
    PUT file:///path/dbt-v1.2.0.zip @release_stage/v1.2.0/;
    CREATE OR REPLACE DBT PROJECT prod_proj
      FROM '@release_stage/v1.2.0/dbt_project/';
```

#### リリース zip の自動生成 (buildspec に追加)

```yaml
  post_build:
    commands:
      - VERSION=$(git describe --tags --always)
      - cd dbt_project && zip -r ../dbt-release-${VERSION}.zip . && cd ..
      - aws s3 cp dbt-release-${VERSION}.zip s3://dbt-releases/${VERSION}/

artifacts:
  files:
    - dbt-release-*.zip
```

## お客さんへの引き渡し成果物

将来の本番リリースを見据えて、リリースパッケージとして以下を一緒に渡す運用を推奨する。

```
dbt-release-v1.2.0/
  ├── dbt_project/            # dbtプロジェクト本体
  ├── RELEASE_NOTES.md        # 変更内容
  ├── deploy.sql              # デプロイ用SQL (お客さんが実行)
  ├── rollback.sql            # ロールバック用SQL
  └── test_results.json       # CIテスト結果 (エビデンス)
```

### deploy.sql の例

```sql
-- =============================================================================
-- dbt-release-v1.2.0 デプロイ手順
-- 実行ロール: DBT_PROD_ROLE
-- =============================================================================

-- 1. リリースファイルをStageにアップロード (事前にPUT済み前提)
-- PUT file:///path/dbt-release-v1.2.0.zip
--   @PROD_DBT_DB.DEPLOYMENTS.RELEASE_STAGE/v1.2.0/;

-- 2. 現行バージョンのバックアップ (ロールバック用)
-- ALTER DBT PROJECT PROD_DBT_DB.DEPLOYMENTS.DBT_PROJ_PROD
--   SET TAG PROD_DBT_DB.TAGS.VERSION = 'v1.1.0';

-- 3. 新バージョンをデプロイ
CREATE OR REPLACE DBT PROJECT PROD_DBT_DB.DEPLOYMENTS.DBT_PROJ_PROD
  FROM '@PROD_DBT_DB.DEPLOYMENTS.RELEASE_STAGE/v1.2.0/dbt_project/'
  EXTERNAL_ACCESS_INTEGRATIONS = (dbt_packages_eai)
  COMMENT = 'Release v1.2.0 deployed on YYYY-MM-DD';

-- 4. 疎通確認 (compile のみ)
EXECUTE DBT PROJECT PROD_DBT_DB.DEPLOYMENTS.DBT_PROJ_PROD
  ARGS='compile --target prod';

-- 5. 本番実行
EXECUTE DBT PROJECT PROD_DBT_DB.DEPLOYMENTS.DBT_PROJ_PROD
  ARGS='build --target prod';
```

## まとめ

### 推奨の進め方

```
今 (Phase 1):
  ・Git ブランチ運用 (develop → release → main)
  ・CodeBuild で CI (SEED_DB clone → テスト)
  ・IT環境へは snow dbt deploy で直接デプロイ
  ・まずこのフローを回して安定させる

将来 (Phase 2):
  ・お客さん環境のネットワーク制約を確認
    ├─ Git連携OK → Git Repository 連携 + main ブランチ参照
    └─ Git連携NG → リリース zip + deploy.sql の引き渡し運用
```

### 推奨の理由

1. Git が唯一の真実: どの環境に何がデプロイされているかがブランチで明確
2. 段階的に成熟できる: IT環境で回してからお客さん環境に拡張
3. お客さんの制約に柔軟対応: Git連携可否に応じて2パターン用意
4. お客さんがデプロイを自走できる: deploy.sql とリリースノートを渡すだけで完結
