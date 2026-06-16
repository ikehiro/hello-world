# ECS RunTask × dbt on Snowflake 実装ガイド

## 1. アーキテクチャ概要

### 全体構成

```
AWS Step Functions
  └─ ECS RunTask (Fargate, awsvpc, EIP 固定)
       └─ execute_dbt.py
            ├─ boto3 → Secrets Manager から接続情報を取得
            ├─ cryptography → 秘密鍵をメモリ上でロード（ディスク書き出しなし）
            ├─ snowflake-connector-python → Snowflake へ key-pair 認証で接続
            └─ EXECUTE DBT PROJECT <project_name> ARGS => '<args>'
```

### 設計方針

- dbt プロジェクトは Snowflake 上にデプロイ済み（Git Integration 等）であり、ECS コンテナは「キック役」に徹する
- Snowflake CLI は使用しない。Python connector で直接 SQL を発行する
- 秘密鍵はメモリ上で完結させ、一時ファイルや config.toml をディスクに一切書き出さない
- ECS Fargate の ENI に EIP をアタッチし、Snowflake Network Policy と整合させる

### ディレクトリ構成

```
ecs-dbt-snowflake/
├── .gitignore
├── Dockerfile
├── execute_dbt.py
├── secrets_template.json         ← コミット禁止
├── register_secret.sh
├── task-definition.json
├── iam-task-role-policy.json
└── stepfunctions-definition.json
```

---

## 2. Secrets Manager 設計

### 2.1 登録する JSON 構造

Secrets Manager に 1 つのシークレットとして以下の JSON を登録する。

```json
{
  "account":    "myaccount.ap-northeast-1.aws",
  "user":       "SVC_DBT_ECS",
  "role":       "FUNC_DBT_ROLE",
  "warehouse":  "DBT_WH",
  "database":   "PROD_DB",
  "schema":     "PUBLIC",

  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvAIBADANBgkqhki...(省略)...\n-----END PRIVATE KEY-----",

  "private_key_passphrase": ""
}
```

| キー | 必須 | 説明 |
|------|:----:|------|
| `account` | ✅ | Snowflake アカウント識別子（`<account>.<region>.aws` 形式） |
| `user` | ✅ | サービスユーザー名（TYPE=SERVICE 推奨） |
| `role` | ✅ | 実行ロール名 |
| `warehouse` | ✅ | 使用ウェアハウス名 |
| `database` | ✅ | 接続先データベース名 |
| `schema` | ✅ | 接続先スキーマ名 |
| `private_key` | ✅ | RSA 秘密鍵（PEM 形式、改行は `\n` で表現） |
| `private_key_passphrase` | | 秘密鍵のパスフレーズ（暗号化している場合のみ） |

### 2.2 RSA キーペア生成

```bash
# 秘密鍵（PKCS#8、パスフレーズなし）
openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out snowflake_rsa_key.p8

# 公開鍵
openssl rsa -in snowflake_rsa_key.p8 -pubout -out snowflake_rsa_key.pub
```

### 2.3 Snowflake ユーザーへ公開鍵を登録

```sql
-- ヘッダー/フッター行（-----BEGIN PUBLIC KEY----- 等）を除いた中身を貼り付ける
ALTER USER SVC_DBT_ECS
  SET RSA_PUBLIC_KEY = 'MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...';
```

### 2.4 登録スクリプト：`register_secret.sh`

