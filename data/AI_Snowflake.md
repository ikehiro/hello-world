ありがとうございます。全機能を使う開発者として、必要な権限を整理します。​​​​​​​​​​​​​​​​

情報が揃いました。整理して説明します。

AI 開発者ロールに必要な権限の全体像
権限は 「Database Role（機能解錠）」 と 「Object 権限（データ・オブジェクトアクセス）」 の2層で考えます。

層1：Database Role（機能を使えるようにする鍵）
Snowflake の Cortex AI 機能へのアクセスは、機能ごとに専用の Database Role で制御されています。 ￼



|Database Role                   |付与する機能                                                    |デフォルト付与先        |
|--------------------------------|----------------------------------------------------------|----------------|
|`SNOWFLAKE.CORTEX_USER`         |AI Functions 全般 ＋ Agents ＋ Search ＋ Fine-tuning ＋ REST API|**PUBLIC（全員）** ⚠️|
|`SNOWFLAKE.AI_FUNCTIONS_USER`   |SQL 関数（AI_COMPLETE 等）のみ。Agent/Search/Fine-tuning は使えない    |未付与             |
|`SNOWFLAKE.CORTEX_AGENT_USER`   |Agents のみに絞った細粒度制御                                        |未付与             |
|`SNOWFLAKE.CORTEX_REST_API_USER`|REST API のみ                                               |未付与             |
|`SNOWFLAKE.CORTEX_EMBED_USER`   |AI_EMBED 等の埋め込み関数のみ                                       |未付与             |
|`SNOWFLAKE.COPILOT_USER`        |Cortex Code（Snowsight チャット）                               |未付与             |

⚠️ CORTEX_USER はデフォルトで PUBLIC に付与されているため、全ユーザーが AI Functions を使える状態になっています。コスト管理の観点からは PUBLIC から剥奪して明示的に付与する運用が推奨です。

層2：Object 権限（データ・スキーマへのアクセス）



|権限                                      |用途                                 |
|----------------------------------------|-----------------------------------|
|`USE AI FUNCTIONS`（アカウント権限）             |AI 関数呼び出しに必須（Database Role とセットで必要）|
|`USAGE ON WAREHOUSE`                    |クエリ実行用 WH の使用                      |
|`USAGE ON DATABASE / SCHEMA`            |学習データや出力先への参照                      |
|`SELECT ON TABLE/VIEW`                  |学習データ・RAG 用データの読み取り                |
|`CREATE MODEL ON SCHEMA`                |Fine-tuning でカスタムモデルを作成            |
|`USAGE ON MODEL`                        |作成済みカスタムモデルを他ロールが使う場合              |
|`CREATE CORTEX SEARCH SERVICE ON SCHEMA`|Cortex Search サービスの作成              |

機能別の必要権限マトリクス



|やること                           |Database Role                                         |Object 権限                                       |
|-------------------------------|------------------------------------------------------|------------------------------------------------|
|**AI_COMPLETE 等を SQL で使う**     |`CORTEX_USER` または `AI_FUNCTIONS_USER`                 |`USE AI FUNCTIONS` ＋ WH/DB/SCHEMA USAGE ＋ SELECT|
|**Cortex Code（Snowsight チャット）**|`COPILOT_USER` ＋ `CORTEX_USER` または `CORTEX_AGENT_USER`|—                                               |
|**Cortex Agents を構築・管理**       |`CORTEX_USER` または `CORTEX_AGENT_USER`                 |Agent オブジェクトへの USAGE ＋ ツール対象への権限                |
|**Cortex Search を構築**          |`CORTEX_USER`                                         |`CREATE CORTEX SEARCH SERVICE ON SCHEMA`        |
|**Fine-tuning でモデル作成**         |`CORTEX_USER`                                         |`CREATE MODEL ON SCHEMA`                        |
|**Fine-tuning 済みモデルを使う**       |`CORTEX_USER`                                         |`USAGE ON MODEL`                                |
|**コスト監視クエリを実行**                |—                                                     |`IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE`     |

本番用ロール設計 SQL

USE ROLE ACCOUNTADMIN;

-- AI 開発者ロールを作成
CREATE ROLE IF NOT EXISTS AI_DEVELOPER;

-- ① 機能解錠：全 AI 機能を使えるようにする
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER       TO ROLE AI_DEVELOPER;
GRANT DATABASE ROLE SNOWFLAKE.COPILOT_USER      TO ROLE AI_DEVELOPER;  -- Cortex Code

