# ADR-001: ELT 変換層の技術選定（dbt on Snowflake + Step Functions）

| 項目 | 内容 |
|------|------|
| ステータス | **承認済み** |
| 作成日 | 2026-06-25 |
| 決定者 | （記入） |
| レビュー者 | （記入） |

---

## コンテキスト

クラウドデータ基盤として Snowflake を採用する前提で、ELT アーキテクチャのデータパイプライン実装方針を定める。本 ADR では以下の 3 点を検討順に決定する。

1. ELT の T（Transform）層をどの手法で実装するか
2. dbt を採用する場合、どの実行形態（Cloud / Core / on Snowflake）を使用するか
3. dbt の定期実行をどのオーケストレーション基盤で制御するか

現在の状況は以下の通り。

- Snowflake を DWH 基盤として本プロジェクトで採用する
- AWS 上で ECS Fargate + EIP 固定 + Secrets Manager 構成を本案件向けに構築する前提であり、当該構成の初期構築負荷も評価対象とする
- 組織に専任の DBA やインフラ運用チームは存在しない（約 200 名規模）
- 運用にかけられるリソース・コストは限定的である
- 将来は全社的なデータ取り込みを見据えており、拡張後も耐えうるアーキテクチャが求められる

---

## 決定 1：ELT 変換層の手法選定 — dbt vs Stored Procedures

### 検討した選択肢

| 選択肢 | 概要 |
|--------|------|
| A. dbt（Data Build Tool） | SQL ベースの変換フレームワーク。モデル間の依存関係を自動解決し、テスト・ドキュメント生成を統合的に提供する |
| B. Snowflake Stored Procedures（SQL 手書き） | Snowflake ネイティブの Stored Procedure で変換ロジックを個別に実装する |

### 評価と比較

| 観点 | A. dbt | B. Stored Procedures |
|------|--------|---------------------|
| 依存関係管理 | `ref()` によるモデル間依存の自動解決。DAG として可視化される | 手動管理。呼び出し順序をスクリプトやドキュメントで管理する必要がある |
| テスト | `not_null`, `unique`, `relationships` 等の標準テストに加え、カスタムテストを YAML で宣言的に定義できる | テスト用 Procedure を個別に作成する必要がある |
| ドキュメント | `dbt docs generate` でデータカタログ・リネージ図を自動生成 | 手動でドキュメントを作成・維持する必要がある |
| コードの再利用 | マクロ・パッケージ（dbt_utils 等）による共通処理の再利用が容易 | 共通処理は Procedure や UDF として個別に実装 |
| バージョン管理 | Git ベースの管理が前提。PR レビュー、差分確認が容易 | Git 管理は可能だが、デプロイの仕組みは自前で構築が必要 |
| 学習コスト | dbt 固有の概念（model, source, ref, macro）の学習が必要 | SQL の知識があれば即座に着手可能 |
| 市場の人材・知見 | データエンジニアリング領域で広く普及。ドキュメント・コミュニティが充実 | Snowflake 固有。Procedure に精通したエンジニアは限定的 |
| スケーラビリティ | モデル数が増えても依存関係管理が破綻しにくい | モデル数が増えると Procedure 間の依存管理が複雑化する |

### 決定

**選択肢 A：dbt を採用する。**

### 根拠

1. **保守性**
   DBA 不在の組織において、変換ロジックが増加した際に Stored Procedure の依存関係を手動管理し続けることは現実的ではない。dbt の `ref()` による自動依存解決はこの問題を構造的に解消する。

2. **品質保証**
   dbt のテスト機能により、データ品質チェックを変換定義と一体で管理できる。Stored Procedure の場合、テスト用コードを別途開発・維持する必要があり、テストが形骸化するリスクが高い。

3. **ドキュメントの自動生成**
   `dbt docs generate` によるリネージ図・データカタログの自動生成は、DBA 不在の組織においてデータの可視性を維持するために不可欠である。

4. **市場の標準性**
   dbt はデータエンジニアリング領域で広く採用されており、将来のチーム拡大時に人材の確保や知見の共有が容易である。

---

## 決定 2：dbt 実行形態の選定 — dbt Cloud vs dbt Core vs dbt on Snowflake

### 検討した選択肢

| 選択肢 | 概要 |
|--------|------|
| A. dbt Cloud | dbt Labs が提供する SaaS。スケジューラ・IDE・CI/CD・ドキュメントホスティングを含む統合環境 |
| B. dbt Core（セルフホスト） | OSS 版の dbt CLI を自前のコンピュート環境（ECS Fargate 等）で実行する方式 |
| C. dbt on Snowflake | dbt プロジェクトを Snowflake 上にデプロイし、Snowflake のコンピュートリソースで直接実行する方式。`EXECUTE DBT PROJECT` コマンドで実行する |

