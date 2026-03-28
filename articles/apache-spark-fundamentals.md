---
title: Apache Spark 徹底入門 — 分散処理エンジンの原理と実践
date: 2025-02-15
tags:
  - data-engineering
  - python
  - sql
  - guide
---

# Apache Spark 徹底入門

分散処理エンジンの原理と実践

## Sparkの歴史と位置づけ

Apache Sparkは、Hadoopエコシステムの分散処理エンジンの統一化という課題から生まれました。2000年代中盤、Hadoop + MapReduceは大規模データ処理の標準でしたが、その後、複数の最適化エンジンが乱立します。

- **Hive** — MapReduceを使用したSQLレイヤー（低速）
- **Impala** — C++での直接実装（高速だが複雑）
- **Presto** — インタラクティブクエリエンジン
- **Tez** — 改善されたDAGエンジン

各エンジンが異なるAPIとメタデータ形式を持つため、統合への課題がありました。そこで**Spark（2011年、UC Berkeleyで開発）** が登場し、以下の革新をもたらしました：

- **統一API** — RDD、DataFrame、SQL、MLlib、Streaming が共存
- **メモリ内処理** — 中間結果をメモリに保持、ディスク I/O削減
- **ストレージの分離** — HDFS、S3など複数のストレージに対応
- **複数デプロイモード対応** — Standalone、YARN、Kubernetes

重要な点として、**Spark は Hadoop の一部ではなく、独立した計算エンジン**です。HDFS等のストレージレイヤーとは分離されており、Spark 自体はストレージに関知しません。

現在のSparkエコシステム（2025年）：

- **Spark SQL** — ANSI SQL:2003準拠、Catalyst最適化
- **MLlib** — 分散機械学習ライブラリ
- **Structured Streaming** — イベント駆動型ストリーム処理
- **GraphX** — グラフ処理フレームワーク

**Databricks**（Spark創設者によって2013年設立）がマネージドSaaSを提供し、Spark 実装の商用化を実現しています。

## アーキテクチャの全体像

Sparkアプリケーションは **Driver-Executor 分散アーキテクチャ** を採用しています。

### コアコンポーネント

**Driver プロセス**

- ユーザーコード（main()メソッド）が実行されるJVM プロセス
- Spark SQL/DataFrameコードを **DAG（有向非巡回グラフ）** に変換
- Executorへのタスク配分、実行監視、結果集約
- Spark UI（ポート4040）でリアルタイム実行状況を提供

**Executor プロセス**

- ワーカーノードで実行されるJVM プロセス（通常 1ノード = 1 Executor）
- 割り当てられたメモリとコアで並列タスク実行
- アプリケーション終了まで生存（動的割り当て有効時は異なる）

**Cluster Manager**

- リソース管理とスケジューリング
- YARN（Hadoop）、Kubernetes、Standalone、Databricks Managed

**SparkSession**

- Spark 2.0以降で導入された統一エントリーポイント
- 従来の SparkContext、SQLContext、HiveContext を統合
- DataFrame/SQL API への唯一のアクセス手段

### 実行フロー例

Spark ユーザーが DataFrame 操作を記述すると、以下のフローで実行されます：

1. Driver で SQL/DataFrame コードを解析
2. DAG を構築し、Catalyst で最適化
3. Driver がタスクをスケジューリング
4. Executor でタスク並列実行
5. 結果をメモリ/ストレージに集約

## Structured API — DataFrame、Dataset、RDD

Spark の API レイヤーは、段階的な進化を遂げています。RDD → DataFrame → Dataset という階層構造で、抽象度と最適化レベルが向上します。

### RDD（Resilient Distributed Dataset）時代の課題

- 汎用的だが、Spark内部が最適化できない（ユーザーコード任せ）
- `Iterator[T]` のため列指定不可、スキーマなし
- シャッフル時のメモリ効率が低い

### DataFrame — 名前付き列とスキーマの統合

**Pandas DataFrameに類似した設計ですが、分散**。

- 名前付き列を持つ、スキーマ定義済みの分散テーブル
- Spark内部（Catalyst最適化）が列構造を把握できる
- SQL と Python/Scala/Java/R の統一API
- **Python、R ユーザーの主な選択肢**（Dataset はJava/Scala向け）

### Dataset — 型安全なAPI

- Scala と Java でのみ利用可（Python不可）
- コンパイル時の型チェック + DataFrame の最適化
- 大規模統型データ処理に向く（例：ケースクラスの集合）

