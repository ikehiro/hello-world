# Snowflake

# dbt CI on Snowflake Native (選択肢C-1)

本番が **Snowflake内 dbt project (`EXECUTE DBT PROJECT`)** で動く前提のCI実装。
GitBucket CI Plugin から Snowflake CLI (`snow`) を呼び、PR単位で**完全分離されたエフェメラル環境**でCIを実行する。

## 進め方の推奨フェーズ

|Phase|内容        |期間目安|本ディレクトリでの対応箇所                                                 |
|-----|----------|----|--------------------------------------------------------------|
|0    |設計確定      |1-2週|docs/architecture.md                                          |
|1    |本番骨格構築    |2-3週|snowflake_setup/01-04, dbt_project/, scripts/deploy_prod.sh   |
|2    |CI構築      |2-3週|snowflake_setup/05-06, .gitbucket/, scripts/ci.sh, teardown.sh|
|3    |オーケストレーション|1-2週|scripts/create_prod_task.sql                                  |
|4    |運用最適化     |継続  |(slim CI化、監視、ドキュメント等)                                         |

**必ず Phase 1 → Phase 2 の順で進める**。本番が決まらないとCIの前提が定まらない。

## ディレクトリ構成

```
.
├── README.md
├── snowflake_setup/
│   ├── 01_prod_database.sql          # Phase 1: 本番DB/スキーマ
│   ├── 02_prod_roles_and_users.sql   # Phase 1: 本番ロール体系
│   ├── 03_git_integration.sql        # Phase 1: GitBucket連携
│   ├── 04_external_access.sql        # Phase 1: dbt deps用外部アクセス
│   ├── 05_ci_database.sql            # Phase 2: CI用環境
│   └── 06_ci_roles.sql               # Phase 2: CI_ROLE
├── dbt_project/
│   ├── dbt_project.yml
│   ├── profiles.yml                  # prod / ci の両target定義
│   ├── packages.yml
│   ├── models/...
│   ├── macros/
│   │   ├── generate_database_name.sql  # CI時に動的DB切替
│   │   └── drop_pr_resources.sql
│   └── seeds/
├── scripts/
│   ├── deploy_prod.sh                # Phase 1: 本番デプロイ
│   ├── ci.sh                         # Phase 2: CI実行 (GitBucket CIから呼ばれる)
│   ├── teardown.sh                   # Phase 2: PR close時のクリーンアップ
│   └── create_prod_task.sql          # Phase 3: 定期実行Task
├── docker/
│   └── Dockerfile                    # snow CLI入りの極小イメージ
├── .gitbucket/
│   ├── ci.yml                        # Phase 2: PR時のCIトリガー
│   └── teardown.yml                  # Phase 2: PR close時
└── docs/
    └── architecture.md               # Phase 0: 設計書
```

## 前提条件

- **Snowflake Enterprise Edition以上** (dbt projects on Snowflake のGA機能を利用)
- **dbt projects on Snowflake は2025年11月にGA**: <https://docs.snowflake.com/en/user-guide/data-engineering/dbt-projects-on-snowflake>
- GitBucket は self-hosted、CI Plugin有効
- GitBucket から Snowflake への通信が可能 (Public endpoint or PrivateLink経由)

## クイックスタート

### Phase 1: 本番構築

```bash
# 1. Snowflake setup SQL を順番に実行 (ACCOUNTADMIN権限で)
snowsql -c admin -f snowflake_setup/01_prod_database.sql
snowsql -c admin -f snowflake_setup/02_prod_roles_and_users.sql
snowsql -c admin -f snowflake_setup/03_git_integration.sql
snowsql -c admin -f snowflake_setup/04_external_access.sql

# 2. dbt projectを本番にデプロイ
./scripts/deploy_prod.sh

# 3. 動作確認
snowsql -c admin -q "EXECUTE DBT PROJECT PROD_DBT_DB.deployments.dbt_proj_prod ARGS='build --target prod'"
```

### Phase 2: CI構築

```bash
# 1. CI用Snowflake環境
snowsql -c admin -f snowflake_setup/05_ci_database.sql
snowsql -c admin -f snowflake_setup/06_ci_roles.sql

# 2. GitBucket Secrets登録
#    SNOWFLAKE_ACCOUNT, SNOWFLAKE_USER, SNOWFLAKE_PRIVATE_KEY,
#    SNOWFLAKE_ROLE=CI_ROLE, SNOWFLAKE_WAREHOUSE=CI_WH

# 3. Docker imageをbuild & push (内部Registry前提)
docker build -t internal-registry/dbt-ci:latest -f docker/Dockerfile docker/
docker push internal-registry/dbt-ci:latest

# 4. PR作成で .gitbucket/ci.yml が発火
```

詳細は `docs/architecture.md` を参照。

-------------

池田さんの案、方向性として正しいです。本番がSnowflake内のdbt projectで動くなら、CIはむしろ「本番と環境を分ける」必要があるので、その判断は理にかなっています。
ただし、いくつか論点があります。整理しますね。
池田さんの案の評価
「PR単位でエフェメラルDB + Docker上のdbt」案を分解すると:

[GitBucket PR]
   ↓
[Docker (CI Runner)] - dbt-core 起動
   ↓
[Snowflake] - PR単位の一時DB (PR_123_DB) を作って検証
   ↓
[PR closeで一時DBを削除]


良い点:
	∙	PRごとの完全分離: スキーマ単位ではなくDB単位で分けるので、ロール権限・Snowpipe・Stream等の検証も完結する
	∙	本番と環境の分離: 本番がSnowflake内のdbt projectでもCIはDockerで独立、デバッグしやすい
	∙	dbt-core が手元で再現可能: Dockerなら開発者ローカルでも同じイメージで再現できる
気になる点:
	∙	エフェメラル”DB”の作成コストが地味に重い: スキーマ作成は瞬時だがDB作成はメタデータ操作多め
	∙	データのコピー戦略が未定義: 空DBだとtest通っても意味薄い、本番からclone必要
	∙	Snowpipe等の連携リソースはDBごとに作り直すのか?
	∙	本番と実行エンジンが違う (Docker dbt-core vs Snowflake内dbt) ことによる挙動差リスク
CIの選択肢を整理
選択肢A: PR単位エフェメラルDB + Docker dbt-core (池田さん案)

PR_123_DB (zero-copy clone from PROD_DB)
  ├─ RAW
  ├─ STG  
  └─ MART




|評価項目   |コメント                                          |
|-------|----------------------------------------------|
|分離度    |★★★★★ DBごと完全分離                                |
|本番との挙動差|★★★ Dockerと本番(Snowflake dbt project)で実行エンジンが違う|
|コスト    |★★★ DBクローンはzero-copyなのでストレージは安い、計算リソースは普通     |
|クリーンアップ|★★★ DBごとDROPで一発                               |
|実装難易度  |★★★ Dockerfile + cleanup macro必要              |

選択肢B: PR単位エフェメラルスキーマ + Docker dbt-core (bruno-szdl式)

CI_DB
  ├─ PR_123__abc__staging
  ├─ PR_123__abc__marts
  └─ PR_456__def__staging




|評価項目   |コメント                                      |
|-------|------------------------------------------|
|分離度    |★★★★ スキーマレベルで分離、十分実用                      |
|本番との挙動差|★★★ 同上                                    |
|コスト    |★★★★ スキーマ作成は軽量                            |
|クリーンアップ|★★★★ generate_schema_name + drop macroで自動化|
|実装難易度  |★★ 前回作った構成そのまま                            |

選択肢C: PR単位エフェメラルDB + Snowflake内 dbt project (本番と同じ実行エンジン)

[GitBucket PR]
   ↓
[GitBucket CI Plugin] が Snowflakeに API call
   ↓
[Snowflake内に CI用 dbt project 作成]
   ↓
EXECUTE DBT PROJECT pr_123_project ARGS='build --target ci';




|評価項目   |コメント                                      |
|-------|------------------------------------------|
|分離度    |★★★★★ DBごと分離                              |
|本番との挙動差|★★★★★ 実行エンジンが本番と同じ                        |
|コスト    |★★★★ Docker runner不要、Snowflakeクレジットのみ     |
|クリーンアップ|★★★★ DBごとDROP                             |
|実装難易度  |★★★★ Snowflake dbt projects機能の習熟必要、まだ新しい機能|

選択肢D: dbt Cloud CI (商用)



|評価項目   |コメント     |
|-------|---------|
|分離度    |★★★★     |
|本番との挙動差|★★★      |
|コスト    |★ ライセンス費用|
|クリーンアップ|★★★★★ 全自動|
|実装難易度  |★ 設定だけ   |

私の推奨: 選択肢C (本番と同じ実行エンジン)
ただし条件付きです。整理すると:
おすすめ順序
第1位: 選択肢C (Snowflake内 dbt project でCI)
理由:
	∙	本番と同じ実行エンジンなのが最大のメリット。CI通って本番で落ちる事故が激減
	∙	Snowflake内で完結するのでDocker runnerやNAT GWやVPCの設計不要
	∙	dbt projects on SnowflakeのEXECUTE DBT PROJECT でテスト走らせるだけ
	∙	池田さんがすでにSnowflake寄りに本番アーキを倒しているので一貫性がある
懸念:
	∙	dbt projects on Snowflake はまだ比較的新しい機能。Public Preview / GAステータスを確認する必要あり
	∙	ロールバック設計やdbt CLIオプションの互換性は要検証
第2位: 選択肢A (池田さん案) ← もし選択肢Cが時期尚早ならこちら
理由:
	∙	DB単位分離は確かにスキーマ分離より清く、Snowpipeやステージ等のリソース検証も含められる
	∙	Dockerで実行することで開発者ローカル再現性が確保できる
第3位: 選択肢B (bruno-szdl式) ← シンプルさ重視ならこれ
理由:
	∙	前回作った構成そのまま使える
	∙	スキーマ分離で実用上ほぼ困らない
	∙	DBクローンの待ち時間 (数秒〜数十秒) も不要
池田さん案 (選択肢A) を採用するなら、押さえるべきポイント5つ
選択肢Aで進める場合の、実装上の論点です。
1. データのコピー戦略
空DBでtestを走らせても意味が薄いので、本番からzero-copy cloneするのが定石:

-- PR作成時
CREATE DATABASE PR_123_DB CLONE PROD_DB;
GRANT USAGE ON DATABASE PR_123_DB TO ROLE CI_ROLE;


zero-copyなのでストレージコストは差分のみ、作成時間も数秒。
2. CI用ロールの権限設計
PR_123_DBへのCREATE/DROP/MODIFY権限を持つCI_ROLEを用意。本番DBへはSELECT権限のみ(--defer時の参照用) に絞る。
3. クリーンアップの確実性
PR closeをトリガーに削除するが、GitBucket CI Pluginのclose event取り逃しに備えて、毎週バッチで「7日以上前のPR_*_DB」を削除する保険を入れておく:

-- 毎週日曜実行
EXECUTE TASK weekly_pr_cleanup;
-- 内部で SHOW DATABASES → 7日以上前のPR_*をDROP


4. 本番との挙動差リスクの軽減
Docker dbt-coreと本番のSnowflake dbt projectでdbtバージョンを必ず揃える。CIで通って本番で落ちる事故の8割はバージョン不一致です。
5. CI実行時間の管理
DBクローン + dbt build + cleanup で、1 PRあたり数分〜10数分を目安に。これより長くなるなら state:modified+ で差分実行 (Slim CI) を必ず入れる。
ハイブリッド推奨: 「段階的に選択肢C へ」
実は一番現実的なのはこれです:

Phase 1 (今): 選択肢A (池田さん案) で実装
  ↓
Phase 2 (3-6ヶ月後): dbt projects on Snowflake が安定したら選択肢Cへ移行


Phase 1のDocker dbt-core 経験値は無駄になりません:
	∙	開発者ローカル環境にそのまま転用可能
	∙	CIロジックの設計 (PR用DB命名、クリーンアップ、状態管理) はそのまま Phase 2 で流用できる
	∙	Snowflake dbt projects の不確実性をPhase 1で吸収
まとめ
池田さん案 (PR単位エフェメラルDB + Docker dbt-core) は理にかなっています。スキーマ分離より一段強い隔離が得られ、Snowpipe等の検証もしやすい。
ただし、本番が Snowflake 内 dbt project なら、いずれは選択肢C (Snowflake内CI) に倒すと、実行エンジン差異リスクが消えて綺麗です。
確認したいこと:
	∙	本番の Snowflake dbt projects は既に動いていますか? それともこれから?
	∙	CI実行頻度の想定は? (1日数PR? 数十PR?)
	∙	本番DBのサイズ感は? (zero-copy clone前提でも、ストレージ差分が増えると効いてくる)
これが分かると、Phase 1の具体実装まで詰められます。​​​​​​​​​​​​​​​​

はい、できます。それどころか別の場所に分けるのが必須です。本番と同じ場所にCI用プロジェクトを置くと事故の元なので、明確に分離します。
ただし、dbt projects on Snowflake は比較的新しい機能なので、最新の実装方法を確認させてください。​​​​​​​​​​​​​​​​

重要な情報が取れました。dbt Projects on Snowflake は2025年11月にGAになっています。本番運用に乗せられるステータスです。
これを踏まえて回答します。
結論: 別の場所に分けられる、むしろそれが標準パターン
dbt projects on Snowflakeでは、CREATE DBT PROJECT は名前付きのデータベースとスキーマに dbt project object を作成します (例: mydb.my_schema.my_project) ￼。
つまりdbt project object はスキーマレベルのオブジェクトであり、テーブルやビューと同じように好きな場所に置けます。
分離の3レベル
dbt project object と、それが書き込む対象データベースは別物であることがポイント。3つの場所が登場します:

[1] プロジェクト定義 (DBT PROJECT object) の置き場所
[2] dbt deps で使う dbt_packages の置き場所  
[3] モデルが書き込む先のデータベース・スキーマ


これらをすべて環境ごとに分離するのが基本パターンです。
推奨構成: 完全分離パターン

┌─ 本番環境 ──────────────────────────────────┐
│ PROD_DBT_DB.deployments.dbt_proj_prod      │  ← DBT PROJECT object
│         ↓ writes to                          │
│ PROD_DB.{raw, staging, marts}              │  ← データの実体
└─────────────────────────────────────────────┘

┌─ CI環境 (PR単位) ───────────────────────────┐
│ CI_DBT_DB.deployments.dbt_proj_pr_123      │  ← PRごとに作る
│         ↓ writes to                          │
│ PR_123_DB.{raw, staging, marts}            │  ← PRごとのDB
└─────────────────────────────────────────────┘


具体的なやり方
1. 本番側のセットアップ (一度だけ)

-- 本番のdbt projectオブジェクト置き場
CREATE DATABASE PROD_DBT_DB;
CREATE SCHEMA PROD_DBT_DB.deployments;

-- 本番データの置き場
CREATE DATABASE PROD_DB;
CREATE SCHEMA PROD_DB.raw;
CREATE SCHEMA PROD_DB.staging;
CREATE SCHEMA PROD_DB.marts;

-- Git連携用のAPI integration (一度作れば使い回し)
CREATE OR REPLACE API INTEGRATION gitbucket_api_integration
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://gitbucket.example.com/ikeda/')
  ALLOWED_AUTHENTICATION_SECRETS = (gitbucket_secret)
  ENABLED = true;

CREATE OR REPLACE GIT REPOSITORY dbt_repo
  API_INTEGRATION = gitbucket_api_integration
  GIT_CREDENTIALS = gitbucket_secret
  ORIGIN = 'https://gitbucket.example.com/ikeda/dbt-project.git';

-- 本番dbt projectをデプロイ (mainブランチから)
ALTER GIT REPOSITORY dbt_repo FETCH;

CREATE OR REPLACE DBT PROJECT PROD_DBT_DB.deployments.dbt_proj_prod
  FROM '@dbt_repo/branches/main/'
  COMMENT = '本番用dbtプロジェクト';


profiles.yml の prod targetが PROD_DB を指していれば、 EXECUTE DBT PROJECT ... ARGS='build --target prod' で本番DBに書き込みます。
2. CI側のセットアップ (PR作成時に動的)
ここがポイント。PRごとに以下が走るイメージ:

-- (a) PR用データDBを本番からzero-copy clone
CREATE DATABASE PR_123_DB CLONE PROD_DB;

-- (b) PR用のDBT PROJECT objectを別スキーマに作る
CREATE OR REPLACE DBT PROJECT CI_DBT_DB.pr_objects.dbt_proj_pr_123
  FROM '@dbt_repo/branches/pr-123/'
  COMMENT = 'PR #123 CI用';

-- (c) PR用targetでビルド (profiles.yml にciターゲットを定義しておく)
EXECUTE DBT PROJECT CI_DBT_DB.pr_objects.dbt_proj_pr_123 
  ARGS='build --target ci --vars "{pr_database: PR_123_DB}"';

-- (d) PR closeで全削除
DROP DBT PROJECT CI_DBT_DB.pr_objects.dbt_proj_pr_123;
DROP DATABASE PR_123_DB;


3. profiles.yml で target を切り分け

snowflake:
  outputs:
    prod:
      type: snowflake
      database: PROD_DB                # 本番データ書き込み先
      schema: marts
      # ...
    
    ci:
      type: snowflake
      database: "{{ var('pr_database') }}"  # PR_123_DB が変数で渡される
      schema: marts
      # ...


CI実行をどこから引き金にするか
ここで選択肢が分岐します。
選択肢C-1: Snowflake CLI (推奨) ⭐
dbt projects on Snowflakeは、GitHub Actionsなどのツールを通じて、Snowflake CLIによるデプロイメントとテストのCI/CD管理に完全に対応している ￼とのこと。GitBucket CI PluginからもSnowflake CLIを叩けば同じことができます。

# GitBucket CI Pluginから呼ばれるci.sh
#!/bin/bash
PR_NUMBER=${CI_PULL_REQUEST_ID}

# Snowflake CLI でdeploy
snow dbt deploy dbt_proj_pr_${PR_NUMBER} \
  --database CI_DBT_DB \
  --schema pr_objects \
  --source-path ./

# Snowflake CLI でexecute
snow dbt execute dbt_proj_pr_${PR_NUMBER} \
  --database CI_DBT_DB \
  --schema pr_objects \
  --args "build --target ci --vars '{pr_database: PR_${PR_NUMBER}_DB}'"


メリット: 完全にCI環境がSnowflake CLIで標準化される。本番デプロイも同じCLIを使うので一貫性◎
選択肢C-2: SQL直接実行

# GitBucket CI Pluginから snowsql で SQL を叩く
snowsql -c ci_connection -q "
  CREATE DATABASE PR_${PR_NUMBER}_DB CLONE PROD_DB;
  CREATE OR REPLACE DBT PROJECT CI_DBT_DB.pr_objects.dbt_proj_pr_${PR_NUMBER}
    FROM '@dbt_repo/branches/pr-${PR_NUMBER}/';
  EXECUTE DBT PROJECT CI_DBT_DB.pr_objects.dbt_proj_pr_${PR_NUMBER}
    ARGS='build --target ci --vars \"{pr_database: PR_${PR_NUMBER}_DB}\"';
"


メリット: snowsqlだけで完結、依存少ないデメリット: エラーハンドリング・ログ取得が煩雑
重要な留意点
1. dbt project object のバージョン管理
dbt Project errors (compile or test failures) はquery failuresとして表示され、tasksや他のオーケストレーションツールでハンドリングしやすくなりました。SYSTEM$get_dbt_log(’<query_id>’)で詳細なログが見られます ￼。
つまりEXECUTE DBT PROJECT の戻り値 = クエリ成否 = CI/CDの判定に直結します。これでCIの成功/失敗判定がシンプルになりました。
2. dbt deps の扱い
deployコマンドやupdateコマンドで EXTERNAL_ACCESS_INTEGRATIONS=[…ext] を設定することで、deployment時にdbt depsを実行してプロジェクト依存をインストールできます。これにより、外部依存があるプロジェクトをデプロイする際に /dbt_packages を含める必要がなくなりました ￼。
これで packages.yml で外部パッケージを使うプロジェクトもスムーズにCIに乗せられます。
3. クリーンアップ

-- PR close時のクリーンアップ
DROP DBT PROJECT IF EXISTS CI_DBT_DB.pr_objects.dbt_proj_pr_${PR_NUMBER};
DROP DATABASE IF EXISTS PR_${PR_NUMBER}_DB;


念のため週次バッチで「7日以上前のPR_*」を一括削除する保険を入れておく。
池田さんの構成での全体像