```bash
#!/bin/bash
# ============================================================
# register_secret.sh
# Secrets Manager へ Snowflake 接続情報を登録する
# ============================================================
set -euo pipefail

SECRET_NAME="${1:-/dbt-snowflake/prod/connection}"
AWS_REGION="${2:-ap-northeast-1}"
KEY_FILE="${3:-./snowflake_rsa_key.p8}"

if [[ ! -f "${KEY_FILE}" ]]; then
  echo "[ERROR] Private key file not found: ${KEY_FILE}"
  echo "Usage: $0 [SECRET_NAME] [REGION] [KEY_FILE]"
  exit 1
fi

# 秘密鍵を JSON 文字列用に整形（改行を \n に変換）
PRIVATE_KEY=$(awk 'NF{printf "%s\\n", $0}' "${KEY_FILE}")

# --- ここを実際の値に書き換えてから実行 ---
SECRET_VALUE=$(python3 -c "
import json
print(json.dumps({
    'account':    'myaccount.ap-northeast-1.aws',
    'user':       'SVC_DBT_ECS',
    'role':       'FUNC_DBT_ROLE',
    'warehouse':  'DBT_WH',
    'database':   'PROD_DB',
    'schema':     'PUBLIC',
    'private_key': '${PRIVATE_KEY}',
    'private_key_passphrase': ''
}))
")

# 既存チェック → 更新 or 新規作成
if aws secretsmanager describe-secret \
    --secret-id "${SECRET_NAME}" \
    --region "${AWS_REGION}" > /dev/null 2>&1; then
  echo "[INFO] Updating existing secret: ${SECRET_NAME}"
  aws secretsmanager put-secret-value \
    --secret-id "${SECRET_NAME}" \
    --secret-string "${SECRET_VALUE}" \
    --region "${AWS_REGION}"
else
  echo "[INFO] Creating new secret: ${SECRET_NAME}"
  aws secretsmanager create-secret \
    --name "${SECRET_NAME}" \
    --description "Snowflake connection for ECS dbt runner" \
    --secret-string "${SECRET_VALUE}" \
    --region "${AWS_REGION}"
fi

echo "[INFO] Done: ${SECRET_NAME}"

# ============================================================
# RSA キーペア新規生成コマンド（参考）
# ============================================================
# openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out snowflake_rsa_key.p8
# openssl rsa -in snowflake_rsa_key.p8 -pubout -out snowflake_rsa_key.pub
#
# Snowflake 側:
# ALTER USER SVC_DBT_ECS SET RSA_PUBLIC_KEY='<公開鍵の中身>';
```

---

## 3. コンテナ実装

### 3.1 Dockerfile

```dockerfile
# ============================================================
# ECS RunTask: dbt on Snowflake 実行コンテナ（Python版）
# snowflake-connector-python で接続し EXECUTE DBT PROJECT を実行
# 秘密鍵はメモリ上で完結（ディスクに一切書き出さない）
# ============================================================
FROM python:3.12-slim

RUN pip install --no-cache-dir \
    snowflake-connector-python[secure-local-storage] \
    cryptography \
    boto3

# 非 root ユーザーで実行
RUN useradd -m -u 1000 -s /bin/bash runner
USER runner
WORKDIR /home/runner

COPY --chown=runner:runner execute_dbt.py ./execute_dbt.py

ENTRYPOINT ["python3", "execute_dbt.py"]
```

### 3.2 メイン実行スクリプト：`execute_dbt.py`

