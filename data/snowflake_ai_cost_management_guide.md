# Snowflake AI 利用料 監視・制限 完全ガイド

> 作成日: 2026-06  
> 対象バージョン: Snowflake 2026年6月時点の機能に基づく  
> 対象読者: Snowflake 管理者・データエンジニア

---

## 目次

1. [Snowflake AI 機能の全体像](#1-snowflake-ai-機能の全体像)
2. [コスト管理の3つの仕組みを理解する](#2-コスト管理の3つの仕組みを理解する)
3. [Resource Monitor vs Budget の違い](#3-resource-monitor-vs-budget-の違い)
4. [監視ビュー一覧（どのビューを見るか）](#4-監視ビュー一覧どのビューを見るか)
5. [セットアップ：本番用 SQL（STEP 1〜8）](#5-セットアップ本番用-sqlstep-18)
6. [日常監視クエリ集](#6-日常監視クエリ集)
7. [運用手順（アラート後の対応フロー）](#7-運用手順アラート後の対応フロー)
8. [重要な注意点・落とし穴](#8-重要な注意点落とし穴)

---

## 1. Snowflake AI 機能の全体像

Snowflakeのすべてのネイティブ AI 機能は **「Snowflake Cortex」** ブランドの傘下にあり、大きく4つの柱で構成されます。

### 柱1：Cortex AI Functions（LLM 関数群）

SQLから直接呼び出せるサーバーレス AI 関数。**トークン課金**が基本。

| カテゴリ | 関数名 | 概要 |
|---|---|---|
| 生成系 | `AI_COMPLETE` | 任意の LLM へのプロンプト実行（汎用） |
| 生成系 | `AI_SUMMARIZE` | テキスト要約 |
| 生成系 | `AI_TRANSLATE` | 翻訳 |
| 分類・分析 | `AI_CLASSIFY` | テキスト/画像の分類 |
| 分類・分析 | `AI_SENTIMENT` | センチメント分析 |
| 分類・分析 | `AI_EXTRACT` | 構造化情報の抽出 |
| マルチモーダル | `AI_TRANSCRIBE` | 音声・動画の文字起こし |
| マルチモーダル | `AI_SIMILARITY` | 埋め込みベクトルの類似度計算 |
| セキュリティ | `AI_REDACT` | PII 自動検出・マスキング |
| 埋め込み | `AI_EMBED` | テキスト/画像のベクトル化 |

> ⚠️ **課金ポイント**：`AI_COMPLETE` 等の生成系は**入力＋出力トークン両方**に課金。モデルによって単価が大幅に異なる（例：Claude Opus 系 = 12 クレジット/100万トークン）。

---

### 柱2：Cortex サービス群（高レイヤー）

| サービス | 概要 | 課金形態 |
|---|---|---|
| **Cortex Search** | セマンティック検索（RAG 基盤） | インデックスサイズ (GB/月) × クエリ数 |
| **Cortex Analyst** | 自然言語 → SQL 変換（REST API） | トークン課金 |
| **Cortex Agents** | 複数 LLM 呼び出しを束ねるオーケストレーター | トークン課金（複数回分） |
| **Snowflake Intelligence** | ビジネスユーザー向けデータエージェント | トークン課金 |
| **Cortex Fine-tuning** | カスタムモデルの作成 | GPU 計算時間 |

> ⚠️ **Cortex Search** はクエリゼロでもインデックスサイズ (GB/月) で課金される「アイドル税」あり。

---

### 柱3：Snowflake ML（機械学習）

| 機能 | 概要 |
|---|---|
| ML Functions | 予測・異常検知・貢献度分析（コード不要） |
| Feature Store | ML モデル向けフィーチャー管理 |
| Snowpark ML | Python/Java でのカスタムモデル開発 |

---

### 柱4：開発者向けアシスト

| 機能 | 概要 | Snowsight での場所 |
|---|---|---|
| **Cortex Code** | SQL 補完・自然言語アシスト（旧 Copilot） | ワークシート右側チャット（✦アイコン） |
| Cortex Code CLI | ターミナル/VS Code からの操作 | CLI |
| Document AI | PDF 等ドキュメントの構造化抽出 | AI & ML メニュー |

> **Snowsight の右側チャットパネル = Cortex Code**（旧称 Snowflake Copilot から 2025年11月に移行）。トークン課金が発生するため監視対象。

---

## 2. コスト管理の3つの仕組みを理解する

```
┌─────────────────────────────────────────────────────┐
│              Snowflake コスト管理の全体像              │
├──────────────────┬──────────────────┬────────────────┤
│ Resource Monitor │     Budget       │  Alert + Task  │
├──────────────────┼──────────────────┼────────────────┤
│ WH のみ対応      │ WH + AI サービス │ AI Functions   │
│ WH 停止が得意    │ 通知・予算管理   │ 即時検知向け   │
│ AI には効かない  │ WH 停止は間接的  │ 自前実装が必要 │
└──────────────────┴──────────────────┴────────────────┘
```

**推奨の組み合わせ：**

```
Resource Monitor  → WH の従来通りのコスト管理（既存設定を維持）
      +
Custom Budget     → AI 専用の月次予算監視・Email 通知
      +
Custom Action     → 90% 到達時に WH 停止（Stored Procedure 経由）
      +
監視クエリ / Task → 日次・リアルタイムの消費確認
```

---

## 3. Resource Monitor vs Budget の違い

| | **Resource Monitor** | **Budget** |
|---|---|---|
| **対象** | Virtual Warehouse のみ | WH ＋ サーバーレス・AI サービス全般 |
| **主な用途** | WH のクレジット消費を監視・停止 | アカウント全体またはグループ単位の予算管理 |
| **WH 停止** | ✅ ネイティブに可能 | ⚠️ Custom Action (Stored Procedure) 経由で可能 |
| **AI Functions 制御** | ❌ 効かない（サーバーレスのため） | ✅ 監視できる |
| **Snowsight の場所** | Admin → Cost Management → Resource Monitors | Admin → Cost Management → Budgets |
| **SQL コマンド** | `CREATE RESOURCE MONITOR` | `CREATE SNOWFLAKE.CORE.BUDGET` |

> **重要**：Snowsight の「90%で WH を止める」設定は **Resource Monitor** です。Budget とは別物。AI のトークン課金には Resource Monitor は効きません。

---

## 4. 監視ビュー一覧（どのビューを見るか）

すべて `SNOWFLAKE.ACCOUNT_USAGE` スキーマ配下。

| ビュー名 | 対象機能 | データ開始日 | 主なカラム |
|---|---|---|---|
| `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` | AI_COMPLETE 等の SQL 関数 | 2026/1/5〜 | FUNCTION_NAME, MODEL_NAME, CREDITS, QUERY_ID, USER_ID |
| `CORTEX_AGENT_USAGE_HISTORY` | Cortex Agents | 2025/11/10〜 | Tokens, Tools |
| `CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY` | Cortex Code（Snowsight チャット） | 2026/2/16〜 | INPUT_TOKENS, OUTPUT_TOKENS |
| `CORTEX_CODE_CLI_USAGE_HISTORY` | Cortex Code（CLI） | 2026/2/16〜 | INPUT_TOKENS, OUTPUT_TOKENS |

> `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` は最大10分遅延（最短5分で反映）。

---

## 5. セットアップ：本番用 SQL（STEP 1〜8）

### 変数定義（最初にここだけ変更する）

```sql
-- ★ ここを環境に合わせて変更 ★
SET AI_BUDGET_DB         = 'MGMT_DB';           -- 管理用 DB
SET AI_BUDGET_SCHEMA     = 'BUDGET_SCHEMA';      -- 管理用 Schema
SET AI_BUDGET_WH         = 'MGMT_WH';           -- Budget 処理用 WH（小サイズで可）
SET AI_TARGET_WH         = 'AI_WH';             -- AI クエリを実行している WH（停止対象）
SET NOTIFY_EMAIL         = 'admin@company.com';  -- 通知先メール（複数の場合はカンマ区切り）
SET MONTHLY_CREDIT_LIMIT = 500;                  -- AI 専用の月間クレジット上限
```

---

### STEP 1：専用ロールの作成

```sql
USE ROLE ACCOUNTADMIN;

-- Budget 管理者ロール（作成・設定変更）
CREATE ROLE IF NOT EXISTS AI_BUDGET_ADMIN;
GRANT APPLICATION ROLE SNOWFLAKE.BUDGET_ADMIN   TO ROLE AI_BUDGET_ADMIN;
GRANT APPLICATION ROLE SNOWFLAKE.BUDGET_CREATOR TO ROLE AI_BUDGET_ADMIN;
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE AI_BUDGET_ADMIN;

-- Budget 閲覧者ロール（参照のみ）
CREATE ROLE IF NOT EXISTS AI_BUDGET_MONITOR;
GRANT APPLICATION ROLE SNOWFLAKE.BUDGET_VIEWER  TO ROLE AI_BUDGET_MONITOR;
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE AI_BUDGET_MONITOR;

-- ロールを自分自身に付与（以降の操作を続けるため）
SET MY_USER = CURRENT_USER();
GRANT ROLE AI_BUDGET_ADMIN   TO USER IDENTIFIER($MY_USER);
GRANT ROLE AI_BUDGET_MONITOR TO USER IDENTIFIER($MY_USER);
```

---

### STEP 2：管理用 DB / Schema の作成と権限付与

```sql
USE ROLE ACCOUNTADMIN;

CREATE DATABASE IF NOT EXISTS IDENTIFIER($AI_BUDGET_DB);
CREATE SCHEMA  IF NOT EXISTS IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA);

-- AI_BUDGET_ADMIN ロールへの権限付与
GRANT USAGE ON DATABASE IDENTIFIER($AI_BUDGET_DB)
    TO ROLE AI_BUDGET_ADMIN;
GRANT USAGE ON SCHEMA IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA)
    TO ROLE AI_BUDGET_ADMIN;
GRANT CREATE SNOWFLAKE.CORE.BUDGET
    ON SCHEMA IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA)
    TO ROLE AI_BUDGET_ADMIN;
GRANT CREATE PROCEDURE
    ON SCHEMA IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA)
    TO ROLE AI_BUDGET_ADMIN;

-- WH 停止権限（Custom Action が WH を Suspend するために必要）
GRANT OPERATE ON WAREHOUSE IDENTIFIER($AI_TARGET_WH) TO ROLE AI_BUDGET_ADMIN;
```

---

### STEP 3：Notification Integration の作成（Email 通知）

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE NOTIFICATION INTEGRATION AI_BUDGET_EMAIL_INT
    TYPE               = EMAIL
    ENABLED            = TRUE
    ALLOWED_RECIPIENTS = ($NOTIFY_EMAIL);
    -- 複数の場合: ALLOWED_RECIPIENTS = ('a@co.com', 'b@co.com')

-- Snowflake アプリケーションが Integration を使えるように権限付与
GRANT USAGE ON INTEGRATION AI_BUDGET_EMAIL_INT TO APPLICATION SNOWFLAKE;
```

> ⚠️ **前提**：`ALLOWED_RECIPIENTS` に指定するメールアドレスは、Snowflake ユーザーとして登録され **Email Verification 済み**であること。未確認アドレスが1件でもあると通知設定全体が失敗します。

---

### STEP 4：Account Budget（アカウント全体）の有効化

```sql
-- すでに有効化済みの場合はこの STEP をスキップ
USE ROLE ACCOUNTADMIN;

-- Account Budget を有効化
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!ACTIVATE();

-- アカウント全体の月間上限クレジット（AI 専用上限は STEP 5 で設定）
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_SPENDING_LIMIT(5000);

-- 通知先メール設定
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_EMAIL_NOTIFICATIONS(
    'AI_BUDGET_EMAIL_INT',
    $NOTIFY_EMAIL
);

-- 通知閾値：予測消費が上限の 80% を超えたら通知
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!SET_NOTIFICATION_THRESHOLD(80);
```

> **Account Budget** はアカウント全体の大枠予算（WH + AI + その他すべて含む）。AI 専用の細かい管理は次の Custom Budget で行います。

---

### STEP 5：AI 専用 Custom Budget の作成

```sql
USE ROLE AI_BUDGET_ADMIN;
USE SCHEMA IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA);

-- Custom Budget 作成
CREATE SNOWFLAKE.CORE.BUDGET IF NOT EXISTS AI_COST_BUDGET();

-- 月間上限クレジット設定（AI 専用）
CALL AI_COST_BUDGET!SET_SPENDING_LIMIT($MONTHLY_CREDIT_LIMIT);

-- 通知先設定
CALL AI_COST_BUDGET!SET_EMAIL_NOTIFICATIONS(
    'AI_BUDGET_EMAIL_INT',
    $NOTIFY_EMAIL
);

-- 通知閾値：80% で Email 通知
CALL AI_COST_BUDGET!SET_NOTIFICATION_THRESHOLD(80);

-- AI クエリを実行する WH を監視対象に追加
-- （AI Functions はサーバーレスだが、WH を紐づけることで
--   WH コストも含めた総合的なコスト管理が可能になる）
CALL AI_COST_BUDGET!ADD_RESOURCE(
    SYSTEM$REFERENCE('WAREHOUSE', $AI_TARGET_WH, 'SESSION', 'true')
);

-- 設定確認
CALL AI_COST_BUDGET!GET_SPENDING_LIMIT();
CALL AI_COST_BUDGET!GET_NOTIFICATION_THRESHOLD();
```

---

### STEP 6：WH 停止 ＋ 通知用 Stored Procedure の作成

Budget の Custom Action から呼び出される Stored Procedure です。

**要件（必須）：**
- `EXECUTE AS OWNER`（caller's rights は不可）
- OUTPUT 引数なし
- 30 分以内に完了
- 冪等性を考慮した設計（Snowflake がリトライするため）

```sql
USE ROLE AI_BUDGET_ADMIN;
USE SCHEMA IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA);

CREATE OR REPLACE PROCEDURE SP_SUSPEND_AI_WH_AND_NOTIFY(
    INTEGRATION_NAME  STRING,
    EMAIL_LIST        STRING,
    WH_NAME           STRING,
    THRESHOLD_PCT     NUMBER
)
RETURNS STRING
LANGUAGE JAVASCRIPT
EXECUTE AS OWNER
AS
$$
try {
    // 1. WH を即時停止
    var suspend_sql = "ALTER WAREHOUSE IDENTIFIER('" + WH_NAME + "') SUSPEND";
    snowflake.execute({ sqlText: suspend_sql });

    // 2. 現在の AI 消費クレジットを取得（当月分）
    var credit_sql = `
        SELECT COALESCE(SUM(CREDITS), 0) AS total_credits
        FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
        WHERE START_TIME >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
    `;
    var stmt    = snowflake.createStatement({ sqlText: credit_sql });
    var rs      = stmt.execute();
    rs.next();
    var credits = rs.getColumnValue(1);

    // 3. メール通知
    var subject = '[Snowflake] AI Budget ' + THRESHOLD_PCT + '% 到達 - WH を停止しました';
    var body    =
        'Snowflake AI 利用料が月次予算の ' + THRESHOLD_PCT + '% に達しました。\n\n' +
        '対象 WH    : ' + WH_NAME + '\n' +
        '当月 AI 消費: ' + credits + ' クレジット\n\n' +
        '対応が完了したら以下を実行して WH を再開してください:\n' +
        '  ALTER WAREHOUSE ' + WH_NAME + ' RESUME;\n\n' +
        '詳細確認クエリ:\n' +
        '  SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY\n' +
        '  WHERE START_TIME >= DATE_TRUNC(\'month\', CURRENT_TIMESTAMP())\n' +
        '  ORDER BY START_TIME DESC LIMIT 50;';

    var email_sql =
        "CALL SYSTEM$SEND_EMAIL('" + INTEGRATION_NAME + "', " +
        "'" + EMAIL_LIST + "', " +
        "'" + subject.replace(/'/g, "''") + "', " +
        "'" + body.replace(/'/g, "''") + "')";
    snowflake.execute({ sqlText: email_sql });

    return 'SUCCESS: WH [' + WH_NAME + '] を停止し、メールを送信しました。AI 消費: ' + credits + ' credits';

} catch (err) {
    return 'ERROR: ' + err.message;
}
$$;

-- Snowflake アプリケーションに Procedure の実行権限を付与
GRANT USAGE ON DATABASE IDENTIFIER($AI_BUDGET_DB)
    TO APPLICATION SNOWFLAKE;
GRANT USAGE ON SCHEMA IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA)
    TO APPLICATION SNOWFLAKE;
GRANT USAGE ON PROCEDURE
    IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA
        || '.SP_SUSPEND_AI_WH_AND_NOTIFY(STRING, STRING, STRING, NUMBER)')
    TO APPLICATION SNOWFLAKE;
```

---

### STEP 7：Custom Action の登録（90% で WH 停止）

```sql
USE ROLE AI_BUDGET_ADMIN;
USE SCHEMA IDENTIFIER($AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA);

-- TRIGGER TYPE の選択：
--   'ACTUAL'    = 実際の消費が閾値を超えたとき（月1回のみ実行）← AI 暴走対策に推奨
--   'PROJECTED' = 予測消費が閾値を超えたとき  （日1回のみ実行）← 事前警告向き

CALL AI_COST_BUDGET!ADD_CUSTOM_ACTION(
    SYSTEM$REFERENCE(
        'PROCEDURE',
        $AI_BUDGET_DB || '.' || $AI_BUDGET_SCHEMA
            || '.SP_SUSPEND_AI_WH_AND_NOTIFY(string, string, string, number)'
    ),
    ARRAY_CONSTRUCT(
        'AI_BUDGET_EMAIL_INT',  -- 引数1: Notification Integration 名
        $NOTIFY_EMAIL,          -- 引数2: 通知先メール
        $AI_TARGET_WH,          -- 引数3: 停止対象 WH 名
        90                      -- 引数4: 閾値（%）
    ),
    'ACTUAL',  -- 実際の消費ベースでトリガー
    90         -- 90% で発動
);

-- 設定確認：登録済み Custom Action の一覧
CALL AI_COST_BUDGET!GET_CUSTOM_ACTIONS();
```

---

## 6. 日常監視クエリ集

### [Q1] 当月の AI 機能別クレジット消費サマリ

```sql
SELECT
    FUNCTION_NAME,
    MODEL_NAME,
    SUM(CREDITS)             AS total_credits,
    COUNT(DISTINCT QUERY_ID) AS query_count,
    ROUND(
        SUM(CREDITS) /
        NULLIF((
            SELECT SUM(CREDITS)
            FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
            WHERE START_TIME >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
        ), 0) * 100, 1
    ) AS pct_of_total
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
WHERE START_TIME >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY total_credits DESC;
```

---

### [Q2] 当月のユーザー別クレジット消費 TOP 10

```sql
SELECT
    u.NAME            AS user_name,
    u.EMAIL,
    SUM(h.CREDITS)    AS total_credits,
    COUNT(DISTINCT h.QUERY_ID) AS query_count
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY h
JOIN SNOWFLAKE.ACCOUNT_USAGE.USERS u ON h.USER_ID = u.USER_ID
WHERE h.START_TIME >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY total_credits DESC
LIMIT 10;
```

---

### [Q3] Cortex Code（Snowsight チャット）の当月消費

```sql
SELECT
    DATE_TRUNC('day', START_TIME) AS usage_date,
    SUM(INPUT_TOKENS)             AS total_input_tokens,
    SUM(OUTPUT_TOKENS)            AS total_output_tokens,
    COUNT(*)                      AS session_count
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY
WHERE START_TIME >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 1 DESC;
```

---

### [Q4] 日別トレンド（過去 30 日）

```sql
SELECT
    DATE_TRUNC('day', START_TIME) AS usage_date,
    SUM(CREDITS)                  AS daily_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
WHERE START_TIME >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 1;
```

---

### [Q5] 当月の予算消化率

```sql
-- $MONTHLY_CREDIT_LIMIT は変数定義セクションの値に置き換える
SELECT
    500                                           AS budget_limit,   -- ← 上限クレジット
    COALESCE(SUM(CREDITS), 0)                     AS used_credits,
    ROUND(COALESCE(SUM(CREDITS), 0) / 500 * 100, 1) AS pct_used
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
WHERE START_TIME >= DATE_TRUNC('month', CURRENT_TIMESTAMP());
```

---

### [Q6] Custom Action の実行履歴確認

```sql
SELECT th.*, ci.name AS budget_name
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY th
JOIN SNOWFLAKE.ACCOUNT_USAGE.CLASS_INSTANCES ci
    ON th.instance_id = ci.id
WHERE ci.class_name  = 'BUDGET'
  AND th.name        ILIKE 'BUDGET_CUSTOM_ACTION_TRIGGER_AT_%'
  AND ci.name        = 'AI_COST_BUDGET'
ORDER BY th.completed_time DESC
LIMIT 20;
```

---

### [Q7] Budget の消費履歴を確認（直近7日）

```sql
CALL MGMT_DB.BUDGET_SCHEMA.AI_COST_BUDGET!GET_SPENDING_HISTORY(
    TIME_LOWER_BOUND => DATEADD('days', -7, CURRENT_TIMESTAMP()),
    TIME_UPPER_BOUND => CURRENT_TIMESTAMP()
);
```

---

## 7. 運用手順（アラート後の対応フロー）

### アラートメール受信時のフロー

```
1. メール受信
   「[Snowflake] AI Budget 90% 到達 - WH を停止しました」

2. 状況確認（Q1〜Q5 を実行）
   - どの関数・モデルが消費しているか
   - どのユーザーが実行しているか

3. 原因特定後、必要に応じて WH を再開
   ALTER WAREHOUSE <AI_TARGET_WH> RESUME;

4. 必要なら対象ユーザーの AI アクセスを一時制限
   REVOKE DATABASE ROLE SNOWFLAKE.CORTEX_USER FROM ROLE <target_role>;

5. 翌月の上限を見直す場合
   CALL MGMT_DB.BUDGET_SCHEMA.AI_COST_BUDGET!SET_SPENDING_LIMIT(1000);
```

---

### その他の運用コマンド集

```sql
-- WH を手動で再開する
ALTER WAREHOUSE <AI_TARGET_WH> RESUME;

-- 上限クレジットを変更する
CALL MGMT_DB.BUDGET_SCHEMA.AI_COST_BUDGET!SET_SPENDING_LIMIT(1000);

-- 通知閾値を変更する（デフォルトに戻す場合は 110 を指定）
CALL MGMT_DB.BUDGET_SCHEMA.AI_COST_BUDGET!SET_NOTIFICATION_THRESHOLD(80);

-- 登録済み Custom Action を確認する
CALL MGMT_DB.BUDGET_SCHEMA.AI_COST_BUDGET!GET_CUSTOM_ACTIONS();

-- Custom Action を削除する（90% のアクションをすべて削除）
CALL MGMT_DB.BUDGET_SCHEMA.AI_COST_BUDGET!REMOVE_CUSTOM_ACTIONS(90);

-- AI ロールを剥奪してアクセスを完全停止する
-- （WH 停止だけではサーバーレスの AI Functions は止まらないため）
REVOKE DATABASE ROLE SNOWFLAKE.CORTEX_USER    FROM ROLE <target_role>;
REVOKE DATABASE ROLE SNOWFLAKE.COPILOT_USER   FROM ROLE <target_role>;  -- Cortex Code も止める場合

-- AI アクセスを復元する
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER    TO ROLE <target_role>;
GRANT DATABASE ROLE SNOWFLAKE.COPILOT_USER   TO ROLE <target_role>;

-- Account Budget の状態確認
CALL SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET!GET_SPENDING_LIMIT();

-- アカウント内の全 Budget 一覧
SELECT SYSTEM$SHOW_BUDGETS_IN_ACCOUNT();
```

---

## 8. 重要な注意点・落とし穴

### ⚠️ 注意点 1：Budget の適用には遅延がある

| モード | 遅延 |
|---|---|
| 通常 Budget | 閾値超過から最大 **8 時間** |
| Low Latency Budget 有効時 | 約 **2 時間** |

→ 暴走クエリの即時制御には、日常監視クエリ（Q1〜Q5）を Snowflake Task でスケジュール実行する仕組みを併用することを推奨。

---

### ⚠️ 注意点 2：WH を止めても AI Functions のトークン課金は続く

AI Functions はサーバーレス課金のため、WH と独立して動作します。

```
WH を SUSPEND → WH コストは止まる
                ただし AI_COMPLETE 等のサーバーレス課金は継続
```

**完全に止める**には WH 停止に加えてロール剥奪が必要です。

```sql
REVOKE DATABASE ROLE SNOWFLAKE.CORTEX_USER FROM ROLE <target_role>;
```

---

### ⚠️ 注意点 3：Resource Monitor は AI に効かない

Snowsight で設定している「90% で WH を止める」設定（Resource Monitor）は **WH の Virtual Compute のみ**対象です。AI Functions のトークン課金は追跡・制御できません。本ガイドの Budget を別途設定することで初めて AI コストが管理できます。

---

### ⚠️ 注意点 4：メールアドレスの事前確認が必須

通知設定（`SET_EMAIL_NOTIFICATIONS`）に指定するメールアドレスは：
1. Snowflake ユーザーとして登録済みであること
2. Admin → Users で **Email Verification 完了済み**であること
3. Notification Integration の `ALLOWED_RECIPIENTS` に含まれていること

上記の3条件が揃わないと通知設定コマンド自体がエラーになります。

---

### ⚠️ 注意点 5：Cortex Search のアイドル課金

Cortex Search はクエリがゼロ件でも、インデックスしているデータ量 (GB/月) に対して課金が発生します。開発環境の Search サービスは使わない期間は SUSPEND 推奨。

```sql
-- Cortex Search サービスを一時停止
ALTER CORTEX SEARCH SERVICE <service_name> SUSPEND;
```

---

### ⚠️ 注意点 6：Cortex Analyst と Fine-tuning は Budget 非対応

2026年6月時点で、以下の機能は Budget での監視が**サポートされていないか未定**です。

| 機能 | Budget 対応 |
|---|---|
| Cortex Analyst | ❌ 非対応（予定なし） |
| Cortex Fine-tuning | ❌ 非対応（予定なし） |
| Cortex Agents | ✅ 対応（Resource Budget） |
| AI Functions | ✅ 対応（Custom Budget） |

Cortex Analyst のコスト監視は `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` ビューや `WAREHOUSE_METERING_HISTORY` を個別に参照してください。

---

## 参考：コスト管理全体の構成図

```
ACCOUNTADMIN
  │
  ├── Resource Monitor（既存）
  │     └── AI_WH が 90% で SUSPEND（WH コストのみ制御）
  │
  ├── Account Budget（アカウント全体）
  │     └── 上限: 5,000 credits/月
  │     └── 80% で Email 通知
  │
  └── AI_COST_BUDGET（AI 専用 Custom Budget）
        ├── 監視対象: AI_WH（WH コスト）＋ AI Functions（サーバーレス）
        ├── 上限: 500 credits/月（変数で設定）
        ├── 80% → Email 通知（SET_NOTIFICATION_THRESHOLD）
        └── 90% → Custom Action 発動
                   └── SP_SUSPEND_AI_WH_AND_NOTIFY 実行
                         ├── AI_WH を SUSPEND
                         └── 管理者に Email 通知
```

---

*本ガイドは Snowflake 公式ドキュメント（docs.snowflake.com）の 2026年6月時点の情報に基づいています。*
*Budget の仕様・ビュー名は今後のリリースで変更される可能性があります。*