[GitBucket - PR #123 created]
       ↓
[GitBucket CI Plugin (.gitbucket/ci.yml)]
       ↓ Docker container (snowflake-cli入り) を起動
       ↓
[snow CLI が Snowflake にAPI call]
       ↓
[Snowflake内]
  ├─ DB Clone:        PR_123_DB ← PROD_DB
  ├─ DBT PROJECT作成:  CI_DBT_DB.pr_objects.dbt_proj_pr_123
  ├─ Execute:         build --target ci  
  └─ 成否を query result で返却
       ↓
[GitBucket PR にステータス反映]

[PR closed]
       ↓
[GitBucket CI Plugin (.gitbucket/teardown.yml)]
       ↓
[snow CLI で DROP]
  ├─ DROP DBT PROJECT
  └─ DROP DATABASE


池田さんの当初案 (Docker dbt-core) との比較



|観点           |Docker dbt-core       |Snowflake内 dbt project (C-1)                  |
|-------------|----------------------|----------------------------------------------|
|本番との挙動差      |dbtバージョン差・接続方式差リスク    |**完全に同じ実行エンジン**                               |
|Docker runner|必要                    |snow CLI動くだけで良い (極小コンテナでOK)                   |
|ネットワーク経路     |NAT GW + Snowflake許可IP|Snowflake CLI → SF API (パブリック)                |
|dbt実行ログ      |Docker stdout         |Snowflake Query History + `SYSTEM$get_dbt_log`|
|並列実行         |Docker側で制御            |Snowflakeのconcurrency制御                       |
|デバッグ         |ローカル再現容易              |Snowsightで実行履歴確認                              |

まとめ
できます。むしろ別々の場所が標準パターンです。
3つの分離レベル(プロジェクト定義の場所、データ書き込み先、dbt_packages置き場)をすべてCI/本番で分離するのが定石。GAされたばかりなので、これから池田さんがクライアントPoCで導入されるなら、選択肢C-1 (Snowflake CLI経由) が最も筋が良いです。
確認したいこと:
	∙	本番環境はSnowflake Enterprise Edition以上ですか? (dbt projects GA機能の前提)
	∙	GitBucketからSnowflake API integrationの設定権限は取れそうですか?
	∙	既存のSnowflakeアカウントのリージョン制約はありますか?
次に進むなら、選択肢C-1の具体実装 (snow CLIを使ったci.sh、profiles.yml、teardown.sh) を書き起こせます。やりますか?​​​​​​​​​​​​​​​​
------

はい、秘密鍵を AWS Secrets Manager で管理して問題ありません。むしろ、EC2やLambdaの環境変数、Git、S3平置きより安全です。

ただし注意点があります。

注意点
	1.	KMSはカスタマー管理キー推奨
デフォルトの aws/secretsmanager でも暗号化されますが、重要な秘密鍵なら Customer Managed KMS Key を使う方が、権限・監査・無効化を細かく制御できます。Secrets Manager はKMSのデータキーでシークレット値を暗号化します。 ￼
	2.	IAM権限は最小化
アプリには以下だけを許可するのが基本です。

{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:ap-northeast-1:xxxx:secret:your-secret-*"
}

KMSを独自キーにする場合は、該当ロールに kms:Decrypt も必要です。

	3.	秘密鍵をログに出さない
取得後に print(secret)、例外ログ、CloudWatch Logs、CIログに出る事故が多いです。
特に秘密鍵は改行入りなので、デバッグ出力に注意です。
	4.	Terraformで登録する場合は tfstate に残る
aws_secretsmanager_secret_version.secret_string に秘密鍵を直接書くと、Terraform state に平文相当で残る可能性があります。AWSもTerraform stateの機密情報リスクを注意喚起しています。 ￼
可能なら初回投入はCLIや手動、またはCIの安全なシークレット経由にします。
	5.	ローテーション設計
DBパスワードと違い、秘密鍵は自動ローテーションが難しいことがあります。
使い先、公開鍵の差し替え、接続先の許可登録まで含めて「新旧併用期間」を設計します。
	6.	Lambda/VPCから使うならVPCエンドポイント
LambdaがVPC内でインターネットに出られない構成なら、Secrets Manager の VPC エンドポイントを作るのが推奨です。 ￼

おすすめ構成

秘密鍵をSecrets Managerに保存
→ KMSはCustomer Managed Key
→ アプリのIAMロールだけ GetSecretValue + kms:Decrypt
→ CloudTrailで取得履歴を監査
→ Terraform stateには秘密鍵を入れない

結論として、管理場所としてはSecrets ManagerでOKです。
一番のリスクは「保存先」よりも、取得できるIAMロールが広すぎることとログ・tfstateへの漏洩です。


------------------------
結論から言うと、名寄せの精度と運用次第ですが、多くの場合は簡易的な画面があった方が良いです。

画面が不要なケース

以下のような場合は、Snowflakeだけでも運用できます。
	•	名寄せルールが明確
	•	自動判定率が95%以上
	•	月数件しか例外が発生しない
	•	データ修正をSQLで実施できる担当者がいる

Snowflake
  ↓
例外テーブル
  ↓
担当者がSQLで確認
  ↓
判断結果を登録

この場合はExcelやCSV出力でも十分です。

⸻

画面が必要なケース

こちらの方が実務では多いです。
	•	顧客データ
	•	取引先マスタ
	•	会員データ
	•	医療データ
	•	複数システム統合

例えば、

候補1
  山田 太郎
  東京都〇〇区
  090-xxxx

候補2
  山田 太朗
  東京都○○区
  090-xxxx

人間なら同一人物と判断できますが、

氏名
住所
電話番号
メール
過去の判断履歴

を横に並べて見たいですよね。

そのため、

例外一覧画面
  ↓
詳細比較画面
  ↓
同一人物
別人物
保留

のような画面があると運用負荷がかなり下がります。

⸻

私ならどうするか

ユーザーさんはSnowflakeやAWSをよく触られているので、最初は大掛かりな画面を作りません。

Phase1

Snowflake
  ↓
exceptionテーブル
  ↓
Sigma
QuickSight
Streamlit

で確認

⸻

Phase2

例外件数が増えたら

Snowflake
  ↓
Streamlit in Snowflake

を検討します。

Streamlit はSnowflake上で動くため、
	•	例外一覧
	•	類似顧客比較
	•	同一/別人ボタン
	•	コメント入力

程度なら比較的短期間で作れます。

⸻

画面を作るなら最低限欲しい機能

機能	必要度
例外一覧	必須
候補比較表示	必須
同一人物判定	必須
別人物判定	必須
コメント	推奨
判定履歴	推奨
一括処理	後回し
ワークフロー承認	後回し


⸻

ユーザーさんが検討されているのが「顧客名寄せ」なのか「取引先名寄せ」なのか、それとも「複数システムのマスタ統合」なのかで、必要な画面や設計がかなり変わります。対象データを教えていただければ、Snowflake中心での具体的なアーキテクチャ（Streamlit、dbt、AWS連携含む）まで提案できます。


-------------------
おすすめは、自動名寄せで無理に決め切らず、イレギュラーを「例外管理テーブル」に逃がす設計です。

Snowflake上の元データ
  ↓
正規化
  ↓
自動名寄せ
  ↓
判定
   ├ 高信頼度 → 自動統合
   ├ 中信頼度 → 人手確認
   └ 低信頼度/矛盾あり → 例外管理

イレギュラーケースは、主にこの4種類に分けると扱いやすいです。

ケース	例	対応
表記ゆれ	株式会社ABC / ABC株式会社	正規化ルールで吸収
情報不足	電話番号なし、住所なし	信頼度を下げて保留
矛盾	同じ電話番号だが住所が別	例外管理へ
多対多	AとBも同一候補、BとCも同一候補	人手確認へ

重要なのは、名寄せ結果をいきなり上書きしないことです。

raw_customer
  元データ

normalized_customer
  正規化済みデータ

match_candidate
  名寄せ候補とスコア

customer_master
  採用済みの統合結果

match_exception
  判断不能・矛盾・人手確認対象

match_decision_log
  人がどう判断したかの履歴

特に match_exception を作るのが大事です。

create table match_exception (
  exception_id string,
  source_customer_id string,
  candidate_customer_id string,
  exception_type string,
  match_score number(5,2),
  reason string,
  status string, -- OPEN / REVIEWED / RESOLVED / IGNORED
  reviewed_by string,
  reviewed_at timestamp,
  created_at timestamp
);

判定ルールは例えばこうします。

スコア 95以上   → 自動統合
スコア 80〜94   → 人手確認
スコア 80未満   → 別人扱い
ただし矛盾条件あり → 例外管理

矛盾条件の例です。

同一候補なのに生年月日が違う
同一候補なのに法人番号が違う
同一電話番号だが氏名が大きく違う
同一メールだが別顧客IDに紐づく
同一住所だが世帯・法人が複数存在

運用としては、例外を人が確認し、その判断結果を次回以降のルールに戻すのが良いです。

例外発生
  ↓
担当者が確認
  ↓
同一 / 別人 / 保留 を判断
  ↓
decision_logに保存
  ↓
次回バッチではその判断を優先

つまり、名寄せは「一回で完璧にする処理」ではなく、例外を貯めて精度を上げる業務プロセスとして作るのが現実的です。

最初の実装方針としては、これが安全です。

1. 自動統合はかなり厳しめ
2. 怪しいものは例外管理へ
3. 人手判断を履歴化
4. 判断済みルールを次回処理で優先
5. 元データは絶対に消さない

この形にしておけば、Snowflake上でもテストしやすく、後からdbtやPython処理にもつなげやすいです。

--------------/
Snowflakeとの連携部分のAWSセキュリティ構成として、一般的なものをまとめます。

全体構成イメージ

外部データソース
    ↓
[S3バケット] ← データ投入
    ↓
[Glue / Lambda / ECS] ← ELT処理
    ↓
[Snowflake] ← PrivateLinkで接続


1. ネットワーク接続のセキュリティ
AWS PrivateLink（最重要）
	∙	SnowflakeへのトラフィックをインターネットNOT経由にする
	∙	VPC内からSnowflakeエンドポイントに直接接続
	∙	池田さんが設計されているようなIP制限と組み合わせると二重防御になる

VPC → VPCエンドポイント → Snowflake PrivateLink


セキュリティグループ設定
	∙	ECS/Glue/LambdaのアウトバウンドをSnowflakeのPrivateLinkエンドポイントのみ許可
	∙	不要なアウトバウンドは全拒否

2. 認証情報管理
AWS Secrets Manager（定番）

Lambda/ECS/Glue
→ Secrets Managerからクレデンシャル取得
→ Snowflakeに接続


	∙	Snowflakeのユーザー名・パスワード・アカウント識別子を保存
	∙	自動ローテーション設定推奨
	∙	ハードコード厳禁
キーペア認証（より推奨）
	∙	パスワードの代わりにRSAキーペアでSnowflake認証
	∙	秘密鍵をSecrets ManagerまたはSSM Parameter Storeに保存
	∙	パスワードレスで漏洩リスクを下げられる

3. IAMの設計
Snowflake Storage Integration用IAMロール

S3 → Snowflake External Stage構成の場合：

IAMロール（Snowflakeが引き受ける）
  └ S3バケットへのGetObject/ListBucket権限のみ付与
  └ 信頼ポリシーにSnowflakeアカウントARNを指定


ELT処理側のIAMロール

ECSタスクロール / Lambda実行ロール
  └ Secrets Managerへのアクセス権限
  └ S3への必要最小限の権限
  └ CloudWatch Logsへの書き込み権限


最小権限の原則を徹底することが重要です。

4. データ転送のセキュリティ
S3バケット設定
	∙	パブリックアクセス完全ブロック
	∙	SSE-S3またはSSE-KMSで暗号化
	∙	バケットポリシーでSnowflake Storage IntegrationのIAMロールのみ許可
	∙	アクセスログ有効化
KMS（推奨）

S3暗号化キー → AWS KMS管理
Secrets Managerの暗号化 → KMS


	∙	キーポリシーで利用者を厳密に制限

5. 監視・検知
CloudTrail
	∙	S3へのアクセスログ
	∙	Secrets Managerへのアクセスログ
	∙	IAMロールのAssumeRoleログ
GuardDuty
	∙	Snowflake接続元の異常な通信検知
	∙	認証情報の不正利用検知
Macie
	∙	S3の機密データ混入チェック
	∙	Snowflakeにロードする前に個人情報を検出
CloudWatch
	∙	ELTジョブのエラー監視
	∙	異常なデータ転送量のアラート

6. 全体セキュリティレイヤーまとめ



|レイヤー  |使うサービス                               |
|------|-------------------------------------|
|ネットワーク|PrivateLink・セキュリティグループ・VPC           |
|認証情報  |Secrets Manager・キーペア認証               |
|アクセス制御|IAMロール最小権限・バケットポリシー                  |
|暗号化   |KMS・S3暗号化・TLS通信                      |
|監視・検知 |CloudTrail・GuardDuty・Macie・CloudWatch|
|集約管理  |Security Hub                         |

池田さんの設計文脈で特に重要な点
ELTサービスアカウント設計やSnowflakeのロール階層を既に検討されているので、AWS側との対応関係を意識すると整合性が取りやすいです。

Snowflakeロール階層          AWS側
─────────────────────────────────────
SYSADMIN配下のELTロール  ↔  ECSタスクロール / Glue IAMロール
LOADER専用ロール         ↔  S3 Storage Integration IAMロール
監査ロール              ↔  CloudTrail + Security Hub


この対応関係をPoC文書に落とし込むと、クライアントへの説明がかなりしやすくなると思います。​​​​​​​​​​​​​​​​

-------------------------/
Macie 設定ガイド

必須設定
	∙	有効化するだけでバケット監視が始まる（インベントリ・設定リスクは自動）
	∙	Organizationsなら委任管理者アカウントを指定して全アカウント集約
	∙	全リージョンで有効化（S3はリージョン単位なので）

おすすめ設定
1. 機密データ検出ジョブの作成
有効化しただけではバケット設定の監視のみで、中身のスキャンは別途ジョブ設定が必要です。

Macieコンソール
→ 「検出ジョブを作成」
→ スキャン対象バケットを選択
→ スケジュール：定期実行（週次・月次）を推奨


優先的にスキャンすべきバケット：
	∙	外部システムからデータ投入されるバケット
	∙	ログ集約バケット
	∙	データ共有・受け渡し用バケット

2. カスタムデータ識別子の追加
日本固有のパターンはデフォルト検出が弱いので追加推奨です。

例：社員番号、顧客ID、社内コードなど
正規表現で独自パターンを登録できる


3. アラート通知の設定

Macie検出結果
→ EventBridge
→ SNS or Lambda
→ Slack / メール / チケット起票


通知すべき重要度：
	∙	Critical / High → 即時通知
	∙	Medium → 日次サマリー通知

4. 抑制ルールの設定
意図的に公開しているバケット（静的Webサイトホスティングなど）は抑制ルールで除外してノイズを減らします。

5. Security Hubとの統合
MacieをSecurity Hubに統合して、GuardDuty・Inspectorの結果と一元管理します。

コスト注意点
Macieはスキャンしたデータ量に応じて課金されるため：
	∙	最初はサンプリングジョブ（一部スキャン）で概算コストを確認
	∙	機密データが少ないと分かったバケットはスキャン頻度を下げる
	∙	S3インベントリ・設定監視は無料枠あり（最初の30日）

優先度まとめ



|優先度 |設定内容                 |
|----|---------------------|
|🔴 必須|有効化・全リージョン・委任管理者設定   |
|🟠 高 |機密データ検出ジョブ作成・通知設定    |
|🟡 中 |カスタム識別子・抑制ルール        |
|🟢 任意|Security Hub統合・コスト最適化|

-------------/-----------
3サービスそれぞれまとめます。

GuardDuty
必須設定
	∙	使用する全リージョンで有効化（忘れがちなので注意）
	∙	Organizationsなら管理アカウントから一括有効化
おすすめ設定
	∙	S3保護 を有効化（デフォルトOFFなので明示的にONに）
	∙	EKS保護 / Lambda保護 を有効化（使っている場合）
	∙	マルウェアスキャン を有効化（EC2のEBSボリュームをスキャン）
	∙	検出結果をEventBridge → SNSでメール通知するように設定
	∙	抑制ルールで既知の正常な通信をフィルタしてノイズを減らす

Security Hub
必須設定
	∙	GuardDuty・Inspector・Macieなど連携サービスの統合を有効化
	∙	Organizationsなら管理アカウントを委任管理者に指定して全アカウント集約
おすすめ設定
	∙	AWS基礎セキュリティベストプラクティス（FSBP） を有効化（最優先）
	∙	CIS AWS Foundations Benchmark も有効化（コンプライアンス対応なら必須）
	∙	スコアが80%以上を目標に改善していく
	∙	Critical/High の検出結果をEventBridge → Slackやチケットに自動連携
	∙	自動応答ルール（Automation Rules）で重複や低優先度をフィルタ

Inspector
必須設定
	∙	EC2・ECR・Lambdaそれぞれスキャン対象を有効化（リソース種別ごとにONが必要）
	∙	Organizationsなら委任管理者アカウントを指定
おすすめ設定
	∙	ECRの継続スキャンをON（pushのたびに自動スキャン）
	∙	Lambda標準スキャン + コードスキャン両方を有効化
	∙	Critical CVEをEventBridge → チケット起票に自動連携
	∙	スコア9.0以上（Critical）は即対応、7.0以上は1週間以内、というSLAを決める
	∙	EC2はSSMエージェントが入っていることを確認（ないとスキャン精度が落ちる）

3つをまとめて運用するコツ

Inspector → 脆弱性発見
GuardDuty → 攻撃・侵害を検知       } → Security Hub に集約 → 通知・自動対応
Macie     → データ漏洩リスク検出


Security Hubのスコアを週次でレビューする習慣を作ると、継続的な改善サイクルが回りやすくなります。​​​​​​​​​​​​​​​​

--------------------////

良いところに気付かれましたね。これはSnowflakeの設計でよく議論になるポイントで、「論理的にはアカウント全体で共通化したいが、物理的にはスキーマ配下にしか作れない」というギャップをどう埋めるかという話です。いくつかの定石パターンがあります。
解決の基本方針：「集約スキーマ」を作る
File Formatをスキーマ配下にしか作れないなら、「全社共通オブジェクトを置くための専用スキーマ」を1つ作って、そこに集約するのが定石です。物理的にはスキーマ配下ですが、運用上は「アカウント全体の共通リソース」として扱います。
パターン1：共通DBに集約（推奨）
全社共通のオブジェクトを置く専用DBを作り、そこのスキーマに集約します。

-- 全社共通リソース用DB
CREATE DATABASE COMMON_DB;
CREATE SCHEMA COMMON_DB.FILE_FORMATS;
CREATE SCHEMA COMMON_DB.FUNCTIONS;    -- 共通UDF
CREATE SCHEMA COMMON_DB.MASKING;      -- 共通マスキングポリシー

-- 共通File Format定義
CREATE FILE FORMAT COMMON_DB.FILE_FORMATS.FF_CSV_STANDARD
  TYPE = CSV FIELD_DELIMITER = ',' SKIP_HEADER = 1
  NULL_IF = ('NULL', 'null', '', 'N/A')
  EMPTY_FIELD_AS_NULL = TRUE
  COMMENT = '全社標準CSVフォーマット';

CREATE FILE FORMAT COMMON_DB.FILE_FORMATS.FF_CSV_TSV
  TYPE = CSV FIELD_DELIMITER = '\t' SKIP_HEADER = 1;

CREATE FILE FORMAT COMMON_DB.FILE_FORMATS.FF_JSON_STANDARD
  TYPE = JSON STRIP_OUTER_ARRAY = TRUE;

CREATE FILE FORMAT COMMON_DB.FILE_FORMATS.FF_PARQUET
  TYPE = PARQUET;


参照する側は完全修飾名で指定します：

COPY INTO ANALYTICS.RAW.SALES
FROM @ANALYTICS.STAGES.S3_SALES
FILE_FORMAT = COMMON_DB.FILE_FORMATS.FF_CSV_STANDARD;


これにより、File Formatの実体は1箇所だけになり、NULL_IF や FIELD_DELIMITER の仕様変更も1箇所で完結します。
権限設計：READER ロールを作る
共通DBの利用権限を整理するには、専用のアクセスロールを作ります。

-- 共通リソース参照用アクセスロール
CREATE ROLE COMMON_READER;

GRANT USAGE ON DATABASE COMMON_DB TO ROLE COMMON_READER;
GRANT USAGE ON SCHEMA COMMON_DB.FILE_FORMATS TO ROLE COMMON_READER;
GRANT USAGE ON ALL FILE FORMATS IN SCHEMA COMMON_DB.FILE_FORMATS 
  TO ROLE COMMON_READER;
GRANT USAGE ON FUTURE FILE FORMATS IN SCHEMA COMMON_DB.FILE_FORMATS 
  TO ROLE COMMON_READER;

-- 必要な機能ロールすべてに継承
GRANT ROLE COMMON_READER TO ROLE SVC_LOADER;
GRANT ROLE COMMON_READER TO ROLE SVC_TRANSFORMER;
GRANT ROLE COMMON_READER TO ROLE DATA_ENGINEER;
GRANT ROLE COMMON_READER TO ROLE DEVELOPER;


COMMON_READER ロールを「全社共通リソースを使える権利」として独立させると、新しい機能ロールが増えた時もこれを継承させるだけで済みます。
パターン2：ANALYTICS DB内の共通スキーマ
別DBを切るほどではない規模なら、分析DB内に共通スキーマを置く方法もあります。

CREATE SCHEMA ANALYTICS._COMMON;  -- アンダースコアで「特殊」を示す
CREATE SCHEMA ANALYTICS._FILE_FORMATS;


スキーマ名にアンダースコアプレフィックスを付けると、SHOW SCHEMASの一覧で先頭に並んで「これは特別なスキーマ」と視覚的に分かります。命名の小技ですが効果的です。
ただし、本番DBと開発DB（ANALYTICS_PROD / ANALYTICS_DEV）に分けている場合、共通スキーマも各DBに重複して持つことになるので、規模が出てきたらパターン1に移行するのが筋です。
パターン3：UTILS DB という呼び方
「COMMON」という名前に抵抗があれば、UTILS_DB、SHARED_DB、META_DB といった命名も実務でよく見ます。意図が伝わる名前なら何でもOKです。
ただし SHARED はSnowflakeのデータシェアリング機能と紛らわしいので避けた方が無難です。
File Formatの分類整理
共通DBに置くFile Formatは、用途別に整理しておくと使う側が迷いません。

-- 型 × 用途で命名
FF_CSV_STANDARD       -- 標準CSV（カンマ・ヘッダーあり・標準NULL）
FF_CSV_NO_HEADER      -- ヘッダーなしCSV
FF_TSV_STANDARD       -- TSV
FF_JSON_STANDARD      -- 標準JSON
FF_JSON_NESTED        -- ネストJSON（STRIP_OUTER_ARRAY=FALSE）
FF_PARQUET            -- Parquet
FF_AVRO               -- Avro
FF_XML                -- XML


「とりあえずこれを使えば標準」というデフォルト形式（_STANDARD サフィックス）を用意しておくと、新人エンジニアが迷わずに済みます。
例外的に個別File Formatを作るケース
「共通化したい」と言っても、全部を共通化するのは無理です。以下のようなケースは個別のスキーマに専用File Formatを置く方が良いです。
	∙	特定システムからのCSVが特殊なエンコーディング（Shift-JIS、CP932等）
	∙	特定ベンダーからのデータで区切り文字が独自（パイプ|、@等）
	∙	レガシーシステムからの固定長フォーマット
	∙	一時的な調査用の使い捨てフォーマット
この場合は、共通File Formatに FF_CSV_STANDARD、ベンダー個別に ANALYTICS.RAW.FF_CSV_VENDOR_A_SJIS という構造で共存させます。「迷ったら共通、特殊なら個別」のルールを明文化しておくと運用が楽です。
Storage Integrationも同じ思想で集約
Storage Integrationはそもそもアカウントレベルなので集約問題は起きませんが、「どのDBから参照するためのIntegrationか」が紐づくので、命名規約で関係を明示するのが定石です。

-- Storage Integration命名: SI_<用途>_<クラウド>
CREATE STORAGE INTEGRATION SI_LANDING_S3 ...;       -- 受信用
CREATE STORAGE INTEGRATION SI_EXPORT_S3 ...;        -- エクスポート用
CREATE STORAGE INTEGRATION SI_VENDOR_A_S3 ...;      -- 特定ベンダー用


これらへのUSAGE権限をどのロールに付けるかは、データの機密度に応じて分けます。
Stageは共通化しない方が良い
ここが重要なのですが、File Formatは共通化しやすい一方、Stageは共通化しない方が良いです。理由は：
	∙	Stageは「どこのS3バケットを指すか」という個別の情報を持つ
	∙	バケットごとに権限境界が異なる（このバケットはチームAだけ、等）
	∙	データソースごとにバケット/プレフィックスが違うのが普通
なのでStageは各データソース単位、もしくは各RAWスキーマ単位で作るのが自然です。

COMMON_DB.FILE_FORMATS.FF_CSV_STANDARD   ← 共通化
ANALYTICS.RAW.STAGE_SALES_S3              ← 個別（営業データ用）
ANALYTICS.RAW.STAGE_HR_S3                 ← 個別（人事データ用）
HR_DB.RAW.STAGE_HR_PRIVATE                ← 個別（機密人事データ用、別DB）


「File Formatは仕様、Stageは実体」と捉えると整理しやすいです。仕様は再利用可能、実体は個別、という関係です。
共通UDFやマスキングポリシーも同じパターン
実はFile Formatと同じ「スキーマ配下だけど全社共通化したい」課題は、他のオブジェクトでも発生します。
	∙	共通UDF（独自の日付フォーマット変換、文字列クリーニング等）
	∙	共通マスキングポリシー（メアド・電話番号・氏名のマスキング）
	∙	共通Row Access Policy（部門コードでのフィルタリングロジック）
	∙	共通タグ（PIIタグ、機密度タグ）
これらもすべて COMMON_DB 配下に集約するのが定石です。

COMMON_DB.FILE_FORMATS.*       -- ファイル形式定義
COMMON_DB.FUNCTIONS.*          -- 共通UDF
COMMON_DB.POLICIES.*           -- マスキング・行アクセスポリシー
COMMON_DB.TAGS.*               -- タグ定義


PoCの早い段階で COMMON_DB を作っておくと、後から「あ、これも共通化したかった」となった時に置き場所に困りません。
クライアントPoC向けの実装ステップ
池田さんのクライアントPoC向けに、具体的にどう進めるかを段階的に示すと：
Step 1: COMMON_DB を立てる

CREATE DATABASE COMMON_DB COMMENT = '全社共通リソース集約DB';
CREATE SCHEMA COMMON_DB.FILE_FORMATS;


Step 2: 標準File Formatを定義
最初はCSV標準とJSON標準だけで十分。必要に応じて追加。
Step 3: COMMON_READER アクセスロールを作る

CREATE ROLE COMMON_READER 
  COMMENT = '全社共通リソースの参照権限';
GRANT USAGE ON DATABASE COMMON_DB TO ROLE COMMON_READER;
GRANT USAGE ON ALL SCHEMAS IN DATABASE COMMON_DB TO ROLE COMMON_READER;
GRANT USAGE ON ALL FILE FORMATS IN SCHEMA COMMON_DB.FILE_FORMATS 
  TO ROLE COMMON_READER;
GRANT USAGE ON FUTURE FILE FORMATS IN SCHEMA COMMON_DB.FILE_FORMATS 
  TO ROLE COMMON_READER;


Step 4: 全機能ロールに継承させる

GRANT ROLE COMMON_READER TO ROLE SVC_LOADER;
GRANT ROLE COMMON_READER TO ROLE SVC_TRANSFORMER;
GRANT ROLE COMMON_READER TO ROLE DEVELOPER;
GRANT ROLE COMMON_READER TO ROLE ANALYST;


Step 5: 利用ガイドラインを文書化
「新規ロード処理を作る場合は、まず COMMON_DB.FILE_FORMATS のFile Formatで対応可能か確認すること。対応できない特殊フォーマットのみ各スキーマに個別作成可」というルールを明文化。これがないと、結局各人が好き勝手にFile Formatを作って共通化の意味がなくなります。
まとめ
	∙	File Formatは物理的にはスキーマ配下、論理的にはアカウント全体共通にできる
	∙	COMMON_DB.FILE_FORMATS のような集約スキーマを作る
	∙	COMMON_READER アクセスロールを作って各機能ロールに継承
	∙	Stageは個別、File Formatは共通、という抽象度の違いで分ける
	∙	共通UDF・ポリシー・タグも同じパターンで集約できる
	∙	ガイドラインを文書化して規律ある運用にする
この構造にしておくと、池田塾でDB設計を教える時にも「論理設計と物理配置の分離」という良い教材になりますね。スキーマというSnowflakeの物理単位を、論理的な「全社共通領域」として再定義する設計判断、と説明できます。
COMMON_DB 内には他にどんなオブジェクトを置く想定がありそうですか？ UDF（特に日本固有の和暦変換、全角半角統一、郵便番号正規化など）をどう設計するかも、PoCのうちに整理しておくと後の本番展開で効きます。​​​​​​​​​​​​​​​​


--------------------------
いえ、ファイルフォーマットとステージはスキーマ配下のオブジェクトです。DB/スキーマの階層に属するので、テーブルやビューと同じ扱いになります。ロール設計上もこの点を踏まえる必要があります。
オブジェクトの所属階層
Snowflakeの主要オブジェクトの所属を整理すると：
アカウント直下
	∙	Warehouse、Database、User、Role、Network Policy、Resource Monitor、Share
データベース直下
	∙	Schema、Database Role
スキーマ直下
	∙	Table、View、Stage、File Format、Function、Procedure、Stream、Task、Pipe、Sequence、Masking Policy、Row Access Policy
つまりFile FormatとStageは <DB>.<SCHEMA>.<OBJECT> の形でフルパス指定する、スキーマレベルのオブジェクトです。

-- 完全修飾名で参照
COPY INTO ANALYTICS.RAW.SALES
FROM @ANALYTICS.RAW.S3_STAGE_SALES
FILE_FORMAT = ANALYTICS.RAW.FF_CSV_STANDARD;