### API の選択基準

- **Python/R ユーザー** → DataFrame
- **Scala/Java（型安全性が必要）** → Dataset
- **低レベル制御が必要** → RDD（稀）

## Spark SQL エンジンと Catalyst オプティマイザ

Spark SQL は**ANSI SQL:2003準拠**の SQL エンジンであり、その核心は Catalyst オプティマイザです。

### 4フェーズのクエリプラン生成

1. **解析（Analysis）** — SQL を解析、テーブル・列名をカタログで検証、スキーマ解決
2. **論理最適化（Logical Optimization）** — 規則ベースの変換（述語押し下げ、列刈り込み）
3. **物理計画（Physical Planning）** — 複数の物理プラン生成、コストベース最適化で選択
4. **コード生成（Code Generation）** — Tungsten による whole-stage Java code generation

### Catalyst の最適化ルール

- **述語押し下げ（Predicate Pushdown）** — フィルタ条件をデータソース直前に移動、不要データ早期除外
- **列刈り込み（Column Pruning）** — SELECT で使用されない列を読み込み対象から除外
- **定数畳み込み（Constant Folding）** — `1 + 2` を実行時ではなくコンパイル時に `3` に
- **ブール式簡略化（Boolean Simplification）** — `a OR a` を `a` に
- **コスト最適化（CBO）** — テーブル統計情報（行数、列の分布）を基に JOIN 順序最適化

### Tungsten — Whole-Stage Code Generation

従来の Spark では各 operator（filter、join等）が個別に呼び出され、中間オブジェクト生成が多発していました。

**利点：**

- 仮想関数呼び出し（virtual call overhead）の削減
- CPU キャッシュ効率向上、レジスタ活用
- 中間オブジェクトのメモリ割り当て削減
- Spark 2.0 以降、デフォルト有効（`spark.sql.codegen.wholeStage`）

## Transformation と Action — 遅延評価の仕組み

### 遅延評価（Lazy Evaluation）

Spark の重要な設計思想：**Transformation（変換）は遅延評価され、Action（アクション）が呼ばれるまで実行されない**。

### Transformation（遅延評価）

- `select()、filter()、map()、join()、groupBy()、union()` など
- 新しい DataFrame/RDD を返すが、実際には計算しない
- DAG にステップを追加するだけ

### Action（即時実行）

- `show()、collect()、count()、write()、take()、first()` など
- 呼び出された瞬間にDAG 実行開始
- 結果をメモリ或いはストレージに具現化

### Narrow vs Wide Transformation

**Narrow Transformation**（狭い依存）：各出力パーティションが1つの入力パーティションに依存。

- `filter()、select()、flatMap()、map()`
- シャッフルなし、単一ステージで処理
- ノード間のデータ移動なし

**Wide Transformation**（広い依存）：各出力パーティションが複数の入力パーティションに依存。

- `groupBy()、reduceByKey()、join()、distinct()、repartition()`
- シャッフル必須（ネットワーク・ディスク I/O 発生）
- 各 wide transformation でステージ区切り

**性能最適化のポイント：** Wide transformation は避けられない場合が多いですが、データスキューに注意。Spark 3.x の AQE（Adaptive Query Execution）は skewed joins を自動検出・最適化します。

### Partition と並列度

**Partition**：データの物理的分散単位。1パーティション = 1Executor 内の1タスク処理。

- 並列度 = パーティション数（コア数 ≥ パーティション数のとき）
- パーティション数 < コア数 → 処理スロット遊び
- パーティション数 >> コア数 → タスク起動オーバーヘッド増加

### repartition() vs coalesce()

- **repartition()** — 新しいパーティション数を指定、シャッフル発生（安全）
- **coalesce()** — パーティション統合のみ、シャッフルなし（効率的だがパーティション数増加不可）

## データソースと I/O

### DataFrameReader/Writer API

Spark の I/O は統一された API で複数フォーマット対応：

### 一般的なフォーマット

- **CSV** — 汎用テキスト形式、スキーマ推論オプション
- **Parquet** — columnar format、圧縮率高、スキーマ保持
- **JSON** — スキーマ推論可能、nested structure 対応
- **Avro** — スキーマ進化対応、binary format
- **ORC** — Hive 向け columnar format
- **Delta/Iceberg** — ACID table format（後述）

### 読み込み・書き込み例

```python
# CSV 読み込み
df = spark.read.csv("data.csv", header=True, inferSchema=True)

# Parquet 書き込み
df.write.mode("overwrite").parquet("output.parquet")

# SQL テーブルに書き込み
df.write.mode("append").saveAsTable("my_table")
```

