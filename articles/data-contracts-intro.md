# Data Contracts — なぜ今データコントラクトが必要か

**Part I: Introduction to the Data Contract Architecture — 学習ノート**

Chad Sanderson, Mark Freeman, B.E. Schmidt著『Data Contracts』（O'Reilly, 2025）Part I 第1〜4章の要約

---

**シリーズ：** Data Contracts 学習ノート | **対象：** Part I（Ch.1–4）

---

## キーメトリクス

| 指標 | 値 |
|------|-----|
| 対象章 | 第1〜4章 |
| 大規模リファクタリングの平均コスト | $663,000（1,500 staff days） |
| 反応的解決ライフサイクルのステップ数 | 11ステップ |
| データ/ソフトウェア開発者比率（欧州上位50社中央値） | 1:4 |

---

## 目次

1. [Part I 概要](#part-i-概要)
2. [Ch.1: Garbage-In, Garbage-Out サイクル](#ch1-garbage-in-garbage-out-サイクル)
3. [Ch.1: データアーキテクチャの進化](#ch1-データアーキテクチャの進化)
4. [Ch.2: データ品質の再定義](#ch2-データ品質の再定義)
5. [Ch.2: データ品質の測定](#ch2-データ品質の測定)
6. [Ch.3: スケーリングの課題](#ch3-スケーリングの課題)
7. [Ch.4: データコントラクトとは何か](#ch4-データコントラクトとは何か)
8. [まとめ — Part I の要点](#まとめ--part-i-の要点)
9. [参考文献](#参考文献)

---

## Part I 概要

Part I（第1〜4章）は、データコントラクトの「なぜ」と「何か」を確立する。データ品質を劣化させてきた歴史的な力を追跡し、データ品質が本当に何を意味するかを定義し、データインフラをスケールさせる際の組織的・技術的課題を検討し、解決策として**データコントラクト・アーキテクチャ**を導入する。

**中心的主張：**

> "Data contracts—agreements between data producers and consumers that are established, updated, and enforced via an API—are necessary for scaling and maintaining data-related change management within an organization."

データ問題は本質的に**複雑なシステム内の変更管理問題**である。データコントラクトは、組織がスケールしてもその変更管理を維持するために必要なメカニズムだ。

### 章の構成

| 章 | テーマ |
|----|--------|
| Chapter 1 | なぜ今データコントラクトが必要か：データ負債・GIGO・アーキテクチャ進化 |
| Chapter 2 | データ品質は完璧なデータについてではない：定義の再構築・OLTP/OLAP問題 |
| Chapter 3 | データインフラをスケールさせる際の課題：変更管理・コスト・組織論 |
| Chapter 4 | データコントラクト入門：定義・ワークフロー・オブザーバビリティとの比較 |

---

## Ch.1: Garbage-In, Garbage-Out サイクル

*Ch.1 / GIGO Cycle*

### データ負債（Data Debt）の定義

> **"Data debt refers to the outcome of incremental technology choices made to expedite the delivery of data infrastructure at the expense of its long-term trustworthiness."**
>
> — Chad Sanderson et al., *Data Contracts* (O'Reilly, 2025) Ch.1

技術的負債（tech debt）に類似しているが、データ負債には2つの重要な違いがある：

**Tech Debt との比較**

| 観点 | Tech Debt（技術的負債） | Data Debt（データ負債） |
|------|------------------------|------------------------|
| トレードオフ | 速度 vs **保守性** | 速度 vs **信頼性**（より深刻） |
| 解消の難易 | 個別マイクロサービスを独立してリファクタリング可能 | 緊密な依存関係で解消困難 |
| 失敗のモード | コードエラーは可視化しやすい | サイレントな失敗（誤った出力が検出されない） |
| 変更のために必要なこと | 当該チームの判断で可能 | 広範な組織的合意と多大なコーディネーション |

> "Data debt is a vicious cycle. It depreciates trust in the data, which attacks the core value proposition of the data team. And unwinding data debt is not easy—not even close."

**データ負債の実例：**
- キーメトリクスが誤解されたデータ値（`customers` と `active_customers` の混同など）に依存している
- テスト用ログがMLモデル訓練に使用されるDBへ混入し、一貫性のない訓練データが生成される
- 当初はデータ品質問題を解決したが、データの前提と合わなくなった `CASE WHEN` などのハードコードされた変換ロジック

### GIGO（Garbage-In, Garbage-Out）サイクル

**GIGO** とは、ステークホルダーの期待を満たさないデータがパイプラインに入り込む状態を指す。GIGOが特に問題なのは、**事後的にしか対処できない**という点だ。

修正が積み重なるにつれ、特定の個人に institutional knowledge hotspot（制度的知識の集積点）が形成される。この個人が退職すると、そのコンテキストを取り戻すことは極めて困難になる。

### ケーススタディ：Convoy（物流スタートアップ）

Convoyのエンジニアリングチームは「ポッド（pod）」に分割されていた（Spotifyの組織モデルから採用）：
- ML モデルでカスタマーサービス問題の優先順位付けを行うポッド
- コア価格モデルを担当するポッド
- 最大パートナーへのリアルタイム出荷ETAアナリティクスを担当するポッド

ポッド間のコミュニケーションは最小限。あるポッドのソフトウェアエンジニアがカラム名変更・カラム削除・ビジネスロジック変更・イベント停止を行っても、データコンシューマーには何も伝わらなかった。

**重要な観察：** スタートアップが約20名以下のエンジニアの段階では、データエンジニアはまだ複数のサイロ化されたドメインに分裂していない。20名を超え始めると、このコミュニケーション構造は機能しなくなる。

---

## Ch.1: データアーキテクチャの進化

*Ch.1 / Evolution*

### プレモダン時代（Pre-2000s）

オンプレミスDWH（Oracle・IBM DB2・Microsoft SQL Server）。高価なETLツール（Informatica、SSIS）。**データアーキテクト**がERD（Entity Relationship Diagram）を設計。

**Bill Inmon の データウェアハウス 4原則：**
1. **主題指向（Subject-oriented）**：組織の主要主題（顧客・製品等）を中心に設計
2. **統合（Integrated）**：多様な上流システムからデータを調達し**単一の真実ソース（SSOT）**を構築
3. **時系列（Time-variant）**：時系列でデータを収集・保存し、過去分析を可能に
4. **非揮発性（Non-volatile）**：一度ロードされたデータは変更されない

**キーコンセプト：**
- **エンティティ（Entity）**：表形式で表現されたビジネス主題（例：`customer` エンティティ）
- **外部キー（Foreign keys）**：テーブル間のデータ結合を可能にする一意識別子
- **カーディナリティ（Cardinality）**：エンティティ間の関係の性質
- **論理データモデル**：エンティティ・次元・外部キー・カーディナリティの理想的なセット
- **物理データモデル**：クエリ可能なテーブル・カラム・インデックス

### Software Eats the World（2000s–2010s）

Marc Andreessen「ソフトウェアが世界を食べている」（WSJ, 2011）。Thoughtworks が普及させたアジャイル手法、Mark Zuckerberg の「Move fast and break things」文化。AWS・Azure・GCPがクラウドインフラを民主化。

**データアーキテクチャは意図的に軽視された**——ERD設計に数ヶ月を費やすことはアジャイルの「移動速度」と相容れなかった。

### マイクロサービス時代

「Microservices」は Thoughtworks コンサルタントが初めて使用し、**Martin Fowler のブログ**で広まった。

> **"Microservices are an architectural pattern that changes how applications are structured by decomposing them into small, autonomous services."**

データアーキテクチャのサイレントな犠牲者：マイクロサービス全体にデータが分散し、ガバナンスのないデータレイクが台頭。これが「**data swamp（データ沼）**」を生んだ。

### モダンデータスタック時代（2012–2022）

**Modern Data Stack (MDS)** の主要プレイヤー：
- **Snowflake**：クラウドベースSQL変換・ストレージの最大手
- **Google BigQuery・Amazon Redshift・Databricks**：代替クラウドDWH
- **ELTツール**：Fivetran・Airbyte（外部ソースからのデータ取り込み）
- **dbt（data build tool）**：バージョン管理・テスト・ドキュメント付きSQL変換CLI

**成長要因（2012–2022）：**
1. **プロダクト主導型成長（PLG）**：SaaSベンダーが従来のIT調達を迂回
2. **VC投資**：データインフラ企業はRevより顧客成長を重視するVCから資金調達
3. **データチームの拡大**：データサイエンスが数十億ドル規模のカテゴリーに成長

**顕在化した問題：**
1. 単一の真実ソースなし（マイクロサービスにデータが分散）
2. データプロデューサーとコンシューマーの関係なし
3. データへの信頼の喪失（上流変更のコストを下流が負担）
4. コストの急騰（使用量ベース価格でコストが急増）

> "Almost overnight, the data team was larger than it had ever been, more expensive than it had ever been, and arguably less trusted than it had ever been."

### 「アクティブ顧客」問題 ——SQLの実例

「今月のアクティブ顧客数は？」というシンプルなクエリが、上流変更によりどれほど複雑になるかを示す実例。

**初期クエリ（シンプル）：**

```sql
WITH visit_counts AS (
  SELECT customer_id, COUNT(*) AS visit_count
  FROM visits
  WHERE DATE_FORMAT(visit_date, '%Y-%m') = DATE_FORMAT(CURDATE(), '%Y-%m')
  GROUP BY customer_id
)
SELECT COUNT(DISTINCT customers.customer_id) AS active_customers
FROM customers
LEFT JOIN visit_counts ON visit_counts.customer_id = customers.customer_id
WHERE COALESCE(visit_counts.visit_count, 0) >= 3;
```

上流変更後（`impression` = 任意の画面遷移、`visit` = 10秒以上の活動期間）：

```sql
WITH impressions_counts AS (
  SELECT customer_id,
    SUM(CASE WHEN duration_seconds >= 10 THEN 1 ELSE 0 END) AS visit_count,
    SUM(CASE WHEN duration_seconds < 10 THEN 1 ELSE 0 END) AS impression_count
  FROM impressions
  WHERE DATE_FORMAT(impression_date, '%Y-%m') = DATE_FORMAT(CURDATE(), '%Y-%m')
  GROUP BY customer_id
  HAVING (visit_count + impression_count) >= 3
)
SELECT COUNT(DISTINCT customers.customer_id) AS active_customers
FROM customers
LEFT JOIN impressions_counts ON impressions_counts.customer_id = customers.customer_id
WHERE COALESCE(impressions_counts.visit_count, 0) >= 3;
```

上流変更が重なるごとにクエリは肥大化し、`CASE`文や `WHERE`句が存在する理由のコンテキストはすべて失われる。新しいデータ開発者は独自の目的のために「車輪の再発明」を行い、コアメトリクスの定義が組織内で増殖する。

### データ中心AI時代（Post-2020）

Andrew Ng の 2021 年キャンペーンにより **Data-Centric AI** が広く受容。

- **Data-Centric AI**：モデルアーキテクチャではなく、訓練データの品質と選定を改善してMLモデル性能を向上させるアプローチ
- **Model-Centric AI**：モデルのチューニング・計算資源の増強で性能向上を図るアプローチ

MLベンダーは 2014 年の約5社から 2024 年には 200 社超に急増（Matt Turck 年次データベンダーレポート）。競争優位は今や**データの品質と独自性**に集中する。

**Shift Left Data**：データパイプラインにおいて、品質・ガバナンス・コントラクトの適用をより上流（データが生産されるソフトウェアエンジニアに近い側）に移動させる動き。

---

## Ch.2: データ品質の再定義

*Ch.2 / Data Quality*

### データ品質の定義

**Wang et al. (1996) の定義（MIT データ品質研究グループ、最も引用される定義）：**

> "Data that are fit for use by data consumers."

**本書の拡張定義：**

> **"[Data quality is] an organization's ability to understand the *degree of correctness* of its data assets, and the trade-offs of operationalizing such data at various degrees of correctness throughout the data lifecycle, as it pertains to being fit for use by the data consumer."**

**重要フレーズの解析：**

1. **「様々な正確性の度合いでのトレードオフ」**：完璧主義より戦略的トレードオフ意識が重要。NoSQL の台頭以来、データ保存の複雑さとトレードオフが増大。
2. **「正確性の度合いを理解する能力」**：データは絶えず劣化しており、完全な完璧さは非現実的。健康指標の計測（血圧計・体重計）は本質的に不正確だが、予防医療において極めて有用——データ品質も同様だ。

**核心的洞察：**

> "Central to this chapter is the idea that data quality is not about data being pristine, but instead being 'fit for use' for your stakeholders. At the end of the day, technical and data debt should ideally be a strategic trade-off where you are balancing effort and robustness with ROI."

### OLTP と OLAP の断絶

| 観点 | OLTP（オンライントランザクション処理） | OLAP（オンライン分析処理） |
|------|--------------------------------------|--------------------------|
| 最適化対象 | 高速データトランザクション・現在状態の維持 | 大量データのスキャン・統計計算 |
| 主要操作 | CRUD + ACID準拠 + 第三正規形（3NF） | 非正規化データ・複雑なビジネスロジック |
| 主要ペルソナ | ソフトウェアエンジニア（データの生産者） | データアナリスト・データサイエンティスト |
| 限界 | 複雑な分析クエリで本番DBが停止するリスク | OLTPからのレプリケーションで複雑性と誤解が増大 |

OLTPからOLAPへのデータレプリケーションはそれぞれの「データ世界観」にサイロを生み出し、誤解を招く。

### ケーススタディ：Kelp（架空の水族館レビューサイト）

Kelpのエンジニアリングチームが「レビューの更新」機能をリリース。データアナリストはこの機能がユーザーセッション時間を増加させるかを分析しようとした。

**Kelp の OLTP データ構造（3NF）：**

```
-- Aquarium Table
|aquarium_id|aquarium_name    |aquarium_location|aquarium_size|adult_admission_price|
|123456     |'Monterey...'    |'Monterey, CA'   |'Large'      |25.99                |

-- User Table
|user_id|user_name|demographic_n|
|1234   |Mark     |'<user info>'|

-- Reviews Table
|aquarium_id|review_id|user_id|number_stars|review_timestamp|review_text|
```

アナリストがOLAP用の非正規化ワイドテーブルを作成した時点で、**解決不能な問い**が生まれる：
- 近接した複数セッションをまたぐ場合、セッション時間はどう計算するか？
- 一人のユーザーが複数アカウントを持つ場合は？
- ビジネスにとって関連するセッション時間の変化はどの程度か？

正解は**機能を最初に構築したソフトウェアエンジニアとのコミュニケーションなしには判断できない**。これがデータコントラクトが重要な理由の核心だ。

> "This difference in constraints and goals between data producers and consumers gets to the crux of why data contracts are important."

---

## Ch.2: データ品質の測定

*Ch.2 / Measuring Quality*

### 先行指標（Leading Indicators）

| 指標 | 説明 |
|------|------|
| **データ負債（Data Debt）** | ドキュメントが存在するデータ資産の割合、依存数の中央値、バックフィルジョブ数、クエリあたり平均フィルタ数 |
| **信頼性（Trustworthiness）** | 四半期サーベイで「データを信頼しているか」等を計測。複製されたデータ資産数も代理指標 |
| **オーナーシップ（Ownership）** | 上流のエラーを解決する可能性と速度の予測指標 |
| **期待値の違反（Violated Expectations）** | スキーマ・セマンティクス・SLA・PIIの4形態で期待値を測定 |

### 後行指標（Lagging Indicators）

| 指標 | 説明 |
|------|------|
| **データダウンタイム（Data Downtime）** | 全インシデントの記録（期間・原因・解決）。平均ダウンタイム・頻度・ビジネスオペレーションへの影響を計算 |
| **四半期インシデント数** | 時系列で「ビジネスインパクト」が最も高いインシデントを追跡。P0〜P3の重大度スケールで分類 |

**インシデント実例：**
- **例1（マーケティング予算の誤配分）**：COVID期間中、あるスタートアップの新規サインアップが25%超減少と見えたが、実際はユーザー行動ではなくサインアップイベントの記録方法の変更が原因。マーケティングチームは予算を大幅増加させた後に気づいた。
- **例2（実物リコール）**：消費財メーカーの生産エンジニアが `item_id` に文字を追加。下流への影響を知らず、誤ったバーコードが印刷・配送されてしまった。

### 誰がデータ品質問題の影響を受けるか

| ペルソナ | 影響 |
|---------|------|
| **データエンジニア** | 全データ問題のデフォルト「オーナー」。オンコールのバックログが積み上がる |
| **データサイエンティスト** | 欠陥データでMLモデルを数ヶ月訓練してから問題を発見 |
| **データアナリスト** | ビジネスロジックが混乱。ステークホルダーへの誤った推奨 |
| **ソフトウェアエンジニア** | 通常は上流の原因（意図的ではない）。変更が適切なデータコンシューマーに届かない |
| **ビジネスチーム** | 最終的な消費者。主体性がほぼなく、上流エラーの帰結を最も大きく受ける |

---

## Ch.3: スケーリングの課題

*Ch.3 / Scale Challenges*

### ソフトウェア開発 vs データ開発

| 観点 | ソフトウェア開発 | データ開発 |
|------|--------------|----------|
| 開始点 | 既知のユーザー体験 | ビジネスの問い |
| プロセス | 要件定義 → 技術仕様 → ビルド | データ探索 → クエリ構築 → 検証 → 安定化 |
| 最重要事前作業 | 要件収集・アーキテクチャ設計 | 仮説の定式化・データの理解 |
| 変更管理 | チーム内で確立（バージョン管理・PR・コードdiff） | クロスチーム間で**確立されたシステムが存在しない** |

データ開発の**データ発見（data discovery）**プロセス——データが存在するか・どこにあるか・誰がオーナーか・何を意味するか・アクセス権があるか——に答えるだけで、1つの問いあたり数時間〜数日を費やす可能性がある。

**スケール統計：**
- データ開発者:ソフトウェア開発者比率 = **1:4**（欧州上位50社の中央値）
- Chadの前職：200超のエンジニアに対してデータエンジニアは10名未満

### 大規模リファクタリングのコスト

Martin Fowler のリファクタリング定義：「観察可能な動作を変えずに、より理解しやすくより変更しやすくするためにソフトウェアの内部構造を変える変更。」

**James Ivers et al. の調査統計：**
- 組織は大規模リファクタリング1件に平均 **1,500 スタッフ日**を費やす
- 平均給与 $115,000・260 勤務日/年で換算 → 単一の大規模リファクタリングに **約$663,000の労務費**

**2種類のリファクタリング：**
- **Floss refactoring（フロスリファクタリング）**：定期的な小規模・機会的な修正（例えるなら定期的な糸ようじ）
- **Root canal refactoring（根管リファクタリング）**：深刻に劣化したコードベースへの大規模・抜本的な手術

### ケーススタディ：Alan（フランスの医療保険会社）

Alanは2016年創業の医療保険会社。データモデルの中心的な前提：**ユーザーと健康保険契約が1対1の関係**（フランスでは標準）。国際展開のため、複数契約ユーザーをサポートするリファクタリングが必要に。

**Alanの7ステップ大規模リファクタリング：**
1. 変更が必要な箇所の特定
2. 変更内容の決定
3. 変更の実装（製品ロードマップへの優先付けと専任クロスファンクショナルチームの結成）
4. 新規テストの生成と既存テストの移行
5. リファクタリングコードの検証
6. 再認定（HIPAA等コンプライアンス要件）
7. ドキュメントの更新

欠陥データモデルの上に **5年間**の技術的負債が積み重なり、リファクタリングは極めて困難なプロジェクトとなった。

### コンセプトドリフトの4類型

（Jie Lu et al.「Learning Under Concept Drift: A Review」より）

| タイプ | 説明 | 例 |
|--------|------|-----|
| **突発的ドリフト（Sudden Drift）** | データ分布の突然の変化 | GDPR対応でEU加盟国ユーザーがトラフィックから消失 |
| **漸進的ドリフト（Gradual Drift）** | 時間とともに緩やかに進むシフト | NetflixがDVD販売からストリーミングへ移行 |
| **段階的ドリフト（Incremental Drift）** | 段階的なデータ分布の変化 | Robloxのユーザー中央年齢が13歳超に上昇 |
| **周期的ドリフト（Recurring Drift）** | 周期的に繰り返すパターン | 景気後退における消費支出の周期的減少 |

### Dunbar の数と Conway の法則

**Dunbar の数（~150）：** 人類学者 Robin Dunbar が提唱。人間が維持できる有意義な関係の認知上限は**約150**。150名の組織には **11,175の潜在的関係**が存在し、1名追加するごとに150の新たな関係が生まれる。

Meta CPO Chris Cox（2016）：「150人を超えると、おかしなことが起き始める」

**Conway の法則：** Melvin Conway「How Do Committees Invent?」

> "Any organization that designs a system…will inevitably produce a design whose structure is a copy of the organization's communication structure."

データシステムは組織のサイロを反映する。これら2つの原理が共に示すのは、**複雑なデータシステムのスケールは人的コーディネーションだけでは達成できない**ということだ。データコントラクトはこの限界を技術的に克服するメカニズムとなる。

### 効果的なデータ変更管理の4要件（Design Surface）

1. **可視性**：データの存在と使用方法の可視性
2. **接続**：データプロデューサーとコンシューマーの接続
3. **タイムリーな通知**：変更通知はブレイキングチェンジの直前に伝える（早すぎても遅すぎても不可）
4. **適切なステークホルダーの特定**：変更管理プロセスに適切な人物を参加させる

### データ変更管理の5要件

1. ビジネスステークホルダーのドメイン知識のコード化
2. データ標準を維持するための意味のある制約の作成
3. データドリフトの発生・種類・ビジネスへの影響の識別
4. 合意閾値超過時の主要ステークホルダーへのアラート
5. データ品質の修正またはドリフト後のビジネスロジックの更新

> "These five requirements are why we strongly believe in data contracts as a needed mechanism to enable data quality at scale."

---

## Ch.4: データコントラクトとは何か

*Ch.4 / What Are Data Contracts?*

### コラボレーションはデータにおいて異なる

ソフトウェアの世界では、各メンバーに変更を真剣に受け止めるインセンティブが組み込まれている——悪いコードを出荷した場合、書いたエンジニアとレビューしたエンジニアの両方が責任を問われる。

**データは異なる：**

> "In most cases, data is stored in source systems owned by product teams, not by data teams. When upstream software engineers make changes to their database, these changes are reviewed by their own team—**not** the downstream teams that leverage this data for a set of totally different use cases. Thanks to this disconnect, data teams are often the last to know when something changes."

> **"The primary goal of data contracts is to solve this problem. Contracts are a mechanism for expanding software-oriented collaboration to data teams, bringing quality to data and enabling greater trust in analytical environments."**

### ステークホルダーの定義

**データプロデューサー（Data Producers）：**
- データの収集・保存を担当するエンジニア。最も一般的な形態は**ソフトウェアエンジニア**
- **トランザクショナルイベント**：顧客または技術が明示的なアクションを取った際に記録（PostgreSQL・MySQL・Kafka）
- **クリックストリームイベント**：ユーザーのウェブ/モバイル操作を追跡（Segment・Rudderstack・Amplitude）

**データコンシューマー（Data Consumers）：**

| 役割 | 説明 |
|------|------|
| **データエンジニア** | ソフトウェアエンジニアのデータ専門版。データの抽出・移動・パイプライン構築 |
| **データサイエンティスト** | ML・統計モデリングに重点 |
| **データアナリスト** | SQL専門家。ダッシュボード・レポートで事業の問いに答える |
| **アナリティクスエンジニア** | ソフトウェア開発の最善実践を分析データに適用（dbtモデル等）|
| **プラットフォームエンジニア** | データインフラの実装・採用・保守を担当（新しい職種） |

コンシューマーが退職すると、データに関して積み上げられたコンテキストを取り戻すことは**極めて困難**。これが**データリネージュ（data lineage）**——分析環境内でデータ資産を相互に結びつける蜘蛛の巣状の接続——が重要な理由の一つだ。

### 11ステップ 反応的解決ライフサイクル（現状の問題）

「これらの数字が少し正常ではないように見えると思います」というフレーズはデータ実践者の恐怖だ：

1. ステークホルダーが問題を発見
2. 問題のトリアージ
3. 要件スコーピング
4. 問題の再現
5. データプロファイリング（タイムライン・nullパターン・スパイク/ドロップ・集計別カウント・データリネージュレビュー）
6. 下流パイプライン調査（SQLバグ・ビジネスロジックの乖離）
7. 上流パイプライン調査（トランザクショナルDBへのリネージュ追跡）
8. 技術ステークホルダーへの相談
9. デプロイ前：DQ修正の実装
10. デプロイ：DQ修正の適用
11. ステークホルダーへのコミュニケーション

> "Note that a majority of these steps are not technical but rather center around communication across teams."

### 7ステップ データコントラクト ワークフロー（プロアクティブ代替案）

> "The data contract workflow moves the data quality resolution process from reactive to proactive, which dramatically changes the incentives and responsibilities of each stakeholder involved."

1. **データコンシューマーがデータ制約を特定**
2. **データコンシューマーがデータ資産のコントラクトをリクエスト**
3. **データプロデューサーがコントラクトの実現可能性を確認**（コントラクトはバージョン管理されたコードとして保存され、履歴も追跡可能）
4. **コントラクトがコードとして確認**（通常 YAML ファイル、バージョン管理）
5. **データプロデューサーがデータ資産変更のプルリクエストを作成**
6. **CI/CD チェックが要求変更がデータコントラクトに違反するか自動検証**
7. **（7a）** CI/CD失敗：データ資産オーナーに違反通知→失敗プロトコル実行 **（7b）** CI/CD成功：変更が下流プロセスに適用

**インセンティブの再構成：**
- データプロデューサー：CI/CDチェックに合格しないとプルリクエストをマージできない。通知を受けたにもかかわらず行動しなかった場合、ビジネスクリティカルな障害のトレースがコード変更に戻る。
- データコンシューマー：データに関する適切な精査を確実に実施。特にエグゼクティブによる知見の受け入れが重要。

### データコントラクト vs データオブザーバビリティ

**データオブザーバビリティ（Gartner 定義）：**
> "[The] ability of an organization to have a broad visibility of its data landscape and multi-layer data flows to minimize data downtime and data outages."

| 観点 | データコントラクト | データオブザーバビリティ |
|------|-----------------|----------------------|
| 主目的 | 特定のデータ品質問題の**防止** | データ品質トレンドの**検出・可視化** |
| CI/CDとの関係 | CI/CDワークフローに**組み込まれる** | CI/CDワークフローを**補完する** |
| ビジネスロジック | ビジネスコンテキストが**入力** | データに**どう捉えられるかを反映**（出力の測定） |
| 可視性の範囲 | **ターゲット可視性**：最重要データ資産のみ | **広範な可視性**：データスタック全体 |
| アラートのタイミング | **変更前**（本番デプロイ前に防止） | **変更後**（発生後の検出が必須） |
| アナロジー | スポットライト（重要な資産に焦点） | 懐中電灯（データシステム全体を照らす） |

> "The key difference between data contracts and data observability is that contracts emphasize the **prevention** of data quality issues, while observability emphasizes **detection**."

**最適な組み合わせ：** オブザーバビリティで広く観測し、コントラクトで特定の制約を強制する。

### データコントラクトの実装ポイント

データコントラクト・アーキテクチャは4つのコンポーネントで構成される：
1. **データ資産（Data assets）**
2. **コントラクト定義（Contract definition）**
3. **検出（Detection）**
4. **防止（Prevention）**

**データライフサイクルにおける実装ポイント：**
- 上流（ソースシステム）：最も推奨されるが最も実装が困難
- CRUD操作（トランザクションDB）：スキーマ・セマンティクスの変更がデータ品質問題の根本
- ETL/ELTパイプライン：多くのデータチームが最初に実装を始める場所
- サードパーティデータ取り込み：外部データとシステムの間にコントラクトを置く
- 分析DB：上流コントラクトが整ったチームが検討
- 下流コンシューマー：ビジネスステークホルダーへ届く前のデータ品質を保証

### データコントラクトの成熟度曲線

| ステージ | 目標 | 重点 |
|---------|------|------|
| **Stage 1: Awareness（認識）** | 下流チームが上流データをどう使用するかの可視性を確立 | プロデューサーが変更の影響を理解する |
| **Stage 2: Collaboration（協働）** | コントラクトによるソースでのデータ保護 | デプロイ前のコンテキストを伴った協働 |
| **Stage 3: Ownership（オーナーシップ）** | データ資産の明確なオーナーシップとアカウンタビリティの確立 | 明示的な変更管理・バージョン管理プロセス |

> "Visibility creates culture change."

---

## まとめ — Part I の要点

*Part I Summary*

### 7つの核心的洞察

**01. データ問題は変更管理問題である**

データの品質問題は表面上は技術的に見えるが、その根本はすべて「複雑なシステム内の変更管理（change management within complex systems）」に行き着く。コードの変更がデータコンシューマーに届かないこと、届く仕組みがないことが問題の本質だ。

**02. データ負債はテック負債より解消が困難**

テック負債は個別マイクロサービスを独立してリファクタリングできる。しかしデータ負債は相互に依存した事業体の上に構築されており、広範な組織合意と多大なコーディネーションなしには変更できない。Alan社の事例は、欠陥データモデルの上に5年間の負債が積み上がり、約663,000ドルのリファクタリングコストが発生することを示す。

**03. データ品質 ≠ 完璧なデータ**

Wang et al. の「利用目的に適合すること（fit for use）」は今なお正しい。本書はこれを「様々な正確性の度合いでデータ資産を運用する際のトレードオフを理解し管理する組織の能力」へと拡張する。データは常に劣化しており、完全な完璧さは非現実的な目標だ。

**04. OLTP/OLAP の断絶が翻訳問題を生む**

OLTPの3NFとOLAPの非正規化された世界観のギャップは、データチームが単独では解決できない解釈の問題を生み出す。Kelpのケースが示すように、正解は最初に機能を構築したソフトウェアエンジニアとのコミュニケーションなしには答えられない。

**05. Dunbar の数と Conway の法則がスケールを阻む**

150人を超えると人間の関係維持能力は限界を迎え（Dunbarの数）、組織のシステムはその通信構造を反映する（Conwayの法則）。スケールした組織ではデータシステムの変更管理が人的コーディネーションだけでは維持できないことを意味する。

**06. コントラクトはオブザーバビリティを置き換えない、補完する**

データオブザーバビリティ（懐中電灯）はデータスタック全体を広く照らすが、イベントが発生した後にのみアラートを出す。データコントラクト（スポットライト）は最重要データ資産に焦点を当て、本番デプロイ前に問題を防ぐ。

**07. シフトレフトは文化変革である**

データコントラクトの採用は技術的な実装だけではなく、データプロデューサーとコンシューマーの間のコラボレーション文化の変革を要求する。「Visibility creates culture change.」成熟度は認識 → 協働 → オーナーシップと段階的に進む。

---

### Part I の本質的なメッセージ

> データ品質問題は**検査（inspection）の問題ではなく、予防（prevention）の問題**だ。業界は反応的（オブザーバビリティ）なアプローチからプロアクティブ（コントラクト）なアプローチへとシフトする必要がある。

**「Most data quality problems are code quality problems.」**

上流のコードの変更が、通知なしに下流のデータ消費者に影響を与える。データコントラクトはこの暗黙の依存関係を明示的・強制可能なAPI契約へと変換することで、ソフトウェア指向の変更管理をデータチームへと拡張する。これがデータコントラクトの存在意義だ。

---

## 参考文献

### 一次資料

1. Chad Sanderson, Mark Freeman, B. E. Schmidt, *Data Contracts: Developing Production-Grade Pipelines at Scale*, O'Reilly Media, Inc., November 2025. ISBN: 9781098157623
   - Ch.1: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ch01.html
   - Ch.2: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ch02.html
   - Ch.3: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ch03.html
   - Ch.4: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ch04.html

### 関連資料

2. Chad Sanderson — "An Engineer's Guide to Data Contracts" (Substack):
   https://dataproducts.substack.com/p/an-engineers-guide-to-data-contracts

3. Chad Sanderson — "The Shift Left Data Manifesto" (Substack):
   https://dataproducts.substack.com/p/the-shift-left-data-manifesto

4. Monte Carlo Data — "Data Contracts Explained":
   https://www.montecarlodata.com/blog-data-contracts-explained/

5. Gable.ai — Data Contracts Book Site:
   https://www.gable.ai/data-contracts-book

### 書籍内で参照された作品

6. Edgar F. Codd, "A Relational Model of Data for Large Shared Data Banks," 1970 — リレーショナルデータベースの基礎論文

7. Richard Y. Wang et al., MIT データ品質研究グループ, 1996 — 定義：「データコンシューマーが使用するのに適したデータ」

8. Marc Andreessen, "Software Is Eating the World," *The Wall Street Journal*, 2011

9. Martin Fowler, *Refactoring: Improving the Design of Existing Code*, Addison-Wesley
   マイクロサービスに関するブログ: https://martinfowler.com/articles/microservices.html

10. Melvin Conway, "How Do Committees Invent?" — Conway の法則の出典

11. Jie Lu et al., "Learning Under Concept Drift: A Review" — コンセプトドリフトの分類体系

12. James Ivers et al. — 大規模リファクタリングに関する調査（平均1,500スタッフ日）

13. Chaïmaa Kadaoui — Alan 社の大規模リファクタリングに関する記事

14. Malcolm Gladwell, *The Tipping Point* (Back Bay Books) — Dunbar の数の参照

15. Matt Turck — 年次データベンダーランドスケープレポート

16. Melody Chien and Ankush Jain, "Data and Analytics Essentials: Data Observability," Gartner, 2023 — データオブザーバビリティの定義

17. Andy Petrella, *Fundamentals of Data Observability*, O'Reilly, 2023

---

*本記事は Chad Sanderson, Mark Freeman, B.E. Schmidt 著『Data Contracts』（O'Reilly Media, 2025）の個人学習ノートです。著作権は原著者および O'Reilly Media に帰属します。*