-- ② アカウント権限：AI 関数呼び出しに必須
GRANT PRIVILEGE USE AI FUNCTIONS TO ROLE AI_DEVELOPER;

-- ③ WH・DB・Schema の利用権限（環境に合わせて変更）
GRANT USAGE ON WAREHOUSE  <AI_WH>              TO ROLE AI_DEVELOPER;
GRANT USAGE ON DATABASE   <DEV_DB>             TO ROLE AI_DEVELOPER;
GRANT USAGE ON SCHEMA     <DEV_DB>.<AI_SCHEMA> TO ROLE AI_DEVELOPER;

-- ④ データアクセス（学習データ・RAG用データ）
GRANT SELECT ON ALL TABLES IN SCHEMA <DEV_DB>.<AI_SCHEMA> TO ROLE AI_DEVELOPER;

-- ⑤ オブジェクト作成権限（開発者なので作成も可）
GRANT CREATE MODEL                ON SCHEMA <DEV_DB>.<AI_SCHEMA> TO ROLE AI_DEVELOPER;
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA <DEV_DB>.<AI_SCHEMA> TO ROLE AI_DEVELOPER;
GRANT CREATE TABLE                ON SCHEMA <DEV_DB>.<AI_SCHEMA> TO ROLE AI_DEVELOPER;

-- ⑥ コスト監視クエリ用（任意）
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE AI_DEVELOPER;

-- ユーザーへのロール付与
GRANT ROLE AI_DEVELOPER TO USER <target_user>;


コスト管理との関係でやっておくべき設定

-- PUBLIC から CORTEX_USER を剥奪して明示的制御に切り替える
-- （デフォルトで全員が AI を使える状態を解消）
USE ROLE ACCOUNTADMIN;
REVOKE DATABASE ROLE SNOWFLAKE.CORTEX_USER FROM ROLE PUBLIC;

-- 必要なロールにのみ付与
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE AI_DEVELOPER;


これをやらないと Budget/Alert でコストを監視していても、誰でも自由に AI 関数を呼べる状態が続きます。

補足：Cortex Agents 編集時の既知の落とし穴
Cortex Agent の編集操作は、そのロールがユーザーのデフォルトロールである必要があります。権限が付与されていても、デフォルトロールでないと編集操作が失敗する既知の制限があります。 ￼
AI_DEVELOPER ロールを使って Agents を管理する場合は、ユーザーのデフォルトロールに設定しておく必要があります：

ALTER USER <target_user> SET DEFAULT_ROLE = AI_DEVELOPER;






---------------

なるほど、理解しました。整理し直します。

修正後の概念整理

DR（Database Role）
  = Snowflake が提供するオブジェクト単位の権限
  = SELECT / USAGE / CREATE など、リソースへの直接アクセス権
  = 自作も可能（例：dr_DEV_DB_READ）

AR（Action Role）
  = DR を束ねて「何ができるか」を表す
  = ARだけで完結することもある（DRを使わず直接Privilege付与）

FR（Functional Role）
  = ARを束ねてユーザーに付与する最終ロール


修正後のロール階層

fr_AI_POC_DEVELOPER
    │
    ├─ ar_AI_USE
    │       ├─ DR: SNOWFLAKE.CORTEX_USER   （Snowflake提供DB Role）
    │       ├─ DR: SNOWFLAKE.COPILOT_USER  （Snowflake提供DB Role）
    │       └─ Privilege: USE AI FUNCTIONS  （Account Privilege）
    │
    ├─ ar_AI_BUILD
    │       ├─ dr_AI_SCHEMA_BUILD（自作DR）
    │       │       ├─ CREATE MODEL ON SCHEMA
    │       │       ├─ CREATE CORTEX SEARCH SERVICE ON SCHEMA
    │       │       ├─ CREATE TABLE ON SCHEMA
    │       │       └─ CREATE PROCEDURE ON SCHEMA
    │       └─ USAGE ON WAREHOUSE（直接付与）
    │
    └─ ar_DEV_DB_READ
            └─ dr_DEV_DB_READ（自作DR）
                    ├─ USAGE ON DATABASE
                    ├─ USAGE ON ALL SCHEMAS
                    ├─ SELECT ON ALL TABLES  (+ FUTURE)
                    └─ SELECT ON ALL VIEWS   (+ FUTURE)


