---
title: Medallion Architecture — データレイクハウスの設計原則
date: 2025-01-15
tags:
  - data-engineering
  - guide
  - infra
---

# Medallion Architecture

データレイクハウスの設計原則 — Bronze・Silver・Gold の3層アーキテクチャで実現するデータ品質管理

初版: 2025-01-15
更新: 2026-03-28

## 3層構造
- Bronze→Silver→Gold

## Delta Lake
- ACIDトランザクション

## Lakehouse
- パラダイム

## 概要
- Medallion
- Bronze層
- Silver層
- Gold層
- Delta Lake
- 実装パターン
- 他のアーキテクチャ
- アンチパターン
- まとめ

## Medallion Architectureとは — 概要と歴史

**Medallion Architecture**は、2020年にDatabricksが提唱したデータレイクハウスの標準アーキテクチャです。生データから分析可能な形式へと段階的に洗練させる3層構造（Bronze・Silver・Gold）によって、スケーラビリティ、データ品質、ガバナンスの課題を同時に解決します。

従来のデータウェアハウス（DWH）は高価で柔軟性に欠け、データレイク（Data Lake）は構造化されず「沼」となってしまう傾向がありました。Medallion Architectureは両者の長所を取り入れ、クラウドネイティブなスケーラビリティと OLAP スタイルの分析が共存する新しいパラダイムを実現します。

> **Key Insight:** Medallion Architectureの本質は「段階的な変換と品質向上」です。各層で明確な責任を持つことで、データパイプラインは複雑さを管理しやすくなり、デバッグやメンテナンスが格段に簡単になります。

**歴史的背景：** Databricksが2020年のブログポスト『Medallion Architecture』で発表して以来、クラウドデータベース業界の事実上の標準となりました。Snowflake、BigQuery、Redshift など主流プラットフォームでも同パターンの採用が推奨されるようになり、データエンジニアリングの共通言語として認識されています。

### Medallion Architecture の基本原理

- **Bronze層：** ソースシステムから のデータを「ありのまま」保存。スキーマはソースに準じ、データ品質チェックはまだ行わない。
- **Silver層：** Bronze から取り込んだデータを クレンジング・標準化・統合。ビジネスロジック適用前の「きれいな単一ソース」を目指す。
- **Gold層：** Silver からさらに集約・非正規化し、ビジネスユースケースに最適化したビュー・テーブルを提供。ダッシュボードやML、分析に即座に使える形。

この3層モデルにより、各ユースケースに応じた柔軟な設計、透明性のある監査証跡、スケーラブルで保守性の高いパイプラインが実現されます。

## Bronze層 — 生データの取り込みと保存

**Bronze層**はすべてのデータの「入口」です。ソースシステム（ERP、CRM、ログシステム、IoT センサ など）から のデータを原則として変更なく保存します。エラーレコードや古いデータも含めて、「捨てない」という原則が重要です。

### 取り込みパターン

| パターン | 説明 | 用途 |
|---------|------|------|
| **バッチ** | 毎日・毎時間など定期的にデータをコピー | 売上データ、マスターデータ |
| **ストリーミング** | イベントが発生した瞬間に取り込み（Kafka、Kinesis等） | ログ、トランザクション、リアルタイム分析 |
| **CDC** | 変更捕捉（Change Data Capture）で差分のみ取得 | データベースの増分レプリケーション |
| **API** | REST/GraphQL経由で外部サービスから取得 | SaaS統合、第三者データ |

### Bronze層の特性

- **スキーマ・オン・リード：** 取り込み時点では型チェック・変換を最小限にし、読み込み側でスキーマ解釈を行う柔軟性を保つ。
- **アペンドオンリー：** 一度書き込んだデータは上書きせず、タイムスタンプ付きで保存。履歴追跡とデバッグが容易。
- **メタデータ保持：** 取り込み日時、ソースシステム、バージョンなどのメタデータを付与。
- **大量保存容易性：** ストレージコストが低いため、「もしかして必要かもしれない」データも保存する余裕が生まれる。

### Pythonコード例：バッチ取り込み