### 評価と比較

| 観点 | A. dbt Cloud | B. dbt Core（セルフホスト） | C. dbt on Snowflake |
|------|-------------|--------------------------|---------------------|
| コンピュート | dbt Labs 管理のインフラ | 自前の ECS Fargate 等 | Snowflake Warehouse |
| データの経路 | Snowflake → dbt Cloud（外部 SaaS）→ Snowflake | 自前環境 → Snowflake | Snowflake 内部完結 |
| ライセンスコスト | 有償（ユーザー数課金、Team $100/月〜） | 無償（OSS） | Snowflake 利用料に含まれる |
| 運用負荷 | 低（SaaS にお任せ） | 高（コンテナ管理、CLI バージョン管理、profiles.yml 管理） | 低（Snowflake に統合） |
| スケジューラ | 内蔵 | なし（外部オーケストレーション必要） | なし（外部オーケストレーション必要） |
| IDE | Cloud IDE 内蔵 | ローカル IDE + dbt CLI | Snowsight 上の Notebook / ローカル IDE |
| Git 統合 | dbt Cloud 経由 | 自前 CI/CD | Snowflake Git Integration |
| ネットワーク | dbt Cloud → Snowflake への接続設計が必要 | 自前 VPC → Snowflake（PrivateLink 等） | Snowflake 内部通信のみ |

### セキュリティ比較（データ経路）

```
A. dbt Cloud:
  Snowflake ←→ dbt Cloud（外部 SaaS）
  → メタデータ・クエリ結果が外部 SaaS を経由する

B. dbt Core:
  Snowflake ←→ ECS Fargate（自社 AWS 環境）
  → データは自社管理環境内に留まるが、コンピュート管理が必要

C. dbt on Snowflake:
  Snowflake 内部完結
  → データもコンピュートも Snowflake 内に閉じる
```

### 決定

**選択肢 C：dbt on Snowflake を採用する。**

### 根拠

1. **データ経路のセキュリティ**
   dbt Cloud を使用した場合、Snowflake 上のメタデータやクエリ結果が外部 SaaS を経由することになる。顧客のセキュリティ要件上、データが外部 SaaS を通過することへの懸念があり、dbt Cloud の採用は困難である。

2. **アーキテクチャのシンプルさ**
   dbt Core をセルフホストする場合、ECS Fargate 上に dbt CLI を含むコンテナを構築・管理する必要があり、以下の運用負荷が発生する。
   - dbt Core のバージョン管理とコンテナイメージの更新
   - `profiles.yml` や接続設定の管理
   - dbt プロジェクトコードのコンテナへの配布
   - Python 環境・依存パッケージの維持

   dbt on Snowflake であれば、dbt プロジェクトは Git Integration で Snowflake 上にデプロイされ、実行も Snowflake のコンピュートリソースで完結する。AWS 側のコンテナは「キック役」に徹するため、構成がシンプルになる。

3. **コスト**
   dbt Cloud のライセンスコスト（ユーザー数課金）が不要である。dbt on Snowflake の実行コストは既存の Snowflake Warehouse 消費に含まれるため、追加の固定費は発生しない。

### 不採用理由の詳細

**dbt Cloud を不採用とした理由：**

- データが外部 SaaS を経由することに対する顧客のセキュリティ懸念
- ユーザー数課金によるライセンスコストの継続的な発生
- dbt Cloud → Snowflake 間のネットワーク設計（IP 許可等）が追加で必要になる

**dbt Core（セルフホスト）を不採用とした理由：**

- dbt CLI・Python 環境を含むコンテナの構築と継続的なメンテナンスが必要
- `profiles.yml` や dbt プロジェクトコードをコンテナに配布する仕組みの構築が必要
- dbt on Snowflake であれば Snowflake 内部で完結し、AWS 側のコンテナは SQL を 1 行発行するだけで済む

---

## 決定 3：オーケストレーション方式の選定 — Step Functions vs Airflow vs Snowflake Tasks

### 検討した選択肢

