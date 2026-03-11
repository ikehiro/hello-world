# データウェアハウス業界の競争構造

データウェアハウス業界では、ここ数年で **「Snowflake vs Databricks」を中心とした競争構造** ができている。  
その背景には **クラウドベンダー（AWS / Google / Microsoft） vs 独立系（Snowflake / Databricks）** という大きな力関係がある。

---

## 1. データ基盤市場の勢力図

大きく3つの陣営が存在する。

| 陣営 | プレイヤー |
|---|---|
| **① クラウドネイティブDWH** | Snowflake / BigQuery / Redshift / Synapse |
| **② レイクハウス** | Databricks / Microsoft Fabric |
| **③ 従来DWH** | Teradata / Oracle / IBM |

> 現在の主戦場は **Snowflake vs Databricks**。

---

## 2. Snowflake vs Databricks 戦争

この2社はデータプラットフォームの主導権争いをしている。

| 比較軸 | Snowflake | Databricks |
|---|---|---|
| 出自 | DWH出身 | Data Lake出身 |
| 中心言語 | SQL | Spark / Python |
| 得意領域 | BI / Analytics | AI / ML |
| 戦略 | DWHをクラウド化 → SQL中心 → BI/Analytics | Data Lake + AI → Spark/Python → ML |

最近の動向：

- **Snowflake** → ML強化・Iceberg対応
- **Databricks** → SQL Warehouse強化

> 両者の境界は急速に消えつつある。

---

## 3. クラウドベンダーとの関係

### Snowflakeはクラウドではない

Snowflakeは AWS / Azure / GCP の **上で動くSaaS** であり、クラウドベンダーに依存している。  
これが業界の裏構造。

---

## 4. AWS が Snowflake を嫌う理由

AWSには **Amazon Redshift** がある。AWSから見ると：

> **Snowflake = AWSの売上を奪う存在**

実際の動き：
- AWS re:Invent では Snowflake はほぼ紹介されない
- AWS は Redshift を猛烈に改良中

---

## 5. Google の BigQuery 戦略

BigQuery は独自モデル：

- **サーバーレス**
- **スキャン課金**

Google の狙い：

```
BigQuery → AI → Vertex AI → Google Cloud lock-in
```

つまり **AIパイプライン** への誘導。

---

## 6. Microsoft の戦略

Microsoft は最も **統合志向**。

```
Fabric
 ├─ Synapse
 ├─ Data Factory
 ├─ Lakehouse
 └─ Power BI
```

> 全部まとめる戦略。

---

## 7. データ基盤の進化（3世代）

| 世代 | カテゴリ | 代表プロダクト |
|---|---|---|
| **第1世代** | オンプレDWH | Teradata / Oracle |
| **第2世代** | クラウドDWH | Snowflake / BigQuery / Redshift |
| **第3世代** | レイクハウス | Databricks / Fabric |

---

## 8. 今後の本命

| ユースケース | 本命 |
|---|---|
| BI中心 | **Snowflake** |
| AI / ML中心 | **Databricks** |

ただし Snowflake は AI + Iceberg へ、Databricks は SQL Warehouse へ拡張しており、境界が消えている。

---

## 9. 現場のリアル（2026年）

企業の採用パターンは大きく3つ：

| パターン | 構成 |
|---|---|
| **①** | Snowflake + dbt + Tableau |
| **②** | Databricks + Delta Lake + ML |
| **③** | BigQuery + Looker |

---

## 10. Snowflake の目指す姿

Snowflake は **「データ共有プラットフォーム」** を目指している。

特徴：
- **Data Sharing**（コピー不要のデータ共有）
- **Data Marketplace**
- **Cross Cloud**

---

# Snowflake Data Sharing アーキテクチャ

Snowflake が他の DWH と本質的に違うポイントである **Data Sharing（データ共有）** について。  
Snowflake が急成長した最大の理由のひとつ。

---

## 1. 従来DWHのデータ共有（問題点）

従来方式：

```
企業A DWH → export → CSV/Parquet → S3/FTP → 企業B DWH → import
```

**問題点：**
- データコピーが必要
- ETLが必要
- データ同期が遅れる
- ストレージが増える

> 「データのコピー文化」だった。

---

## 2. Snowflake の Data Sharing

Snowflake はこの問題を **アーキテクチャレベル** で解決。

```
      Snowflake Storage Layer
              │
       ┌──────┴──────┐
       │              │
   Company A      Company B
   Warehouse      Warehouse
```

**ポイント：**
- データコピー不要
- 同じストレージを参照
- リアルタイム

---

## 3. Snowflake の3層アーキテクチャ

```
       ┌──────────────────┐
       │  Cloud Services   │
       │  Metadata / Auth  │
       └────────┬─────────┘
                │
       ┌────────┴─────────┐
       │ Virtual Warehouse │
       │   Compute Layer   │
       └────────┬─────────┘
                │
       ┌────────┴─────────┐
       │  Storage Layer    │
       │  MicroPartition   │
       └──────────────────┘
```

- **Compute 分離** — これが Data Sharing の鍵
- **Storage 共有**

---

## 4. Data Share の仕組み

```
データ → Share Object → Consumer Account
```

### SQL 例

```sql
-- 共有を作成
CREATE SHARE sales_share;

-- テーブルへの SELECT 権限を付与
GRANT SELECT ON TABLE sales_data TO SHARE sales_share;

-- 相手アカウントを追加
ALTER SHARE sales_share ADD ACCOUNT = 'companyB';
```

Consumer 側：

```sql
SELECT * FROM shared_db.sales_data;
```

> **コピーなし** でクエリ可能。

---

## 5. Data Marketplace

Snowflake は Data Sharing を発展させ **Data Marketplace** を構築。

| 提供データ例 |
|---|
| Bloomberg（金融） |
| 気象データ |
| マーケティングデータ |

> `SUBSCRIBE → 即クエリ` が可能。

---

## 6. なぜ他社は真似できないのか

理由は Snowflake の **ストレージ設計** にある。

- **Micro Partition** + **Immutable Storage**
- データを変更しない → 参照共有が安全にできる

---

## 7. Databricks のアプローチ

Databricks は **Delta Sharing** を開発。

| | Snowflake | Databricks |
|---|---|---|
| 共有方式 | **built-in**（プラットフォーム内蔵） | **外部仕様**（オープンプロトコル） |

> 思想が根本的に異なる。

---

## 8. 実務での典型構成

```
ETL → Snowflake → BI（Tableau / Power BI / Looker）
                 → データ共有 → パートナー企業
```

---

## 9. Snowflake の最大の特徴（まとめ）

| # | 特徴 |
|---|---|
| 1 | **Storage / Compute 分離** |
| 2 | **Micro Partition** |
| 3 | **Data Sharing** |
| 4 | **Marketplace** |

---

## 10. Snowflake の真の戦略

Snowflake は DWH会社ではない。目指しているのは：

> **Global Data Cloud**

```
企業A ─┐
企業B ─┼─ Snowflake ─ 世界のデータを繋ぐプラットフォーム
企業C ─┘
```