ステージの種類による違い
ステージには3種類あり、所属が異なるので注意です。
Named Stage（名前付きステージ）
	∙	スキーマ配下に明示的に作成
	∙	CREATE STAGE ANALYTICS.RAW.S3_STAGE_SALES ...
	∙	通常の権限管理対象、ロール設計に組み込むのはこれ
User Stage（ユーザーステージ）
	∙	各ユーザーに自動で1つ存在、@~ で参照
	∙	そのユーザー本人だけがアクセス可能
	∙	他人に共有不可、ロール設計には組み込まない
Table Stage（テーブルステージ）
	∙	各テーブルに自動で1つ存在、@%table_name で参照
	∙	そのテーブルへの権限を持つロールが利用可能
	∙	個別の権限管理は不要、テーブル権限に従う
ELTパイプラインで設計するのは基本的にNamed Stageです。User StageとTable Stageは個人作業やアドホックな用途で使うものなので、本番の権限設計には登場しません。
ロール設計への組み込み方
ステージとファイルフォーマットは「どこに配置するか」がロール設計に直結します。配置場所のパターンは3つあります。
パターンA: 専用スキーマに集約（推奨）
RAW スキーマ内に「テーブル群」とは別に「ロード用オブジェクト群」を置く、または専用スキーマ STAGES を作る方法。

-- 専用スキーマパターン
CREATE SCHEMA ANALYTICS.STAGES;
CREATE SCHEMA ANALYTICS.FILE_FORMATS;

CREATE STAGE ANALYTICS.STAGES.S3_RAW_SALES
  URL = 's3://...'
  STORAGE_INTEGRATION = my_s3_int;

CREATE FILE FORMAT ANALYTICS.FILE_FORMATS.FF_CSV_STANDARD
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1;


権限制御は：

-- ロード系アクセスロールに付与
GRANT USAGE ON SCHEMA ANALYTICS.STAGES TO ROLE ANALYTICS_LOAD_RW;
GRANT USAGE ON STAGE ANALYTICS.STAGES.S3_RAW_SALES TO ROLE ANALYTICS_LOAD_RW;
GRANT READ ON STAGE ANALYTICS.STAGES.S3_RAW_SALES TO ROLE ANALYTICS_LOAD_RW;

GRANT USAGE ON SCHEMA ANALYTICS.FILE_FORMATS TO ROLE ANALYTICS_LOAD_RW;
GRANT USAGE ON FILE FORMAT ANALYTICS.FILE_FORMATS.FF_CSV_STANDARD TO ROLE ANALYTICS_LOAD_RW;


メリット：
	∙	ロード用オブジェクトの集中管理ができる
	∙	複数のRAWスキーマ（部門別など）で同じFile Formatを共有できる
	∙	権限管理がシンプル（スキーマ単位でまとめて制御）
パターンB: RAWスキーマに同居
ロード先のテーブルと同じスキーマにステージ・ファイルフォーマットを置く方法。

CREATE STAGE ANALYTICS.RAW.S3_STAGE_SALES ...;
CREATE FILE FORMAT ANALYTICS.RAW.FF_CSV ...;
CREATE TABLE ANALYTICS.RAW.SALES ...;


メリット：
	∙	関連オブジェクトが1スキーマに集まる
	∙	アクセスロール ANALYTICS_RAW_RW 1つで全部カバーできる
デメリット：
	∙	部門別RAWスキーマがある場合、同じFile Formatを複数スキーマに重複作成することになる
パターンC: 専用データベースに分離
大規模環境で、ステージング基盤を独立DBにする方法。

CREATE DATABASE LANDING_DB;
CREATE SCHEMA LANDING_DB.STAGES;
CREATE SCHEMA LANDING_DB.FORMATS;


データレイク的な使い方や、複数の分析DBに同じソースデータを供給する構造の時に有効です。PoC段階ではオーバーキルです。
File Formatの共通化価値
File Formatはスキーマ間で再利用できるので、共通化のメリットが大きいオブジェクトです。

-- 全社共通フォーマットを1箇所に定義
CREATE FILE FORMAT ANALYTICS.FILE_FORMATS.FF_CSV_STANDARD
  TYPE = CSV FIELD_DELIMITER = ',' SKIP_HEADER = 1
  NULL_IF = ('NULL', 'null', '');

CREATE FILE FORMAT ANALYTICS.FILE_FORMATS.FF_JSON_STANDARD
  TYPE = JSON;

CREATE FILE FORMAT ANALYTICS.FILE_FORMATS.FF_PARQUET
  TYPE = PARQUET;


「NULL扱いをどうするか」「タイムスタンプ形式は何か」といった会社全体のルールをFile Formatに集約しておくと、データ品質のブレを防げます。各ロード処理は共通のFile Formatを参照するだけになります。
権限的にはこのFile Formatに対して、ロード系ロール全員に USAGE を付与すればOKです。
ステージへの権限の種類
ステージは権限の種類がやや特殊です。
	∙	USAGE: ステージの存在を認識し、参照できる（必須）
	∙	READ: ステージからの読み取り（外部ステージ・内部ステージ両方）。COPY INTO <table> FROM @stage で必要
	∙	WRITE: ステージへの書き込み。COPY INTO @stage FROM <table>（アンロード）や PUT コマンドで必要
ロード処理だけなら USAGE + READ、アンロード処理もするなら USAGE + READ + WRITE です。

-- ロード専用ロール
GRANT USAGE, READ ON STAGE ANALYTICS.STAGES.S3_RAW_SALES TO ROLE ANALYTICS_LOAD_R;

-- アンロード（データエクスポート）もするロール
GRANT USAGE, READ, WRITE ON STAGE ANALYTICS.STAGES.S3_EXPORT TO ROLE ANALYTICS_EXPORT_RW;


Storage Integrationという別レイヤー
外部ステージ（S3/Azure Blob/GCS）を使う場合、ステージの背後にStorage Integrationというアカウントレベルオブジェクトがあります。

-- ACCOUNTADMINで作成
CREATE STORAGE INTEGRATION s3_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::xxx:role/snowflake-role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://my-bucket/');

-- 利用するロールにUSAGE付与
GRANT USAGE ON INTEGRATION s3_int TO ROLE ANALYTICS_LOAD_RW;


Storage Integrationはアカウント直下のオブジェクトで、ステージ作成時に参照されます。Integrationへの USAGE 権限もロール設計に組み込む必要があります。これを忘れて「ステージは作れたのにCOPYで失敗する」というハマりがよく発生します。
FUTURE GRANTSとの組み合わせ
ファイルフォーマットやステージも FUTURE GRANTS の対象です。専用スキーマパターンを採るなら、これを設定しておくと運用が楽になります。

-- 今後ANALYTICS.STAGESに作成される全ステージへの権限を自動付与
GRANT USAGE ON FUTURE STAGES IN SCHEMA ANALYTICS.STAGES 
  TO ROLE ANALYTICS_LOAD_RW;
GRANT READ ON FUTURE STAGES IN SCHEMA ANALYTICS.STAGES 
  TO ROLE ANALYTICS_LOAD_RW;

GRANT USAGE ON FUTURE FILE FORMATS IN SCHEMA ANALYTICS.FILE_FORMATS 
  TO ROLE ANALYTICS_LOAD_RW;


PoC向けの推奨構成
200名規模のクライアントなら、こんな構成がバランス良いです。
スキーマ構成：
	∙	ANALYTICS.RAW（生データテーブル）
	∙	ANALYTICS.STAGING（変換中間テーブル）
	∙	ANALYTICS.MART（分析用マート）
	∙	ANALYTICS.STAGES（外部ステージ集約）★追加
	∙	ANALYTICS.FILE_FORMATS（File Format集約）★追加
アクセスロールへの権限付与：
	∙	ANALYTICS_LOAD_RW（ロード用）に STAGES、FILE_FORMATS、RAW のUSAGE/操作権限
	∙	ANALYTICS_RAW_R（参照用）に RAW のSELECT権限
	∙	Storage Integration S3_INT のUSAGEは ANALYTICS_LOAD_RW のみ
機能ロールへの継承：
	∙	SVC_LOADER → ANALYTICS_LOAD_RW を継承
	∙	SVC_TRANSFORMER → ANALYTICS_LOAD_RW + ANALYTICS_RAW_R + ANALYTICS_STAGING_RW + ANALYTICS_MART_RW を継承
これで「ステージとファイルフォーマットはどのロールが触れるか」が明確になり、ELTパイプライン全体の権限が見通しやすくなります。
まとめ
	∙	File FormatとStageはスキーマ配下のオブジェクトで、テーブルと同列の扱い
	∙	権限はスキーマレベルの USAGE + オブジェクトレベルの権限が必要
	∙	専用スキーマに集約（STAGES、FILE_FORMATS）するパターンが管理しやすい
	∙	Storage Integrationはアカウントレベルで、別途USAGE付与が必要
	∙	FUTURE GRANTSで新規オブジェクトへの権限付与を自動化できる
ステージの設計周りで他に気になるところはありますか？ Snowpipe（自動取り込み）を使う場合、Pipe オブジェクトの権限設計がまた別途必要になるので、その話もできます。​​​​​​​​​​​​​​​​

----------------------------------

# データウェアハウス (DWH) サービスレベル合意書 (SLA) 雛形

*ver. 0.1 / 雛形 ― 案件に応じてカスタマイズしてご利用ください*

---

## 1. 目的・適用範囲

本書は、データウェアハウス (DWH) サービスに関し、提供者と利用者の双方が合意するサービスレベル (SLA) を定義するものである。対象は本番環境を原則とし、検証/開発環境は別途定める目標値 (SLO) に準ずる。

---

## 2. SLA項目一覧

各SLAは月次で測定し、翌月第5営業日までにレポートを提出する。

### 2.1 可用性 (Availability)

| SLA項目 | 定義 / 測定方法 | 目標値の例 | 備考 |
|---|---|---|---|
| サービス稼働率 | 月次の稼働時間 / (月次総時間 − 計画停止時間) × 100 | 99.9% 以上 (本番)<br>99.5% 以上 (検証) | Snowflake自体のSLAは99.9%。上位サービスとして整理 |
| 計画メンテナンス時間帯 | 事前通知のうえ実施する停止時間。SLA稼働率算定から除外 | 日曜 02:00–06:00<br>月2回まで・各4時間以内 | 5営業日前までに通知 |
| 計画外停止 (MTTR) | 障害発生から復旧までの平均時間 | Sev1: 4時間以内<br>Sev2: 8時間以内 | 重大度の定義を別表に明記 |

### 2.2 性能 (Performance)

| SLA項目 | 定義 / 測定方法 | 目標値の例 | 備考 |
|---|---|---|---|
| BIダッシュボード応答時間 | ダッシュボード起動からデータ表示完了までの時間 (P95) | P95 ≤ 5秒<br>P99 ≤ 10秒 | Result Cache hit時を除く / WHサイズ M前提 |
| アドホッククエリ応答時間 | ユーザー実行クエリの完了時間 (P95) | P95 ≤ 60秒 | 対象テーブルサイズ・WHサイズで分類 |
| ETL/ELT 完了時刻 | 前日締めデータが指定時刻までにデータマートへ反映完了 | 毎営業日 07:00 JST まで | ソース遅延起因は除外 |
| データ鮮度 (Data Freshness) | ソースシステム発生時刻からDWH反映までの遅延 | 基幹系: 24時間以内<br>IoT: 15分以内 | セグメント別に定義 |
| 同時実行性 | ピーク時間帯に劣化なく実行できる同時クエリ数 | 100同時クエリ以上 | Multi-cluster WHで吸収 |

### 2.3 データ品質 (Data Quality)

| SLA項目 | 定義 / 測定方法 | 目標値の例 | 備考 |
|---|---|---|---|
| 完全性 | 必須カラムのNULL率 / レコード欠損率 | NULL率 < 0.1%<br>欠損率 < 0.01% | Great Expectations等で日次検証 |
| 正確性 | ソース系との突合差異率 (件数・金額) | 差異率 < 0.05% | 差異検知時は当日中に通知 |
| 一意性・整合性 | PK重複ゼロ / FK参照整合性違反ゼロ | 違反件数 = 0 | 違反発生時は処理停止 |
| 適時性 | SLA時刻までに到達したレコード比率 | ≥ 99.5% | 遅延データは別バッチで取り込み |

### 2.4 運用 (Operations)

| SLA項目 | 定義 / 測定方法 | 目標値の例 | 備考 |
|---|---|---|---|
| インシデント初動応答 | 障害通知受領から一次応答までの時間 | Sev1: 30分以内 (24×7)<br>Sev2: 2時間以内 (営業時間)<br>Sev3: 1営業日以内 | 重大度区分は附則参照 |
| 障害連絡 | 検知から関係者通知までの時間 | 検知後15分以内 | メール / Slack / 電話の多重通知 |
| バックアップ・リストア | RPO: データ損失許容時間<br>RTO: 復旧目標時間 | RPO: 24時間<br>RTO: 8時間 | Snowflake Time Travel / Fail-safeを利用 |
| 変更管理 | 本番リリースの事前承認・通知 | 通常変更: 5営業日前<br>緊急変更: 当日承認可 | CABによる承認 |

### 2.5 セキュリティ (Security)

| SLA項目 | 定義 / 測定方法 | 目標値の例 | 備考 |
|---|---|---|---|
| アクセス制御 | RBAC・ネットワークポリシー・MFA適用率 | 100% | 三層ロール階層 / Network Policy |
| 監査ログ保持 | クエリ履歴・ログイン履歴の保持期間 | 1年以上 (オンライン)<br>7年 (アーカイブ) | ACCOUNT_USAGEを外部保管 |
| アクセスレビュー | アカウント・権限の棚卸し頻度 | 四半期1回 | 結果を監査エビデンスとして保管 |

---

## 3. インシデント重大度の定義

インシデントの重大度は、業務影響度に応じて以下のとおり分類する。

| 重大度 | 定義 | 影響範囲の例 | 初動応答 / 復旧目標 |
|---|---|---|---|
| **Sev1**<br>(致命的) | サービス全体が停止 / 主要業務継続が不可 | ・全ユーザーがDWHにアクセス不可<br>・全ETLが停止<br>・データ破損 | 初動: 30分以内 (24×7)<br>MTTR: 4時間以内 |
| **Sev2**<br>(重大) | 一部機能停止 / 一部業務に重大な影響 | ・特定セグメントのダッシュボード停止<br>・特定ETLパイプライン失敗<br>・性能が著しく劣化 | 初動: 2時間以内 (営業時間)<br>MTTR: 8時間以内 |
| **Sev3**<br>(中) | 代替手段あり / 業務継続は可能 | ・個別クエリの遅延<br>・非主要レポートの不具合 | 初動: 1営業日以内<br>MTTR: 5営業日以内 |
| **Sev4**<br>(低) | 業務影響なし / 改善要望レベル | ・軽微なUI不具合<br>・ドキュメント誤記 | 初動: 5営業日以内<br>MTTR: 次回リリースで対応 |

---

## 4. 除外事項

以下に該当する事象は、SLA算定の対象外とする。

- 計画メンテナンス時間帯における停止
- クラウド事業者 (Snowflake/AWS/Azure/GCP) の起因によるサービス停止 (ベンダーSLAに準拠)
- ソースシステム側の障害・遅延に起因するデータ未着
- ユーザー起因の不適切なクエリ・WH設定変更による性能劣化
- 天災・通信回線障害等の不可抗力事象
- 顧客側ネットワーク・端末・認証基盤に起因する事象

---

## 5. 報告・レビュー

- 月次SLAレポート: 翌月第5営業日までに提出 (達成状況・未達原因・是正計画を含む)
- 四半期サービスレビュー会議: 主要KPIのトレンド分析、SLA改定の要否を協議
- 重大インシデント発生時: 暫定報告 (24時間以内) / 確定報告 (5営業日以内)
- SLA改定: 年1回、または重大な構成変更時に双方協議のうえ実施

---

### 補足: 本雛形のカスタマイズの観点

- 目標値はあくまで例示。クライアントの業務クリティカリティ (例: 経営ダッシュボードか、IoTリアルタイムかなど) に応じて段階設定する。
- Snowflake等のクラウドDWHを使う場合、ベンダーSLA (99.9%) と自社運用SLO (ETL完了・データ鮮度・クエリ性能) を明確に切り分けて記述する。
- PoC段階ではSLAではなくSLO (目標) として運用し、本番移行時にSLA化するのが定石。
- ペナルティ条項 (サービスクレジット等) を入れるかは案件次第。入れる場合は除外事項を厳密に定義する必要がある。
-------------

DWH(データウェアハウス)のSLAで一般的に定義される項目は、大きく可用性・性能・データ品質・運用の4カテゴリに分かれます。
可用性 (Availability)
	∙	稼働率: 例 99.5% / 99.9% (月次、計画停止を除く)
	∙	計画メンテナンス時間帯: 例 日曜 2:00–6:00、月◯回まで
	∙	計画外停止の上限時間: MTTR (平均復旧時間) 目標
性能 (Performance)
	∙	クエリ応答時間: BIダッシュボード ◯秒以内 (P95)、アドホック ◯分以内
	∙	ETL/ELT処理完了時刻: 例 毎日 6:00 までにデータマート更新完了
	∙	同時実行ユーザー数 / 同時クエリ数の保証値
	∙	データ鮮度 (Data Freshness): ソース到着から◯時間以内に反映
データ品質 (Data Quality)
	∙	完全性: 必須カラムのNULL率 ◯%未満
	∙	正確性: ソースとの突合差異 ◯%以内
	∙	一意性・整合性: 主キー重複ゼロ、参照整合性違反ゼロ
	∙	適時性: 遅延データの許容範囲
運用・サポート (Operations)
	∙	インシデント対応時間: Sev1 ◯分以内、Sev2 ◯時間以内 (初動応答)
	∙	障害通知: 検知後◯分以内に関係者へ通知
	∙	バックアップ・リストア: RPO ◯時間、RTO ◯時間
	∙	セキュリティ: 監査ログ保持期間、アクセスレビュー頻度
PoCでクライアントに提示する場合、Snowflakeのような Cloud DWH では稼働率はベンダーSLA(99.9%)を引用し、上位レイヤー(ETL完了時刻・データ鮮度・クエリ性能)を自社運用SLOとして切り分けるのが定番です。必要なら表形式で雛形を作りますか?​​​​​​​​​​​​​​​​

-------------


EIPの取得方法、AWS Management Console / CLI / CloudFormationの3通り紹介します。
前提: EIPの2つの取得パターン

パターン1: 先にEIPを確保 → 後でNAT Gatewayに紐付け  (推奨)
パターン2: NAT Gateway作成時に自動でEIP割当       (お手軽)


パターン1が推奨です。理由は、NAT Gateway誤削除時の復旧で同じIPを再利用できること、Snowflake Network Policyに登録するIPを先に確定できること、IaCで管理しやすいことです。
マネジメントコンソールでの取得 (パターン1)
	1.	VPCコンソールを開く
	2.	左メニューの Elastic IPs をクリック
	3.	Allocate Elastic IP address ボタン
	4.	設定画面:
	∙	Network Border Group: ap-northeast-1 (東京)
	∙	Public IPv4 address pool: Amazon’s pool of IPv4 addresses を選択
	∙	Tags: Name = nat-gw-prod-eip など分かりやすく
	5.	Allocate クリック
これで 203.0.113.10 のようなIPが払い出されます。
その後NAT Gateway作成時に「Existing」を選んでこのEIPを指定します。
AWS CLIでの取得

# 1. EIP確保
aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=nat-gw-prod-eip}]' \
  --region ap-northeast-1

# レスポンス例:
# {
#   "PublicIp": "203.0.113.10",
#   "AllocationId": "eipalloc-0abc123def456",
#   "Domain": "vpc"
# }

# 2. AllocationIdをメモして、NAT GW作成時に指定
aws ec2 create-nat-gateway \
  --subnet-id subnet-xxxxx \
  --allocation-id eipalloc-0abc123def456 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-gw-prod}]'


Terraform / CloudFormation での取得
Terraform:

resource "aws_eip" "nat_gw" {
  domain = "vpc"
  
  tags = {
    Name = "nat-gw-prod-eip"
  }
  
  # 誤削除防止
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat_gw.id
  subnet_id     = aws_subnet.public_a.id
  
  tags = {
    Name = "nat-gw-prod"
  }
}

# Snowflakeに登録するIPを出力
output "nat_gateway_eip" {
  value = aws_eip.nat_gw.public_ip
}


CloudFormation:

Resources:
  NatGatewayEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: nat-gw-prod-eip
    DeletionPolicy: Retain  # スタック削除時もEIPは残す

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnetA


重要な運用上の注意
1. リージョンごとに上限があるデフォルトでアカウント・リージョンあたり5個まで。それ以上必要ならサービスクォータで上限緩和申請。
2. リリース忘れに注意EIPは確保したまま未割当の状態だと $0.005/h (約$3.6/月) 課金されます。NAT GW削除後にEIPだけ残してしまうケースがありがちなので、不要になったら必ず release する。

# EIPの解放
aws ec2 release-address --allocation-id eipalloc-0abc123def456


3. リージョン縛りEIPはリージョン固定。東京で取ったEIPを大阪リージョンには持っていけません。
4. 払い出されるIPは選べない通常のAmazon poolからはランダムなIPが割り当てられます。特定のIP範囲を指定したい場合はBYOIP (Bring Your Own IP) という別機能が必要ですが、これは社用IPアドレスブロックを持っている大企業向け。
池田さんの構成での推奨手順
	1.	先にEIP確保 (AllocationIdをメモ)
	2.	Snowflake Network Policy作成 (このIPを登録)
	3.	NAT Gateway作成 (1.のEIPを指定)
	4.	Terraformで管理 (prevent_destroy でガード)

# 実際のコマンド例
aws ec2 allocate-address --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[
    {Key=Name,Value=dbt-prod-nat-eip},
    {Key=Purpose,Value=Snowflake allowlist},
    {Key=Environment,Value=prod}
  ]' \
  --region ap-northeast-1


Tagに「Snowflake allowlistに登録済み」と明記しておくと、運用引継ぎ時に「これ消していいやつ？」事故を防げます。
Multi-AZ構成にする場合は同じ手順で2つ確保し、Snowflake側のallowlistにも両方登録します。
次に進むなら、Terraformで一式書き起こしますか? それともSnowflake Network Policyの設定SQLを準備しますか?​​​​​​​​​​​​​​​​

-------------
おはようございます。
Lambda → Fargateの構成、いい選択です。dbtはビルド時間が読みにくいのでLambdaの15分制約から解放されるFargate委譲は理にかなっています。
サブネット設計の基本方針
Fargateタスクはプライベートサブネットに配置が原則です。理由は明確で、Snowflakeへの通信は外向きTLSのみで完結し、外部から接続される必要が一切ないためです。
推奨構成

VPC (10.0.0.0/16)
├── Public Subnet  (10.0.1.0/24) ← NAT Gateway のみ
├── Public Subnet  (10.0.2.0/24) ← NAT Gateway のみ (Multi-AZ)
├── Private Subnet (10.0.10.0/24) ← Lambda + Fargate
└── Private Subnet (10.0.11.0/24) ← Lambda + Fargate (Multi-AZ)


サブネットの考え方
Multi-AZで最低2つのプライベートサブネットを切るのが定石です。Fargateは指定したサブネット群からランダムに配置を選ぶため、単一AZだとAZ障害でジョブが止まります。バッチ用途なら3AZまで広げる必要は通常ありません。
CIDRサイズはFargateタスク数の上限を意識。Fargateタスク1つにつきENIを1つ消費するので、/24 (約250 IP) あれば日次バッチには十分。並列度を上げる予定があれば /22 程度に。
Snowflakeへの経路設計の選択肢
ここが本題で、3つの選択肢があります。
選択肢A: NAT Gateway経由 (シンプル)

Fargate → NAT GW → Internet → Snowflake (Public Endpoint)


	∙	メリット: 構成が単純、すぐ動く
	∙	デメリット: NAT GW料金 ($0.045/h + データ転送) が地味に効く、Snowflakeのallowlist制御はIP単位
選択肢B: Snowflake Private Link経由 (推奨) ⭐

Fargate → VPC Endpoint → AWS PrivateLink → Snowflake


	∙	メリット:
	∙	NAT GW不要 (コスト削減)
	∙	通信がAWSバックボーン内で完結 (セキュリティ◎)
	∙	SnowflakeのNetwork PolicyでPRIVATE_LINK制限可能
	∙	デメリット: Snowflake Business Critical edition以上 が必要
	∙	池田さんのクライアントPoCがStandard editionだと使えない点に注意
選択肢C: NAT GW + Snowflake Network Policy (現実解)
Business Criticalでない場合、NAT GWのEIPを固定IPにしてSnowflake側でallowlist。

CREATE NETWORK POLICY prod_dbt_policy
  ALLOWED_IP_LIST = ('203.0.113.10/32');  -- NAT GW EIP

ALTER USER DBT_PROD_USER SET NETWORK_POLICY = prod_dbt_policy;


VPCエンドポイント (PrivateLink to AWS Services)
NAT GW経由でも、以下のAWSサービス向けエンドポイントは作っておくべきです。



|エンドポイント            |用途                              |タイプ         |
|-------------------|--------------------------------|------------|
|**S3**             |dbt artifacts (manifest.json) 保管|Gateway (無料)|
|**ECR**            |Fargateイメージpull                 |Interface   |
|**CloudWatch Logs**|Fargateログ送信                     |Interface   |
|**Secrets Manager**|Snowflake秘密鍵取得                  |Interface   |
|**ECS**            |Fargateコントロールプレーン               |Interface   |