```python
"""
execute_dbt.py
ECS RunTask で Snowflake に接続し EXECUTE DBT PROJECT を実行する。

セキュリティ設計:
  - 秘密鍵はメモリ上でのみ扱い、ディスクに一切書き出さない
  - Secrets Manager からランタイムに取得し、プロセス終了とともに消滅する
"""

import json
import logging
import os
import sys
import time

import boto3
import snowflake.connector
from cryptography.hazmat.primitives import serialization

# ============================================================
# ログ設定
# ============================================================
logging.basicConfig(
    level=logging.INFO,
    format="[%(levelname)s] %(asctime)s %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger(__name__)


# ============================================================
# 環境変数
# ============================================================
def get_env(key: str, required: bool = True, default: str = "") -> str:
    value = os.environ.get(key, default)
    if required and not value:
        logger.error(f"Required environment variable is missing: {key}")
        sys.exit(1)
    return value


SECRET_ARN = get_env("SECRET_ARN")
AWS_REGION = get_env("AWS_REGION")
DBT_PROJECT_NAME = get_env("DBT_PROJECT_NAME")
DBT_COMMAND = get_env("DBT_COMMAND", required=False, default="run")
DBT_SELECT = get_env("DBT_SELECT", required=False, default="")


# ============================================================
# Secrets Manager からシークレット取得
# ============================================================
def fetch_secret(secret_arn: str, region: str) -> dict:
    logger.info("Fetching secret from Secrets Manager...")
    client = boto3.client("secretsmanager", region_name=region)
    resp = client.get_secret_value(SecretId=secret_arn)
    secret = json.loads(resp["SecretString"])

    required_keys = ["account", "user", "private_key", "role", "warehouse", "database", "schema"]
    missing = [k for k in required_keys if k not in secret]
    if missing:
        logger.error(f"Missing keys in secret: {missing}")
        sys.exit(1)

    logger.info("Secret retrieved successfully.")
    return secret


# ============================================================
# 秘密鍵をメモリ上でロード（ディスク書き出しなし）
# ============================================================
def load_private_key(pem_text: str, passphrase: str = ""):
    # JSON 内の \\n を実際の改行に変換
    pem_bytes = pem_text.replace("\\n", "\n").encode("utf-8")

    password = passphrase.encode("utf-8") if passphrase else None

    private_key = serialization.load_pem_private_key(pem_bytes, password=password)

    # snowflake-connector-python が受け付ける DER 形式に変換
    private_key_der = private_key.private_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption(),
    )

    logger.info("Private key loaded in memory (no disk write).")
    return private_key_der


# ============================================================
# Snowflake 接続
# ============================================================
def create_connection(secret: dict, private_key_der: bytes):
    logger.info(
        f"Connecting to Snowflake... "
        f"account={secret['account']}, user={secret['user']}, "
        f"role={secret['role']}, warehouse={secret['warehouse']}"
    )

    conn = snowflake.connector.connect(
        account=secret["account"],
        user=secret["user"],
        private_key=private_key_der,
        role=secret["role"],
        warehouse=secret["warehouse"],
        database=secret["database"],
        schema=secret["schema"],
    )

    logger.info("Connected to Snowflake.")
    return conn


# ============================================================
# dbt project 実行
# ============================================================
def execute_dbt(conn, project_name: str, command: str, select: str):
    # EXECUTE DBT PROJECT の SQL を組み立て
    # args は dbt CLI 引数形式で渡す
    args_parts = [command]
    if select:
        args_parts.extend(["--select", select])
    args_str = " ".join(args_parts)

    sql = f"EXECUTE DBT PROJECT {project_name} ARGS => '{args_str}'"

    logger.info("=" * 60)
    logger.info(f"Executing: {sql}")
    logger.info("=" * 60)

    start_time = time.time()

    cur = conn.cursor()
    try:
        cur.execute(sql)
        results = cur.fetchall()

        elapsed = time.time() - start_time
        logger.info(f"Execution completed in {elapsed:.1f}s")

        # 結果の出力
        if results:
            columns = [desc[0] for desc in cur.description] if cur.description else []
            if columns:
                logger.info(f"Columns: {columns}")
            for row in results:
                logger.info(f"  {row}")

        return results

    finally:
        cur.close()


# ============================================================
# 接続テスト
# ============================================================
def test_connection(conn):
    logger.info("Testing Snowflake connection...")
    cur = conn.cursor()
    try:
        cur.execute("SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA()")
        row = cur.fetchone()
        logger.info(
            f"  Role={row[0]}, Warehouse={row[1]}, "
            f"Database={row[2]}, Schema={row[3]}"
        )
    finally:
        cur.close()
    logger.info("Connection test passed.")


# ============================================================
# メイン
# ============================================================
def main():
    logger.info("=" * 60)
    logger.info("ECS dbt on Snowflake Runner (Python)")
    logger.info(f"  DBT_PROJECT_NAME : {DBT_PROJECT_NAME}")
    logger.info(f"  DBT_COMMAND      : {DBT_COMMAND}")
    logger.info(f"  DBT_SELECT       : {DBT_SELECT or '<all models>'}")
    logger.info("=" * 60)

    # 1. シークレット取得
    secret = fetch_secret(SECRET_ARN, AWS_REGION)

    # 2. 秘密鍵をメモリ上でロード
    private_key_der = load_private_key(
        secret["private_key"],
        secret.get("private_key_passphrase", ""),
    )

    # 3. Snowflake 接続
    conn = create_connection(secret, private_key_der)

    try:
        # 4. 接続テスト
        test_connection(conn)

        # 5. dbt project 実行
        execute_dbt(conn, DBT_PROJECT_NAME, DBT_COMMAND, DBT_SELECT)

    finally:
        conn.close()
        logger.info("Snowflake connection closed.")

    logger.info("=" * 60)
    logger.info("dbt project completed successfully.")
    logger.info("=" * 60)


if __name__ == "__main__":
    try:
        main()
    except snowflake.connector.errors.ProgrammingError as e:
        logger.error(f"Snowflake SQL error: {e}")
        sys.exit(1)
    except snowflake.connector.errors.DatabaseError as e:
        logger.error(f"Snowflake connection error: {e}")
        sys.exit(1)
    except Exception as e:
        logger.error(f"Unexpected error: {e}", exc_info=True)
        sys.exit(1)
```