```python
# PySpark を使ったCSVファイル → Bronze の取り込み
from pyspark.sql import SparkSession
from datetime import datetime

spark = SparkSession.builder.appName("bronze_ingest").getOrCreate()

# ソースCSVを読み込み
source_df = spark.read.csv(
    "/source/sales_data.csv",
    header=True,
    inferSchema=True
)

# メタデータカラムを追加
from pyspark.sql.functions import lit, current_timestamp
bronze_df = source_df.withColumn(
    "ingestion_date", current_timestamp()
).withColumn(
    "source_system", lit("SAP_ERP")
).withColumn(
    "version", lit("1.0")
)

# Bronze に書き込み（append mode）
bronze_df.write.mode("append").format("delta").save(
    "/lakehouse/bronze/sales"
)

print(f"Ingested {source_df.count()} records to Bronze")
```

> **Best Practice:** Bronze層はストレージ効率よりも「取得の確実性」を優先します。圧縮はしても、フォーマット変換は最小限に。エラーハンドリングは Silver で行うメンタルモデルを持つことが大切です。

### CDC（Change Data Capture）の取り込み例

```python
# Debezium/Kafka 経由でオンプレデータベースの変更を取り込み
# CDC フィードから Bronze へ
cdc_stream = spark.readStream.format("kafka").option(
    "kafka.bootstrap.servers", "kafka:9092"
).option(
    "subscribe", "db_changes"
).option(
    "startingOffsets", "earliest"
).load()

# JSON形式のCDCメッセージをパース
from pyspark.sql.functions import from_json, col
parsed = cdc_stream.select(
    from_json(col("value").cast("string"),
              "op STRING, table STRING, before MAP<STRING,STRING>, after MAP<STRING,STRING>").alias("cdc")
).select("cdc.*")

# Bronze に書き込み（streaming write）
query = parsed.writeStream \
    .format("delta") \
    .option("checkpointLocation", "/lakehouse/checkpoints/cdc") \
    .outputMode("append") \
    .start("/lakehouse/bronze/customers_cdc")

query.awaitTermination()
```

## Silver層 — データ品質とクレンジング

**Silver層**は Bronze の粗データを精製し、組織全体で信頼できる「単一の正解」（SSOT: Single Source of Truth）を実現します。ここが最もエンジニアリング負荷が高い層であり、データ品質の大半はこの層で担保されます。

### Silver層で実施する主な処理

- **データクレンジング：** NULL値の処理、形式の統一、外字・特殊文字の除去。
- **重複排除：** データソース側の重複や、取り込みエラーによる重複を検出・削除。
- **スキーマ標準化：** 複数のソースを統一スキーマにマッピング。日付形式、通貨、単位などの統一。
- **データ型の厳密化：** スキーマ・オン・リードから スキーマ・オン・ライト へ。型定義を明示。
- **ビジネスキーの定義：** 自然キー（顧客ID、注文番号など）を明確にし、SCD（Slowly Changing Dimensions）対応を準備。
- **品質メトリクスの計算：** 記録数、NULL比率、異常値フラグなどを記録。

### Silver層の設計パターン

| パターン | 説明 | メリット |
|---------|------|---------|
| テーブル毎 | bronze/customers → silver/customers | シンプル、変更影響が限定的 |
| ドメイン毎 | bronze/sales, bronze/inventory → silver/crm_unified | クロス・ドメイン統合が容易 |
| 階層化 | bronze → silver_raw → silver_cleaned → gold | 段階的品質向上、中間ステップでのデバッグ容易 |

### SQLコード例：顧客マスターの標準化

```sql
-- Bronze 顧客データから Silver へのETL
CREATE OR REPLACE TABLE silver.customers AS
SELECT
-- ビジネスキー（重複排除に使用）
customer_id,
-- クレンジング済みカラム
LOWER(TRIM(first_name)) as first_name,
LOWER(TRIM(last_name)) as last_name,
CAST(birth_date AS DATE) as birth_date,
CASE
WHEN email LIKE '%@%.%' THEN LOWER(TRIM(email))
ELSE NULL
END as email_validated,
-- 正規化：電話番号をE.164形式へ
REGEXP_REPLACE(phone_number, '[^0-9+]', '') as phone_e164,
-- ステータスの標準化
CASE
WHEN status IN ('ACTIVE', 'A', 'Y', '1') THEN 'ACTIVE'
WHEN status IN ('INACTIVE', 'I', 'N', '0') THEN 'INACTIVE'
ELSE 'UNKNOWN'
END as status_normalized,
-- 品質指標
CASE
WHEN email IS NULL OR phone_number IS NULL THEN TRUE
ELSE FALSE
END as has_missing_contact,
-- 監査カラム
FROM_UNIXTIME(created_timestamp) as created_at,
FROM_UNIXTIME(updated_timestamp) as updated_at,
current_timestamp() as silver_processed_at,
-- ハッシュ：行全体の変更検知用（SCD実装に必要）
MD5(CONCAT_WS('|',
first_name, last_name, email, phone_number, status
)) as row_hash
FROM bronze.customers
WHERE
-- エラーレコードをスキップ（フラグ付けは金）
customer_id IS NOT NULL
AND created_timestamp > 0
ORDER BY customer_id, updated_timestamp DESC
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_timestamp DESC) = 1;
```