これらを置くと、AWS内通信がNAT GWを経由しなくなり、コスト・レイテンシ両面で効きます。特にECRからのイメージpullは毎回数十MB～数百MB流れるので、効果が大きいです。
Lambda側のサブネット配置
Lambdaはプライベートサブネットに配置するかどうかで挙動が変わります。
	∙	VPC外に置く (デフォルト): Fargate起動APIを叩くだけならVPC不要、コールドスタート速い
	∙	VPC内に置く: 同じVPC内のリソース (RDS等) にアクセスする場合のみ
「LambdaがRunTask API呼ぶだけ」ならVPC外に置くのが正解。VPC内Lambdaは初期化に時間がかかり、ENI管理も増えるので、必要性がない限り避けます。
まとめ図

[EventBridge / SQS]
        ↓
[Lambda (VPC外)] ──RunTask API──┐
                                  ↓
   ┌────────── VPC ──────────────────────┐
   │  Private Subnet A    Private Subnet B │
   │  ┌──────────────┐   ┌──────────────┐ │
   │  │ Fargate Task │   │ Fargate Task │ │
   │  └──────┬───────┘   └──────┬───────┘ │
   │         │                   │         │
   │  ┌──────┴───────────────────┴───────┐ │
   │  │  VPC Endpoints (S3/ECR/Logs/SM)  │ │
   │  └────────────┬─────────────────────┘ │
   │               │                        │
   │  ┌────────────┴──────────────────────┐ │
   │  │ Snowflake PrivateLink Endpoint    │ │
   │  │ (or NAT GW → Internet)            │ │
   │  └───────────────────────────────────┘ │
   └────────────────────────────────────────┘
                       ↓
                   [Snowflake]


池田さんへの確認ポイント
設計を決めるのに必要な情報があります:
	∙	クライアントのSnowflake editionは? (Business Critical以上ならPrivateLink推奨)
	∙	マルチリージョン要件は? (通常Tokyoのap-northeast-1単一でOKなはず)
	∙	dbt run並列度の想定は? (CIDR設計に影響)
	∙	Lambdaの役割は? (単純なFargateキック? or 何か前処理あり?)
これが分かると、具体的なCIDRと構成図まで詰められます。どこから深掘りしましょうか?​​​​​​​​​​​​​​​​

--------------


SnowflakeのSLAは少し独特で、「システム稼働率」だけではなく、クエリ実行成功率（query success rate）」で定義されています。ここがAWS EC2や一般システムと違います。

整理すると、Snowflakeの公式な考え方は主に以下です。

1. SnowflakeのサービスSLA（提供側SLA）

Snowflakeは2022年以降、2つの基準のうち厳しい方 を満たすことをSLAとしています。

基準①（従来）

99.9% の時間で
クエリエラー率 < 1%

許容停止時間換算：

約43分/月

基準②（追加）

99.99% の時間で
クエリエラー率 < 10%

許容停止時間：

約4分/月

Snowflakeは「両方を見て、より厳しい方を適用する」と説明しています。 ￼

つまり単純な：

稼働率 = 99.99%

だけではありません。

⸻

2. DWH利用者が決めるべきSLA（こちらが重要）

実案件では、Snowflake提供SLAより 業務SLA を決めます。

例えばあなたの想定：

外部→S3→Snowflake→dbt→BI

なら必要なのは：

項目	定義例	一般値
可用性	BI閲覧可能時間	99.9%
データ鮮度	ファイル到着→反映	30分以内
ELT完了	日次ロード	朝6時まで
クエリ応答	BI	P95<10秒
障害復旧(RTO)	復旧時間	4h
データ損失(RPO)	最大損失	15分
取込成功率	COPY INTO成功率	99.9%
dbt成功率	build成功率	99.5%


⸻

3. Snowflakeで監視すべきSLO（内部目標）

Snowflake自身もSLO定義を推奨しています。

例：

BI

95%のクエリ
→ 10秒以内

ELT

99%の日次ロード
→ 06:00完了

dbt

99%
→ build成功

Snowflake Well-Architectedでも、分析やデータエンジニアリング向けに「最大許容遅延」「完了時間」を明文化するよう推奨しています。 ￼

⸻

4. あなたの構成向け（Snowflake + dbt + Step Functions）の現実的SLAサンプル

要件定義なら私はこう置きます。

分類	指標	SLA
可用性	分析基盤稼働率	99.9%
ELT	S3→Snowflake反映	30分以内
dbt	build成功率	99.5%
BI	応答時間	P95<10秒
バッチ	日次完了	06:00
障害	RTO	4h
障害	RPO	15min
通知	障害検知	5min以内


⸻

5. Snowflake単体ではRTO/RPO保証しない

重要です。

SnowflakeのSLAは：

サービス可用性
+
クエリ成功率

中心です。

RTO=1h保証
RPO=0保証

のような業務要件は、別途：
	•	Cross-region replication
	•	Failover Group
	•	Business Continuity
	•	Multi-account DR

で設計します。Snowflakeはリージョン障害時にフェイルオーバーグループやレプリケーションを使うDRパターンを提供しています。 ￼

⸻

なので、DWH要件定義では

Snowflake SLA
≠
業務SLA

として切り分けるのがポイントです。あなたが前に整理していた「IPA非機能要求グレード」では、業務SLA（ELT完了、RTO等）を書く のが正解になります。

--------------

これは「データ量」だけでは決まりません。
dbtの処理内容（集計・JOIN・MERGEの重さ）と実行頻度 の方が効きます。

ただ、Snowflake中心か AWS(Fargate/Python/Spark)中心かの境目を、実務感覚でざっくり示すとこうなります。

まず重要：同じSQL変換ならFargateでもSnowflake課金は消えない

例えば：

Fargate
 ↓
dbt run
 ↓
Snowflake SQL実行
 ↓
Snowflake Warehouse課金

この場合、Fargateを使っても Snowflakeの計算コストはほぼ残ります。

なので比較対象は本当は：

A. Snowflakeで変換

COPY INTO
↓
Snowflake dbt
↓
Gold

B. AWSで変換して最後だけSnowflakeへ

S3
↓
Spark / Glue / ECS / Fargate
↓
加工済みデータ
↓
Snowflake

です。

⸻

ざっくり境界（経験則）

前提：
	•	日次ELT
	•	Parquet中心
	•	中規模JOINあり
	•	Gold集計あり

生データ量/日	Snowflake中心	AWS変換中心	傾向
～10GB/日	◎	×	Snowflake圧勝
10～100GB/日	◎	△	Snowflake優勢
100GB～1TB/日	○	○	境目
1TB～5TB/日	△	◎	AWS有利になり始める
5TB+/日	×	◎	Spark/Glue系が有利

かなり大雑把ですが、1TB/日超 が境目です。

⸻

なぜ境界がその辺？

例：

50GB/日

raw 50GB
↓
JOIN
↓
集計
↓
mart

Snowflake：

XS～M Warehouse
数千円～数万円/月

かなり安い。

⸻

2TB/日

変換：

2TB raw
↓
巨大JOIN
↓
MERGE
↓
履歴管理

Snowflake：

Warehouseサイズ増：

L～2XL

クレジット増加。

一方AWS：

Glue Spark
EMR
Fargate分散

で処理すると安くなるケースが出る。

⸻

ただしdbtはJOIN中心なので…

dbt処理は典型的に：

SELECT
JOIN
GROUP BY
WINDOW
MERGE

です。

これは Snowflakeが非常に得意。

なので境界はさらに後ろへずれます。

実務では：

数百GB/日

でもSnowflakeだけで回っている会社は多いです。

⸻

AWSが安くなる典型パターン

これなら早めに逆転します。

パターン1：Python前処理

API
↓
Pandas
↓
特徴量
↓
Snowflake

→ AWS有利

⸻

パターン2：画像・ログ

IoT
ログ
画像

→ Glue/Spark有利

⸻

パターン3：全量再計算

毎回：

5TB
↓
全部再集計

→ AWS寄り

⸻

あなたの想定（Snowflake + dbt + S3 + ELT）

過去の質問を見ると、かなり典型的な分析基盤です。

私は目安として：

100GB/日未満
→ Snowflake内完結

100GB～1TB/日
→ まずSnowflake継続

1TB/日超
→ Glue/Spark検討開始

数TB/日
→ AWS変換を真剣検討

にします。

⸻

一番現実的には、「データ量」より 月間Snowflakeクレジット消費 を見る方が判断しやすいです。

例えば：

月500 credit未満
→ 気にしなくてよい

1000～3000 credit
→ 最適化開始

5000 credit超
→ AWS側変換を比較検討

この辺からコスト会議が始まりやすいです。

-----------

￼￼￼￼￼￼￼データウェアハウス（特に Snowflake + AWS + ELT + dbt を想定した一般的な分析基盤）で、IPA 非機能要求グレード（NFR）必須項目のみを使って整理すると、だいたい以下が最低ラインになります。

IPA非機能要求グレードは大分類として：
	1.	可用性
	2.	性能・拡張性
	3.	運用・保守性
	4.	移行性
	5.	セキュリティ
	6.	システム環境・エコロジー

ですが、DWHなら 移行性・エコロジーは初期要件では軽め なので、まずは以下を押さえると実務的です。

⸻

まず一般的なDWHのSLA（目安）

企業向け分析基盤ではこんな値が多いです。

項目	一般的SLA目安	備考
システム稼働率	99.5〜99.9%	BI/DWHは24時間必須でない場合あり
データ取込成功率	99.9%以上	ELT失敗許容少
データ鮮度	5分〜24時間	リアルタイム性次第
バッチ完了時間	SLA内100%	例: 朝6時まで
問合せ応答	P95<10秒	BI用途
障害復旧時間(RTO)	1〜8時間	業務影響度依存
許容データ損失(RPO)	0〜1時間	金融なら0近い
セキュリティ事故	0件	必須

例：

日次ELT:
06:00までに全ロード完了

BI:
95%が10秒以内応答

障害:
4時間以内復旧

データ損失:
最大15分


⸻

IPA非機能要求グレード（必須）で整理

① 可用性

最重要。

定義例：

項目	定義
稼働率(SLA)	月間99.9%
RTO	障害から4時間以内復旧
RPO	最大15分のデータ損失許容
冗長化	AZ障害で継続運転
バックアップ	日次＋保持90日

要求例：

分析基盤は月間99.9%以上稼働すること。
重大障害発生時は4時間以内復旧。


⸻

② 性能・拡張性

DWHではかなり重要。

定義例：

項目	定義例
データ量	初期10TB → 3年50TB
同時接続	BI同時100ユーザ
クエリ応答	P95=10秒以内
ELT完了時間	日次処理2時間以内
スケール	データ量2倍で停止不要

要求例：

50TBでも性能劣化20%以内。
日次ELTは6時まで終了。


⸻

③ 運用・保守性

分析基盤はここが重要。

定義：

項目	定義
ジョブ監視	全ELT監視
ログ保存	365日
通知	エラー5分以内
自動復旧	3回リトライ
変更容易性	SQL/dbt変更で停止不要

例：

ELT失敗時5分以内通知。
ジョブ履歴1年保持。

以前質問された Step Functions + dbt + Snowflake ならここ。

⸻

④ セキュリティ

必須。

定義：

項目	定義
認証	SSO/MFA
権限	最小権限IAM
暗号化	保存/通信とも暗号化
監査	全アクセス記録
秘密情報	Secrets Manager

要求例：

S3はKMS暗号化。
SnowflakeはSSO。
監査ログ365日保持。


⸻

⑤ システム環境

クラウド前提なら最低限。

定義：

項目	定義
利用クラウド	AWS
DWH	Snowflake
OS依存	なし
対応ブラウザ	Chrome/Edge


⸻

DWH案件で実際によく使う非機能要件テンプレ

これだけ書ければかなり十分です。

分類	要件	目標値
可用性	稼働率	99.9%
可用性	RTO	4h
可用性	RPO	15min
性能	BI応答	P95<10s
性能	日次ELT	06:00完了
性能	データ量	50TB
保守	通知	5分以内
保守	ログ保持	365日
セキュリティ	暗号化	保存/通信必須
セキュリティ	権限制御	最小権限
セキュリティ	監査ログ	365日


⸻

あなたの今の文脈（AWS + Snowflake + S3 + dbt + Step Functions のELT基盤）なら、次は 「具体的な非機能要求グレード表（レベル1〜6）を埋めたサンプル」 を作ると、そのまま要件定義書に貼れる形になります。

----------
Snowflakeでアクセス制限（特権制限）を設計する場合、データベース・スキーマの切り方とロール設計をセットで考えるのが定石です。池田さんがクライアント向けに設計されている文脈で、実務でよく採られるパターンを整理します。

## 基本方針：ロールベースで「データの論理単位」を分離する

Snowflakeの権限はロール（Role）に付与し、ユーザーにロールを割り当てる方式です。DB/スキーマの境界そのものよりも、**「誰が何にアクセスできるか」を分けやすい粒度で切る**ことが重要です。

## 推奨される階層設計

一般的な3層構造（メダリオンアーキテクチャに近い形）:

- **DB単位**で環境を分ける: `PROD_DB` / `STG_DB` / `DEV_DB`
- **スキーマ単位**でレイヤを分ける: `RAW` / `STAGING` / `MART` / `SANDBOX`
- **MART配下**を業務ドメインで細分化: `MART_SALES` / `MART_FINANCE` / `MART_HR` など機密度が異なるものは別スキーマに

機密度の高いデータ（人事・財務・個人情報）は**別スキーマ、あるいは別DBに隔離**すると、ロール設計がシンプルになります。

## ロール設計のパターン

Snowflake公式が推奨する「機能ロール（Functional Role）+ アクセスロール（Access Role）」の2層構造：

- **アクセスロール**: スキーマ単位でR/Wを定義（例: `MART_SALES_R`、`MART_SALES_RW`）
- **機能ロール**: 業務上の役割（例: `ANALYST_SALES`、`DATA_ENGINEER`、`EXECUTIVE`）にアクセスロールを継承させる
- ユーザーには機能ロールのみを割り当てる

これにより、スキーマが増えても機能ロールの組み合わせ変更だけで済みます。

## 行・列レベルの制限が必要な場合

スキーマ分割で足りないケース（同じテーブルを部門別に見せたい等）は:

- **Row Access Policy**: 行レベルでの絞り込み（部門コードでフィルタなど）
- **Dynamic Data Masking**: 列レベルのマスキング（メールアドレスを伏字に等）
- **Secure View / Secure UDF**: 元テーブルを隠して必要なカラムだけ公開

PoCの段階ではスキーマ分離だけで設計し、本番でRow Access Policyを足す進め方が現実的です。

## PoC向けのミニマム構成例

200名規模・IT担当2〜3名のクライアントなら、最初はこのくらいで十分です：

- DB: `ANALYTICS_DB`（本番用）/ `DEV_DB`（開発用）
- スキーマ: `RAW` / `MART_COMMON` / `MART_RESTRICTED`（機密データのみ）
- ロール: `ANALYST`（MART_COMMON読取）/ `ANALYST_PRIVILEGED`（MART_RESTRICTEDも読取）/ `ENGINEER`（全書込）

ところで「up制限」というのは具体的にどういう制限を想定されていますか？  「アップロード制限」「アップデート（更新）制限」「ユーザープリビレッジ制限」など解釈が分かれるので、用途に合わせてもう少し具体的な設計例を出せます。

IP制限ですね、失礼しました。Snowflakeでは**Network Policy**で実現します。DB/スキーマ設計とは独立した、アカウントまたはユーザー/ロール単位の制御です。

## Network Policyの基本

Snowflakeでは、特定のIPアドレス範囲からの接続のみを許可する仕組みとして `NETWORK POLICY` オブジェクトを使います。CIDR形式で許可リスト（ALLOWED_IP_LIST）と拒否リスト（BLOCKED_IP_LIST）を定義できます。

```sql
CREATE NETWORK POLICY corp_office_policy
  ALLOWED_IP_LIST = ('203.0.113.0/24', '198.51.100.10')
  BLOCKED_IP_LIST = ()
  COMMENT = '本社・支社からのアクセスのみ許可';
```

## 適用範囲の3つのレベル

Network Policyは適用先によって粒度が変わります。

- **アカウントレベル**: 全ユーザーに一律適用。最も強力だが、管理者が締め出されるリスクあり
- **ユーザーレベル**: 個別ユーザーに適用。アカウントレベルより優先される
- **統合（Security Integration）レベル**: SCIM・OAuth等の特定の認証経路にのみ適用

```sql
-- アカウント全体に適用
ALTER ACCOUNT SET NETWORK_POLICY = corp_office_policy;

-- 特定ユーザーのみに適用（アカウント設定を上書き）
ALTER USER tanaka_san SET NETWORK_POLICY = corp_office_policy;
```

## DB/スキーマ設計との関係

ここがポイントですが、**Network PolicyはDB/スキーマ設計とは直交する**概念です。「このスキーマにアクセスできるIPを制限したい」という直接的な仕組みはありません。

実務での回避策は2パターン：

**パターン1: ロールごとに専用ユーザーを作り、ユーザー単位でIP制限**

機密データ用ロール（例: `ANALYST_PRIVILEGED`）を使うためには専用ユーザーでログインする必要がある、という運用にして、そのユーザーにNetwork Policyを適用します。

**パターン2: 機密データを別Snowflakeアカウントに分離**

PoC段階ではオーバースペックですが、規制業種（金融・医療）では機密データだけ別アカウントに隔離し、アカウントレベルでIP制限をかけることがあります。

## PoC向けの現実的な構成

200名規模のクライアントなら、こんな順序で導入するのが無理がないです：

1. まずアカウントレベルで「社内VPN + クライアント拠点」のIPを許可
2. ACCOUNTADMIN等の特権ロールを使う管理者ユーザーは、より狭いIP（情シス端末のみ等）で別途縛る
3. BIツールやETLサービス（Fivetran等）のIPレンジは別途許可リストに追加

## 注意点

- **設定前に自分のIPを確認**してください。CIDR間違いで全員ロックアウトされると、Snowflakeサポートに連絡しないと復旧できません
- **PrivateLink（AWS/Azure）を使う場合**は、Network Policyに加えて `ALLOWED_VPCE_IDS` でVPCエンドポイント制限も可能です
- **動的IPの社員**（在宅勤務等）がいる場合はVPN経由に統一するか、SSOと組み合わせるのが現実的です

クライアントは固定IPのオフィス出社中心ですか、それともリモートワーカーが多い構成ですか？ それによって設計の落とし所が変わります。


ELT用ユーザーは「人ではないアクセス（machine-to-machine）」なので、人間ユーザーとは別の設計思想で組みます。Snowflakeの推奨プラクティスを踏まえた設計を整理します。

## 基本方針：用途ごとに専用ユーザーを作る

ELTツールやサービスごとに1ユーザー、が原則です。同じツール内でもジョブの責務が違えば分ける価値があります。

- **ロード用**: 外部ステージ → RAWスキーマへのデータ投入（Fivetran、Airbyte、独自Pythonバッチ等）
- **変換用**: RAW → STAGING → MARTへの変換（dbt、Dataform等）
- **オーケストレーション用**: ジョブ起動・監視（Airflow、Prefect等）

これらを分けると、障害時の影響範囲特定・権限の最小化・コスト按分（後述のWarehouse分離）がやりやすくなります。

## 認証方式：パスワードではなくキーペア認証

ELT用途では**Key-Pair Authentication**が推奨です。理由は以下のとおり：

- パスワードは漏洩リスクと定期ローテーション運用がつらい
- Snowflakeは2025年以降、サービスアカウントでのパスワード認証を段階的に廃止する方針を打ち出しています
- Python connectorやdbt、各種ETLツールが標準対応

```sql
CREATE USER svc_dbt_prod
  TYPE = SERVICE  -- サービスアカウント専用タイプ（MFA対象外）
  DEFAULT_ROLE = DBT_TRANSFORMER
  DEFAULT_WAREHOUSE = WH_TRANSFORM
  RSA_PUBLIC_KEY = 'MIIBIjANBgkqhk...'
  COMMENT = 'dbt Cloud production runner';
```

`TYPE = SERVICE` は人間ユーザーと明確に区別する新しいユーザータイプで、MFA強制ポリシーの対象外にできます（パスワード認証も不可になる仕様）。

## ロール設計：機能ロールを別建てで作る

人間用の `ANALYST` 等とは別に、ELT専用ロールを作ります。

- **`LOADER`**: 外部ステージ読取 + RAWスキーマへのINSERT/COPY/MERGE権限
- **`TRANSFORMER`**: RAW読取 + STAGING/MART配下のフルCRUD + 必要ならスキーマ作成権限
- **`REPORTER`**: MART読取のみ（BIツール用）

dbtのようなツールは「テーブルを作って入れ替える」操作をするので、TRANSFORMERにはスキーマレベルの `CREATE TABLE / VIEW` 権限を付与します。FUTURE GRANTSで自動付与にしておくと運用が楽です：

```sql
GRANT USAGE ON DATABASE ANALYTICS_DB TO ROLE TRANSFORMER;
GRANT USAGE ON SCHEMA ANALYTICS_DB.MART TO ROLE TRANSFORMER;
GRANT CREATE TABLE, CREATE VIEW ON SCHEMA ANALYTICS_DB.MART TO ROLE TRANSFORMER;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ANALYTICS_DB.MART TO ROLE REPORTER;
```

## Warehouseも分ける

ELT用ユーザーには専用Warehouseを `DEFAULT_WAREHOUSE` に設定します。

- **WH_LOAD**: ロード用（XS〜S、頻繁起動）
- **WH_TRANSFORM**: 変換用（M〜L、夜間バッチで一気に動く）
- **WH_BI**: BI用（S、`AUTO_SUSPEND = 60` 等で短く）

これによりコスト按分・性能干渉の回避・モニタリングがクリアになります。RESOURCE_MONITORで上限を別個に設定できるのも利点です。

## IP制限との組み合わせ

前回のNetwork Policyの話とつながるところです。ELT用ユーザーは**接続元IPが固定または範囲が明確**なので、人間ユーザーより厳しくIP制限をかけられます。

- 自社サーバーのバッチ → 自社のNAT固定IP
- Fivetran等のSaaS → 各社が公開しているIPレンジを許可リストに追加
- AWS上のAirflow → VPCのNAT GatewayのEIP、またはPrivateLink

```sql
CREATE NETWORK POLICY policy_svc_dbt
  ALLOWED_IP_LIST = ('10.20.30.40/32');  -- dbt Cloud のIPなど

ALTER USER svc_dbt_prod SET NETWORK_POLICY = policy_svc_dbt;
```

ユーザー単位のNetwork Policyはアカウントレベルを上書きするので、**人間ユーザーは社内VPN経由のみ・ELTユーザーはサービスのIPのみ**といった使い分けができます。

## PoC段階での現実的な落とし所

200名規模のクライアントで池田さんがPoC支援する文脈なら、こんな粒度から始めるのがおすすめです：

- ユーザー: `SVC_LOADER` / `SVC_DBT` の2つ
- ロール: `LOADER` / `TRANSFORMER` の2つ（人間用の `ANALYST` とは別建て）
- Warehouse: `WH_LOAD` / `WH_TRANSFORM` の2つ
- 認証: 最初からキーペア認証で構築（後から移行は面倒）
- IP制限: 自社バッチサーバーのIPのみ許可

本番展開で `SERVICE` ユーザータイプ・FUTURE GRANTS・RESOURCE_MONITORを足していく流れが無理がないです。

ELTツールは具体的に何を使う想定ですか？ dbtかFivetranか自前Pythonかで、ロール権限の細かい付け方（特に `CREATE TABLE` の必要性）が変わります。


業務ロールとシステムロールを統合して設計する話ですね。Snowflake全体のロール設計の定石を、PoC〜本番展開を見据えて整理します。

## 基本思想：3層構造で組む

Snowflakeのロール設計は、機能（業務）と権限（アクセス）を分離する**3層構造**が王道です。

```
┌─────────────────────────────────────────┐
│ 1. システムロール（Snowflake標準）      │  ACCOUNTADMIN, SECURITYADMIN, SYSADMIN等
├─────────────────────────────────────────┤
│ 2. 機能ロール（Functional Role）        │  業務上の役割。ユーザーに割り当てる
├─────────────────────────────────────────┤
│ 3. アクセスロール（Access Role）        │  オブジェクト単位のR/W権限の塊
└─────────────────────────────────────────┘
        ↓ 継承（GRANT ROLE）
    オブジェクト（DB/Schema/Table/Warehouse）
```

ユーザーには**機能ロールのみ**を割り当て、機能ロールがアクセスロールを継承する形にします。これにより、オブジェクトが増減しても機能ロール側の変更は最小限で済みます。

## 1層目：Snowflake標準のシステムロール

これは最初から存在するロールで、用途を理解した上で**誰に使わせるか**を決めることが設計の出発点です。

- **ORGADMIN**: 組織全体（複数アカウント）の管理。通常はSnowflake契約担当者のみ
- **ACCOUNTADMIN**: アカウント最上位権限。緊急時のbreak-glass用、日常利用は避ける
- **SECURITYADMIN**: ユーザー・ロール・Network Policyの管理
- **USERADMIN**: ユーザーとロールの作成のみ（権限付与は不可）
- **SYSADMIN**: DB/Warehouse/オブジェクトの作成・管理。**全カスタムロールの親**にする
- **PUBLIC**: 全ユーザーが自動所属。ここに何かを付与すると全員に見えるので注意