### Managed vs Unmanaged テーブル

- **Managed**：データとメタデータが Spark Warehouse に保存
- **Unmanaged（External）**：メタデータは Spark、データはユーザー指定パス

### キャッシング（メモリ最適化）

```python
df.cache()  # メモリに保持（初アクセスで遅延ロード）
df.unpersist()  # キャッシュ削除
```

## クエリの最適化とチューニング

### Spark 3.x 以降の改革：AQE（適応的クエリ実行）

従来 Spark は実行前にクエリプラン確定し、実行中は不変でした。Spark 3.0（2020年）で**AQE（Adaptive Query Execution）**が導入され、

- **動的パーティション数調整** — シャッフル後、パーティションサイズ不適切なら自動 coalesce
- **JOIN 戦略動的選択** — 片側テーブルが予想より小さければ broadcast join に変更
- **Skew join の自動分割** — データスキューが検出されれば該当パーティション分割

### Dynamic Partition Pruning（DPP）

large_table と small_table の JOIN で、small_table フィルタ値が large_table のパーティション剪定に使われる。

### JOIN 戦略の選択

Spark は以下の順で JOIN 戦略を選択：

1. **Broadcast Join** — 片側テーブル が小さい（< spark.sql.broadcastTimeout）→ 全 executor にブロードキャスト
2. **Shuffle Hash Join** — テーブルサイズ中程度、ハッシュテーブル構築
3. **Sort Merge Join** — デフォルト・最も一般的、ソート後にマージ

### Spark UI でのプロファイリング

- **http://driver_host:4040** — Jobs、Stages、Tasks タブで実行詳細確認
- **Task Duration** — 遅いタスクがあれば data skew の可能性
- **Shuffle Metrics** — 読み書きバイト数で network 実合評価
- **Stage Duration** — 各 wide transformation の実コスト確認

**推奨設定（Spark 3.4+）：** AQE はデフォルト有効です。特別な設定不要ですが、過度な skew への `spark.sql.adaptive.skewJoin.enabled=true` 確認推奨。

## データレイクと Open Table Format

### データレイク的課題

2010年代中盤、企業は

- **Data Lake**（HDFS + Spark）と
- **Data Warehouse**（Snowflake、BigQuery）の2層

を併存していました。課題：

- ACID トランザクション不足（中間状態可視、重複問題）
- スキーマ進化への対応不足
- 変更データキャプチャ（CDC）未対応
- パーティション管理の手作業負荷

### Lakehouse Paradigm

**Lakehouse** = Data Lake の scale + Data Warehouse の ACID 特性

**Medallion Architecture** との組み合わせ：

- **Bronze**：raw data（変換なし、全カラム保持）
- **Silver**：cleaned data（スキーマ検証、重複排除、JOIN結果）
- **Gold**：business-ready（集計、モデル）

### Open Table Format の比較

**Delta Lake**（Databricks）
- Spark SQL のデフォルト format
- ACID transaction + time travel
- Parquet ベース、スキーマ進化対応
- Databricks で商用サポート

**Apache Iceberg**（Netflix等）
- schema evolution + partition evolution
- Hidden partition （内部パーティション最適化）
- Multi-engine 対応（Spark、Flink、Presto等）
- オープンネス優位

**Apache Hudi**（Uber）
- Streaming incremental processing
- record-level updates/deletes
- Copy-on-Write vs Merge-on-Read
- Streaming-heavy ワークロード向け

**現状（2025-2026）：** Iceberg がオープンネス・portability で優位、Delta が Spark コミュニティで主流、Hudi は Uber等 streaming-heavy 企業向け。

## Spark on Databricks — OSS Spark との相違

### Photon Engine

Databricks は**Photon**というプロプライエタリエンジンを開発。JVM ベース Spark の上に C++ ベース実行レイヤーを追加。

- **ガベージコレクション排除** — JVM GC pause なし
- **ベクトル化実行** — CPUベクトル命令（SIMD）活用
- **パフォーマンス** — 3~8倍高速化（TPC-DS ベンチマーク：2倍）
- **互換性** — Spark API そのまま、drop-in 置き換え

### Databricks Runtime

Apache Spark に独自の最適化を含む runtime：

- **DBR（Databricks Runtime）** — Apache Spark + Photon + 独自 optimizer
- **Dynamic allocation が強化** — YARN より高速スケール
- **統合パイプライン** — Delta Lake、MLflow、Notebooks 統合