### PySpark：SCD Type 2（履歴追跡）の実装

```python
from pyspark.sql.functions import *
from pyspark.sql.window import Window

# Silver から新しい顧客データを取得
new_customers = spark.read.table("silver.customers_staging")

# 現在の SCD テーブルを読み込み
try:
    current_scd = spark.read.table("silver.customers_scd_type2")
except:
    # 初回実行時は空のテーブルを作成
    current_scd = spark.createDataFrame([], """
        customer_id STRING,
        first_name STRING,
        email_validated STRING,
        status_normalized STRING,
        row_hash STRING,
        valid_from DATE,
        valid_to DATE,
        is_current BOOLEAN
    """)

# 既存レコードと新規データをマージ
merged = new_customers.join(
    current_scd.where(col("is_current") == True).select(
        "customer_id", "row_hash"
    ),
    "customer_id",
    "left_outer"
)

# ハッシュが異なる = 属性が変わったレコード
changed = merged.filter(
    (col("row_hash") != col("row_hash")) | (col("row_hash").isNull())
)

# 既存の "is_current" レコードを "valid_to" を今日に設定して非活性化
to_close = current_scd.where(
    current_scd.customer_id.isin([row.customer_id for row in changed.select("customer_id").collect()])
    & (col("is_current") == True)
).withColumn(
    "valid_to", current_date()
).withColumn(
    "is_current", lit(False)
)

# 新しいレコードを挿入（valid_from = 今日、valid_to = 9999-12-31）
new_records = changed.select(
    "customer_id", "first_name", "email_validated", "status_normalized", "row_hash"
).withColumn(
    "valid_from", current_date()
).withColumn(
    "valid_to", lit("9999-12-31").cast("date")
).withColumn(
    "is_current", lit(True)
)

# 変更のなかったレコードはそのまま
unchanged = current_scd.join(
    new_customers.select("customer_id", "row_hash"),
    ["customer_id", "row_hash"],
    "inner"
)

# 3つを union して新しい SCD テーブルを作成
final_scd = (
    to_close.union(new_records).union(unchanged)
)

final_scd.write.mode("overwrite").saveAsTable("silver.customers_scd_type2")
```

> **Deep Dive:** SCD（Slowly Changing Dimensions）はデータウェアハウジングの重要な概念です。Type 1（上書き）は履歴喪失、Type 2（履歴追跡）は行数増加のトレードオフがあります。ビジネス要件に応じて使い分けることが重要です。

## Gold層 — ビジネスに最適化されたデータ

Gold層は分析やMLの「直前」のステージです。ここでのデータは ユーザーがすぐに使える形 に最適化されています。集約、非正規化、キャッシング など、パフォーマンスとビジネス要件を両立させます。

### Gold層の責務

- **集約テーブル：** 日次・月次の売上集計、顧客ライフタイムバリュー（LTV）など、計算コストが高い指標を事前計算。
- **非正規化（Denormalization）：** Silver の正規化された構造を、分析用に非正規化。JOIN の手数を削減。
- **スター・スキーマ：** 事実テーブル（Fact）と次元テーブル（Dimension）を設計。古典的な BI スタイル。
- **ビジネスロジック適用：** マージン計算、セグメント分類、異常検知フラグ など、ドメイン知識の具現化。
- **キャッシング・マテリアライズド・ビュー：** 頻繁にアクセスされるビューを物理テーブルとして保存。

### Gold層のパターン