---

## 4. ECS タスク定義

### 4.1 タスク定義：`task-definition.json`

```json
{
  "family": "dbt-snowflake-runner",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",

  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn":      "arn:aws:iam::123456789012:role/dbt-snowflake-task-role",

  "containerDefinitions": [
    {
      "name": "dbt-runner",
      "image": "123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/dbt-snowflake-runner:latest",
      "essential": true,

      "environment": [
        { "name": "AWS_REGION",       "value": "ap-northeast-1" },
        { "name": "DBT_PROJECT_NAME", "value": "MY_DBT_PROJECT" },
        { "name": "DBT_COMMAND",      "value": "run" },
        { "name": "DBT_SELECT",       "value": "" }
      ],

      "secrets": [
        {
          "name":      "SECRET_ARN",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:/dbt-snowflake/prod/connection"
        }
      ],

      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group":         "/ecs/dbt-snowflake-runner",
          "awslogs-region":        "ap-northeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      },

      "readonlyRootFilesystem": false,
      "privileged": false,
      "user": "1000"
    }
  ]
}
```

### 4.2 環境変数一覧

| 変数名 | 必須 | 説明 | 例 |
|--------|:----:|------|-----|
| `AWS_REGION` | ✅ | Secrets Manager のリージョン | `ap-northeast-1` |
| `SECRET_ARN` | ✅ | Secrets Manager の ARN | `arn:aws:secretsmanager:...` |
| `DBT_PROJECT_NAME` | ✅ | Snowflake 上の dbt プロジェクト名 | `MY_DBT_PROJECT` |
| `DBT_COMMAND` | | dbt コマンド（デフォルト: `run`） | `run` / `test` / `build` |
| `DBT_SELECT` | | 実行対象モデルの絞り込み | `staging.users` / `tag:daily` |

---

## 5. IAM 設計

### 5.1 タスクロール用ポリシー：`iam-task-role-policy.json`

ECS タスクロール（`dbt-snowflake-task-role`）にアタッチするポリシー。Secrets Manager の GetSecretValue と KMS の Decrypt のみを許可する最小権限設計。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GetSnowflakeSecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:/dbt-snowflake/prod/connection*"
      ]
    },
    {
      "Sid": "DecryptSecret",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": [
        "arn:aws:kms:ap-northeast-1:123456789012:key/<KMS_KEY_ID>"
      ],
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.ap-northeast-1.amazonaws.com"
        }
      }
    }
  ]
}
```

### 5.2 ロール構成図

```
ecsTaskExecutionRole（実行ロール）
  └─ ECR イメージの pull
  └─ CloudWatch Logs への書き込み
  └─ Secrets Manager からの SECRET_ARN 値の取得（ECS Agent 用）

dbt-snowflake-task-role（タスクロール）
  └─ secretsmanager:GetSecretValue（接続情報の取得）
  └─ kms:Decrypt（シークレット復号、KMS 暗号化時のみ）