### Databricks と自前 Spark on YARN/K8s の選択

| 観点 | Databricks | YARN | Kubernetes |
|------|-----------|------|------------|
| 初期構築コスト | 低（SaaS） | 中 | 高（DevOps重い） |
| ランニングコスト | 高（オンデマンド） | 中（既存Hadoop利用） | 中（K8s保守必須） |
| パフォーマンス | 最高（Photon） | 標準 | 標準（正設定時） |
| オープンネス | 低（ベンダーロック） | 高 | 最高 |

## デプロイモード比較

### Local モード

- **開発・テスト向け**、単一マシンで driver + executor 実行
- `spark-submit --master local[4]` で4スレッド並列

### Standalone

- Spark付属のシンプルな cluster manager
- 小～中規模クラスタ向け、オーバーヘッド最小
- HA（High Availability）には ZooKeeper 別途必要

### YARN（Hadoop Ecosystem）

- **Hadoop エコシステム統合** — HBase、Hive と共存可能
- エンタープライズ標準（既存 Hadoop 環境での自然な選択）
- **課題** — YARN scheduling latency、動的割り当ての複雑さ

### Kubernetes

- **クラウドネイティブ** — Docker コンテナベース、pod 単位
- 水平スケーリング迅速（30秒以内）、CI/CD 統合
- **課題** — 初期設定複雑、リソース監視が別必要
- パフォーマンス — YARN と同等（正しく設定時）

### Databricks

- 完全マネージド cluster manager、自動スケーリング
- Job cluster（単一ジョブ）vs All-purpose cluster（複数セッション）
- Serverless オプション（起動 < 10秒）

## まとめ — 10年腐らない Spark の原理原則

Spark は **2011年の誕生以来、基本設計は変わっていません**。これは同時に、その設計が極めて堅牢であることの証明です。

### 変わらない原理

- **遅延評価** — Action が呼ばれるまで DAG を構築し、全体最適化を可能に
- **不変性（Immutability）** — RDD/DataFrame は変更不可、新規オブジェクト生成が安全な並列処理を実現
- **DAG スケジューリング** — 細粒度 task へ分割、失敗パーティションの independent 再実行可能
- **Catalyst 最適化** — ユーザーコードはロジックに専念、Spark が自動最適化

### 技術進化の歴史

- **Spark 1.x** — RDD 中心、Python support 追加
- **Spark 2.x** — DataFrame/SQL 統一、SparkSession 導入、Tungsten whole-stage codegen
- **Spark 3.x** — AQE（適応的最適化）、DPP、GPU support 実験的追加
- **Spark 4.0+（2025）** — Photon 標準化検討、Iceberg/Delta の完全統合

### データ量増加への対応

データが 100GB → 100TB → ペタバイト規模へ増加しても、Spark の基本原理は変わりません：

- partition/parallelism の管理ルール一貫
- shuffle の物理メモリ制約は同じ（分散アルゴリズムで回避）
- Catalyst の最適化ロジック健全

### これから学ぶべき最優先事項

**重要：**

1. **DataFrame API の習得** — 将来すべての高レベルスクリプティングはこれ
2. **Catalyst 最適化の理解** — `explain()` を読む癖をつける
3. **Partition 戦略** — performance tuning の 9割はここ
4. **Open Table Format 選択** — Delta/Iceberg どちらにしろ、ACID table format は必須

## 参考資料・外部リンク

- [Apache Spark Cluster Mode Overview — Official Documentation](https://spark.apache.org/docs/latest/cluster-overview.html)
- [Deep Dive into Spark SQL's Catalyst Optimizer — Databricks Blog](https://www.databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html)
- [Performance Tuning — Apache Spark Official Guide](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [Understanding AQE and DPP in PySpark — Medium](https://medium.com/@ty.berretty/understanding-adaptive-query-execution-aqe-and-dynamic-partition-pruning-dpp-in-pyspark-53c1a255e82c)
- [Delta Lake vs Iceberg vs Hudi — Open Table Format Comparison](https://www.onehouse.ai/blog/apache-hudi-vs-delta-lake-vs-apache-iceberg-lakehouse-feature-comparison)
- [Databricks Photon Engine — Official Product Page](https://www.databricks.com/product/photon)
- [Mastering Spark Partitioning: Coalesce vs Repartition — Big Data Performance](https://bigdataperformance.substack.com/p/mastering-apache-spark-partitioning)