本番用 SQL

USE ROLE ACCOUNTADMIN;

-- ============================================================
-- Database Role 1: 開発DBの参照権限（自作DR）
-- ============================================================
CREATE DATABASE ROLE IF NOT EXISTS <DEV_DB>.dr_DEV_DB_READ;

GRANT USAGE  ON DATABASE  <DEV_DB>                         TO DATABASE ROLE <DEV_DB>.dr_DEV_DB_READ;
GRANT USAGE  ON ALL SCHEMAS IN DATABASE <DEV_DB>           TO DATABASE ROLE <DEV_DB>.dr_DEV_DB_READ;
GRANT USAGE  ON FUTURE SCHEMAS IN DATABASE <DEV_DB>        TO DATABASE ROLE <DEV_DB>.dr_DEV_DB_READ;
GRANT SELECT ON ALL TABLES  IN DATABASE <DEV_DB>           TO DATABASE ROLE <DEV_DB>.dr_DEV_DB_READ;
GRANT SELECT ON ALL VIEWS   IN DATABASE <DEV_DB>           TO DATABASE ROLE <DEV_DB>.dr_DEV_DB_READ;
GRANT SELECT ON FUTURE TABLES IN DATABASE <DEV_DB>         TO DATABASE ROLE <DEV_DB>.dr_DEV_DB_READ;
GRANT SELECT ON FUTURE VIEWS  IN DATABASE <DEV_DB>         TO DATABASE ROLE <DEV_DB>.dr_DEV_DB_READ;

-- ============================================================
-- Database Role 2: AIスキーマへのオブジェクト作成権限（自作DR）
-- ============================================================
CREATE DATABASE ROLE IF NOT EXISTS <DEV_DB>.dr_AI_SCHEMA_BUILD;

GRANT CREATE MODEL                 ON SCHEMA <DEV_DB>.<AI_SCHEMA> TO DATABASE ROLE <DEV_DB>.dr_AI_SCHEMA_BUILD;
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA <DEV_DB>.<AI_SCHEMA> TO DATABASE ROLE <DEV_DB>.dr_AI_SCHEMA_BUILD;
GRANT CREATE TABLE                 ON SCHEMA <DEV_DB>.<AI_SCHEMA> TO DATABASE ROLE <DEV_DB>.dr_AI_SCHEMA_BUILD;
GRANT CREATE PROCEDURE             ON SCHEMA <DEV_DB>.<AI_SCHEMA> TO DATABASE ROLE <DEV_DB>.dr_AI_SCHEMA_BUILD;

-- ============================================================
-- Action Role 1: AI機能の実行（DRを束ねる）
-- ============================================================
CREATE ROLE IF NOT EXISTS ar_AI_USE;

GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER  TO ROLE ar_AI_USE;
GRANT DATABASE ROLE SNOWFLAKE.COPILOT_USER TO ROLE ar_AI_USE;
GRANT PRIVILEGE USE AI FUNCTIONS           TO ROLE ar_AI_USE;

-- ============================================================
-- Action Role 2: AIオブジェクトの構築（DR + WH権限）
-- ============================================================
CREATE ROLE IF NOT EXISTS ar_AI_BUILD;

GRANT DATABASE ROLE <DEV_DB>.dr_AI_SCHEMA_BUILD TO ROLE ar_AI_BUILD;
GRANT USAGE ON WAREHOUSE <DEV_WH>               TO ROLE ar_AI_BUILD;

-- ============================================================
-- Action Role 3: 開発DBデータ参照（DRを束ねる）
-- ============================================================
CREATE ROLE IF NOT EXISTS ar_DEV_DB_READ;

GRANT DATABASE ROLE <DEV_DB>.dr_DEV_DB_READ TO ROLE ar_DEV_DB_READ;

-- ============================================================
-- Functional Role: ユーザーに付与するロール
-- ============================================================
CREATE ROLE IF NOT EXISTS fr_AI_POC_DEVELOPER;

GRANT ROLE ar_AI_USE     TO ROLE fr_AI_POC_DEVELOPER;
GRANT ROLE ar_AI_BUILD   TO ROLE fr_AI_POC_DEVELOPER;
GRANT ROLE ar_DEV_DB_READ TO ROLE fr_AI_POC_DEVELOPER;