```

---

## 6. Step Functions 定義

### 6.1 ステートマシン定義：`stepfunctions-definition.json`

```json
{
  "Comment": "dbt on Snowflake - ECS RunTask オーケストレーション",
  "StartAt": "RunDbtProject",

  "States": {
    "RunDbtProject": {
      "Type": "Task",
      "Resource": "arn:aws:states:::ecs:runTask.sync",
      "Parameters": {
        "LaunchType": "FARGATE",
        "Cluster": "arn:aws:ecs:ap-northeast-1:123456789012:cluster/dbt-cluster",
        "TaskDefinition": "arn:aws:ecs:ap-northeast-1:123456789012:task-definition/dbt-snowflake-runner:1",
        "NetworkConfiguration": {
          "AwsvpcConfiguration": {
            "Subnets": [
              "subnet-xxxxxxxxxxxxxxxxx"
            ],
            "SecurityGroups": [
              "sg-xxxxxxxxxxxxxxxxx"
            ],
            "AssignPublicIp": "DISABLED"
          }
        },
        "Overrides": {
          "ContainerOverrides": [
            {
              "Name": "dbt-runner",
              "Environment": [
                {
                  "Name": "DBT_PROJECT_NAME",
                  "Value.$": "$.dbt_project_name"
                },
                {
                  "Name": "DBT_COMMAND",
                  "Value.$": "$.dbt_command"
                },
                {
                  "Name": "DBT_SELECT",
                  "Value.$": "$.dbt_select"
                }
              ]
            }
          ]
        }
      },
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 30,
          "MaxAttempts": 2,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "NotifyFailure",
          "ResultPath": "$.error"
        }
      ],
      "Next": "NotifySuccess"
    },

    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:ap-northeast-1:123456789012:dbt-notifications",
        "Message.$": "States.Format('dbt project {} completed successfully.', $.dbt_project_name)"
      },
      "End": true
    },

    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:ap-northeast-1:123456789012:dbt-notifications",
        "Message.$": "States.Format('dbt project {} FAILED. Error: {}', $.dbt_project_name, $.error)"
      },
      "Next": "JobFailed"
    },

    "JobFailed": {
      "Type": "Fail",
      "Error": "DbtExecutionFailed",
      "Cause": "ECS task for dbt project returned non-zero exit code."
    }
  }
}
```

### 6.2 実行入力例

フル実行：

```json
{
  "dbt_project_name": "MY_DBT_PROJECT",
  "dbt_command": "run",
  "dbt_select": ""
}
```

特定モデルのみ実行：

```json
{
  "dbt_project_name": "MY_DBT_PROJECT",
  "dbt_command": "run",
  "dbt_select": "tag:daily"
}
```

テストのみ実行：

```json
{
  "dbt_project_name": "MY_DBT_PROJECT",
  "dbt_command": "test",
  "dbt_select": ""
}
```

---

## 7. デプロイ手順

### 7.1 Secrets Manager 登録

```bash
chmod +x register_secret.sh
./register_secret.sh /dbt-snowflake/prod/connection ap-northeast-1 ./snowflake_rsa_key.p8
```

### 7.2 ECR リポジトリ作成 & イメージ push

```bash
ACCOUNT_ID=123456789012
REGION=ap-northeast-1
REPO=dbt-snowflake-runner

# リポジトリ作成
aws ecr create-repository --repository-name ${REPO} --region ${REGION}

# ECR ログイン
aws ecr get-login-password --region ${REGION} \
  | docker login --username AWS \
    --password-stdin ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

# ビルド & push
docker build -t ${REPO} .
docker tag ${REPO}:latest ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${REPO}:latest
docker push ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${REPO}:latest
```

### 7.3 ECS タスク定義登録

```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --region ap-northeast-1
```

### 7.4 ローカル動作確認

```bash
docker run --rm \
  -e AWS_REGION=ap-northeast-1 \
  -e SECRET_ARN=arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:/dbt-snowflake/prod/connection \
  -e DBT_PROJECT_NAME=MY_DBT_PROJECT \
  -e DBT_COMMAND=run \
  -e DBT_SELECT="" \
  -e AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}" \
  -e AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}" \
  -e AWS_SESSION_TOKEN="${AWS_SESSION_TOKEN}" \
  dbt-snowflake-runner:latest
```

---

## 8. .gitignore

```gitignore
# 秘密鍵・認証情報
*.p8
*.pem
snowflake_rsa_key.*
secrets_template.json

# Python
__pycache__/
*.pyc

# IDE
.vscode/
.idea/
```

---

## 9. セキュリティ設計まとめ

| 観点 | 実装内容 |
|------|----------|
| 秘密鍵 | メモリ上のみ。PEM → DER 変換もメモリ内で完結 |
| 設定ファイル | config.toml 不要（Snowflake CLI を使用しない） |
| 一時ファイル | 一切生成しない。trap/cleanup も不要 |
| コンテナ実行ユーザー | 非 root（UID 1000） |
| IAM ポリシー | SecretsManager:GetSecretValue + KMS:Decrypt のみ（最小権限） |
| ネットワーク | Fargate ENI + EIP → Snowflake Network Policy に IP 登録 |
| 認証方式 | key-pair 認証（パスワード認証は使用しない） |
| シークレットのスコープ | 1 シークレット = 1 環境（prod/dev で分離） |

### データフロー

```
[ECS タスクロール]
      │
      ▼
[Secrets Manager] ──取得──► [execute_dbt.py メモリ空間]
                                    │
                         PEM → DER 変換（メモリ内）
                                    │
                                    ▼
                            [Snowflake 接続]
                                    │
                         EXECUTE DBT PROJECT
                                    │
                                    ▼
                            [結果ログ出力]
                                    │
                         プロセス終了 → メモリ解放
```