| 選択肢 | 概要 |
|--------|------|
| A. AWS Step Functions + ECS RunTask | AWS マネージドのサーバーレスオーケストレーションサービスを使用し、ECS Fargate タスクとして dbt 実行用の Python コンテナを起動する方式 |
| B. Apache Airflow（Amazon MWAA） | OSS のワークフローエンジンである Apache Airflow の AWS マネージドサービス（MWAA）を使用し、DAG として dbt 実行ワークフローを定義する方式 |
| C. Snowflake Tasks 単体 | Snowflake ネイティブの Task スケジューラを使用し、AWS 側のオーケストレーションを介さずに dbt を定期実行する方式 |

### 評価と比較

#### 運用コスト

| 観点 | A. Step Functions | B. Airflow (MWAA) | C. Snowflake Tasks |
|------|:-:|:-:|:-:|
| 固定費 | なし（従量課金のみ） | $300〜$500+/月（常駐インフラ） | なし（Warehouse 消費のみ） |
| インフラ管理 | 不要 | Scheduler / Worker / DB の管理が必要 | 不要 |
| バージョン管理 | 不要 | MWAA のバージョン追従が必要 | 不要 |
| 障害点 | 少ない | Airflow 自体が障害点になる | 少ない |

#### 開発・学習コスト

| 観点 | A. Step Functions | B. Airflow (MWAA) | C. Snowflake Tasks |
|------|:-:|:-:|:-:|
| 学習曲線 | 低（JSON ベースの定義） | 高（DAG, Executor, Provider 等） | 低（SQL ベース） |
| dbt 統合 | 手動（SQL 発行で実行） | cosmos で DAG 自動生成可 | 手動（SQL で実行） |
| AWS サービス統合 | ネイティブ統合 | Provider 経由 | なし |

#### セキュリティ・ネットワーク

| 観点 | A. Step Functions | B. Airflow (MWAA) | C. Snowflake Tasks |
|------|:-:|:-:|:-:|
| EIP 固定（Network Policy） | ECS Fargate ENI で対応済み | MWAA 用 VPC + NAT 設計が追加で必要 | 不要（Snowflake 内部） |
| 認証情報管理 | Secrets Manager + メモリ内処理 | Airflow Connection に保存（暗号化） | Snowflake 内部 |
| IAM 統合 | ネイティブ | MWAA 実行ロール設計が必要 | なし |

#### 拡張性

| 観点 | A. Step Functions | B. Airflow (MWAA) | C. Snowflake Tasks |
|------|:-:|:-:|:-:|
| 多様な配信先対応 | Lambda で個別実装 | Provider で統一的に対応可 | Snowflake 外への配信は困難 |
| 将来の規模拡大 | Map ステートで並列化可 | Worker スケールが必要 | Task DAG で対応 |

#### 実装方針との整合と構築負荷

| 観点 | A. Step Functions | B. Airflow (MWAA) | C. Snowflake Tasks |
|------|:-:|:-:|:-:|
| 本案件向け AWS 構成の構築負荷 | ECS + EIP + Secrets Manager の初期構築が必要（中） | ECS 構成に加えて MWAA 基盤（VPC / NAT / 環境）の追加構築が必要（高） | AWS 側構成は不要（低） |
| 段階的構築 | 本案件向けに構築する ECS 構成を基点に最小差分で実現 | 大幅な追加構築が必要 | AWS オーケストレーションが不要になる |

### 決定

**選択肢 A：AWS Step Functions + ECS RunTask を採用する。**

### 根拠

1. **運用リソースの制約**
   専任のインフラ運用チームが存在しないため、Airflow のような常駐インフラの管理は現実的ではない。Step Functions はサーバーレスであり、管理対象が最小限に抑えられる。

2. **固定費ゼロ**
   MWAA は最小構成でも月額 $300〜$500 の固定費が発生する。Step Functions は実行回数に応じた従量課金のみであり、固定費を抑えつつ運用できる。

3. **実装方針との整合と構築負荷**
   ECS Fargate + EIP 固定 + Secrets Manager + Network Policy の構成は本案件向けに初期構築が必要である。Step Functions はこの構成に最小差分で統合できるため、追加の実装負荷を抑えられる。一方で Airflow を選択した場合、VPC 設計の見直しや MWAA 用インフラの追加構築が必要となり、総構築負荷が増える。

4. **現時点の要件に対する適合性**
   現在の要件は「dbt プロジェクトを定時実行し、結果を通知する」ことである。この要件に対して Airflow はオーバースペックであり、Step Functions で十分に実現できる。

### 不採用理由の詳細

**Airflow (MWAA) を不採用とした理由：**