-- ============================================================
-- ユーザー作成・ロール付与
-- ============================================================
CREATE USER IF NOT EXISTS DEV_AI_POC
    TYPE                 = PERSON
    DEFAULT_ROLE         = fr_AI_POC_DEVELOPER
    DEFAULT_WAREHOUSE    = <DEV_WH>
    DEFAULT_NAMESPACE    = <DEV_DB>.<AI_SCHEMA>
    MUST_CHANGE_PASSWORD = TRUE;

GRANT ROLE fr_AI_POC_DEVELOPER TO USER DEV_AI_POC;


考え方のまとめ



|層 |命名      |何を入れるか                                      |
|--|--------|--------------------------------------------|
|DR|`dr_XXX`|SELECT・USAGE・CREATEなどリソース権限。Snowflake提供DRもここ|
|AR|`ar_XXX`|DRを束ねて「行為」を表す。WHなど直接Privilegeもここ            |
|FR|`fr_XXX`|ARを束ねてユーザーに付与                               |

ar_AI_USE のように Snowflake 提供の DR（CORTEX_USER 等）を AR で束ねるパターンと、自作 DR（dr_DEV_DB_READ）を AR で束ねるパターンの両方が共存する形になります。​​​​​​​​​​​​​​​​

-----------

## – ============================================================
– AI POC 開発者ユーザー作成スクリプト
– Snowflake CLI (snow) Jinja2 テンプレート版
– ============================================================
– 【呼び出し方】

## –   snow sql -f create_ai_poc_developer.sql   
–            -D db=DEV_DB   
–            -D name=IKEDA   
–            -D wh=DEV_WH

## – 【パラメータ】
–   db   : 対象データベース名
–   name : ユーザー個人名（ロール名・ユーザー名に使用）
–   wh   : 開発用ウェアハウス名

## – 【作成されるオブジェクト】（例: db=DEV_DB, name=IKEDA）

## –   Database Roles（<db> に紐づく）:
–     dr_<db>*RAW_READ
–     dr*<db>*TRANSFORM_READ
–     dr*<db>*MART_READ
–     dr*<db>*COMMON_READ
–     dr*<db>*AUDIT_READ
–     dr*<db>*WEB_READ
–     dr*<db>_SANDBOX_READWRITE

## –   Action Roles（個人名入り）:
–     ar_<name>*AI_USE          … CORTEX_USER + COPILOT_USER + USE AI FUNCTIONS
–     ar*<name>*AI_BUILD        … CREATE MODEL + CREATE CORTEX SEARCH SERVICE
–     ar*<name>*DB_READ         … 参照スキーマ DR 6本を束ねる
–     ar*<name>_SANDBOX_BUILD   … sandbox DR + WH USAGE

## –   Functional Role:
–     fr_<name>_AI_POC_DEVELOPER

## –   User:
–     DEV_AI_<name>

– 【前提条件】
–   - ACCOUNTADMIN ロールで実行すること
–   - <wh> が存在すること
–   - <db> に raw/transform/mart/common/audit/web/sandbox スキーマが存在すること
– ============================================================

USE ROLE ACCOUNTADMIN;

– ============================================================
– STEP 1: Database Roles の作成（参照スキーマ × 6）
– ============================================================

– [raw] 参照専用
CREATE DATABASE ROLE IF NOT EXISTS {{ db }}.dr_{{ db }}*RAW_READ;
GRANT USAGE  ON SCHEMA                   {{ db }}.raw TO DATABASE ROLE {{ db }}.dr*{{ db }}*RAW_READ;
GRANT SELECT ON ALL TABLES    IN SCHEMA  {{ db }}.raw TO DATABASE ROLE {{ db }}.dr*{{ db }}*RAW_READ;
GRANT SELECT ON ALL VIEWS     IN SCHEMA  {{ db }}.raw TO DATABASE ROLE {{ db }}.dr*{{ db }}*RAW_READ;
GRANT SELECT ON FUTURE TABLES IN SCHEMA  {{ db }}.raw TO DATABASE ROLE {{ db }}.dr*{{ db }}*RAW_READ;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA  {{ db }}.raw TO DATABASE ROLE {{ db }}.dr*{{ db }}_RAW_READ;