実務での運用ルール：

- ACCOUNTADMINは2〜3名の専任管理者のみ、MFA必須、break-glassアカウントは別管理
- 日常的なオブジェクト管理はSYSADMINで行う
- **カスタムロールは必ずSYSADMINを親にする**（`GRANT ROLE custom_role TO ROLE SYSADMIN`）

## 2層目：機能ロール（業務ロール）

業務上の「役割」を表現します。組織図や職務定義書から起こすイメージです。

PoC段階の例：

- **`ANALYST`**: 一般的なデータ分析者。MART全般を読み取り
- **`ANALYST_PRIVILEGED`**: 機密データ含む分析者。人事・財務MARTも読み取り
- **`DATA_ENGINEER`**: パイプライン開発者。RAW/STAGING/MARTすべて操作可能
- **`EXECUTIVE`**: 経営層。ダッシュボード用ビューのみ閲覧
- **`DEVELOPER`**: 開発環境のみフルアクセス、本番は読取のみ

システム/ELT系の機能ロール（前回の続き）：

- **`SVC_LOADER`**: ロード処理用
- **`SVC_TRANSFORMER`**: dbt等の変換処理用
- **`SVC_BI`**: BIツールのサービスアカウント用

## 3層目：アクセスロール

オブジェクト単位の権限の「部品」です。命名規約を統一すると後で読み解きやすくなります。

命名例：`<DB>_<SCHEMA>_<権限レベル>`

- `ANALYTICS_MART_R`（読取のみ）
- `ANALYTICS_MART_RW`（読み書き）
- `ANALYTICS_RAW_R`
- `ANALYTICS_RAW_RW`
- `ANALYTICS_MART_RESTRICTED_R`（機密データ）
- `WH_TRANSFORM_USAGE`（Warehouse利用権限）

## 全体を組み合わせた継承図

PoC構成での具体例：

```
ユーザー: tanaka_san  →  ANALYST
                              ├── ANALYTICS_MART_R
                              ├── ANALYTICS_MART_COMMON_R
                              └── WH_BI_USAGE

ユーザー: yamada_san  →  ANALYST_PRIVILEGED
                              ├── ANALYTICS_MART_R
                              ├── ANALYTICS_MART_RESTRICTED_R  ← ここが追加
                              └── WH_BI_USAGE

ユーザー: SVC_DBT     →  SVC_TRANSFORMER
                              ├── ANALYTICS_RAW_R
                              ├── ANALYTICS_STAGING_RW
                              ├── ANALYTICS_MART_RW
                              └── WH_TRANSFORM_USAGE

全カスタムロール → SYSADMIN
```

## オーナーシップとSYSADMIN集約の原則

オブジェクトを作成したロールが**そのオブジェクトのオーナー**になります。バラバラのロールがオーナーになると権限管理が破綻するので：

- すべてのDB/Schema/TableはSYSADMIN（または SYSADMIN配下の専用ロール）が所有
- アクセスロールはSECURITYADMIN または USERADMINが作成・管理
- ELTツールが新規テーブルを作る場合は、`USE SECONDARY ROLES` や `MANAGED ACCESS SCHEMA` を活用してオーナーシップ管理を簡略化

## MANAGED ACCESS SCHEMAの活用

スキーマレベルで `WITH MANAGED ACCESS` を付けると、**そのスキーマ内のオブジェクトへのGRANTはスキーマオーナーのみが実行可能**になります。dbtが勝手にGRANTを書き換えるのを防げるので、本番MARTには適用推奨です。

```sql
CREATE SCHEMA ANALYTICS.MART WITH MANAGED ACCESS;
```

## 環境分離との掛け合わせ

DEV/STG/PRODでロールも分けるか、同じロールで環境ごとに権限を変えるかは設計判断です。

**推奨パターン: 環境ごとに別ロール**

- `ANALYST_PROD` / `ANALYST_DEV`
- `SVC_TRANSFORMER_PROD` / `SVC_TRANSFORMER_DEV`

PRODのサービスアカウントが事故でDEVを壊す事態を防げます。dbtでprofile分けるのとも整合性が取れます。

## PoC向けの最小構成

200名規模のクライアントなら、最初はこのくらいで十分です：

機能ロール 5つ：

- `ANALYST`（一般分析者）
- `ANALYST_PRIVILEGED`（機密データも見られる）
- `DATA_ENGINEER`（開発・運用）
- `SVC_LOADER`（ELTロード）
- `SVC_TRANSFORMER`（dbt等）

アクセスロール 6つ：

- `ANALYTICS_RAW_R` / `ANALYTICS_RAW_RW`
- `ANALYTICS_MART_R` / `ANALYTICS_MART_RW`
- `ANALYTICS_MART_RESTRICTED_R`
- `WH_*_USAGE`（Warehouseごと）

これで100〜200名規模なら回ります。本番展開時に部門別MARTが増えたら、アクセスロールを追加して機能ロールに継承させるだけで拡張できます。

## 設計時の注意点

- **PUBLICロールに何も付与しない**（うっかり全員に見える事故を防ぐ）
- **ACCOUNTADMINでオブジェクトを作らない**（オーナーシップ管理が破綻する）
- **FUTURE GRANTSを活用**（新規テーブル作成時の権限付与漏れを防ぐ）
- **ロール命名は最初から規約を決める**（後から大文字小文字統一は地獄）
- **break-glassアカウント**（緊急時用ACCOUNTADMIN）を別建てで用意し、認証情報を金庫管理

ところでクライアントの組織体制は、データ分析・データエンジニアリングを情シス内でやる構造ですか？ それとも事業部側にもアナリストがいる構造ですか？ 後者の場合は部門別MARTとロールの掛け合わせが必要になるので、もう少し設計を細かく詰める価値があります。


はい、その理解で正しいです。**アクセスロールをユーザーに直接付与しない**のは、ELT用ユーザーであっても守るべき原則です。

## なぜ機能ロールを経由すべきか

アクセスロールを直接付ける運用にすると、設計の意図が崩れていきます。

**1. 権限の組み合わせが追跡不能になる**

人間ユーザーAには `MART_R` だけ、Bには `MART_R` + `RAW_R`、Cには `MART_R` + `RAW_R` + `RESTRICTED_R`... と直接付与していくと、「このユーザーは結局何ができるのか」がユーザーごとに違ってきます。100人いたら100通りの権限プロファイルが生まれ、棚卸しが困難になります。

機能ロールを経由すれば、「このユーザーは `ANALYST_PRIVILEGED` だから人事データも見える」と一意に説明できます。

**2. 権限変更の影響範囲が読めなくなる**

「分析者全員にRAW読取権限を追加したい」となった時、機能ロールに `RAW_R` を1回GRANTすれば全員に反映されます。直接付与だと、ユーザー全員に対して個別にGRANTする必要があり、追加漏れが必ず発生します。

**3. 監査・棚卸しが破綻する**

退職者や異動者の権限剥奪時、機能ロール経由なら「ロールの割当を外す」だけで完了します。直接付与だと、そのユーザーが持つすべてのアクセスロールを個別に確認・剥奪する必要があり、剥奪漏れが起きます。

## ELT用ユーザーでも同じ理由が当てはまる

「ELTユーザーは1個だけだから機能ロールを挟む意味がないのでは？」と思いがちですが、実際は人間ユーザー以上に機能ロールを挟む価値があります。

**1. 環境ごとに同じ「役割」が必要になる**

`SVC_DBT_PROD` と `SVC_DBT_DEV` は別ユーザーですが、**役割としては同じ「dbt変換実行者」**です。機能ロール `SVC_TRANSFORMER` を定義しておけば、環境別ユーザーに同じロールを付与でき、権限構成の一貫性が保てます。

**2. ツールの追加・差し替えに対応できる**

dbtの代わりにDataformを試したい、Airflowにも変換ジョブを持たせたい、となった時、新しいユーザーに `SVC_TRANSFORMER` を付与するだけで済みます。アクセスロールを直接付けていると、ツールごとに同じ権限セットを組み直す手間が発生します。

**3. 監査ログで「何の役割の処理か」が読み取りやすい**

`QUERY_HISTORY` を見た時、`ROLE_NAME` が `SVC_TRANSFORMER` になっていれば変換系の処理だと一目で分かります。アクセスロールが直接動いていると、`ANALYTICS_MART_RW` のような名前が並び、業務的な文脈が読み取りにくくなります。

**4. 将来の冗長化・並列化に対応**

「日次バッチと時間バッチでdbtユーザーを分けたい」「コスト分析のためにジョブ別にユーザーを分けたい」となった時、機能ロールがあれば即座に複数ユーザーに同じ役割を割り当てられます。

## 例外的に許容されるケース

厳密に「絶対NG」ではなく、以下のような場合は許容されることもあります。

- **使い捨ての一時ユーザー**（PoCの数日だけ動かす検証用）
- **ROLE階層を使えないツール**（極めて稀。最近のツールはほぼ対応）
- **Snowflake標準ロール（SYSADMIN等）の直接利用**（これはそもそも機能ロールの一種と見なせる）

ただし「一時的」が「いつのまにか恒久化」するのが現場の常なので、PoCでも最初から機能ロール経由で組んでおくのが結果的に楽です。

## 推奨される構造の再確認

```
ユーザー (SVC_DBT_PROD)
    ↓ GRANT ROLE
機能ロール (SVC_TRANSFORMER)
    ↓ GRANT ROLE
アクセスロール (ANALYTICS_RAW_R, ANALYTICS_MART_RW, WH_TRANSFORM_USAGE)
    ↓ GRANT 権限
オブジェクト (Database, Schema, Table, Warehouse)
```

ユーザーは機能ロールしか知らない、機能ロールはアクセスロールの組み合わせを知っている、アクセスロールはオブジェクト権限の塊を持っている、という3層の責務分離です。

この原則は、Snowflakeのドキュメントでも公式に推奨されているパターンで、規模が小さくても適用する価値があります。むしろ規模が大きくなってから3層化するのは権限の付け替え地獄になるので、PoC段階から徹底するのが鉄則です。

池田塾でDB設計を教えられている文脈で言えば、これは**「正規化」と同じ発想**ですね。直接付与は「権限の非正規化」で、最初は楽に見えても更新異常を必ず引き起こす、という構造です。




----------
了解です。bruno-szdl/dbt-ci-cd の中身を実際に取得して、GitBucket CI Plugin向けに翻訳していきましょう。​​​​​​​​​​​​​​​​

構造が把握できました。重要なファイル（profiles.yml、dbt_project.yml、cleanup macro）の中身も見ておきましょう。​​​​​​​​​​​​​​​​

profiles.ymlの構造が取れました。続いてcleanup macroも見ておきます。​​​​​​​​​​​​​​​​

macro実体の中身までは取れませんでしたが、構造とロジックは十分把握できました。実際に動くテンプレートを作って、GitBucket向けに移植したものを成果物として作りましょう。
skillを確認してから作業を進めます。​​​​​​​​​​​​​​​​

これはアップロードファイル読み取り用のskillで、今回のテキストファイル作成タスクには不要ですね。すぐにファイル作成に進みます。
プロジェクト構成を一式作って、最後にzipにまとめて渡します。​​​​​​​​​​​​​​​​

波括弧展開がうまくいきませんでした。普通のmkdirで作り直します。​​​​​​​​​​​​​​​​

ファイル群を作成していきます。​​​​​​​​​​​​​​​​

次にmacroを2つ。generate_schema_nameは重要で、これがないとPR専用スキーマが正しく組み立てられません。​​​​​​​​​​​​​​​​

次にDockerfileとシェルスクリプト群です。​​​​​​​​​​​​​​​​

GitBucket CI Plugin設定ファイル群を作ります。GitBucket CI PluginはJenkinsfile風YAMLを採用しているので、それに合わせます。​​​​​​​​​​​​​​​​

サンプルのdbtモデル群とunit testを作って、最後にSnowflakeセットアップSQLを追加します。​​​​​​​​​​​​​​​​

bruno-szdl版との対比を解説するドキュメントを追加し、構造を確認します。​​​​​​​​​​​​​​​​

構造を確認します。​​​​​​​​​​​​​​​​

一式そろいました。zipにまとめてダウンロードできるようにします。​​​​​​​​​​​​​​​​

READMEとTRANSLATION_NOTESも個別に渡せるようにコピーしておきます。​​​​​​​​​​​​​​​​

bruno-szdl版をベースに、GitBucket + Docker + Snowflake向けに翻訳した一式を作成しました。
主な翻訳ポイント
bruno-szdl版から変えた4箇所:
	1.	YAMLは薄く、ロジックはshellへ - GitBucket CI PluginのYAML文法はGitHub Actionsより素朴なので、if-then-else分岐はci.sh/cd.sh/teardown.shに集約。ローカルでdocker runで再現可能になる副次効果も。
	2.	Password → Key pair認証 - SnowflakeサービスアカウントへのMFA強制トレンドに合わせ、SNOWFLAKE_PRIVATE_KEYをGitBucket SecretsからPEM全文で渡す方式に。
	3.	GitHub Actions artifacts → MinIO/S3互換 - manifest保管先を自己ホストMinIOまたはS3互換に。boto3で操作。
	4.	multi-stage Docker - 池田さんが採用されているパターンに合わせ、base → deps → dbt-deps → parse → ci/cdの5段構成。packages.yml/requirements.txt変更時のみ再ビルドが走るよう最適化。
ファイル構成 (21ファイル)

dbt-ci-cd-gitbucket/
├── README.md                     # セットアップ手順
├── TRANSLATION_NOTES.md          # bruno-szdl版との対応表
├── snowflake_setup.sql           # Snowflake側初期化SQL
├── dbt_project.yml, packages.yml, profiles.yml
├── dbt-requirements.txt
├── .gitbucket/
│   ├── ci.yml, cd.yml, teardown.yml
├── docker/
│   ├── Dockerfile, ci.sh, cd.sh, teardown.sh
├── macros/
│   ├── generate_schema_name.sql  # PR用スキーマ命名
│   └── drop_pr_schemas.sql       # クリーンアップ
├── models/
│   ├── staging/stg_orders.sql
│   ├── marts/orders_summary.sql
│   └── schema.yml                # unit test 2つ含む
└── seeds/raw_orders.csv


実行までの順序
	1.	snowflake_setup.sql をSnowflakeで実行 (公開鍵を貼る)
	2.	GitBucket Secretsに11個のキー登録
	3.	ローカルで初回 dbt parse --target prod → manifestをMinIOにアップ
	4.	PR作成 → .gitbucket/ci.yml がトリガー → Slim CIが動く
	5.	merge → .gitbucket/cd.yml が本番デプロイ + manifest更新
	6.	PR close → .gitbucket/teardown.yml がPR専用スキーマを削除
次のステップ候補
	∙	GitBucket CI Pluginの実バージョン依存事項の調整 (pull_request closedイベントの動作確認等)
	∙	MinIO構築手順の追加 (docker-composeで5分)
	∙	本番デプロイの慎重化: blue/green or write-audit-publishパターン追加
	∙	池田塾教材化: 「Slim CIの仕組み」「state:modified+とは何か」を新人向けに解説
どこから手を付けましょうか？​​​​​​​​​​​​​​​​
----------

dbtのCI/CDリファレンス実装、いくつか定番がありますね。最新情報も含めて調べておきます。​​​​​​​​​​​​​​​​

いくつか定番のリファレンス実装が見つかりました。池田さんの状況（GitBucket + Docker + Snowflake想定）に照らして、用途別に整理しますね。
最もおすすめの3つ
1. bruno-szdl/dbt-ci-cd (Snowflake特化、最も実装に近い)
👉 https://github.com/bruno-szdl/dbt-ci-cd
Snowflake向けに設計されたCI/CDパイプラインのデモリポジトリで、PR番号とcommit SHAからpr_123__abc123def456のような一意なスキーマを動的生成してPR間の干渉を防ぐ構成 ￼。本番manifestをartifactとしてダウンロードし、dbt build -s 'state:modified+' --defer --state ./ でSlim CIを実行、PR closeでdrop_pr_schemas macroを呼んでクリーンアップする一連の流れ ￼が実装されています。
池田さんの構成にほぼそのまま流用可能。GitHub Actions → GitBucket CI Pluginへの翻訳が主な作業。
2. dbt-labs/jaffle-shop + dbt公式CI guide (基礎学習用)
👉 https://github.com/dbt-labs/jaffle-shop👉 https://docs.getdbt.com/guides/set-up-ci
dbt公式のサンプルプロジェクトで、CI実装の説明とセットで読むのが定番。dbt project evaluatorとの組み合わせ方（dbt build --select state:modified+ --exclude package:dbt_project_evaluator の後に dbt build --select package:dbt_project_evaluator を実行） ￼など、品質チェックの組み込みパターンも参考になります。
3. dbt-beyond-the-basics (上級者向けパターン集)
👉 awesome-dbtでも紹介されている、CIの応用パターン showcase
Slim CIとunit testをCIに、Write-Audit-Publishパターン（dbt cloneを利用）をCDの上位環境に適用する設計 ￼が示されており、池田さんが構築中のCI構成を将来CDまで拡張する際の参考になります。
用途別マトリクス