| パターン | 例 | 特徴 |
|---------|----|----|
| 集約テーブル | daily_sales, monthly_revenue | 時間軸で粒度を落とし、計算量削減 |
| スター・スキーマ | fact_orders + dim_customers, dim_products | BI ツール統合に最適、JOIN最小化 |
| キューブ・テーブル | sales_cube（年×月×製品×地域） | 多次元分析、OLAP ドリルダウン対応 |
| 特徴テーブル | customer_features（ML学習用） | ML パイプライン向け、時間スライス対応 |

### SQLコード例：売上スター・スキーマ

```sql
-- Gold層：売上スター・スキーマ
-- fact_orders：事実テーブル
CREATE OR REPLACE TABLE gold.fact_orders AS
SELECT
    o.order_id,
    o.customer_id,
    o.product_id,
    o.store_id,
    CAST(o.order_date AS DATE) as order_date,
    EXTRACT(YEAR FROM o.order_date) as order_year,
    EXTRACT(MONTH FROM o.order_date) as order_month,
    EXTRACT(QUARTER FROM o.order_date) as order_quarter,
    -- 数値指標（ファクト）
    o.quantity,
    o.unit_price,
    ROUND(o.quantity * o.unit_price, 2) as gross_amount,
    -- ビジネスロジック：マージン率計算
    ROUND(
        (o.quantity * o.unit_price - o.quantity * p.cost_price) /
        NULLIF(o.quantity * o.unit_price, 0),
        4
    ) as margin_rate,
    -- 割引・キャンペーンフラグ
    CASE
        WHEN o.discount_code IS NOT NULL THEN TRUE
        ELSE FALSE
    END as has_discount
FROM silver.orders o
INNER JOIN silver.products p ON o.product_id = p.product_id
WHERE o.order_date >= DATE_SUB(CURRENT_DATE, INTERVAL 24 MONTHS);
```

## Delta Lake — トランザクション層

Delta Lakeは Medallion Architecture を支える技術基盤です。ACID トランザクション保証により、複数のジョブが同時にテーブルを更新する場合でもデータ整合性を保証します。

### Delta Lake の主要機能

- **ACID トランザクション：** データベース品質のトランザクション保証。
- **スキーマ検証：** 不正なデータは自動的に拒否。
- **タイムトラベル：** 過去のバージョンに遡及可能。デバッグやロールバックに強力。
- **変更フィード（Change Feed）：** テーブルの変更差分をキャプチャ。ストリーミングパイプラインに活用。

### PySpark：Delta Lake の基本操作

```python
from delta.tables import DeltaTable

# テーブル作成
spark.createDataFrame(
    [("A", 1), ("B", 2)],
    ["name", "value"]
).write.format("delta").mode("overwrite").save("/delta/sample")

# タイムトラベル：3時間前のデータを読み込み
spark.read.format("delta").option(
    "timestampAsOf", "2025-01-15 12:00:00"
).load("/delta/sample").show()

# Change Feed：変更差分を取得
spark.read.format("delta").option(
    "readChangeData", "true"
).option(
    "startingVersion", 0
).load("/delta/sample").show()

# 更新：MERGE 操作でアップサート
delta_table = DeltaTable.forPath(spark, "/delta/sample")
delta_table.alias("existing").merge(
    spark.createDataFrame([("A", 10)], ["name", "value"]).alias("updates"),
    "existing.name = updates.name"
).whenMatchedUpdate(set={"value": "updates.value"}).execute()
```

## 実装パターン — 実践的な設計

### パターン1：日次バッチ ETL

```python
# 毎日深夜に実行される典型的なパイプライン
from datetime import datetime, timedelta

# Bronze 取り込み
def ingest_to_bronze(source_path, target_table):
    df = spark.read.parquet(source_path)
    df.withColumn("ingestion_date", lit(datetime.now())) \
        .write.mode("append") \
        .format("delta") \
        .save(target_table)

# Silver 変換
def transform_to_silver():
    bronze_df = spark.read.table("bronze.customers")
    silver_df = bronze_df.select(
        col("customer_id"),
        lower(trim(col("name"))).alias("customer_name"),
        # ... その他の変換
    ).distinct()
    silver_df.write.mode("overwrite").format("delta").save("silver.customers")

# Gold 集約
def aggregate_to_gold():
    silver_df = spark.read.table("silver.customers")
    gold_df = silver_df.groupBy("segment") \
        .agg(count("*").alias("count"), avg("ltv").alias("avg_ltv"))
    gold_df.write.mode("overwrite").format("delta").save("gold.customer_summary")

# 実行
ingest_to_bronze("/landing/customers_*.parquet", "bronze/customers")
transform_to_silver()
aggregate_to_gold()
```