– [transform] 参照専用
CREATE DATABASE ROLE IF NOT EXISTS {{ db }}.dr_{{ db }}*TRANSFORM_READ;
GRANT USAGE  ON SCHEMA                   {{ db }}.transform TO DATABASE ROLE {{ db }}.dr*{{ db }}*TRANSFORM_READ;
GRANT SELECT ON ALL TABLES    IN SCHEMA  {{ db }}.transform TO DATABASE ROLE {{ db }}.dr*{{ db }}*TRANSFORM_READ;
GRANT SELECT ON ALL VIEWS     IN SCHEMA  {{ db }}.transform TO DATABASE ROLE {{ db }}.dr*{{ db }}*TRANSFORM_READ;
GRANT SELECT ON FUTURE TABLES IN SCHEMA  {{ db }}.transform TO DATABASE ROLE {{ db }}.dr*{{ db }}*TRANSFORM_READ;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA  {{ db }}.transform TO DATABASE ROLE {{ db }}.dr*{{ db }}_TRANSFORM_READ;

– [mart] 参照専用
CREATE DATABASE ROLE IF NOT EXISTS {{ db }}.dr_{{ db }}*MART_READ;
GRANT USAGE  ON SCHEMA                   {{ db }}.mart TO DATABASE ROLE {{ db }}.dr*{{ db }}*MART_READ;
GRANT SELECT ON ALL TABLES    IN SCHEMA  {{ db }}.mart TO DATABASE ROLE {{ db }}.dr*{{ db }}*MART_READ;
GRANT SELECT ON ALL VIEWS     IN SCHEMA  {{ db }}.mart TO DATABASE ROLE {{ db }}.dr*{{ db }}*MART_READ;
GRANT SELECT ON FUTURE TABLES IN SCHEMA  {{ db }}.mart TO DATABASE ROLE {{ db }}.dr*{{ db }}*MART_READ;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA  {{ db }}.mart TO DATABASE ROLE {{ db }}.dr*{{ db }}_MART_READ;

– [common] 参照専用
CREATE DATABASE ROLE IF NOT EXISTS {{ db }}.dr_{{ db }}*COMMON_READ;
GRANT USAGE  ON SCHEMA                   {{ db }}.common TO DATABASE ROLE {{ db }}.dr*{{ db }}*COMMON_READ;
GRANT SELECT ON ALL TABLES    IN SCHEMA  {{ db }}.common TO DATABASE ROLE {{ db }}.dr*{{ db }}*COMMON_READ;
GRANT SELECT ON ALL VIEWS     IN SCHEMA  {{ db }}.common TO DATABASE ROLE {{ db }}.dr*{{ db }}*COMMON_READ;
GRANT SELECT ON FUTURE TABLES IN SCHEMA  {{ db }}.common TO DATABASE ROLE {{ db }}.dr*{{ db }}*COMMON_READ;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA  {{ db }}.common TO DATABASE ROLE {{ db }}.dr*{{ db }}_COMMON_READ;

– [audit] 参照専用
CREATE DATABASE ROLE IF NOT EXISTS {{ db }}.dr_{{ db }}*AUDIT_READ;
GRANT USAGE  ON SCHEMA                   {{ db }}.audit TO DATABASE ROLE {{ db }}.dr*{{ db }}*AUDIT_READ;
GRANT SELECT ON ALL TABLES    IN SCHEMA  {{ db }}.audit TO DATABASE ROLE {{ db }}.dr*{{ db }}*AUDIT_READ;
GRANT SELECT ON ALL VIEWS     IN SCHEMA  {{ db }}.audit TO DATABASE ROLE {{ db }}.dr*{{ db }}*AUDIT_READ;
GRANT SELECT ON FUTURE TABLES IN SCHEMA  {{ db }}.audit TO DATABASE ROLE {{ db }}.dr*{{ db }}*AUDIT_READ;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA  {{ db }}.audit TO DATABASE ROLE {{ db }}.dr*{{ db }}_AUDIT_READ;

– [web] 参照専用
CREATE DATABASE ROLE IF NOT EXISTS {{ db }}.dr_{{ db }}*WEB_READ;
GRANT USAGE  ON SCHEMA                   {{ db }}.web TO DATABASE ROLE {{ db }}.dr*{{ db }}*WEB_READ;
GRANT SELECT ON ALL TABLES    IN SCHEMA  {{ db }}.web TO DATABASE ROLE {{ db }}.dr*{{ db }}*WEB_READ;
GRANT SELECT ON ALL VIEWS     IN SCHEMA  {{ db }}.web TO DATABASE ROLE {{ db }}.dr*{{ db }}*WEB_READ;
GRANT SELECT ON FUTURE TABLES IN SCHEMA  {{ db }}.web TO DATABASE ROLE {{ db }}.dr*{{ db }}*WEB_READ;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA  {{ db }}.web TO DATABASE ROLE {{ db }}.dr*{{ db }}_WEB_READ;

