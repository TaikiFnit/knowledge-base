# Unity Catalog ガバナンス — モダンガバナンススタック｜Ch.1 学習ノート

> **UNITY CATALOG GOVERNANCE — CHAPTER 01**
>
> Ch.1: データガバナンスの基礎、レイクハウスの進化、Unity Catalogによる統合ガバナンス
>
> Kiran Sreekumar & Karthik Subbarao 著（O'Reilly, 2025）

📘 **Data Governance with Unity Catalog on Databricks — Chapter 1 of 12**

| Stat | Value |
|------|-------|
| 全章数 | 12 Chapters |
| ページ数 | 384ページ |
| 出版年 | 2025（O'Reilly Media） |
| Key Topic | Databricks Unity Catalog |

学習日: 2026-04-01 ｜ 読了目安: ~30分

---

## 目次

1. [データガバナンスとは](#01-データガバナンスとは)
2. [データのライフサイクル](#02-データのライフサイクル)
3. [非構造化データとAIアセットのガバナンス](#03-非構造化データとaiアセットのガバナンス)
4. [規制への対応](#04-規制への対応)
5. [レイクハウスの夜明け](#05-レイクハウスの夜明け)
6. [レイクハウスパラダイム](#06-レイクハウスパラダイム)
7. [Unity Catalog: 統合ガバナンスの実現](#07-unity-catalog-統合ガバナンスの実現)
8. [Databricksプラットフォームアーキテクチャ](#08-databricksプラットフォームアーキテクチャ)
9. [データ共有とコラボレーション](#09-データ共有とコラボレーション)
10. [まとめ — Ch.1の要点](#10-まとめ--ch1の要点)
11. [参考文献](#11-参考文献)

---

**シリーズ：** Unity Catalog ガバナンス 学習ノート（全12章）
- **Ch.1 — モダンガバナンススタック** ◀ 現在の記事
- Ch.2 — Under The Hood（Coming Soon）

---

## 01. データガバナンスとは

> "Data is the new oil." — Clive Humby

第1章は、データがなぜ「新しい石油」と呼ばれるほど価値を持つようになったのか、そしてその価値あるデータを**適切に管理しなかった場合に何が起きるか**を実世界のインシデントで示すところから始まる。

### 実世界のインシデント

| インシデント | 詳細 |
|------------|------|
| **Securitas事件（2021年10月）** | SafetyDetectivesが、ストックホルム本社のSecuritasの保護されていないAmazon S3バケットを発見。100万件以上のファイルが漏洩し、空港従業員の機密データが含まれていた。 |
| **Capital One漏洩事件（2019年7月）** | クラウドファイアウォール設定の脆弱性を突かれ、1億件以上の顧客レコードにアクセスされた。 |
| **S3設定ミスによる大量漏洩** | ヒューマンエラーによる設定ミスで約2,300万件のファイルが漏洩。トルコのLPPD違反。 |

### 重要統計データ

- **82%** — セキュリティ侵害のうち人的要素が関与（Verizon 2022 DBIR）
- **>50%** — 重大サイバーインシデントのうち人材不足・人的エラーが原因になるとGartnerが予測

### データガバナンスの定義

> **Data governance** とは、組織のデータを管理するための包括的なアプローチであり、データの**可用性（availability）**、**使用可能性（usability）**、**完全性（integrity）**、**セキュリティ（security）**を確保する。

### データガバナンスの主要コンポーネント

1. **ポリシーと標準** (Policies and Standards) — データの取り扱いルール・組織横断的な標準
2. **データ品質管理** (Data Quality Management) — データの正確性・完全性・一貫性の継続的な監視・改善
3. **メタデータ管理** (Metadata Management) — データに関するデータの一元管理
4. **アクセス制御とセキュリティ** (Access Controls & Security) — 暗号化、ファイアウォール、侵入検知、データ最小化
5. **データライフサイクル管理とリネージ** (Lifecycle Management & Lineage) — データの生成から廃棄までの全過程の追跡
6. **監査とコンプライアンス監視** (Auditing & Compliance Monitoring) — すべてのデータ操作の記録

### 効果的なデータガバナンスの利点

- 人的エラーの削減
- 強力なアクセス制御と認証の統合的な実装
- データ分類とリスク評価
- データ取扱ポリシー（保存、処理、転送、廃棄）
- データ品質と信頼性の向上
- 規制コンプライアンスの強化
- 運用効率の向上とコスト削減
- 消費者からの信頼向上

ガバナンスは**人間中心（human-centric）**であることが重要。定期的な従業員トレーニング、明確な役割と責任の定義が不可欠。

---

## 02. データのライフサイクル

データには4つの主要なライフサイクルステージがある。

### 4段階のライフサイクル

| Stage | 名称 | 説明 |
|-------|------|------|
| 1 | **Data Creation / Ingestion** | データは3つの方法で生まれる：収集（ユーザーサインアップ等）、生成（マシンシグナル等）、取り込み（外部データ） |
| 2 | **Processing** | クレンジング、結合・マージ、集計、匿名化 |
| 3 | **Active Duty** | アプリケーション、レポーティング、高度なアナリティクスでの利用 |
| 4 | **Offboarding** | 規制要件とコストに基づき、パージ（完全削除）またはアーカイブ（長期保存） |

### 処理段階の重要な概念

- **Entity Resolution（エンティティ解決）**: 複数のシステムから同一のデータを統合し、冗長性を排除
- **Data Harmonization（データ調和）**: 様々なソースからのデータを一貫性のある標準化されたフォーマットに統一
- **Aggregation（集約）**: 大規模データセットから意味のある情報を抽出
- **Anonymization（匿名化）**: コンプライアンス要件に基づくデータのマスキング（墨消しまたは仮名化）

### PII と PHI の区別

- **PII** (Personally Identifiable Information): 個人を特定できるあらゆる情報（氏名、住所、社会保障番号、メールアドレス等）
- **PHI** (Protected Health Information): PIIのサブセット。医療分野における個人の健康情報。米国ではHIPAAにより保護される。

---

## 03. 非構造化データとAIアセットのガバナンス

従来のガバナンスは構造化データを主な対象として設計されてきた。しかし：

- **組織データの80%以上**が非構造化データ（メール、テキストファイル、ログ、メディアファイル等）
- 管理されない非構造化データは**データ沼（data swamp）**を形成
- AIの台頭により、非構造化データの価値が急速に高まっている

### AIが拡大するガバナンス対象

AIは以下の領域をカバーする：
- **予測分析（Predictive Analytics）** — 過去データから将来を予測
- **処方分析（Prescriptive Analytics）** — 最適なアクションを推奨
- **従来型ML（Traditional Machine Learning）** — 分類・回帰等
- **生成AI（Generative AI）** — テキスト・画像・コードの自動生成

### AIアセットという新しいガバナンス対象

データセットを用いて学習されたMLモデルは**AIアセット（AI assets）**であり、データとは異なる新しい種類のアセットだ。これらにもガバナンスが必要だが、既存のガバナンスソリューションの多くは対応できていない。Unity CatalogはデータとAIアセットの**両方**を統合的にガバナンスする。

---

## 04. 規制への対応

ChatGPTがAIを一般大衆に普及させたことで、AI規制が世界的に加速している。

| 規制 | 概要 |
|------|------|
| **EU AI Act** | 2024年7月公布。AIの利用を規制。リスクレベル別分類。世界初の包括的AI規制。 |
| **GDPR** | EU域内の個人データ保護規制。データ主体の権利（アクセス権、削除権、ポータビリティ権）を定義。 |
| **CCPA** | カリフォルニア州のプライバシー保護法。消費者にデータの収集・共有・削除に関する権利を付与。 |

モダンなガバナンスソリューションは、データとAIアセットを含む拡大した規制範囲に包括的に対応する必要がある。

---

## 05. レイクハウスの夜明け

### BI vs 高度なアナリティクス

| 観点 | BI | Advanced Analytics |
|------|-----|-----|
| インサイト | 過去・現在から | 予測・処方的 |
| 出力 | レポート、ダッシュボード | MLモデル、統計分析 |
| データ型 | 構造化データ | 半構造化・非構造化データ |
| 実行環境 | データウェアハウス | データレイク |

### ビッグデータの3V

> Doug Laney（Gartner, 2001）がビッグデータの3つのV—**Volume**（量）、**Velocity**（速度）、**Variety**（多様性）—を定義した。

VolumeとVelocityはコモディティ化された。最も挑戦的なのは**Variety（多様性）**。

### データウェアハウス vs データレイク

| 観点 | Data Warehouse | Data Lake |
|------|---------------|-----------|
| 主な用途 | BIとレポーティング | 高度なアナリティクス、ML |
| データ型 | 構造化データ | 半構造化・非構造化データ |
| 代表例 | Azure Synapse, Redshift, BigQuery | S3, ADLS, GCS |

> "If you think of a data mart as a store of bottled water — cleansed and packaged, and structured for easy consumption — the data lake is a large body of water in a more natural state." — James Dixon（Pentaho元CTO）

**Data Mart（データマート）**: 単一のビジネスユニット向けにキュレーションされたデータベース。

### Hadoop: 偽りの夜明け

- 著者Karthikは2010〜2011年頃、フィンテックスタートアップでHadoopに携わった
- HDFS + MapReduce — 分析データのパラダイムシフト
- Cloudera, MapR, Hortonworksが設立。LinkedIn, Yahoo, Facebookが採用
- **問題**: ディスクベース = 現代のシステムと比較して低速 → レガシー化
- **良い結果**: UC BerkeleyでApache Sparkが誕生

### データサイロとフラクチャードガバナンス

- 複数の事業部門が異なるツール・プラットフォームを選択 → **データサイロ**形成
- 2つの分離されたスタック（ウェアハウス + レイク）= **分断されたガバナンス**（fractured governance）
- 各スタックに異なるアクセス制御、メタデータ管理、監査ログ → 一貫したガバナンスが不可能

---

## 06. レイクハウスパラダイム

> **Lakehouse** とは、データレイクとデータウェアハウスの最良の要素を組み合わせた**オープンなデータ管理アーキテクチャ**であり、単一のプラットフォーム上で統合ガバナンスとAI駆動アナリティクスを実現する。

目標：
1. すべてのデータワークロード（BI、高度なアナリティクス等）
2. すべてのデータ型（構造化、半構造化、非構造化）
3. 統合されたセキュリティとガバナンス

「data lakehouse」が初めて文書化されたのは2017年。

### オープンテーブルフォーマット

| フォーマット | 開発元 | 特徴 |
|------------|--------|------|
| **Apache Hudi** | Uber | インクリメンタル処理に強み |
| **Apache Iceberg** | Netflix | 大規模データ向け設計 |
| **Delta Lake** | Databricks | Sparkエコシステムの基盤 |

すべてACIDトランザクション、スキーマ進化、タイムトラベル、効率的なメタデータ管理を提供。

### ACID vs BASE

| 特性 | ACID | BASE |
|------|------|------|
| 正式名称 | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventually consistent |
| 一貫性モデル | 強整合性 | 結果整合性 |
| 適用場面 | トランザクション処理、金融データ | 高可用性が優先される分散システム |

### メダリオンレイクハウスアーキテクチャ

**Medallion architecture**（マルチホップアーキテクチャ）— データ品質が段階的に向上するレイヤード処理パターン。

| Layer | 名称 | 説明 |
|-------|------|------|
| **Bronze** | ブロンズ | ソースシステムからの生データ。変換は最小限。 |
| **Silver** | シルバー | 検証・変換・エンリッチメント済み。重複排除、スキーマ適用。 |
| **Gold** | ゴールド | 読み取り最適化。BIダッシュボード、KPI、MLフィーチャーストア向け。 |

### Databricks Data Intelligence Platform

コアコンポーネント：
1. **コスト効率の高いオブジェクトストレージ** — すべてのデータ/AIアセット
2. **オープンソースストレージレイヤー（Delta Lake）** — ACID、スキーマ進化、タイムトラベル
3. **統合カタログ（Unity Catalog）** — メタストア、ACL、監査、リネージ
4. **インテリジェンスエンジン** — データ・コード・使用状況のコンテキスト活用
5. **全ワークロード対応** — BI、データエンジニアリング、データサイエンス、ML、リアルタイム

---

## 07. Unity Catalog: 統合ガバナンスの実現

### タイムライン

| 時期 | イベント |
|------|---------|
| 2021 | Data and AI Summit で発表 |
| 2022 Summit | Azure/AWS向けゲーテッドパブリックプレビュー |
| 2022年8月 | 一般提供開始（GA） |

### なぜUnity Catalogが必要か

Unity Catalog以前は、Hive Metastore（HMS）がメタデータリポジトリとして使用され、アクセス制御はワークスペースごとに管理されていた。数十〜数百のワークスペースを運用する段階で、**複雑で、エラーが発生しやすく、スケーラブルではない**という限界を露呈した。

### Unity Catalogの特性

- **Operational（運用カタログ）**: メタデータ管理を超えた統合運用
- **Unified（統合）**: 全データ/AIアセットに対する単一のガバナンスモデル。Delta, Iceberg, Hudi, CSV, JSON, Parquet, テキスト, バイナリ対応
- **Open（オープン）**: オープンソース、ベンダーロックインなし。Credential vending、Catalog federation対応

> **UC OSS** = スタンドアロンの統合データ/AIカタログ
> **Databricks UC** = Databricksプラットフォームに組み込まれたガバナンスソリューション

### データ/AIアセットの種類

| アセット種別 | 説明 |
|------------|------|
| **Tables** | 行と列で構成される構造化データ |
| **Views** | テーブル/ビューから作成される読み取り専用データ |
| **Volumes** | 任意のファイルの論理エンティティ（非構造化データ対応） |
| **Models** | AIモデル（MLflow管理） |
| **Functions** | ユーザー定義ロジック（UDF） |
| **Foreign Catalogs** | 外部DB、ウェアハウス、カタログ（例：AWS Glue） |

### コアガバナンス機能（6つ）

1. **Identity Management（ID管理）** — Entra ID、Google、Okta等と統合
2. **Catalog（カタログ）** — 全データ/AIアセットのメタストア
3. **Access Control（アクセス制御）** — アイデンティティに対する認可
4. **Search and Discovery（検索と発見）** — 自然言語検索
5. **Audit（監査）** — 全操作の自動ログ記録
6. **Lineage（リネージ）** — テーブル、ビュー、ボリューム、モデルのリネージ自動生成

### 高度なガバナンス機能（3つ）

1. **Row-level filtering & Column-level masking（CLM）** — 機密データのきめ細かな保護
2. **Attribute-Based Access Control（ABAC）** — アイデンティティ属性、リソース属性、リクエスト属性に基づく条件付きアクセス制御
3. **Data Monitoring** — テーブルレベルの統計的特性とデータ品質の自動モニタリング

---

## 08. Databricksプラットフォームアーキテクチャ

### 4つの主要コンポーネント

| コンポーネント | 説明 |
|--------------|------|
| **Users & Applications** | プラットフォームにアクセスするユーザー、サービスプリンシパル、外部アプリ |
| **Control Plane** | Databricks管理サービス：Web UI、クラスター管理、ワークフロー管理、UCサービス、ノートブックサーバー |
| **Compute Plane** | データ処理の実行場所。サーバーレス（Databricks管理）またはクラシック（顧客管理）。**Databricks Runtime（DBR）**が最適化されたSparkを提供 |
| **Storage** | 全データ/AIアセットの永続化。クラウドオブジェクトストレージ |

**DBR注意点**: Unity Catalogとの連携にはDBR 11.3以上が必要。最新LTS推奨。一部機能（Databricks SQL Serverless等）はSparkを直接使用しない。

### Unity Catalogの3コンポーネント

1. **Unity Catalog UI** — Databricks UIに統合されたインターフェース
2. **Unity Catalog Service** — コントロールプレーン上で稼働するバックエンドサービス（REST API提供）
3. **Databricks Runtime統合** — クエリ実行時にUCサービスと連携し、アクセス権限確認・監査ログ記録を自動実行

---

## 09. データ共有とコラボレーション

### Delta Sharing

**Delta Sharing**はセキュアなデータ共有のための**オープンプロトコル**。オープンソースでスタンドアロン利用可能。Databricks上ではマネージドソリューションとして提供。

プロバイダーとレシピエント間に信頼関係を確立し、Unity Catalogがアクセス制御とガバナンスを適用。

### Delta Sharing上に構築されたサービス

- **Databricks Marketplace** — データプロダクトの公開・消費のためのオープンフォーラム
- **Databricks Clean Rooms** — 機密データを共有するためのプライバシー保護環境。複数組織が生データを公開せず共同分析を実行

詳細は第8章で解説。

---

## 10. まとめ — Ch.1の要点

1. **ガバナンスはデータの量と多様性の増大に比例して重要になる** — 82%のセキュリティ侵害に人的要素が関与。human-centricなガバナンスが必要。

2. **データには明確なライフサイクルがあり、各段階でガバナンスが必要** — 作成・取得 → 処理 → 利用 → 退役の4段階すべてで管理が求められる。

3. **非構造化データとAIアセットがガバナンスの新たなフロンティア** — 80%以上が非構造化。MLモデル等のAIアセットもガバナンス対象に。

4. **2つの分離されたスタックは分断されたガバナンスを生む** — ウェアハウスとレイクの並行運用はデータサイロを形成。

5. **レイクハウスパラダイムが統合データ管理の解答** — データレイクの柔軟性とウェアハウスのACID/パフォーマンスを統合。メダリオンアーキテクチャで品質を段階的に向上。

6. **Unity Catalogはレイクハウスの統合ガバナンスレイヤー** — 運用的・統合的・オープン。6つのコア機能 + 3つの高度な機能。

7. **Delta Sharingがセキュアなデータコラボレーションを実現** — Marketplace + Clean Rooms。データの価値は共有によって倍増。

> データガバナンスは単なるセキュリティ対策ではなく、組織の**データ資産から価値を引き出す能力そのもの**を決定する。レイクハウスパラダイムはデータの格納・処理の統合を実現し、Unity Catalogはその上で統合ガバナンスを適用する。

---

## 11. 参考文献

1. Kiran Sreekumar, Karthik Subbarao, *Data Governance with Unity Catalog on Databricks*, O'Reilly Media, Sep 2025. ISBN: 9781098179625 — https://learning.oreilly.com/library/view/data-governance-with/9781098179625/
2. Verizon, *2022 Data Breach Investigations Report (DBIR)* — https://www.verizon.com/business/resources/reports/dbir/
3. Clive Humby — "Data is the new oil" (2006)
4. Doug Laney, "3D Data Management: Controlling Data Volume, Velocity and Variety," Gartner, 2001
5. James Dixon (former CTO of Pentaho) — Data lake vs data mart water analogy
6. Databricks — Unity Catalog documentation — https://docs.databricks.com/en/data-governance/unity-catalog/index.html
7. European Union — EU AI Act (July 2024) — https://artificialintelligenceact.eu/
8. Apache Spark, Delta Lake, Apache Hudi, Apache Iceberg — https://spark.apache.org/

---

*本記事は Kiran Sreekumar & Karthik Subbarao 著『Data Governance with Unity Catalog on Databricks』（O'Reilly Media, 2025）の個人学習ノートです。書籍の内容を要約・解説したものであり、著作権は原著者および O'Reilly Media に帰属します。*