### パターン2：リアルタイム ストリーミング

```python
# Kafka からの ストリーミング取り込み
kafka_stream = spark.readStream.format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "events") \
    .load()

# Bronze に保存
kafka_stream.select(col("value").cast("string")) \
    .writeStream \
    .format("delta") \
    .option("checkpointLocation", "/checkpoint/events_bronze") \
    .start("bronze/events")

# Silver で集約（1分ごと）
silver_stream = spark.readStream.format("delta").load("bronze/events") \
    .withWatermark("timestamp", "10 minutes") \
    .groupBy(window(col("timestamp"), "1 minute")) \
    .count()

silver_stream.writeStream \
    .format("delta") \
    .option("checkpointLocation", "/checkpoint/events_silver") \
    .outputMode("update") \
    .start("silver/events_per_minute")
```

## 他のアーキテクチャとの比較

| アーキテクチャ | 特徴 | 適用例 |
|-------------|------|-------|
| **Medallion（3層）** | 段階的品質向上、監査証跡、柔軟性 | クラウドデータレイク、モダンデータスタック |
| Lambda | バッチ + ストリーム + 統合層 | リアルタイム + 履歴分析が必須 |
| Kappa | ストリームメインで統一 | リアルタイム系が主流 |
| ELT | Data Warehouse 中心 | 規制厳密、高スキーマ統制 |

## アンチパターン — 避けるべき設計

### アンチパターン1：全層で同じスキーマを使用

```python
# 悪い例
bronze_df.write.mode("overwrite").format("delta").save("silver/customers")

# 良い例：各層で明確な責務分割
# Bronze：原則として変更なし
# Silver：クレンジング・標準化
# Gold：分析用に最適化
```

### アンチパターン2：Bronze に変換ロジックを詰め込む

```python
# 悪い例
bronze_df = df.withColumn("processed_date", ...) \
              .withColumn("adjusted_amount", ...) \
              .filter(col("amount") > 0)

# 良い例：Bronze は取り込みのみ
bronze_df = df.withColumn("ingestion_date", current_timestamp()) \
              .write.mode("append") \
              .format("delta") \
              .save("bronze/sales")

# Silver で変換
silver_df = spark.read.table("bronze.sales") \
    .withColumn("adjusted_amount", col("amount") * 1.1) \
    .filter(col("amount") > 0)
```

### アンチパターン3：Gold 層を複数の異なるスキーマで設計

```python
# 悪い例：Gold が統一性を欠く
spark.read.table("silver.customers") \
    .select(col("id"), col("name"), col("ltv")) \
    .write.format("delta").mode("overwrite").save("gold/customers_v1")

spark.read.table("silver.customers") \
    .select(col("customer_id"), col("customer_name"), col("lifetime_value")) \
    .write.format("delta").mode("overwrite").save("gold/customers_v2")

# 良い例：メタデータの統一
# スター・スキーマに統一、分析ツール向けメタデータを付与
```

## まとめ

Medallion Architecture は単なる「3層の箱」ではなく、**段階的な品質向上とデータガバナンスの哲学**です。

### 重要なポイント

1. **各層の責務を明確に：** Bronze = 取り込み、Silver = 品質、Gold = 最適化
2. **品質チェックは Silver で：** 上流で完全に排除しない、問題を可視化
3. **スケーラビリティを優先：** Snowflake・BigQuery・Delta Lake など、スケーラブルな基盤を選択
4. **メタデータを保持：** 監査証跡、タイムスタンプ、ソース追跡が後の分析を楽にする
5. **ドメイン知識は Gold で：** テクニカルな品質は Silver で、ビジネス最適化は Gold で

Medallion Architecture を採用することで、データエンジニアリングは単なる「データ移動」から「データ品質の段階的向上」へシフトし、組織全体が信頼できるデータに基づいた意思決定が可能になります。