– ============================================================
– STEP 2: Database Role の作成（sandbox：読み書き＋作成）
– ============================================================

CREATE DATABASE ROLE IF NOT EXISTS {{ db }}.dr_{{ db }}*SANDBOX_READWRITE;
GRANT USAGE  ON SCHEMA {{ db }}.sandbox TO DATABASE ROLE {{ db }}.dr*{{ db }}*SANDBOX_READWRITE;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES    IN SCHEMA {{ db }}.sandbox TO DATABASE ROLE {{ db }}.dr*{{ db }}*SANDBOX_READWRITE;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA {{ db }}.sandbox TO DATABASE ROLE {{ db }}.dr*{{ db }}*SANDBOX_READWRITE;
GRANT CREATE TABLE     ON SCHEMA {{ db }}.sandbox TO DATABASE ROLE {{ db }}.dr*{{ db }}*SANDBOX_READWRITE;
GRANT CREATE VIEW      ON SCHEMA {{ db }}.sandbox TO DATABASE ROLE {{ db }}.dr*{{ db }}*SANDBOX_READWRITE;
GRANT CREATE PROCEDURE ON SCHEMA {{ db }}.sandbox TO DATABASE ROLE {{ db }}.dr*{{ db }}*SANDBOX_READWRITE;
GRANT CREATE STAGE     ON SCHEMA {{ db }}.sandbox TO DATABASE ROLE {{ db }}.dr*{{ db }}_SANDBOX_READWRITE;

– ============================================================
– STEP 3: Action Roles の作成
– ============================================================

– [ar_<name>*AI_USE] AI機能実行系
–   Snowflake提供 DR と Account Privilege を束ねる
CREATE ROLE IF NOT EXISTS ar*{{ name }}*AI_USE;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER  TO ROLE ar*{{ name }}*AI_USE;
GRANT DATABASE ROLE SNOWFLAKE.COPILOT_USER TO ROLE ar*{{ name }}*AI_USE;
GRANT PRIVILEGE USE AI FUNCTIONS           TO ROLE ar*{{ name }}_AI_USE;

– [ar_<name>*AI_BUILD] AI オブジェクト構築系
–   sandbox スキーマへの AI オブジェクト作成 + WH 使用権限
CREATE ROLE IF NOT EXISTS ar*{{ name }}*AI_BUILD;
GRANT CREATE MODEL                 ON SCHEMA {{ db }}.sandbox TO ROLE ar*{{ name }}*AI_BUILD;
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA {{ db }}.sandbox TO ROLE ar*{{ name }}*AI_BUILD;
GRANT USAGE ON WAREHOUSE {{ wh }}                             TO ROLE ar*{{ name }}_AI_BUILD;

– [ar_<name>*DB_READ] 参照スキーマへのアクセス
–   DR 6本 + DB USAGE を束ねる
CREATE ROLE IF NOT EXISTS ar*{{ name }}*DB_READ;
GRANT USAGE          ON DATABASE {{ db }}                              TO ROLE ar*{{ name }}*DB_READ;
GRANT DATABASE ROLE  {{ db }}.dr*{{ db }}*RAW_READ                    TO ROLE ar*{{ name }}*DB_READ;
GRANT DATABASE ROLE  {{ db }}.dr*{{ db }}*TRANSFORM_READ              TO ROLE ar*{{ name }}*DB_READ;
GRANT DATABASE ROLE  {{ db }}.dr*{{ db }}*MART_READ                   TO ROLE ar*{{ name }}*DB_READ;
GRANT DATABASE ROLE  {{ db }}.dr*{{ db }}*COMMON_READ                 TO ROLE ar*{{ name }}*DB_READ;
GRANT DATABASE ROLE  {{ db }}.dr*{{ db }}*AUDIT_READ                  TO ROLE ar*{{ name }}*DB_READ;
GRANT DATABASE ROLE  {{ db }}.dr*{{ db }}*WEB_READ                    TO ROLE ar*{{ name }}_DB_READ;

