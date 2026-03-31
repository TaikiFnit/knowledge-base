# Data Engineering Summit「激動の2025年、Modern Data Stackの最新技術動向」要約

2025年11月6日、ファインディ株式会社（Findy）主催の [Data Engineering Summit](https://data-engineering-summit.findy-tools.io/2025) にて、クラスメソッド株式会社の相樂 悟（さがら）氏がSpecial Speaker枠として登壇した。セッションタイトルは「激動の2025年、Modern Data Stackの最新技術動向」。同年9月30日のDES#31で披露した10分クイズ形式のMDS動向紹介が大きな反響を呼んだことを受け、同テーマを45分のフルセッションへと大幅に拡充した、いわば「正式版・完全版」である。アーカイブ動画はFindy Conferenceのプラットフォーム上で登録後に視聴可能（YouTube非公開）となっており、本記事はスライド44枚の内容とClassmethodブログに掲載された登壇レポートのQ&Aを元に構成している。

- [アーカイブ視聴（要登録）](https://conference.findy-code.io/conferences/data-engineering-summit/8/archives/245)
- [スライド（Speaker Deck）](https://speakerdeck.com/sagara/ji-dong-no2025nian-modern-data-stacknozui-xin-ji-shu-dong-xiang)
- [登壇レポート（DevelopersIO）](https://dev.classmethod.jp/articles/data-engineering-summit-modern-data-stack-2025/)
- [関連記事: DES#31 公開企画会議](/articles/data-engineering-study-31-advisory.html)

---

## 登壇者: 相樂 悟（さがら）

**会社:** クラスメソッド株式会社  
**役職:** Modern Data Stackチーム テックリード  
**入社:** 2020年9月  
**資格:** Snowflake Data Superhero (2025)、dbt Cloud Architect認定  
**Twitter/X:** [@SS_chneider](https://twitter.com/SS_chneider)  
**Speaker Deck:** [speakerdeck.com/sagara](https://speakerdeck.com/sagara)

MDS製品の技術支援・プリセールスを担当しながら、2週間ごとに各製品のブログ・リリースノートをチェックし、最新技術情報をDevelopersIOで発信し続けるエキスパート。業界横断的なMDSエコシステムの動向把握においては国内屈指の専門家の一人である。今回のData Engineering SummitではSpecial Speaker（招待枠）として登壇した。

<iframe class="speakerdeck-iframe" style="border: 0px; background: padding-box padding-box rgba(0, 0, 0, 0.1); margin: 0px; padding: 0px; border-radius: 6px; box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 40px; width: 100%; height: auto; aspect-ratio: 560 / 315;" frameborder="0" src="https://speakerdeck.com/player/2477887347934f5ca930fc26f5071267" title="激動の2025年、Modern Data Stackの最新技術動向" allowfullscreen="true" data-ratio="1.7777777777777777"></iframe>

---

## Modern Data Stackとは

さがら氏はMDSを「**最新のクラウドネイティブの技術やサービスを組み合わせたデータ基盤、あるいはその考え方や該当するサービス群**」と定義する。重要なのは「特定の製品」ではなく「組み合わせ方の思想」であるという点だ。

本セッションでは、さがら氏が日常的にウォッチしている**9分野・50以上の製品**のうち、2025年に特に動きが大きかったトピックを厳選して解説した。調査サイクルは2週間ごとのブログ・リリースノートチェックという、継続的かつ体系的なもの。

### 調査対象カテゴリと製品一覧

| カテゴリ | 主な対象製品 |
|---|---|
| Data Extract/Load | Fivetran、Airbyte、Meltano、Snowplow、dlt、Omnata |
| Data Warehouse/Data Lakehouse | Snowflake、BigQuery、Redshift、Databricks、ClickHouse、Onehouse、DuckDB/MotherDuck、Starburst、LakeFS、Bauplan |
| Data Transform | dbt、Datacoves、Dataform、Coalesce |
| Semantic Layer | dbt Semantic Layer、Cube、AtScale、Honeydew |
| Business Intelligence | Looker、Tableau、Power BI、Sigma、ThoughtSpot、Hex、Preset、Lightdash、Omni、Steep、Evidence |
| Data Catalog | Atlan、Select Star、Secoda、Alation、OpenMetadata |
| Data Activation | Hightouch、Census |
| Data Observability/Quality | Monte Carlo、Sifflet、Acceldata、Decube、Anomalo、Soda、Datafold、Great Expectations、Recce、Elementary、re_data |
| Data Orchestration | Dagster、Prefect、Astronomer(Airflow)、Kestra、Mage、Orchestra |

---

## 2025年の企業買収・合併 — エコシステムの収束

2025年のMDS界隈を語るうえで最も象徴的なのが**買収・合併の嵐**だ。DES#31（2025年9月）の時点でさがら氏は「2025年だけで8社が買収された」と述べていたが、今回の45分セッションではその背景と各社の狙いをより詳細に掘り下げた。

### 主要な買収・合併

| 買収元 | 対象 | 時期 | 意義 |
|---|---|---|---|
| dbt Labs | SDF | 2025年1月 | Rust製データ変換エンジンを獲得 → dbt Fusionとして統合 |
| Fivetran | Census | 2025年5月 | Reverse ETL機能を取り込み |
| Fivetran | Tobiko Data | 2025年9月 | SQL Mesh（データ変換）開発会社を統合 |
| Salesforce | Informatica | 2025年5月 | 包括的データ統合管理ソリューションを傘下に |

中でも**Fivetranの動き**は業界に衝撃を与えた。もともと「データロードに特化したツール」だったFivetranが、リバースETL（Census）とデータ変換（Tobiko Data/SQL Mesh）を相次いで買収したことで、パイプラインの上流から下流まで一貫して担うプラットフォームへと急速に変貌した。DES#31でさがら氏が「FivetranがリバースETLとデータ変換を取り込んだのは衝撃だった」と語っていた通りの展開だ。

2025年10月のdbt Coalesce（年次カンファレンス）では、Fivetranとdbt Labsが共同で新ビジョン **「Open data infrastructure」** を発表。買収・統合という単純な話にとどまらず、エコシステム全体の設計思想が再定義されつつある。

DES#31時点でさがら氏が予告していた「**Data Catalog・Observability分野での買収継続**」については、DES#31時点でのMetaphor、CastorDoc、Satori、Metaplaneの買収に続き、本セッションでも2026年以降の動向予測として引き続き言及されている。

---

## Data Extract/Load

### セキュアネットワーク接続の主流化

オンプレミスやVPC内のデータソースへの接続において、**Dockerコンテナエージェントを用いたアーキテクチャが主流**になってきた。

- **Fivetran Hybrid Deployment**: 自社ネットワーク内にエージェントをデプロイし、Fivetranのクラウド管理機能と組み合わせるハイブリッド構成
- **Airbyte Enterprise Flex**: 同様のコンテナエージェント型でセキュアな接続を実現

従来の「全データをクラウドベンダーのネットワーク経由で転送」ではなくなり、セキュリティ要件の厳しい企業でも採用しやすくなった。

### GenAI × データロードの登場

生成AIとのインテグレーションが、データ取り込みの開発体験を変え始めた。

- **Fivetran**: Claude Codeを活用することで、**1時間未満でカスタムコネクタを開発**できるデモを披露
- **dlt（data load tool）**: Claude Codeでパイプラインを構築し、DuckDBに格納してVisivoダッシュボードで可視化するまでのフローをデモ

コネクタ開発というこれまで専門的なスキルが必要だった作業が、LLMの支援によって大幅に民主化されつつある。

### SnowflakeとDatabricksのExtract/Load参入

主要DWHベンダーもデータ取り込み層への展開を強化している。

- **Snowflake Openflow**: 非構造化データ・構造化データの双方に対応したデータ取り込みフレームワーク
- **SAP Business Data Cloud**: ゼロコピーによる双方向データ共有をSAPとSnowflake/Databricks間で実現

---

## Data Warehouse / Data Lakehouse

2025年のデータウェアハウス・レイクハウス分野は、**AI機能の統合**・**OLTP対応**・**オープンフォーマット**の3軸で大きく動いた。

### AI導線の強化 — 各社の自然言語BI

3大クラウドDWHがほぼ同時期に「自然言語でデータにアクセスできる」機能を打ち出した。DES#31ではさがら氏がこの対比を「SnowflakeはシンプルさResp、DatabricksはUnity Catalogを軸としたエンジニア向け」と整理していたが、今回の45分セッションではより具体的な機能の詳細が示された。

| プラットフォーム | 機能名 | 内容 |
|---|---|---|
| Snowflake | Snowflake Intelligence | Cortex Search・Cortex Analystを統合したエージェント型BI |
| Databricks | Databricks One | 自然言語入力でダッシュボードを生成・表示 |
| BigQuery | Gemini Enterprise | Data Insights agentプレビュー（Gemini連携） |

### 開発者向け機能の拡充

- **Snowflake**: Git連携IDE「Workspace」、「dbt Projects on Snowflake」（Snowflake UI上でdbtプロジェクトを直接管理）
- **Databricks**: 「Lakeflow Pipelines Editor」によるパイプライン構築UIの改善

### OLTP対応 — DWHからの「越境」

データウェアハウスが、従来はRDBMSの領域だったOLTP（オンライントランザクション処理）への対応を本格化させた。

- **Snowflake Postgres**: 未リリースながら100% PostgreSQL互換を目指す。既存のPostgreSQLアプリをそのまま接続できる
- **Databricks Lakebase**: プレビュー段階。低レイテンシ（10ms未満）・高同時実行（10,000 qps超）を実現。Neonの技術を採用し、ブランチ機能でdev/prodの分離が可能

OLAP（分析）とOLTP（トランザクション）の境界が溶け始めており、「分析用DWHと業務系RDBは別物」という常識が揺らいでいる。

### 非構造化データへの対応

- **FILE型の一般提供**: PDFや画像などのファイルをDWH内にネイティブに保存・管理
- **AI_EXTRACT関数**: PDFなどの非構造化ドキュメントから情報を抽出して構造化データとして扱う

### Apache Iceberg — オープンフォーマットの覇権

| プラットフォーム | Iceberg対応 |
|---|---|
| Snowflake | 外部Iceberg RESTカタログへの書き込みサポート |
| Databricks | Unity Catalog Managed Iceberg tablesがパブリックプレビュー |

主要クラウドDWHがいずれもIcebergに注力しており、「データはユーザー管理のS3に持つ」というアーキテクチャの実現可能性が高まっている。詳細はQ&Aセクションも参照。

### 自動最適化機能

- **Amazon Redshift**: MDDL（Materialized Data Distribution Layer）が一般提供。動的なソートキー最適化を自動化
- **BigQuery**: CMETA（自動列メタデータインデックス）により、クエリ最適化をシステム側が自律的に行う

### 注目の新興プレイヤー

- **Onehouse — OneFlow**: データをユーザー自身のストレージに保存し、OneSyncによる同期機能を提供。ベンダー依存を下げた設計
- **DuckDB/MotherDuck — DuckLake**: DBファイルのメタデータをDuckDBで管理する新しいレイクハウスフォーマット。Icebergに対するオルタナティブとして注目

---

## Data Transform / Semantic Layer

### dbt Fusion — 競合買収が生んだ革新

2025年1月にdbt Labsが買収したSDF（Rust製データ変換エンジン）が、**dbt Fusion**として製品化された。BetaからPublic Previewへと進んだdbt Fusionの主な特徴：

- **Rustベースの高速エンジン**: 大規模プロジェクトでの実行速度が大幅に向上
- **カラムレベルリネージ**: モデル間での列単位のデータ系譜を追跡可能に
- **SDF由来の静的解析**: SQL実行前の検証が可能

dbtの競合として独自路線を歩んでいたSDFが、dbt自身に買収されて「Fusion」として組み込まれるというMDSエコシステムの収束を象徴する動きだ。

### dbt × AI の盛り上がり

2025年を通じてTokyo dbt Meetupで連続開催されたLLM×dbtをテーマとしたイベントが示すように、dbtとAIの組み合わせへの関心が国内でも急上昇している。

### Open Semantic Interchange（OSI）

**Snowflake・Salesforce・dbt Labsの3社が主導**するSemantic Layerの標準化プロジェクト。各ツール間でセマンティック定義を相互に交換可能にすることを目指す。各社がそれぞれ独自のSemantic Layer実装を持っている現状に対し、相互運用性を確保するための業界標準として注目される。

---

## Business Intelligence

### Tableau Next

SalesforceがTableauの次世代版として発表した**Tableau Next**。Salesforceプラットフォームとの深い統合を前提とした設計で、AIアシストによる分析機能が中心となる。Salesforceによる一連のデータ統合戦略（Informatica買収、OSI主導なども含む）の一環として位置づけられる。

### Looker

Googleが継続的に機能強化を進めている。

- **Conversational Analytics**: 自然言語での問い合わせに対応
- **period_over_period**: LookMLで前期比較を宣言的に定義できる新機能。期間比較の定義コストが大幅に低下

### 資金調達が続くBI新興勢

| ツール | 調達ラウンド | 時期 | 金額 | 備考 |
|---|---|---|---|---|
| Hex | シリーズC | 2025年5月 | **7,000万USD** | データサイエンス・AI統合型ノートブックBI |
| Omni | シリーズB | 2025年3月 | **6,900万USD** | 評価額6.5億USD（前年比8倍）、創業3年 |

特にOmniの評価額の急成長（前年比8倍）は業界の注目を集めた。その理由についてはQ&Aセクションで詳述する。

---

## Data Catalog / Data Orchestration

### MCPサーバーという革命

2025年にAI界隈を席巻したMCP（Model Context Protocol）が、Data Catalogツールにも波及した。

**Atlan・Select Star・Secoda・OpenMetadataなど主要製品が軒並みMCP Serverをリリース**。これによって、AIエージェントからデータカタログに収録されたメタデータ・系譜・ドキュメント情報に自然言語でアクセスできるようになった。「データカタログはメタデータを人間が閲覧するもの」から「AIエージェントがコンテキストを取得するもの」へと役割が拡張されつつある。

### Kestraの台頭

オーケストレーションツールにおいて**Kestra**が存在感を急速に高めた。

- **2025年9月にv1.0をリリース**
- **GitHubスターでDagster・Prefectを超える**急成長
- **YAML宣言型**によるシンプルなパイプライン定義が特徴

従来のAirflow・Dagster・Prefectという三強に対して、シンプルさと成長スピードで割って入った形だ。

---

## 2026年以降の動向予想

さがら氏はセッションの締めくくりとして、2026年以降の見通しを4点に整理した。

### 1. 買収・統合の継続

**Data Catalog・Data Observabilityの分野**で買収が継続すると予測。DES#31でさがら氏が名指しで注目していたAtlan、OpenMetadata、Monte Carloといったプレイヤーの動向が引き続き焦点となる。2025年に8社が動いたのに続き、エコシステムのさらなる収束が進む見込みだ。

### 2. Snowflake / Databricks 2強構図の継続

両社が機能・生態系ともに他を引き離した状態が続く。DES#31のパネルディスカッションで「どちらを選んでも外れない」と語ったさがら氏の認識は本セッションでも一貫しており、技術的な優劣よりも「フィーリングと組織適合」が選定基準になるという状況は変わらないとみられる。

### 3. AI-Ready基盤への意識が必須に

2026年以降、データ基盤の構築指針として**AI-Readyであること**が前提条件になる。具体的には以下の5つの要素が挙げられた：

- **データモデリング**: 整合性の高い論理モデルの整備
- **Semantic Layer**: メトリクス・指標定義の統一
- **データ品質**: 信頼できるデータだけをAIに渡す
- **非構造化データ対応**: PDFや画像も含めた情報の活用
- **ドメイン知識のドキュメント化**: AIのコンテキストとして機能させる

### 4. 技術は手段

「技術は手段であり、**データ基盤の普及と事業貢献への意識**が本質的に重要」というメッセージ。ツールの最新動向を追うことよりも、それを使って何を実現するかという視点を失わないことが肝要だと結んだ。

---

## Q&Aセッション

アーカイブが視聴登録制のため、ここで紹介する[Classmethod登壇レポート（DevelopersIO）](https://dev.classmethod.jp/articles/data-engineering-summit-modern-data-stack-2025/)のQ&A記録が「スライドに映らない知見」として特に価値を持つ。さがら氏の実務経験と日常的なプロダクトウォッチから生まれた、率直な回答が読みどころだ。

---

### Q1. SnowflakeとDatabricksにベンダーロックインしないためにはどういうことができますか？

> **A.** Icebergなどのオープンテーブルフォーマットを取り入れることで、データ自体はユーザー管理のS3に保持されるため移行もしやすく、別のクエリエンジンからも柔軟にクエリができる。ベンダーロックインを多少避けることはできる。

DWH選定で避けがたいベンダーロックインの問題に対し、さがら氏は**Icebergによるデータ層の分離**を現実的な解として提示した。アプリケーション層はSnowflakeやDatabricksに乗りながら、データそのものはS3という中立ストレージに保持することで、将来的なクエリエンジンの乗り換えコストを下げるアプローチだ。

---

### Q2. IcebergがレイクハウスフォーマットのDe Facto Standardとなるのでしょうか？

> **A.** ここ1〜2年の動きを見ていると、AWS・Google Cloud・Snowflake・Databricks、いずれもIcebergに最も力を入れている印象があるため、今はIcebergが筆頭。一方で、DuckLakeのように新しいレイクハウスフォーマットも出てきているので、Icebergを見つつ、類似技術の動向はウォッチすることをおすすめ。

クラウド4強がIceberg推しで足並みを揃えているという事実は重い。ただし「De Facto = 競争終了」ではなく、DuckLakeのような新参者も登場しているという現実を踏まえた、バランスの良い回答だ。

---

### Q3. Omniはなぜそこまで伸びているのでしょうか？

> **A.** これまでのセルフサービス型BIとSemantic Layer型BIの良いところ取りをしているから。セルフサービス型BIはすぐに分析できるがガバナンスが難しい。Semantic Layer型BIは定義を統一できるがスピードが落ちる。Omniは「GUIベースですぐにデータに繋いで計算フィールドや結合定義を行い、裏側でSemantic Layerのコードに変換される」アプローチがユーザーから評価を受けている。

Omniの評価額が創業3年で前年比8倍に達した理由の核心がここにある。GUIで触ればすぐ動き、裏側では定義が自動的にコード化されてガバナンスも確保されるという「速さと整合性の両立」が市場に刺さった。セルフサービスBIが抱えてきた長年の課題への実用的な解答として支持を集めている。

---

### Q4. Semantic Layerでは何がおすすめでしょうか？

> **A.** 最近触った中でOmniをお勧め。豊富なAI用コンテキスト設定機能があり、Omni内でも多くの生成AIを用いた機能がある。有償なので気軽に試すならSnowflakeのSemantic View、DatabricksのUnity CatalogのMetric Viewを第一歩として。

実際に触った経験に基づく率直な推薦だ。「まずは既存DWHに付属の機能で体験して、本格導入ならOmni」という段階的アプローチの提示は実務的に参考になる。OmniがQ3（評価額急成長）とQ4（専門家推薦）の両方で登場することで、その評価の確かさが裏付けられている。

---

## DES#31からの進化 — 10分と45分の違い

| 観点 | DES#31（10分クイズ形式） | Data Engineering Summit（45分フルセッション） |
|---|---|---|
| 買収情報 | 8社を一覧列挙 | 各買収の背景・意義・業界インパクトを詳説 |
| Snowflake vs Databricks | 「どちらも外れない」という方向性の整理 | Intelligence / Databricks Oneなど具体的機能差を比較、OLTP対応の新動向も |
| Iceberg | Adaptive Data Warehouse・Polaris CatalogのOSS化に言及 | 各社のIceberg戦略を体系的に整理、Q&Aでロックイン回避の文脈で深掘り |
| dbt Fusion | 競合SDFが「Fusion」としてベータリリースと紹介 | Beta → Public Previewへの進化、Rustエンジン・カラムリネージの詳細 |
| BI | 触れる範囲が限定的 | Tableau Next、Looker新機能、Hex/Omniの資金調達額まで詳述 |
| Data Catalog / Orchestration | ほぼ未言及 | MCPサーバー革命、Kestraの台頭を専門セクションで解説 |
| Semantic Layer | 言及なし | OSI（業界標準化）、Omni推薦、Q&Aでの実務的アドバイスまで |
| 2026年予測 | DataCatalog・Observabilityの買収継続を予告 | AI-Ready基盤の5要素など具体的なロードマップを提示 |
| Q&A | イベント性格上、公開情報は限定的 | Classmethdブログで4問のQ&Aが詳細に公開 |

DES#31はクイズという制約のある10分間でMDSの「見出し集」を届けるコンパクトな登壇だった。本セッションはその同じ題材を45分かけて展開した「フルテキスト版」であり、背景・文脈・実務的含意が格段に充実している。さがら氏が閉会の言葉として語った「**SnowflakeとDatabricksは強い。Modern Data Stackというワードもトレンドの節目を迎えている印象（ちょっと寂しい）**」というコメントに、エコシステムを5年以上ウォッチしてきた者の複雑な感情が滲む。

---

## 関連リンク

- [アーカイブ視聴（要登録）](https://conference.findy-code.io/conferences/data-engineering-summit/8/archives/245)
- [スライド（Speaker Deck）](https://speakerdeck.com/sagara/ji-dong-no2025nian-modern-data-stacknozui-xin-ji-shu-dong-xiang)
- [登壇レポート（DevelopersIO）](https://dev.classmethod.jp/articles/data-engineering-summit-modern-data-stack-2025/)
- [Data Engineering Summit 2025 イベントページ](https://data-engineering-summit.findy-tools.io/2025)
- [関連記事: DES#31 公開企画会議](/articles/data-engineering-study-31-advisory.html)
- [さがら氏 Speaker Deck](https://speakerdeck.com/sagara)
- [さがら氏 Twitter/X (@SS_chneider)](https://twitter.com/SS_chneider)
- [クラスメソッド株式会社](https://classmethod.jp/)