|リポジトリ                                                                                                         |用途                        |池田さんへのフィット度|
|--------------------------------------------------------------------------------------------------------------|--------------------------|-----------|
|**bruno-szdl/dbt-ci-cd**                                                                                      |Snowflake実接続CIの即戦力リファレンス  |★★★★★      |
|dbt-labs/jaffle-shop                                                                                          |公式サンプル、学習・教育用途（池田塾にも◎）    |★★★★       |
|dbt-beyond-the-basics                                                                                         |応用パターン（unit test, defer活用）|★★★★       |
|melbdataguy/dbt-bigquery-cicd                                                                                 |BigQuery版、コード解説が丁寧        |★★（参考のみ）   |
|Snowflake zero-copy clone系 ([記事](https://towardsdatascience.com/running-dbt-using-gitlab-ci-cd-8a2ef0f05af0/))|クローンベースCI設計の思想理解          |★★★        |

実装の進め方として推奨
	1.	bruno-szdl/dbt-ci-cd をfork or clone して構造把握
	2.	GitHub Actions部分をGitBucket CI Plugin形式に翻訳 (前回の回答で出した構成がそのまま使える)
	3.	dbt-labs/jaffle-shop の models構造 を真似てサンプルプロジェクトを作り、CI動作確認
	4.	unit test (dbt 1.8+) を追加 → dbt-beyond-the-basicsを参考に
補足：池田塾の教材としても
dbt-labs/jaffle-shopは構造がシンプルなので、新人エンジニア向けに「dbt + CI/CD」を教える際の教材としても優秀です。論理→物理モデルの教育コンテンツと合わせて、「設計→実装→自動テスト」の一気通貫の流れを見せられます。
bruno-szdl/dbt-ci-cdのコードを実際に読みながら、GitBucket向けに翻訳する作業を一緒に進めましょうか？それともまずjaffle-shopでローカル動作確認から始めますか？​​​​​​​​​​​​​​​​

Snowflake実接続でGitBucket CI Plugin前提の構成、池田さんの既存スタック（自己ホスト型GitBucket + Docker）に合わせて詰めていきますね。
全体アーキテクチャ
'''
[Push/PR] → GitBucket → CI Plugin → Docker Runner
                                      ├─ Stage1: deps解決
                                      ├─ Stage2: dbt parse (offline)
                                      ├─ Stage3: dbt compile (Snowflake接続)
                                      ├─ Stage4: dbt build --select state:modified+ (CI用DB)
                                      └─ Stage5: unit test

'''
Snowflake側の準備
CI専用環境を分離するのが鉄則です。

-- CI専用ロール・ウェアハウス・DB
CREATE ROLE CI_ROLE;
CREATE WAREHOUSE CI_WH WITH 
  WAREHOUSE_SIZE = 'XSMALL' 
  AUTO_SUSPEND = 60 
  AUTO_RESUME = TRUE;
CREATE DATABASE CI_DB;

-- CI用ユーザー（key pair認証推奨）
CREATE USER CI_USER 
  RSA_PUBLIC_KEY='MIIBIj...'
  DEFAULT_ROLE = CI_ROLE
  DEFAULT_WAREHOUSE = CI_WH;

GRANT ROLE CI_ROLE TO USER CI_USER;
GRANT USAGE ON WAREHOUSE CI_WH TO ROLE CI_ROLE;
GRANT ALL ON DATABASE CI_DB TO ROLE CI_ROLE;

-- 本番からのzero-copy clone権限（state:modified+ で参照用）
GRANT IMPORTED PRIVILEGES ON DATABASE PROD_DB TO ROLE CI_ROLE;


ポイント: パスワード認証ではなくkey pair認証にすると、秘密鍵をGitBucketのSecretsに格納してCIから安全に使えます。
Dockerfile (multi-stage)

# ===== Stage 1: base =====
FROM python:3.11-slim AS base
RUN pip install --no-cache-dir uv
WORKDIR /dbt

# ===== Stage 2: deps =====
FROM base AS deps
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# ===== Stage 3: dbt deps =====  
FROM deps AS dbt-deps
COPY packages.yml dbt_project.yml ./
RUN uv run dbt deps

# ===== Stage 4: parse (オフライン構文チェック) =====
FROM dbt-deps AS parse
COPY . .
RUN uv run dbt parse  # 接続不要、manifest生成

# ===== Stage 5: CI実行 (Snowflake接続あり) =====
FROM parse AS ci
ARG SNOWFLAKE_ACCOUNT
ARG SNOWFLAKE_USER
ARG SNOWFLAKE_ROLE=CI_ROLE
ARG SNOWFLAKE_WAREHOUSE=CI_WH
ARG SNOWFLAKE_DATABASE=CI_DB
ARG CI_SCHEMA  # PR番号などで動的に

ENV DBT_PROFILES_DIR=/dbt/.ci
COPY .ci/profiles.yml /dbt/.ci/profiles.yml

# 秘密鍵はbuild時にsecret mountで注入(後述)
RUN --mount=type=secret,id=sf_private_key,target=/tmp/key.p8 \
    cp /tmp/key.p8 /dbt/.ci/key.p8 && chmod 600 /dbt/.ci/key.p8


profiles.yml (CI用)

# .ci/profiles.yml
my_project:
  target: ci
  outputs:
    ci:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      private_key_path: /dbt/.ci/key.p8
      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE') }}"
      schema: "{{ env_var('CI_SCHEMA') }}"  # PR_123 等
      threads: 8
      query_tag: "ci_build_{{ env_var('CI_BUILD_ID') }}"


GitBucket CI Plugin設定

# .gitbucket/ci.yml (またはJenkinsfile風スクリプト)
stages:
  - name: build-and-test
    image: docker:24
    commands:
      # PR番号から一意なスキーマ名を生成
      - export CI_SCHEMA="PR_${CI_PULL_REQUEST_ID}_$(git rev-parse --short HEAD)"
      
      # 秘密鍵をsecretから取り出してbuild
      - echo "$SNOWFLAKE_PRIVATE_KEY" > /tmp/key.p8
      - |
        docker build \
          --target ci \
          --build-arg SNOWFLAKE_ACCOUNT=$SNOWFLAKE_ACCOUNT \
          --build-arg SNOWFLAKE_USER=$SNOWFLAKE_USER \
          --build-arg CI_SCHEMA=$CI_SCHEMA \
          --secret id=sf_private_key,src=/tmp/key.p8 \
          -t dbt-ci:$CI_BUILD_ID .
      
      # 本番manifestを取得してstate比較
      - aws s3 cp s3://dbt-artifacts/prod/manifest.json ./prod-manifest/
      
      # 差分のみbuild + test
      - |
        docker run --rm \
          -v $(pwd)/prod-manifest:/dbt/prod-manifest \
          dbt-ci:$CI_BUILD_ID \
          uv run dbt build \
            --select "state:modified+" \
            --defer --state /dbt/prod-manifest \
            --fail-fast
      
      # unit test (接続不要分は別途分離可)
      - |
        docker run --rm dbt-ci:$CI_BUILD_ID \
          uv run dbt test --select "test_type:unit"

  - name: cleanup
    when: always  # 成否に関わらず実行
    commands:
      - |
        docker run --rm dbt-ci:$CI_BUILD_ID \
          uv run dbt run-operation drop_ci_schema \
            --args "{schema: $CI_SCHEMA}"


設計上のキーポイント
1. PR単位のスキーマ分離
CI_SCHEMA = PR_123_abc1234 のように動的生成し、PR間の干渉を防ぐ。マージ後 or PR closeで必ずdrop。
2. state:modified+ + --defer
変更されたモデルとその下流だけを実行。未変更モデルは本番(PROD_DB)を参照するので、CI時間とSnowflakeクレジットを大幅節約。本番のmanifest.jsonをS3等に保管しておくのが前提。
3. zero-copy cloneで初期データ準備
大規模テーブルの場合、CREATE SCHEMA CI_PR_123 CLONE PROD.SCHEMA をon-run-startフックで実行すればストレージコストほぼゼロで本番同等データでテスト可能。
4. unit test と data test の分離
	∙	dbt test --select "test_type:unit" → ロジック検証(seed固定値)
	∙	dbt build 内の通常test → 実データ品質検証
5. クリーンアップ用macro

-- macros/drop_ci_schema.sql
{% macro drop_ci_schema(schema) %}
  {% do run_query("DROP SCHEMA IF EXISTS " ~ target.database ~ "." ~ schema ~ " CASCADE") %}
{% endmacro %}


GitBucket Secretsに登録するもの
	∙	SNOWFLAKE_ACCOUNT
	∙	SNOWFLAKE_USER (CI_USER)
	∙	SNOWFLAKE_PRIVATE_KEY (PEM全体)
	∙	AWS credentials (manifest取得用、IAM Roleが使えればそちら)
想定される詰まりどころ
GitBucket CI PluginのDocker-in-Docker: pluginの実行モードによってはDinD設定が必要。Docker socketをmountする方式の方がシンプルです。
manifest.json管理: 本番デプロイ時に必ずS3/MinIO等にuploadする運用フローが別途必要。これがないと--deferが機能しません。
Snowflakeクレジット監視: query_tagを仕込んでおくと、CI由来のクエリだけ集計できて費用管理が楽になります。
ここから先、どこを深掘りしましょうか？
	∙	本番デプロイ側のpipeline (manifest upload含む)
	∙	on-run-startでのzero-copy clone実装
	∙	DinD回避でDocker socket mount方式の具体手順
	∙	pre-commit + dbt-checkpoint との組み合わせ​​​​​​​​​​​​​​​​



Snowflakeの通常テーブルは、ユーザーが物理パーティションを設計する⽅式ではなく、Snowflakeが⾃動
で「マイクロパーティション」に分割して保存する⽅式です。マイクロパーティションは、Snowflakeのストレージ
層で管理される⼩さな連続ストレージ単位で、各パーティションには50MB〜500MBの⾮圧縮データが⼊
り、実際には圧縮されて保存されます。さらに、各マイクロパーティションの中では**列指向
（columnar）**でデータが保持され、列ごとに個別圧縮されます。
アーキテクチャ全体で⾒ると、Snowflakeは⼤きくストレージ層・コンピュート層・クラウドサービス層に分かれ
ています。保存データは中央のストレージ層に置かれ、クエリ実⾏は仮想ウェアハウスが担います。つまり、
データの格納と計算資源が分離されているのが基本思想です。
データの格納⽅法をイメージで⾔うと、たとえばテーブルに⼤量の⾏を INSERT / COPY すると、
Snowflakeはその投⼊順序をもとに⾏群を⾃動的にマイクロパーティションへ割り当てます。ユーザーが
PARTITION BY のように物理分割を細かく管理する必要はありません。Snowflake公式も、テーブルは
挿⼊・ロード順に従って透過的にマイクロパーティション化されると説明しています。
各マイクロパーティションには、データ本体だけでなくメタデータも保持されます。代表例は、各列の値の最
⼩値・最⼤値の範囲、distinct 値数、その他の最適化⽤情報です。このメタデータのおかげで、クエリ時
に不要なマイクロパーティションを読み⾶ばすプルーニングができます。たとえば⽇付条件やID範囲条件が
あると、該当しないパーティションをスキャンしないで済むので、⼤規模表でも効率が出ます。
重要なのは、Snowflakeのマイクロパーティションは⾃動管理であり、従来型DBの静的パーティションのよ
うに事前設計・⼿動保守を前提にしないことです。また、Snowflake公式は、マイクロパーティションは値
範囲が互いに重なり得ると説明していて、これが偏りの緩和にも役⽴つとしています。
性能⾯では、列指向格納と列単位圧縮の組み合わせが効きます。つまり SELECT col1, col2 ... のよう
なクエリでは、参照した列だけを重点的に読むことができ、⾏全体を毎回読む必要がありません。加え
て、圧縮⽅式はSnowflakeが列ごとに⾃動選択します。
また、テーブルの並びがクエリ条件に合っていない場合や、⻑期間のDMLで⾃然な並びが崩れた場合に
は、クラスタリングキーで「似た値を同じマイクロパーティションに寄せる」ことができます。これは特に⾮常に
⼤きいテーブルで、フィルタ列に偏ったアクセスがあるときに有効です。なお、クラスタリングは万能ではなく、
維持のために計算資源を使うので、必要な表だけに使うべきです。
整理すると、Snowflakeのデータ格納⽅法は次の理解でほぼ合っています。
⾏データをロードする → Snowflakeが⾃動で⼩さなマイクロパーティションに分ける → 各パーティション内
では列指向・圧縮保存する → パーティションごとの統計メタデータを持つ → 実⾏時はそのメタデータで不
要領域を⾶ばして読む、という流れです。

補⾜ですが、これは通常のSnowflakeテーブルの話です。Snowflakeには例外として、Hybrid Table は
⾏指向ストレージエンジン、Iceberg Table は外部クラウドストレージにデータ/メタデータを保持という別系
統もあります。なので「Snowflakeは全部マイクロパーティション」と⾔うときは、通常テーブルの説明として
理解するのが正確です。
情報源は主に以下です。すべてSnowflake公式です。
• Snowflake key concepts and architecture
• Micro-partitions & Data Clustering
• Clustering Keys & Clustered Tables
• Hybrid tables
• Apache Iceberg tables
必要なら次に、
「RDBの物理ページ/索引との違い」
または
「INSERT・UPDATE・DELETEしたときマイクロパーティションがどう増減するか」
まで踏み込んで説明します。
では、先ほどの続きとして
## ① RDB（Oracle / PostgreSQL など）との物理構造の違い
## ② INSERT / UPDATE / DELETE時にマイクロパーティションがどう変化するか
を、Snowflakeの内部構造ベースで説明します。
公式資料も併記します。
## ① 従来RDBとSnowflakeの物理構造の違い
従来RDB（例：Oracle / PostgreSQL）

従来DBの基本構造は ページ（ブロック）＋インデックスです。
TABLE
```
├─ Page (8KBなど)
│ ├─ row1
│ ├─ row2
│ └─ row3
├─ Page
│ ├─ row4
│ ├─ row5
│ └─ row6
特徴
• ⾏指向（row store）
• ページサイズは数KB
• B-tree indexが必須
• UPDATEはページを書き換える
• VACUUMやREORGが必要
検索は基本的に
WHERE id = 100
↓
index lookup
↓
page read
↓
row read
つまり
インデックスが性能の中⼼
になります。
Snowflake
```

Snowflakeは構造が根本的に違います。
TABLE
```
├─ Micro-partition
│ ├─ column A
│ ├─ column B
│ ├─ column C
│
├─ Micro-partition
│ ├─ column A
│ ├─ column B
│ ├─ column C
特徴
• 列指向（column store）
• 50〜500MB ⾮圧縮
• インデックス不要
• メタデータで探索
Snowflakeは
各マイクロパーティションに統計情報を持っています
例
partition1
date min = 2024-01-01
date max = 2024-01-10
partition2
date min = 2024-01-11
date max = 2024-01-20
クエリ
```sql
SELECT *
FROM orders
WHERE date = '2024-01-15'
```
Snowflakeは
partition1 → skip
partition2 → read
```

になります。
これを
micro-partition pruning
と呼びます。
つまり
項⽬
RDB
Snowflake
探索 index metadata
単位 page micro-partition
格納 row column
管理 ⼿動 ⾃動
です。
## ② INSERT / UPDATE / DELETE時の内部動作
Snowflakeは
immutable storage
という設計です。
つまり
既存データを書き換えません。

| 項⽬ |  |  |
| --- | --- | --- |
| RDB |  |  |
| Snowflake |  |  |
| 探索 | index | metadata |
| 単位 | page | micro-partition |
| 格納 | row | column |
| 管理 | ⼿動 | ⾃動 |


INSERT
INSERTすると
新しいマイクロパーティションが生成
例
partition1
partition2
partition3 ← new
Snowflakeは
ロード順でパーティションを作る
ので
```sql
COPY INTOなどで⼤量ロードすると
```
⾃然にデータが整列します。
UPDATE
UPDATEは
書き換えではなく再⽣成
になります。
例
元データ

partition1
row1
row2
row3
UPDATE
```sql
UPDATE table
SET price = 100
WHERE id = 2
```
内部では
partition1 (old) → 無効化
partition4 (new) → 新しいデータ
になります。
イメージ
partition1 (obsolete)
partition4 (active)
Snowflakeは
Time Travel を実現するために
古いパーティションを保持します。
DELETE
DELETEも同様です。
```sql
DELETE FROM table
WHERE id = 5
```
結果
旧パーティション → obsolete
新パーティション → id=5を除いたデータ

つまり
Snowflakeは
常に新しいマイクロパーティションを⽣成
します。
## ③ マイクロパーティションが増える理由
⼤量の
• UPDATE
• DELETE
• MERGE
を⾏うと
old partitions
new partitions
が増えます。
その結果
クラスタリングが崩れる
ことがあります。
そこで使うのが
CLUSTER BY
です。
これにより

Snowflakeがバックグラウンドで
パーティションを再編成
します。
## ④ Snowflakeのストレージ構造（全体）
全体構造はこうなります。
Client
```
│
SQL
│
Cloud Services
│
Query Optimizer
│
Virtual Warehouse
│
Scan Micro-partitions
│
Cloud Storage
データ⾃体は
AWS S3
Azure Blob
GCS
に保存されています。
```
## ⑤ マイクロパーティションのメタデータ

Snowflakeは各パーティションに
以下を持っています。
metadata
内容
min 最⼩値
max 最⼤値
distinct count ユニーク数
null count NULL数
これがあるので
index不要
になります。
## ⑥ Snowflakeが速い理由（本質）
Snowflakeの速さの本質は
3つ
です。
## ① Columnar storage
## ② Micro-partition pruning
## ③ Massive parallel scan
つまり
必要な列だけ読む
```
↓
```

| metadata |  |
| --- | --- |
| 内容 |  |
| min | 最⼩値 |
| max | 最⼤値 |
| distinct count | ユニーク数 |
| null count | NULL数 |


必要なパーティションだけ読む
```
↓
大量並列スキャン
公式情報源
Snowflake公式
Micro-partitions
```
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
Snowflake architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
Clustering keys
https://docs.snowflake.com/en/user-guide/tables-clustering-keys
Table storage
https://docs.snowflake.com/en/user-guide/tables-intro

もし興味あれば（ここからが本当に重要）
Snowflakeを設計する⼈が必ず知るべきテーマがあります。
実務ではむしろこっちが重要です。
１ ⃣ マイクロパーティションサイズの実態
２ ⃣ なぜORDER BYでロードすると速くなるのか
３ ⃣ クラスタリングキーはいつ必要か
４ ⃣ MERGEがSnowflakeで遅い理由
５ ⃣ Pruningが効かないアンチパターン
このあたりは
Snowflakeを設計するエンジニアなら必須知識です。
もしよければ
Snowflake内部アーキテクチャ（上級編）
を図付きで解説します。
かなり⾯⽩いと思います。
では Snowflake内部アーキテクチャ（上級編） を、実際にSnowflakeの設計思想に近い形で整理し
ます。
データレイク設計や性能設計をする⼈が理解しているレベルの内容です。
（図は理解⽤の簡略図です）
1 Snowflakeのストレージ内部構造
Snowflakeのデータはクラウドオブジェクトストレージに保存されています。
Snowflake Table
```
│
▼
```

Micro-partition
```
│
▼
Column segments
│
▼
Compressed column files
│
▼
Cloud Object Storage (S3 / Blob / GCS)
つまりSnowflakeのテーブルは実体として
⼤量の⼩さいカラムファイル集合
です。
内部イメージ
table_orders
├─ partition_001
│ ├─ col_order_id
│ ├─ col_date
│ ├─ col_price
│
├─ partition_002
│ ├─ col_order_id
│ ├─ col_date
│ ├─ col_price
特徴
• 各列は個別圧縮
• カラムごとに最適圧縮
• カラム単位で読み込み
参考
Snowflake Docs: Micro-partitions
```
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions

2 マイクロパーティション内部
1つのマイクロパーティションの中は
Micro-partition (50–500MB uncompressed)
metadata
```
├─ column min
├─ column max
├─ distinct count
├─ null count
data
├─ column A (compressed)
├─ column B (compressed)
├─ column C (compressed)
ポイント
metadataがクエリ⾼速化の本体
です。
3 Snowflake Query実⾏フロー
SQLが実⾏されると内部ではこう動きます。
Client
│
▼
Cloud Services Layer
│
├─ Authentication
├─ Metadata management
├─ Query optimizer
│
▼
```

Virtual Warehouse (Compute)
```
│
▼
Micro-partition scan
│
▼
Object storage
Snowflakeの特徴
ComputeとStorageが完全分離
です。
参考
Snowflake Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
4 Partition Pruning（超重要）
Snowflakeが速い理由の中⼼です。
例
テーブル
partition_1
date min = 2024-01-01
date max = 2024-01-10
partition_2
date min = 2024-01-11
date max = 2024-01-20
partition_3
date min = 2024-01-21
date max = 2024-01-30
```

クエリ
```sql
SELECT *
FROM orders
WHERE date = '2024-01-15'
```
Snowflakeの動き
partition_1 → skip
partition_2 → scan
partition_3 → skip
この処理は
メタデータだけで判断
します。
つまり
ストレージアクセス前に除外
できます。
これが
Micro-partition pruning
です。
5 Parallel Scan
Snowflakeはマイクロパーティションを
並列スキャン
します。
Warehouse node 1 → partition 1

Warehouse node 2 → partition 2
Warehouse node 3 → partition 3
Warehouse node 4 → partition 4
つまり
scan speed ∝ compute nodes
になります。
6 なぜ ORDER BY でロードすると速いか
これは実務で超重要です。
例
```sql
COPY INTO orders
FROM stage
ORDER BY order_date
```
すると
partition_1
date 1-10
partition_2
date 11-20
partition_3
date 21-30
になります。
つまり
partitionが⾃然にクラスタリング
されます。
結果

WHERE order_date = '2024-01-15'
の時
1パーティションだけ読む
ことになります。
7 Pruningが効かないアンチパターン
Snowflakeが遅い原因の80%はこれです。
①関数をかける
悪い例
WHERE YEAR(order_date) = 2024
理由
min/max metadata が使えない
良い例
WHERE order_date
BETWEEN '2024-01-01'
AND '2024-12-31'
②⽂字列LIKE
WHERE email LIKE '%gmail.com'
これもpruning不可です。

③JSON
VARIANT列
WHERE data:user_id = 10
もpruning効かないことがあります。
8 クラスタリングキーの本当の意味
クラスタリングキーは
indexではありません
役割
similar values
```
↓
same micro-partition
例
CLUSTER BY (order_date)
すると
date 1-10
date 11-20
date 21-30
のように整理されます。
参考
```
https://docs.snowflake.com/en/user-guide/tables-clustering-keys

9 SnowflakeのTime Travel構造
Snowflakeは
copy-on-write
です。
例
```sql
UPDATE table
SET price = 100
WHERE id = 1
```
内部
old partition
new partition
になります。
partition1 (old)
partition2 (new)
これで
```sql
SELECT *
```
AT (TIMESTAMP => '1 hour ago')
が可能です。
10 Snowflakeの設計思想（核⼼）
Snowflakeの思想は
Cheap storage

Massive parallel scan
Metadata pruning
つまり
indexを捨ててスキャンを⾼速化
しています。
まとめ（Snowflake理解の核⼼）
Snowflakeの本質
Table
```
↓
Micro-partitions
↓
Columnar compressed files
↓
Object storage
クエリ
metadata pruning
↓
parallel scan
↓
result
もし興味あれば（実務で⼀番重要）
データレイクやSnowflake設計では
次の理解がかなり重要です。
```

Snowflake設計の核⼼
１ ⃣ なぜ⼩さいファイルはダメなのか
２ ⃣ Snowflakeの理想パーティション数
３ ⃣ COPY INTO のベストサイズ
４ ⃣ SnowflakeでMERGEが遅い理由
５ ⃣ dbt + Snowflakeの最適設計
ここを理解すると
Snowflakeの設計レベルが⼀段上がります。
もしよければ
Snowflakeアーキテクチャの「プロ設計編」
（AWSデータレイク設計レベル）も解説します。
かなり⾯⽩いと思います。
では Snowflakeアーキテクチャ「プロ設計編」 をまとめます。
実際に データレイク・DWH設計をするエンジニアが理解しているレベルで解説します。
（Snowflake + AWS データレイク構成も含めます）
1 Snowflakeの性能を決める本質
Snowflakeの性能は実は 3つだけで決まります。
## ① Micro-partition pruning
## ② File size
## ③ Parallel scan
つまり
速さ = (スキャン量) / (並列度)
になります。
Snowflakeは

index → 使わない
scan → 高速化
という思想です。
2 Snowflake理想ファイルサイズ
Snowflakeは内部でマイクロパーティションを作りますが、
その元になるのが ロードファイルサイズです。
理想サイズ
サイズ
評価
50MB〜250MB ベスト
10MB以下 NG
1GB以上 NG
理由
⼩さすぎる
file1
file2
file3
file4
結果
micro-partition 爆増
metadata増大
scan overhead増加

| サイズ |  |
| --- | --- |
| 評価 |  |
| 50MB〜250MB | ベスト |
| 10MB以下 | NG |
| 1GB以上 | NG |


⼤きすぎる
1つの巨大パーティション
結果
parallelism低下
Snowflake公式も推奨しています。
参考
Snowflake Data Loading Best Practices
https://docs.snowflake.com/en/user-guide/data-load-considerations
3 COPY INTO のベスト設計
Snowflakeでは
```sql
COPY INTO
```
が最重要ロード⼿段です。
理想構成
ETL
```
↓
S3 stage
↓
```sql
COPY INTO
```
↓
Snowflake table
ファイル構成
s3://bucket/orders/
orders_0001.parquet
orders_0002.parquet
```

orders_0003.parquet
サイズ
100MB 前後
これが理想です。
4 SnowflakeでMERGEが遅い理由
Snowflakeで最もよくある問題です。
原因
```sql
UPDATE / MERGE
```
```
↓
copy-on-write
↓
micro-partition 再生成
つまり
MERGE = rewrite
になります。
例
10TB table
↓
MERGE 1000 rows
↓
100GB rewrite
ということが起きます。
5 プロがやるSnowflake設計
```

MERGEを減らします。
代わりに
append-only
を使います。
設計
raw table
```
↓
append
↓
transform
↓
final table
6 dbt + Snowflakeの理想構成
Snowflakeでは
ELT
が基本です。
構成
Source system
↓
S3 stage
↓
Snowflake RAW
↓
dbt transform
↓
Snowflake MART
```

イメージ
RAW
```
├─ events_raw
├─ orders_raw
STAGING
├─ events_clean
├─ orders_clean
MART
├─ fact_orders
├─ dim_customer
7 Snowflakeの理想テーブルサイズ
Snowflakeは
巨大テーブル
に向いています。
理想
1TB〜100TB
逆に
小さいテーブル多数
は性能悪化します。
理由
metadata管理
```

8 Snowflake + AWSデータレイク構成
プロ設計ではこの構成が多いです。
```
┌──────────────┐
│ Source DB │
└──────┬───────┘
│
▼
┌──────────────┐
│ AWS DMS │
└──────┬───────┘
│
▼
┌──────────────┐
│ S3 │
│ Data Lake │
└──────┬───────┘
│
▼
┌──────────────┐
│ Snowflake │
│ ExternalStage│
└──────┬───────┘
│
▼
┌──────────────┐
│ COPY INTO │
└──────┬───────┘
│
▼
┌──────────────┐
│ Snowflake DWH│
└──────────────┘
9 Snowflake External Table設計
```

Snowflakeは
External table
も使えます。
S3
```
↓
Parquet
↓
External table
メリット
データコピー不要
デメリット
性能は内部テーブルより遅い
10 Snowflake設計アンチパターン
実務で多い失敗です。
```
## ① ⼩さいファイル
1MB files
→ 1000倍遅くなる
## ② UPDATE多発
OLTP的利用

→ Snowflake向いてない
## ③ 過剰クラスタリング
CLUSTER BY 10 columns
→ コスト爆増
11 Snowflakeのコスト最適化
コスト
Compute
Storage
Cloud services
最重要
warehouse size
プロは
Small warehouse
+
Auto scale
を使います。
12 Snowflake設計の最終形
プロのSnowflake設計はこうなります。

```
┌─────────────┐
│ Source DB │
└──────┬──────┘
│
▼
┌─────────────┐
│ Kafka │
└──────┬──────┘
│
▼
┌─────────────┐
│ S3 │
│ Data Lake │
└──────┬──────┘
│
▼
┌─────────────┐
│ Snowflake │
│ RAW layer │
└──────┬──────┘
│
▼
┌─────────────┐
│ dbt │
│ Transform │
└──────┬──────┘
│
▼
┌─────────────┐
│ Data Mart │
└─────────────┘
Snowflake設計の核⼼まとめ
Snowflakeは
OLTP DBではない
```

です。
思想
Append
Scan
Parallel
です。
参考資料
Snowflake公式
Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
Micro-partitions
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
Loading best practices
https://docs.snowflake.com/en/user-guide/data-load-considerations
Clustering
https://docs.snowflake.com/en/user-guide/tables-clustering-keys
もしよければですが、
あなたが最初に質問してくれた
AWSでデータレイク構築 + Snowflake
このテーマには さらに重要な話があります。
実はプロ設計では

Snowflakeだけでデータレイクは作りません。
理由があります。
もし興味あれば次に
Snowflake + AWSデータレイクの本当の設計（超重要）
を解説します。
（多くの会社がここを間違えています）
では **Snowflake + AWS データレイクの「本当の設計」**を解説します。
これは実際に⼤規模データ基盤でよく採⽤されるアーキテクチャです。
ポイントは **「Snowflakeをデータレイクの代わりにしない」**ことです。
1 なぜSnowflakeだけでデータレイクを作らないのか
まず前提です。
Snowflakeは本質的に **DWH（分析基盤）**です。
データレイクとは役割が違います。
役割
Snowflake
データレイク
⽬的 分析 データ保存
データ形式 構造化 ⾮構造化OK
コスト ⾼め ⾮常に安い
保存期間 分析期間 ⻑期保存
そのためプロ設計では

| 役割 |  |  |
| --- | --- | --- |
| Snowflake |  |  |
| データレイク |  |  |
| ⽬的 | 分析 | データ保存 |
| データ形式 | 構造化 | ⾮構造化OK |
| コスト | ⾼め | ⾮常に安い |
| 保存期間 | 分析期間 | ⻑期保存 |


S3 = Data Lake
Snowflake = Data Warehouse
という分担になります。
2 プロが作るデータ基盤の基本構造
典型的な構成です。
```
┌───────────────┐
│ Source Systems │
│ DB / API / App │
└───────┬───────┘
│
▼
┌────────────┐
│ Ingestion │
│ DMS/Kafka │
└──────┬─────┘
│
▼
┌────────────┐
│ S3 │
│ Data Lake │
└──────┬─────┘
│
┌────────────┼────────────┐
▼ ▼ ▼
Raw Zone Processed Analytics
Zone Zone
│
▼
┌────────────┐
│ Snowflake │
│ DWH │
└────────────┘
```

3 S3データレイクのレイヤー構造
データレイクでは通常 3層構造にします。
S3 Data Lake
/raw
/processed
/analytics
Raw
sourceのデータそのまま
例
s3://lake/raw/orders/
s3://lake/raw/events/
形式
JSON
CSV
Parquet
Avro
Rawの特徴
• 永久保存
• 変換しない
• 再処理可能
Processed

データを整形します。
例
JSON → Parquet
型変換
カラム整理
s3://lake/processed/orders/
ここで
partition
compression
schema
を整えます。
Analytics
分析⽤に最適化
fact table
dimension table
例
s3://lake/analytics/orders_daily/
4 Snowflakeはここで使う
Snowflakeは 分析層です。
S3 (Data Lake)
```
│
```

```
▼
External Stage
│
▼
```sql
COPY INTO
```
│
▼
Snowflake Tables
Snowflakeの役割
SQL
BI
Analytics
ML
5 なぜS3にデータを残すのか
理由は3つあります。
①コスト
Snowflake storage
$40/TB/month
S3
$23/TB/month
さらに
Glacier
なら
$4/TB
```

②ベンダーロック回避
Snowflakeだけだと
Snowflake依存
になります。
S3なら
Athena
Spark
Databricks
Snowflake
全部使えます。
③再処理
ETLミスが起きた場合
rawから再生成
できます。
6 Snowflake + S3連携⽅法
Snowflakeでは
External Stage

を作ります。
例
```sql
CREATE STAGE orders_stage
```
URL='s3://data-lake/orders'
STORAGE_INTEGRATION = my_s3_integration;
その後
```sql
COPY INTO orders
FROM @orders_stage
FILE_FORMAT = (TYPE = PARQUET);
```
7 External Table
S3データを直接読む⽅法もあります。
```sql
CREATE EXTERNAL TABLE orders_ext
```
WITH LOCATION=@orders_stage
FILE_FORMAT=(TYPE=PARQUET);
構造
S3
```
↓
Parquet
↓
External table
↓
SQL
ただし
性能は内部テーブル > External table
です。
```

8 Snowflake設計の実務ベストパターン
最もよく使われる構成
Source DB
```
│
▼
CDC / Kafka
│
▼
S3
(Raw Data Lake)
│
▼
Snowflake RAW
│
▼
dbt Transform
│
▼
Data Mart
│
▼
BI (Tableau)
9 Snowflake + AWSのプロ構成
実務ではこうなります。
┌───────────┐
│ RDS / DB │
└─────┬─────┘
│
▼
┌───────────┐
```

│ AWS DMS │
```
└─────┬─────┘
│
▼
┌───────────┐
│ S3 │
│ DataLake │
└─────┬─────┘
│
▼
┌───────────┐
│ Snowflake │
│ RAW │
└─────┬─────┘
│
▼
┌───────────┐
│ dbt │
└─────┬─────┘
│
▼
┌───────────┐
│ BI Tools │
└───────────┘
10 よくある失敗
❌
Snowflakeに直接データ投⼊
source → snowflake
→ Raw消失
```

❌
⼩さいファイル
1MB files
→ Snowflake性能劣化
❌
UPDATE中⼼
Snowflakeは
append型
が基本です。
まとめ
Snowflake + Data Lake の本質
S3 = Data Lake
Snowflake = Compute + Analytics
Snowflakeは
Lakehouse
ではなく
Cloud Data Warehouse
です。
もし興味あればですが、
あなたが質問してくれた内容はかなりレベルが⾼いので、次に

Snowflake内部アーキテクチャ（さらに深い話）
を説明できます。
これは普通のエンジニアは知らない内容で
• マイクロパーティションの実際の数
• Snowflakeのメタデータ管理
• Query Optimizer
• Result Cache
• Warehouse内部構造
まで解説できます。
かなり⾯⽩いので、希望あれば続けます。
では **Snowflake内部アーキテクチャ（さらに深いレベル）**を、
①内部コンポーネント / ②クエリ実⾏ / ③キャッシュ / ④メタデータ管理 / ⑤マイクロパーティション実態
の順で説明します。
できるだけ 公式ドキュメントと論⽂系資料に基づいて解説します。
1 Snowflakeの3層アーキテクチャ
Snowflakeは基本的に 3レイヤー構造です。
Client
```
│
▼
Cloud Services Layer
│
▼
Compute Layer (Virtual Warehouse)
│
▼
```

Storage Layer (Object Storage)
Storage Layer
データ保存
AWS S3
Azure Blob Storage
Google Cloud Storage
ここに
micro-partition
columnar data
compressed files
が保存されます。
Compute Layer
SQL処理
Virtual Warehouse
ここが
• SQL実⾏
• join
• aggregation
• scan
を担当します。
Cloud Services Layer
メタデータ管理
Authentication

Query optimizer
Metadata services
Access control
Transaction management
参考
Snowflake Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
Snowflake Paper
The Snowflake Elastic Data Warehouse
2 Virtual Warehouse内部構造
Virtual Warehouseは実体として
複数のノードクラスター
です。
Virtual Warehouse
```
├─ Node 1
├─ Node 2
├─ Node 3
└─ Node 4
Warehouseサイズ
```

Size
Nodes
XS 1
S 2
M 4
L 8
XL 16
（概念図）
Query
```
│
▼
Task Scheduler
│
▼
Distributed Execution
│
├─ node1 → partition scan
├─ node2 → partition scan
├─ node3 → partition scan
└─ node4 → partition scan
参考
```
https://docs.snowflake.com/en/user-guide/warehouses-overview
3 Snowflake Query実⾏フロー
SQLが実⾏されると内部ではこう動きます。
1 SQL submit
2 Parse
3 Optimization

| Size |  |
| --- | --- |
| Nodes |  |
| XS | 1 |
| S | 2 |
| M | 4 |
| L | 8 |
| XL | 16 |


4 Execution plan
5 Partition pruning
6 Parallel scan
7 Result aggregation
具体的には
Client
```
│
▼
Cloud Services
│
├─ SQL parser
├─ Optimizer
└─ Execution planner
│
▼
Virtual Warehouse
│
▼
Micro-partition scan
参考
Snowflake Query Processing
https://docs.snowflake.com/en/user-guide/querying-overview
4 Snowflakeのキャッシュ構造
Snowflakeは 3種類のキャッシュを持っています。
```
## ① Result Cache
完全⼀致クエリのキャッシュ

```sql
SELECT * FROM orders
```
同じクエリを実⾏すると
Result Cache
から即返ります。
特徴
• 24時間保持
• Warehouse不要
• 完全⼀致SQL
参考
https://docs.snowflake.com/en/user-guide/querying-persisted-results
## ② Local Disk Cache
Virtual Warehouse内のSSDキャッシュです。
Node SSD
保存
micro-partition
そのため
同じwarehouse
で再実⾏すると速いです。
参考
Snowflake performance optimization
https://docs.snowflake.com/en/user-guide/performance-query-warehouse-cache

## ③ Remote Disk Cache
クラウドストレージから取得したデータを
cluster cache
として共有する仕組みです。
5 Snowflakeメタデータ管理
Snowflakeの特徴は
メタデータ中⼼設計
です。
管理される情報
table schema
micro-partition metadata
query statistics
access control
マイクロパーティションのメタデータ
min value
max value
distinct count
null count
これにより
partition pruning
が可能になります。
参考

Snowflake Micro-partitions
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
6 マイクロパーティションの実際
1パーティション
50MB – 500MB
(uncompressed)
例
1TB table
の場合
約2000〜20000 partition
になります。
内部構造
Micro-partition
metadata
```
├─ column min
├─ column max
├─ distinct count
data
├─ column1 compressed
├─ column2 compressed
├─ column3 compressed
参考
Snowflake table storage
https://docs.snowflake.com/en/user-guide/tables-intro
```

7 Time Travel内部構造
Snowflakeは
copy-on-write
です。
UPDATE
```sql
UPDATE orders
SET price = 100
WHERE id = 1
```
内部
old partition → preserved
new partition → created
そのため
```sql
SELECT *
```
AT (TIMESTAMP => '1 hour ago')
が可能です。
参考
https://docs.snowflake.com/en/user-guide/data-time-travel
8 Snowflake Query Optimizer
Snowflakeのoptimizerは
cost based optimizer
です。

考慮要素
partition pruning
join strategy
broadcast join
hash join
実⾏例
large table
```
↓
partition pruning
↓
parallel scan
参考
Snowflake Query Optimization
```
https://docs.snowflake.com/en/user-guide/performance-query-optimization
9 Snowflake Result Pipeline
Snowflakeでは結果も
distributed aggregation
で処理されます。
node1 → partial result
node2 → partial result
node3 → partial result
```
↓
final aggregation
```

まとめ
Snowflakeの本質
Object storage
+
Massively parallel compute
+
Metadata pruning
です。
つまり
index不要
scan高速化
という設計です。
主な情報源
Snowflake公式ドキュメント
• Snowflake Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
• Micro-partitions
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
• Query Optimization
https://docs.snowflake.com/en/user-guide/performance-query-optimization
• Time Travel
https://docs.snowflake.com/en/user-guide/data-time-travel
• Warehouses
https://docs.snowflake.com/en/user-guide/warehouses-overview
研究論⽂

• The Snowflake Elastic Data Warehouse
もしよければですが、
ここまで理解すると Snowflakeの本当の核⼼が⾒えてきます。
実はエンジニアでもあまり知らないのですが、次のテーマがさらに重要です。
Snowflakeの内部設計の核⼼
1. なぜSnowflakeはインデックスを作らないのか
2. なぜBigQueryより⾼速なケースがあるのか
3. SnowflakeのOptimizerの特殊仕様
4. マイクロパーティションの実際の数
5. Snowflakeの「検索最適化サービス」
このあたりは データ基盤設計レベルの知識になります。
では続きとして、**Snowflake内部設計の核⼼（さらに深い設計思想）**を説明します。
今回は特に重要な次の5つを扱います。
1. なぜSnowflakeはインデックスを使わないのか
2. SnowflakeのOptimizerの特徴
3. Search Optimization Service（疑似インデックス）
4. マイクロパーティション数の実態
5. BigQueryなどとのアーキテクチャ⽐較
可能な限り公式資料・論⽂ベースで説明します。
1 なぜSnowflakeはインデックスを使わないのか

従来のRDBでは
table
```
├─ data pages
└─ index (B-tree)
検索は
index lookup
↓
row read
です。
しかしSnowflakeでは
table
└─ micro-partitions
└─ metadata
を使います。
メタデータ
min value
max value
distinct count
null count
このメタデータにより
partition pruning
が可能になります。
なぜindexを作らないか
理由は3つです。
```
## ① massive scan型
Snowflakeは

parallel scan
を前提に設計されています。
つまり
scan > index lookup
という思想です。
## ② index管理コスト
インデックスは
insert
update
delete
のたびに更新されます。
Snowflakeは
immutable storage
なので
index管理が相性悪いです。
## ③ micro-partition metadata
Snowflakeは
automatic partition statistics
を常に持っています。
つまり

index = 不要
になります。
参考
Snowflake micro-partitions
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
論⽂
The Snowflake Elastic Data Warehouse
2 Snowflake Optimizer
Snowflakeのoptimizerは
cost based optimizer
です。
主な判断要素
partition pruning
join order
join strategy
statistics
join戦略
hash join
broadcast join
merge join
例
large table join small table
なら
broadcast join
を選択します。

3 Search Optimization Service
Snowflakeには
擬似インデックス
があります。
それが
Search Optimization Service
です。
Search Optimization Service
⽬的
point lookup高速化
例
```sql
SELECT *
FROM users
WHERE email = 'abc@gmail.com'
```
普通
table scan
ですが
Search Optimizationを使うと
metadata lookup
になります。
使い⽅

ALTER TABLE users
ADD SEARCH OPTIMIZATION
ON EQUALITY(email);
参考
https://docs.snowflake.com/en/user-guide/search-optimization-service
4 マイクロパーティション数の実態
Snowflakeの実際のパーティション数はかなり多いです。
例
1TB table
だと
4000〜20000 partitions
になります。
理由
50MB〜500MB
だからです。
イメージ
table
```
├─ partition1
├─ partition2
├─ partition3
├─ partition4
├─ partition5
この数が多いほど
parallel scan
が効きます。
```

5 Snowflake vs BigQuery
アーキテクチャ⽐較
項⽬
Snowflake
BigQuery
Compute 仮想Warehouse Dremel
Storage object storage Colossus
Execution MPP cluster serverless
Partition micro-partition partition table
Snowflake
compute-storage separation
BigQuery
serverless execution
参考
BigQuery Architecture
Google BigQuery
6 SnowflakeのResult Cache
Snowflakeの強⼒な機能です。

| 項⽬ |  |  |
| --- | --- | --- |
| Snowflake |  |  |
| BigQuery |  |  |
| Compute | 仮想Warehouse | Dremel |
| Storage | object storage | Colossus |
| Execution | MPP cluster | serverless |
| Partition | micro-partition | partition table |


query result
をキャッシュします。
特徴
same query
same data
なら
0秒で返る
仕組み
query hash
```
↓
result cache
参考
```
https://docs.snowflake.com/en/user-guide/querying-persisted-results
7 Snowflakeの本当の設計思想
Snowflakeの設計思想
cheap storage
massive parallel compute
metadata pruning
つまり
index → 不要
scan → 高速化
という
完全に新しいDWH設計
です。

まとめ
Snowflakeの内部構造
Object Storage
```
│
Micro-partitions
│
Metadata pruning
│
Massively parallel scan
参考資料
Snowflake公式
• Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
• Micro-partitions
```
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
• Query optimization
https://docs.snowflake.com/en/user-guide/performance-query-optimization
• Search Optimization
https://docs.snowflake.com/en/user-guide/search-optimization-service
論⽂
• The Snowflake Elastic Data Warehouse

もしよければですが、
あなたの質問レベルはかなり⾼いので、次に Snowflakeを理解する上で⼀番重要なテーマを解説できま
す。
実はSnowflakeの内部で⼀番重要なのは
「クラスタリング深度（clustering depth）」
です。
これを理解すると
• Snowflakeのパフォーマンス設計
• クラスタリングキー設計
• Snowflakeチューニング
がすべて理解できます。
かなり⾯⽩いテーマなので、希望あれば解説します。
では **Snowflakeの「クラスタリング深度（Clustering Depth）」**を、
内部構造 → なぜ重要か → 実際のチューニング → 確認⽅法の順で解説します。
これはSnowflakeの性能設計でかなり重要な概念です。
1 クラスタリングとは何か
Snowflakeのテーブルは通常、ロード順でマイクロパーティションに分割されます。
INSERT順
```
↓
micro-partition
例
```

partition1
date 2024-01-01〜2024-01-03
partition2
date 2024-02-10〜2024-02-12
partition3
date 2024-01-05〜2024-01-08
この状態では
WHERE order_date = '2024-01-06'
を実⾏すると
partition1
partition2
partition3
全部読む可能性があります。
2 良いクラスタリング状態
理想状態は
partition1
2024-01-01〜2024-01-05
partition2
2024-01-06〜2024-01-10
partition3
2024-01-11〜2024-01-15
この状態なら
WHERE order_date = '2024-01-06'
は
partition2

だけ読みます。
これが
Micro-partition pruning
です。
参考
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
3 クラスタリング深度とは
Snowflakeはクラスタリング状態を
Clustering Depth
で表します。
概念
depth = overlapping partitions
図
悪い例
partition1 1-10
partition2 5-15
partition3 8-18
partition4 12-22
検索
WHERE date = 9
結果
partition1
partition2

partition3
読む必要があります。
この時
clustering depth = 3
になります。
4 理想的な深度
Snowflake公式の⽬安
depth
状態
1 理想
2〜5 良い
10以上 悪い
深度が⼤きいほど
scan partitions ↑
query cost ↑
になります。
参考
https://docs.snowflake.com/en/user-guide/tables-clustering-keys
5 深度が悪化する原因

| depth |  |
| --- | --- |
| 状態 |  |
| 1 | 理想 |
| 2〜5 | 良い |
| 10以上 | 悪い |


主に3つです。
## ① INSERT順がランダム
例
orders
customer
date
のように
ランダム挿入
すると
partition overlap
が増えます。
## ② UPDATE / MERGE
Snowflakeは
copy-on-write
なので
UPDATE
```
↓
new partitions
になります。
結果
clustering崩壊
します。
```

## ③ ⻑期間運⽤
時間が経つと
data fragmentation
が発⽣します。
6 クラスタリングキー
Snowflakeでは
CLUSTER BY
が使えます。
例
```sql
CREATE TABLE orders
CLUSTER BY(order_date);
```
Snowflakeが
similar values
```
↓
same partition
になるように
⾃動再配置します。
参考
```
https://docs.snowflake.com/en/user-guide/tables-clustering-keys

7 クラスタリング再構築
Snowflakeは
automatic reclustering
を⾏います。
内部処理
old partitions
```
↓
rebuild partitions
↓
better clustering
8 クラスタリング深度の確認
Snowflakeでは
SYSTEM$CLUSTERING_INFORMATION
で確認できます。
例
```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('orders');
```
結果
average_depth
partition_count
overlap
が表⽰されます。
参考
```

https://docs.snowflake.com/en/sql-reference/functions/system_clustering_information
9 実務での設計ポイント
クラスタリングキーは
よくフィルタされる列
にします。
例
date
customer_id
region
逆に
random id
uuid
はNGです。
10 Snowflakeチューニングの優先順位
プロのSnowflakeチューニング順序
## ① partition pruning
## ② clustering
## ③ file size
## ④ warehouse size
です。

つまり
scan量を減らす
ことが最重要です。
まとめ
クラスタリング深度
Clustering Depth
```
↓
Partition overlap
↓
Query scan量
深度が低いほど
高速
になります。
主な情報源
Snowflake公式
• Micro-partitions
```
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
• Clustering keys
https://docs.snowflake.com/en/user-guide/tables-clustering-keys
• Clustering information function
https://docs.snowflake.com/en/sql-reference/functions/system_clustering_information

• Snowflake Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
論⽂
• The Snowflake Elastic Data Warehouse
もし興味あればですが、ここまで理解すると Snowflakeの内部設計のほぼ核⼼まで来ています。
さらにその先のテーマとして
Snowflakeの「マイクロパーティションの実装」
があります。
これはかなり⾯⽩くて、
• 実際のパーティションサイズ
• Parquetとの関係
• なぜSnowflakeは速いのか（本当の理由）
• Databricks / Icebergとの違い
まで理解できます。
では最後のテーマとして、**Snowflakeのマイクロパーティションの実装（内部構造）**を解説します。
これは Snowflake を深く理解するうえで⾮常に重要な部分です。
内容は次の順序で説明します。
1. マイクロパーティションの実装イメージ
2. Parquetとの関係
3. Snowflakeが⾼速な本当の理由
4. Databricks / Icebergとの違い
5. マイクロパーティションの⽣成アルゴリズム

公式資料や論⽂も併記します。
1 マイクロパーティションの実装イメージ
Snowflakeのテーブルは、実際には オブジェクトストレージ上のカラムファイル集合です。
Snowflake Table
```
│
▼
Micro-partitions
│
▼
Column segments
│
▼
Compressed column data
│
▼
Object storage
(S3 / Blob / GCS)
つまりSnowflakeのテーブルは
table
├─ partition1
│ ├─ columnA
│ ├─ columnB
│ ├─ columnC
│
├─ partition2
│ ├─ columnA
│ ├─ columnB
│ ├─ columnC
という構造になります。
特徴
```

• 列ごと圧縮
• 列単位スキャン
• メタデータ付き
参考
Snowflake Micro-partitions
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
2 Parquetとの関係
よくある質問ですが
Snowflake内部フォーマットはParquetではありません。
ただし
⾮常に似た構造です。
⽐較
項⽬
Snowflake
Parquet
格納 columnar columnar
圧縮 column compression column compression
metadata partition metadata row group metadata
Snowflakeは
custom columnar format
です。

| 項⽬ |  |  |
| --- | --- | --- |
| Snowflake |  |  |
| Parquet |  |  |
| 格納 | columnar | columnar |
| 圧縮 | column compression | column compression |
| metadata | partition metadata | row group metadata |


理由
Snowflakeは
micro-partition metadata
を⾮常に重視しています。
論⽂でも説明されています。
参考
The Snowflake Elastic Data Warehouse
3 Snowflakeが速い本当の理由
Snowflakeの速度は
次の3つです。
1 Metadata pruning
2 Massive parallel scan
3 Column compression
つまり
Scan量削減
+
並列処理
です。
処理イメージ
SQL
```
│
▼
Query Optimizer
│
▼
Partition pruning
```

```
│
▼
Parallel scan
│
▼
Aggregation
参考
Snowflake Query Optimization
```
https://docs.snowflake.com/en/user-guide/performance-query-optimization
4 Databricks / Icebergとの違い
最近よく⽐較されるのが
• Snowflake
• Databricks
• Iceberg
です。
構造⽐較
Snowflake
Iceberg
Databricks
Storage proprietary open format parquet
Metadata internal manifest files delta log
Execution warehouse spark spark
Snowflake

```
|  |  |  |  |
```
| --- | --- | --- | --- |
| Snowflake |  |  |  |
| Iceberg |  |  |  |
| Databricks |  |  |  |
| Storage | proprietary | open format | parquet |
| Metadata | internal | manifest files | delta log |
| Execution | warehouse | spark | spark |


closed architecture
Iceberg
open lakehouse
です。
Iceberg構造
table
```
├─ manifest
│
├─ parquet file
├─ parquet file
参考
Apache Iceberg
5 マイクロパーティション⽣成アルゴリズム
Snowflakeはロード時に
⾃動パーティション化
します。
例
```sql
COPY INTO orders
FROM @stage/orders.parquet
```
内部
file read
↓
rows grouped
↓
micro-partition created
```

サイズ
50MB〜500MB (uncompressed)
参考
Snowflake table storage
https://docs.snowflake.com/en/user-guide/tables-intro
6 COPYロード時の内部処理
Snowflakeのロード処理
stage file
```
│
▼
reader
│
▼
partition builder
│
▼
micro-partition
最終的に
object storage
に書き込まれます。
7 Snowflakeの実際のパーティション数
例
10TB table
```

の場合
50000〜200000 partitions
になります。
しかし
metadata pruning
により
読むのは
数十 partitions
だけです。
8 Snowflakeの最⼤の特徴
Snowflakeは
ストレージとコンピュートの完全分離
です。
Object storage
```
│
▼
Micro-partitions
│
▼
Virtual Warehouse
│
▼
Parallel execution
この設計により
• 無限スケール
```

• ⾼並列
• コスト分離
が可能になります。
参考
Snowflake Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
まとめ
Snowflake内部構造
Table
```
│
▼
Micro-partitions
│
▼
Columnar compressed data
│
▼
Object storage
クエリ
Metadata pruning
↓
Parallel scan
↓
Aggregation
参考資料
```

Snowflake公式
• Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
• Micro-partitions
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
• Table storage
https://docs.snowflake.com/en/user-guide/tables-intro
• Query optimization
https://docs.snowflake.com/en/user-guide/performance-query-optimization
論⽂
• The Snowflake Elastic Data Warehouse
関連技術
• Apache Iceberg
もしよければですが、ここまで理解すると Snowflakeの設計思想の90%くらいまで来ています。
ただ実はもう⼀つだけ 超重要なテーマがあります。
それは
「Snowflakeがなぜ“データレイクハウス”ではなく“DWH”なのか」
です。
これは
• Snowflake
• Databricks
• Iceberg

の思想の違いに直結します。
もし興味があれば、この3つのアーキテクチャの本質的な違いを図付きで解説します。
（データ基盤設計ではかなり重要です）
では最後に、Snowflake / Databricks / Iceberg のアーキテクチャ思想の違いを整理します。
これはデータ基盤設計では⾮常に重要で、現在の DWH vs Lakehouse 議論の中⼼でもあります。
説明の流れは次の通りです。
1. DWH と Data Lake の歴史
2. Snowflake の思想
3. Lakehouse（Databricks）の思想
4. Iceberg / Delta の役割
5. 実務でのアーキテクチャ選択
参考情報源も併記します。
1 DWHとデータレイクの歴史
もともとデータ基盤は次の2つに分かれていました。
種類
⽬的
DWH BI分析
Data Lake データ保存
典型構造
ETL

| 種類 |  |
| --- | --- |
| ⽬的 |  |
| DWH | BI分析 |
| Data Lake | データ保存 |


```
│
▼
Data Warehouse
(Teradata / Oracle)
問題
高コスト
スケール限界
そこで登場したのが
Data Lake
構造
S3
├─ parquet
├─ json
├─ csv
しかしData Lakeには問題がありました。
schema管理
ACID
query性能
参考
The Snowflake Elastic Data Warehouse
2 Snowflakeの思想（Cloud DWH）
Snowflakeは
Cloud Data Warehouse
として設計されました。
特徴
storage / compute separation
構造
```

Object Storage
```
│
▼
Micro-partitions
│
▼
Virtual Warehouse
Snowflakeは
structured analytics
に最適化されています。
特徴
• SQL
• BI
• analytics
• performance
参考
https://docs.snowflake.com/en/user-guide/intro-key-concepts
3 Databricksの思想（Lakehouse）
Databricksは
Data Lake + Warehouse
を統合する
Lakehouse
という概念を提案しました。
構造
S3
```

```
│
▼
Delta Lake
│
▼
Spark SQL
イメージ
Data Lake
+
ACID
+
SQL
これが
Lakehouse
です。
参考
```
Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores
4 Iceberg / Delta の役割
Lakehouseでは
テーブルフォーマット
が重要です。
例
• Delta Lake
• Iceberg
• Hudi

Iceberg構造
table
```
├─ metadata.json
├─ manifest list
├─ manifest file
└─ parquet files
つまり
data = parquet
metadata = table format
です。
参考
Apache Iceberg
5 Snowflake vs Lakehouse
アーキテクチャ⽐較
Snowflake
Lakehouse
Storage proprietary parquet
Metadata internal iceberg/delta
Engine proprietary spark
Format closed open
Snowflake
managed system
Lakehouse
```

```
|  |  |  |
```
| --- | --- | --- |
| Snowflake |  |  |
| Lakehouse |  |  |
| Storage | proprietary | parquet |
| Metadata | internal | iceberg/delta |
| Engine | proprietary | spark |
| Format | closed | open |


open ecosystem
です。
6 実務での選択
現在の業界は
3つのパターンがあります。
## ① Snowflake中⼼
S3
```
│
▼
Snowflake
│
▼
BI
⽤途
• BI
• analytics
• SQL
```
## ② Lakehouse
S3
```
│
▼
```

Iceberg / Delta
```
│
▼
Spark
⽤途
• ML
• ETL
• data engineering
```
## ③ ハイブリッド
最近多いです。
S3
```
│
├─ Iceberg
│
▼
Databricks
│
▼
Snowflake
役割
システム
役割
Databricks ETL / ML
Snowflake BI
```

| システム |  |
| --- | --- |
| 役割 |  |
| Databricks | ETL / ML |
| Snowflake | BI |


7 Snowflakeの強み
Snowflakeが強い理由
managed
performance
simplicity
つまり
SQL analytics
に最適です。
8 Lakehouseの強み
Lakehouseの強み
open ecosystem
ML
streaming
つまり
data platform
向きです。
まとめ
データ基盤の3つの思想
DWH

Data Lake
Lakehouse
整理すると
Snowflake = Cloud DWH
Databricks = Lakehouse
Iceberg = table format
です。
参考情報
Snowflake
https://docs.snowflake.com/en/user-guide/intro-key-concepts
Snowflake論⽂
The Snowflake Elastic Data Warehouse
Lakehouse論⽂
Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores
Iceberg
Apache Iceberg
もしよければですが、
あなたの質問レベルを⾒ると データ基盤アーキテクト視点の話がかなり向いていそうです。
次のテーマを解説すると理解が⼀気に深まります。
「Snowflakeがなぜ“SQLエンジン”として強いのか（内部実装）」

具体的には
• distributed query engine
• joinアルゴリズム
• broadcast join
• shuffle join
• Snowflake execution plan
このあたりです。
希望あれば続けます。
では続けて、**Snowflakeがなぜ「SQLエンジンとして強いのか」**を、内部の分散クエリエンジンの観点か
ら解説します。
主に次の内容です。
1. Snowflakeの分散クエリエンジン
2. クエリ実⾏計画（Execution Plan）
3. Joinアルゴリズム
4. Broadcast Join と Shuffle Join
5. Snowflakeのパイプライン実⾏
参考資料として、公式ドキュメントや研究論⽂も併記します。
1 Snowflakeの分散クエリエンジン
Snowflakeは MPP (Massively Parallel Processing) 型のクエリエンジンです。
Query
```
│
▼
Query Optimizer
│
▼
Execution Plan
```

```
│
▼
Distributed Execution
│
├─ Node1
├─ Node2
├─ Node3
└─ Node4
Virtual Warehouseは内部的に
cluster
├─ compute node
├─ compute node
├─ compute node
で構成されています。
各ノードは
• マイクロパーティションをスキャン
• ローカル処理
• 結果を集約
します。
参考
Snowflake Warehouses
```
https://docs.snowflake.com/en/user-guide/warehouses-overview
2 クエリ実⾏計画
SQLを実⾏すると
SQL
```
↓
Parser
↓
```

Logical Plan
```
↓
Physical Plan
↓
Execution
になります。
例
```sql
SELECT *
FROM orders o
JOIN customers c
ON o.customer_id = c.id
```
Optimizerは
join order
scan strategy
partition pruning
を決定します。
参考
Snowflake Query Optimization
```
https://docs.snowflake.com/en/user-guide/performance-query-optimization
3 SnowflakeのJoinアルゴリズム
Snowflakeでは主に3種類のJoinがあります。

Join
特徴
Hash Join ⼀般的
Broadcast Join ⼩さいテーブル
Merge Join ソート済み
Hash Join
⼀般的なJoinです。
Table A
Table B
処理
hash table build
```
↓
probe
イメージ
Build side
└─ hash table
Probe side
└─ scan
4 Broadcast Join
```

| Join |  |
| --- | --- |
| 特徴 |  |
| Hash Join | ⼀般的 |
| Broadcast Join | ⼩さいテーブル |
| Merge Join | ソート済み |


⼩さいテーブルの場合
small table
を
全ノードにコピー
します。
Node1 ─┐
Node2 ─┼─ small table copy
Node3 ─┘
その後
local join
します。
メリット
shuffle不要
⾼速です。
5 Shuffle Join
⼤きいテーブル同⼠の場合
partition redistribution
が必要です。
Table A partition
Table B partition
を
hash(key)
で分配します。
イメージ

Node1 ──┐
Node2 ──┼─ shuffle
Node3 ──┘
その後
local join
します。
6 Snowflakeのパイプライン実⾏
Snowflakeは
pipeline execution
を使います。
つまり
scan
join
aggregation
を
同時並行
で実⾏します。
イメージ
scan partitions
```
↓
partial join
↓
partial aggregation
結果
memory usage ↓
```

latency ↓
になります。
参考
The Snowflake Elastic Data Warehouse
7 Snowflake Execution Pipeline
実際の処理
partition scan
```
│
▼
filter
│
▼
join
│
▼
aggregate
│
▼
result
各処理は
distributed operator
です。
8 Result Aggregation
最終結果は
```

partial results
を
final node
が集約します。
node1 partial
node2 partial
node3 partial
```
↓
final result
9 Snowflakeが速い理由（SQLエンジン）
SnowflakeのSQL性能は
次の3つです。
1 partition pruning
2 distributed execution
3 pipeline processing
つまり
scan最小化
+
並列処理
です。
まとめ
```

Snowflake SQLエンジン
SQL
```
│
▼
Optimizer
│
▼
Distributed Execution
│
├─ scan partitions
├─ join
├─ aggregate
実⾏モデル
MPP
+
pipeline
です。
主な情報源
Snowflake公式
• Warehouses
```
https://docs.snowflake.com/en/user-guide/warehouses-overview
• Query Optimization
https://docs.snowflake.com/en/user-guide/performance-query-optimization
• Query Processing
https://docs.snowflake.com/en/user-guide/querying-overview
研究論⽂
• The Snowflake Elastic Data Warehouse

もしよければですが、ここまで理解すると Snowflakeアーキテクチャのほぼ全体です。
ただ、実務ではさらに重要なテーマがあります。
それは
「Snowflakeで遅いクエリが発⽣する本当の理由」
です。
実際にはほとんどの場合
• partition pruningが効いていない
• join戦略ミス
• clustering崩壊
が原因です。
希望あれば Snowflakeパフォーマンスチューニング完全版を解説します。
（実務で⼀番役に⽴つ内容です）
では Snowflake パフォーマンスチューニング完全版をまとめます。
実務で Snowflake の遅いクエリの原因の 90%はこの5つです。
1. Partition pruning
2. クラスタリング設計
3. Join戦略
4. ファイルサイズ / ロード⽅法
5. Warehouse設計
Snowflake公式資料や論⽂を根拠に整理します。

1 Partition Pruning（最重要）
Snowflakeの性能は基本的に
Scan量
で決まります。
Snowflakeは
micro-partition metadata
を使って
不要パーティションを読み飛ばす
仕組みです。
イメージ
partition1 2024-01-01〜2024-01-05
partition2 2024-01-06〜2024-01-10
partition3 2024-01-11〜2024-01-15
クエリ
```sql
SELECT *
FROM orders
WHERE order_date = '2024-01-07'
```
結果
partition2のみ読む
これが
Micro-partition pruning
です。
参考
Snowflake Micro-partitions
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions

2 Pruningが効かない典型例
NG① 関数
WHERE YEAR(order_date) = 2024
これは
metadata min/max
が使えません。
良い例
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
NG② LIKE
WHERE email LIKE '%gmail.com'
prefix検索以外は
full scan
になります。
NG③ CAST
WHERE CAST(order_date AS STRING)
これも pruning 不可。

3 クラスタリング設計
クラスタリングとは
似た値を同じmicro-partitionに置く
ことです。
悪い例
partition1
2024-01
2024-07
2024-03
良い例
partition1 2024-01-01〜2024-01-10
partition2 2024-01-11〜2024-01-20
クラスタリングキー
CLUSTER BY(order_date)
参考
Snowflake Clustering Keys
https://docs.snowflake.com/en/user-guide/tables-clustering-keys
4 Join戦略チューニング
Snowflake Joinは主に

Join
⽤途
Broadcast join ⼩さいテーブル
Shuffle join ⼤きいテーブル
Broadcast join
small table
```
↓
全ノードコピー
⾼速です。
例
fact_orders
join
dim_customer
dim_customerが⼩さい場合
broadcast join
になります。
参考
Snowflake Query Optimization
```
https://docs.snowflake.com/en/user-guide/performance-query-optimization
5 Join順序
Optimizerは
small table first
を選びます。

| Join |  |
| --- | --- |
| ⽤途 |  |
| Broadcast join | ⼩さいテーブル |
| Shuffle join | ⼤きいテーブル |


しかし統計が悪いと
join explosion
が起きます。
例
fact_orders
join
fact_events
これはかなり重いです。
6 ファイルサイズ
Snowflakeでは
```sql
COPY INTO
```
のファイルサイズが重要です。
理想
100MB 前後
悪い例
1MB files
これを
small files problem
と⾔います。
理由
micro-partition爆増
metadata増加
参考
Snowflake Data Loading Best Practices

https://docs.snowflake.com/en/user-guide/data-load-considerations
7 MERGEの落とし⽳
Snowflakeは
copy-on-write
です。
つまり
UPDATE
MERGE
は
新micro-partition作成
になります。
例
10TB table
MERGE
すると
大量パーティション再生成
になります。
8 Warehouseチューニング
Warehouseサイズ

Size
Nodes
XS 1
S 2
M 4
L 8
Warehouseを⼤きくすると
parallel scan
が増えます。
しかし
scan量が大きい
と
warehouseを大きくしても遅い
です。
9 Result Cache
Snowflakeは
result cache
を持っています。
同じクエリなら
0秒
で返ります。

| Size |  |
| --- | --- |
| Nodes |  |
| XS | 1 |
| S | 2 |
| M | 4 |
| L | 8 |


条件
query identical
data unchanged
参考
Snowflake Result Cache
https://docs.snowflake.com/en/user-guide/querying-persisted-results
10 Query Profileの使い⽅
Snowflakeの調査は
Query Profile
を⾒ます。
⾒るポイント
scan bytes
partitions scanned
join type
例
Table Scan
Partitions scanned: 12000
これは
pruning failure
です。
11 Snowflakeチューニング優先順位

Snowflakeエンジニアの定⽯
## ① Partition pruning
## ② Clustering
## ③ Join戦略
## ④ File size
## ⑤ Warehouse
です。
つまり
scan量削減
が最重要です。
まとめ
Snowflakeチューニングの本質
scan data
```
↓
reduce scan
↓
parallelize scan
です。
主な情報源
Snowflake公式
```

Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
Micro-partitions
https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
Query optimization
https://docs.snowflake.com/en/user-guide/performance-query-optimization
Clustering
https://docs.snowflake.com/en/user-guide/tables-clustering-keys
Data loading
https://docs.snowflake.com/en/user-guide/data-load-considerations
Result cache
https://docs.snowflake.com/en/user-guide/querying-persisted-results
研究論⽂
• The Snowflake Elastic Data Warehouse
もしよければですが、ここまで理解すると Snowflakeアーキテクトレベルの知識です。
次に解説するとさらに理解が深まるのが
「Snowflake内部のメタデータ管理アーキテクチャ」
です。
これは普通のエンジニアは知らない内容で
• catalog設計
• transaction管理
• snapshot isolation
• metadata scaling

に関係します。
かなり⾯⽩いテーマなので、希望あれば解説します。
では最後に、Snowflake内部のメタデータ管理アーキテクチャを解説します。
これはSnowflakeのスケーラビリティやトランザクションの仕組みを理解するうえで重要です。
内容は次の流れで説明します。
1. Snowflakeのメタデータとは
2. メタデータサービスの役割
3. スナップショット分離（Snapshot Isolation）
4. Time Travel と Fail-safe
5. メタデータスケーリング
主にSnowflake公式資料と論⽂を基にまとめています。
1 Snowflakeのメタデータとは
Snowflakeでは、データそのものはクラウドストレージに置かれますが、
データを管理する情報（メタデータ）はCloud Services層が管理します。
Snowflake Architecture
Client
```
│
▼
Cloud Services
├─ Metadata
├─ Query optimizer
├─ Transaction manager
│
▼
Compute (Virtual Warehouse)
```

```
│
▼
Storage (Object Storage)
メタデータの例
• テーブルスキーマ
• マイクロパーティション情報
• クエリ履歴
• アクセス制御
• 統計情報
参考
Snowflake Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
2 メタデータサービスの役割
SnowflakeのCloud Services層には
Metadata Services
があります。
主な役割
table catalog
schema management
query planning
access control
transaction management
イメージ
Metadata Catalog
├─ database
│ ├─ schema
│ │ ├─ table
```

```
│ │ │ ├─ micro-partitions
│ │ │ └─ statistics
このカタログ情報があるため
• クエリ計画
• パーティションプルーニング
• トランザクション管理
が可能になります。
参考
Snowflake Metadata
https://docs.snowflake.com/en/user-guide/intro-key-concepts
3 Snapshot Isolation（スナップショット分離）
Snowflakeのトランザクションは
Snapshot Isolation
で動作します。
概念
transaction start
↓
snapshot created
↓
query sees consistent data
例
ユーザーA
```sql
SELECT * FROM orders;
```
ユーザーB
```

```sql
UPDATE orders SET price = 100;
```
この場合
User A → 旧データ
User B → 新データ
をそれぞれ⾒ます。
これは
copy-on-write
⽅式で実現されています。
参考
Snowflake Transactions
https://docs.snowflake.com/en/user-guide/transactions
4 Time Travel
Snowflakeの特徴的な機能の⼀つです。
```sql
SELECT *
FROM orders
AT (TIMESTAMP => '2026-03-10 10:00');
```
内部では
old micro-partition
new micro-partition
が保存されています。
Time Travel期間

エディション
保持
Standard 1⽇
Enterprise 最⼤90⽇
参考
Snowflake Time Travel
https://docs.snowflake.com/en/user-guide/data-time-travel
5 Fail-safe
Time Travelの後に
Fail-safe
という期間があります。
⽬的
災害復旧
です。
期間
7日
この間は
Snowflakeサポートのみ復旧可能
です。
参考
Snowflake Fail-safe
https://docs.snowflake.com/en/user-guide/data-failsafe

| エディション |  |
| --- | --- |
| 保持 |  |
| Standard | 1⽇ |
| Enterprise | 最⼤90⽇ |


6 メタデータスケーリング
Snowflakeは⼤量のテーブル・パーティションを管理します。
例
10TB table
```
↓
50000+ micro-partitions
そのため
metadata distributed architecture
を採⽤しています。
イメージ
Metadata Service
├─ partition catalog
├─ statistics store
├─ query history
これにより
millions of partitions
を管理できます。
論⽂でも説明されています。
参考
The Snowflake Elastic Data Warehouse
7 Snowflakeメタデータの重要性
```

Snowflakeでは
metadata = performance
です。
理由
partition pruning
query planning
statistics
すべてメタデータに依存します。
まとめ
Snowflakeの内部管理構造
Object Storage
```
│
▼
Micro-partitions
│
▼
Metadata Catalog
│
▼
Cloud Services
そして
Snapshot isolation
+
copy-on-write
により
• Time Travel
• ⾼並⾏トランザクション
• 分散クエリ
```

が実現されています。
主な情報源
Snowflake公式
Architecture
https://docs.snowflake.com/en/user-guide/intro-key-concepts
Transactions
https://docs.snowflake.com/en/user-guide/transactions
Time Travel
https://docs.snowflake.com/en/user-guide/data-time-travel
Fail-safe
https://docs.snowflake.com/en/user-guide/data-failsafe
研究論⽂
• The Snowflake Elastic Data Warehouse
もし興味があれば、次に
「Snowflake内部のクエリオプティマイザ」
を解説できます。
これは実際のデータ基盤設計でかなり重要で、
• 統計情報
• join reorder

• cost model
などが関係します。
---------------------------------------
はい。結論から⾔うと、Snowflakeを「業務アプリのデータストア」として使うなら、まずそのアプリが OLTP
型なのか、分析寄りなのかを切り分けるのが最重要です。
Snowflake の通常テーブルは分析基盤向けで、近年は Hybrid Tables により低遅延・⾼スループットの
ランダム read/write、⾏ロック、主キー/外部キー制約といったトランザクション寄りの要件にも対応してい
ます。ただし、Hybrid Tables には機能差やクォータがあり、⼀般的なRDBをそのまま置き換える前提で
は評価が必要です。
考慮点は、実務では次の順番で整理すると失敗しにくいです。
1. まず「どの種類の処理」を載せるのか
• 集計、検索、レポート、分析API、ダッシュボードが中⼼なら、Snowflakeはかなり相性がいいです。
• ⼀⽅で、⾼頻度な単票更新、厳しいミリ秒応答、複雑なトランザクション制御、強いOLTP要件が
主役なら、通常テーブル前提では厳しく、Hybrid Tables 前提での検証が必要です。Hybrid
Tables はトランザクション⽤途向けですが、標準テーブルとは別物として設計を考えるべきです。
2. テーブル種別を最初に決める
Snowflake には少なくとも次の考え分けがあります。
• 標準テーブル: 分析・ELT・集計向け
• Hybrid Tables: 低遅延の point read/write と⾼並⾏な業務処理向け
• Interactive tables / interactive warehouses: ⾼並⾏・低遅延の対話型クエリ向け。ただし利
⽤可能リージョンやSQL上の制約を確認すべきです。
なので、アプリの中でも

• 台帳・注⽂・ステータス更新 → Hybrid Tables 候補
• 参照系・分析系 → 標準テーブルや interactive 系候補
のように、⽤途ごとに使い分ける設計が⾃然です。
3. レイテンシ要件を⽢く⾒ない
Snowflakeは「速い」ですが、何に対して速いかが重要です。
Hybrid Tables ですら、まずは実際のドライバ経由の負荷試験で確認する必要があります。Snowflake
公式でも、Snowsight 上の表⽰値は driver-based workload の性能評価には向かないと明記して
います。つまり、画⾯で速く⾒えても、業務アプリの本番SDK/ドライバ経由では別です。
4. 同時実⾏数と更新パターンを確認する
Hybrid Tables には既定で、1データベースあたり約8,000 ops/sec の balanced workload ⽬安や、ハ
イブリッドストレージ 2TB/DB などのクォータがあります。⼤量同時更新や急激なスパイクがある業務で
は、この制約が設計判断に直結します。
このため、次を事前に数値化した⽅がいいです。
• 秒間 read / write 件数
• 同時接続数
• ピーク時間帯
• 1件更新か、まとめ更新か
• 参照:更新の⽐率
これが曖昧なまま選ぶと、後で「使えなくはないが苦しい」状態になりやすいです。これは上記クォータ
と Snowflake の推奨評価⼿順からの実務的な帰結です。
5. 制約・SQL互換性・機能差を確認する
Hybrid Tables は魅⼒的ですが、制限事項や未対応機能があります。Snowflake ⾃⾝が limitations
ページを分けているので、採⽤前提なら必読です。特に、既存アプリが PostgreSQL / MySQL / Oracle
前提で書かれていると、SQLや機能の差分確認が必要です。

6. コストモデルを理解する
Snowflakeは⼀般的なDBのように「常時1台」ではなく、ストレージとコンピュートを分離して考えます。
Hybrid Tables には専⽤のストレージ課⾦もあり、クエリや処理のためのコンピュートも別で考える必要が
あります。低遅延要件を満たすために常時稼働寄りにすると、想像よりコストが伸びることがあります。
特に業務アプリ⽤途では、
• 24時間アクセスがあるか
• 夜間停⽌できるか
• バッチ中⼼か常時オンラインか
でコスト感が⼤きく変わります。これは Snowflake の分離アーキテクチャからくる本質です。
7. セキュリティ/権限制御を最初に整理する
Snowflakeはロールベースの権限管理が中⼼です。テーブル、ビュー、スキーマ、DB、ウェアハウスごとに権限
を整理し、アプリ⽤ロールと管理⽤ロールを分けるのが基本です。Hybrid Tables も権限付与は通常の
TABLE として扱います。
実務では、
• アプリ実⾏ロール
• ETL/バッチロール
• 開発者参照ロール
• 運⽤管理ロール
を分離し、本番データへの直接更新権限を絞る設計が重要です。これは Snowflake の権限モデル
に沿った王道です。
8. 運⽤監視をどうするか
Hybrid Tables では、SHOW TRANSACTIONS、SHOW LOCKS、LOCK WAIT HISTORY などでトラ
ンザクションやロック待ちを監視できます。つまり、アプリ⽤データストアとして使うなら、単にSQLを打つだけ
でなく、ロック競合や待ち時間を⾒る運⽤設計が必要です。

9. 「Snowflakeだけで全部やる」かを⾒極める
実務では、全部Snowflakeに寄せるより、次のような分離がよく合います。
• SoR / 厳密OLTP は別DB
• 分析・統合・配信・参照強化 は Snowflake
• もしくは 更新系の⼀部だけ Hybrid Tables
Snowflake 公式も、Hybrid Tables を「トランザクションと分析を⼀つのプラットフォームで扱う」ため
の選択肢として位置づけていますが、制約つきなので、全⾯置換より適材適所の混在設計が安全
です。
あなたが最初に決めるべきチェック項⽬
これを埋めると判断しやすいです。
• 1リクエストの許容応答時間は何msか
• 1秒あたりの read/write 件数はどれくらいか
• 更新は単⼀⾏中⼼か、複数表トランザクションか
• 業務の正本DBにしたいのか、参照/統合ストアにしたいのか
• データ量はどれくらいか
• 24時間オンラインか、営業時間中⼼か
• 監査/権限制御/暗号化の要求はどの程度か
• 既存アプリはどのDB前提で作られているか
これらは、Hybrid Tables の性能特性・制約・権限・コスト構造に直結します。
かなり実務的な結論
Snowflakeを「業務アプリのデータストア」にすること⾃体は可能ですが、
• 分析・参照・統合基盤として使うのか

• 更新系の本番DBとして使うのか
で難易度が⼤きく違います。
更新系の本番DBとして使うなら、Hybrid Tables 前提で、性能試験・制限確認・コスト試算・ロック監
視設計までやってから決めるのが安全です。
⼀⽅、参照系や分析系が中⼼なら、Snowflakeはかなり強い候補です。
あなたの想定が
「業務アプリの正本DB」 なのか、
「分析も兼ねた参照⽤ストア」 なのかで、設計の勘所がかなり変わります。
その前提に合わせて、次は メリデメを業務アプリ⽬線で具体化 して整理できます。
Snowflake へのデータ登録（取り込み）は、実務では ⼤きく6パターンに分かれます。
⽤途（リアルタイム / バッチ / 外部連携 / データレイク）で選びます。
1. COPY INTO（最も⼀般的：バッチ）
⼀番よく使われる⽅法です。
S3 / Azure Blob / GCS に置いたファイルを Snowflake にロードします。
S3 → Snowflake
```sql
COPY INTO my_table
FROM @my_stage/data/
FILE_FORMAT = (TYPE = CSV);
```
特徴
• ⼤量データに強い
• Snowflake公式の王道
• データレイク連携と相性良い

よくある構成
外部システム
```
↓
S3
↓
Snowflake Stage
↓
```sql
COPY INTO
```
↓
Table
AWSデータレイク構成ではほぼこの⽅式です。
2. Snowpipe（⾃動取り込み）
ファイル到着をトリガーに⾃動ロードします。
S3 → Snowpipe → Snowflake
構成
S3
↓ (イベント通知)
SNS / SQS
↓
Snowpipe
↓
Table
特徴
```

• ⾃動ロード
• ほぼリアルタイム
• バッチ不要
典型⽤途
• データレイク
• ログ収集
• IoT
• 外部システムファイル連携
3. Snowpipe Streaming（リアルタイム）
Snowflakeの最新のリアルタイム取り込み機能
アプリ → Snowpipe Streaming → Snowflake
特徴
• 秒単位でデータ反映
• Kafka不要
• APIベース
典型⽤途
• イベントデータ

• トレードログ
• センサー
• アプリログ
4. INSERT / MERGE（アプリ連携）
普通のSQLでデータ登録します。
```sql
INSERT INTO table VALUES (1,'abc');
```
または
```sql
MERGE INTO target t
USING source s
ON t.id = s.id
WHEN MATCHED THEN UPDATE
WHEN NOT MATCHED THEN INSERT
```
特徴
• RDBと同じ
• OLTP的
• 少量データ向き
注意
Snowflakeは OLTP⽤途には最適ではない
（Hybrid Tablesを除く）

5. コネクタ / ETLツール
外部ツールから直接ロードする⽅法。
例
• Fivetran
• Airbyte
• dbt
• Informatica
• Matillion
• Talend
典型構成
RDB
```
↓
ETL
↓
Snowflake
最近は ELT（Snowflake側で変換） が主流。
6. Kafka / Streaming連携
ストリームデータの取り込み。
Kafka → Snowflake Connector
または
```

Kafka → Snowpipe Streaming
特徴
リアルタイムデータ基盤
実務で⼀番多い構成
ほとんどの企業はこの形です。
業務システム
```
↓
データレイク(S3)
↓
Snowpipe / COPY INTO
↓
Snowflake
↓
BI / 分析
理由
• データレイクを中⼼にできる
• 再処理可能
• 安全
• スケールする
AWS × Snowflake の典型アーキテクチャ
```

外部システム
```
↓
S3
↓
EventBridge
↓
Snowpipe
↓
Snowflake
↓
BI / アプリ
この構成が モダンデータアーキテクチャ です。
取り込み⽅法の選び⽅
⽅法
向いている⽤途
```sql
COPY INTO ⼤量バッチ
```
Snowpipe ⾃動ファイル連携
Snowpipe Streaming リアルタイム
INSERT 少量更新
ETLツール SaaS / DB連携
Kafka ストリーム
Snowflake設計で重要なポイント
```

| ⽅法 |  |
| --- | --- |
| 向いている⽤途 |  |
| COPY INTO | ⼤量バッチ |
| Snowpipe | ⾃動ファイル連携 |
| Snowpipe Streaming | リアルタイム |
| INSERT | 少量更新 |
| ETLツール | SaaS / DB連携 |
| Kafka | ストリーム |


Snowflakeでは
データ登録⽅法 = アーキテクチャ
になります。
最初に決めるべきこと
• リアルタイムか
• バッチか
• データレイク併⽤か
• CDCが必要か
もしよければ教えてください。
かなり重要なので聞きたいです。
1. Snowflakeは
• データレイクの分析基盤？
• 業務アプリのDB？
2.
3. データの元
• RDB
• 外部API
• ファイル
• SaaS
4.
5. 更新頻度
• リアルタイム
• 数分
• ⽇次
6.
これが分かると、AWS + Snowflake の最適アーキテクチャをかなり具体的に提案できます。

（実務でよくある構成を図付きで説明できます。）
131