- 常駐インフラの運用負荷を許容できない
- 月額 $300〜$500+ の固定費が運用制約に対して過大
- 学習コストが高く、チームへの展開に時間がかかる
- 現時点の要件に対してオーバースペック

**Snowflake Tasks を不採用とした理由：**

- AWS 側での前後処理（通知、ファイル連携等）との統合が困難
- エラーハンドリングやリトライ制御が Step Functions に比べて限定的
- 本案件向けに構築する AWS 構成を活用しにくい

---

## 最終アーキテクチャ

3 つの決定を踏まえた全体構成は以下の通り。

```
EventBridge (スケジュール)
  └─ Step Functions                          ← 決定 3
       └─ ECS RunTask (Fargate, awsvpc, EIP 固定)
            └─ execute_dbt.py
                 ├─ boto3 → Secrets Manager から接続情報を取得
                 ├─ cryptography → 秘密鍵をメモリ上でロード
                 ├─ snowflake-connector-python → key-pair 認証で接続
                 └─ EXECUTE DBT PROJECT      ← 決定 2（dbt on Snowflake）
                      └─ Snowflake 内部で dbt モデル実行  ← 決定 1（dbt 採用）
```

### セキュリティ設計

- Snowflake 認証は key-pair 方式を採用し、パスワード認証は使用しない
- 秘密鍵は Secrets Manager に保管し、実行時にメモリ上で DER 形式に変換して使用する。ディスクへの書き出しは行わない
- ECS Fargate ENI に EIP をアタッチし、Snowflake Network Policy に登録することで接続元 IP を固定する
- ECS タスクロールは Secrets Manager の GetSecretValue のみを許可する最小権限設計とする
- データは Snowflake 内部で完結し、外部 SaaS を経由しない

---

## 将来の拡張パス

```
現在
  Step Functions → ECS RunTask → EXECUTE DBT PROJECT

Phase 2（配信先が増えた場合）
  Step Functions → ECS RunTask → EXECUTE DBT PROJECT
                → Lambda → S3 / SFTP 配信

Phase 3（大規模化・多様化した場合）
  Airflow (MWAA) への移行を検討
    → dbt 実行は cosmos で DAG 自動生成
    → 配信は Provider で統一
```

---

## 影響範囲

- ECS クラスター、タスク定義、ECR リポジトリの構築が必要
- Step Functions ステートマシンの作成が必要
- IAM ロール（タスクロール、実行ロール、Step Functions 実行ロール）の作成が必要
- Secrets Manager へのシークレット登録が必要
- Snowflake 側でサービスユーザー、ロール、公開鍵の登録が必要
- Snowflake Git Integration の設定と dbt プロジェクトのデプロイが必要

---

## 再検討トリガー

以下のいずれかに該当した場合、本 ADR の関連する決定を見直す。

### 決定 1（dbt 採用）の再検討

- dbt on Snowflake の機能制約により、Stored Procedure でしか実現できない変換処理が多数発生した場合

### 決定 2（dbt on Snowflake）の再検討

- dbt on Snowflake の機能が dbt Core / Cloud に比べて大幅に遅延し、必要な機能（増分モデル、スナップショット等）が利用できない場合
- Snowflake 以外の DWH（BigQuery, Redshift 等）への並行展開が必要になり、マルチプラットフォーム対応が求められた場合

### 決定 3（Step Functions）の再検討

- データ配信先が 5 種類以上に増加し、Lambda 個別実装の保守コストが増大した場合
- dbt モデル数が 50 以上に達し、モデル単位の依存関係制御や部分リトライが必要になった場合
- インフラ運用の専任リソースが確保され、MWAA の運用コストを許容できるようになった場合
- マルチクラウド対応が求められ、AWS 以外の環境でも同一ワークフローを実行する必要が生じた場合

---

## 決定サマリー

| # | 検討テーマ | 決定 | 主な根拠 |
|---|-----------|------|---------|
| 1 | ELT 変換層の手法 | **dbt** を採用（Stored Procedures は不採用） | 依存関係の自動管理、テスト・ドキュメント自動生成、DBA 不在組織での保守性 |
| 2 | dbt の実行形態 | **dbt on Snowflake** を採用（Cloud / Core は不採用） | データが外部 SaaS を経由しない、Snowflake 内部完結によるシンプルさ、追加ライセンスコスト不要 |
| 3 | オーケストレーション | **Step Functions + ECS RunTask** を採用（Airflow / Snowflake Tasks は不採用） | サーバーレスで運用負荷が最小、固定費ゼロ、本案件向け AWS 構成の初期構築を含めても総負荷が小さい |