– [ar_<name>*SANDBOX_BUILD] sandbox 読み書き＋作成
–   DR 1本 + DB USAGE + WH を束ねる
CREATE ROLE IF NOT EXISTS ar*{{ name }}*SANDBOX_BUILD;
GRANT USAGE         ON DATABASE {{ db }}                               TO ROLE ar*{{ name }}*SANDBOX_BUILD;
GRANT DATABASE ROLE {{ db }}.dr*{{ db }}*SANDBOX_READWRITE            TO ROLE ar*{{ name }}*SANDBOX_BUILD;
GRANT USAGE ON WAREHOUSE {{ wh }}                                      TO ROLE ar*{{ name }}_SANDBOX_BUILD;

– ============================================================
– STEP 4: Functional Role の作成
– ============================================================

CREATE ROLE IF NOT EXISTS fr_{{ name }}*AI_POC_DEVELOPER;
GRANT ROLE ar*{{ name }}*AI_USE        TO ROLE fr*{{ name }}*AI_POC_DEVELOPER;
GRANT ROLE ar*{{ name }}*AI_BUILD      TO ROLE fr*{{ name }}*AI_POC_DEVELOPER;
GRANT ROLE ar*{{ name }}*DB_READ       TO ROLE fr*{{ name }}*AI_POC_DEVELOPER;
GRANT ROLE ar*{{ name }}*SANDBOX_BUILD TO ROLE fr*{{ name }}_AI_POC_DEVELOPER;

– ACCOUNTADMIN への付与（管理用）
GRANT ROLE fr_{{ name }}_AI_POC_DEVELOPER TO ROLE ACCOUNTADMIN;

– ============================================================
– STEP 5: ユーザーの作成
– ============================================================

CREATE USER IF NOT EXISTS DEV_AI_{{ name }}
TYPE                 = PERSON
DEFAULT_ROLE         = fr_{{ name }}_AI_POC_DEVELOPER
DEFAULT_WAREHOUSE    = {{ wh }}
DEFAULT_NAMESPACE    = {{ db }}.sandbox
MUST_CHANGE_PASSWORD = TRUE
COMMENT              = ‘AI POC 開発者アカウント’;

GRANT ROLE fr_{{ name }}*AI_POC_DEVELOPER TO USER DEV_AI*{{ name }};

– ============================================================
– STEP 6: 作成結果の確認
– ============================================================

SHOW GRANTS TO ROLE fr_{{ name }}*AI_POC_DEVELOPER;
SHOW GRANTS TO USER DEV_AI*{{ name }};

## – ============================================================
– 【クリーンアップ手順】PoC 終了時はこちらを実行

## –   snow sql -f create_ai_poc_developer.sql   
–            -D db=DEV_DB   
–            -D name=IKEDA   
–            -D wh=DEV_WH   
–            –define cleanup=true

## –   ※ 以下を別ファイル cleanup_ai_poc_developer.sql として保存し
–      同じ -D オプションで呼び出してもよい

–   DROP USER  IF EXISTS DEV_AI_{{ name }};
–   DROP ROLE  IF EXISTS fr_{{ name }}*AI_POC_DEVELOPER;
–   DROP ROLE  IF EXISTS ar*{{ name }}*AI_USE;
–   DROP ROLE  IF EXISTS ar*{{ name }}*AI_BUILD;
–   DROP ROLE  IF EXISTS ar*{{ name }}*DB_READ;
–   DROP ROLE  IF EXISTS ar*{{ name }}*SANDBOX_BUILD;
–   – DR は他ユーザーと共用のため、全員削除後にまとめて実行
–   – DROP DATABASE ROLE IF EXISTS {{ db }}.dr*{{ db }}*RAW_READ;
–   – DROP DATABASE ROLE IF EXISTS {{ db }}.dr*{{ db }}*TRANSFORM_READ;
–   – DROP DATABASE ROLE IF EXISTS {{ db }}.dr*{{ db }}*MART_READ;
–   – DROP DATABASE ROLE IF EXISTS {{ db }}.dr*{{ db }}*COMMON_READ;
–   – DROP DATABASE ROLE IF EXISTS {{ db }}.dr*{{ db }}*AUDIT_READ;
–   – DROP DATABASE ROLE IF EXISTS {{ db }}.dr*{{ db }}*WEB_READ;
–   – DROP DATABASE ROLE IF EXISTS {{ db }}.dr*{{ db }}_SANDBOX_READWRITE;
– ============================================================


