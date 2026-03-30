# Data Contracts — 実装アーキテクチャの全体像｜Part II 学習ノート

**シリーズ：** Data Contracts 学習ノート — [Part I](data-contracts-intro.html) | **Part II（当記事）** | Part III（近刊）

**書籍：** Chad Sanderson, Mark Freeman, B.E. Schmidt 著  
**タイトル：** *Data Contracts: Developing Production-Grade Pipelines at Scale*  
**出版：** O'Reilly Media, November 2025 | ISBN: 978-1-098-15763-0  
**O'Reilly URL：** https://learning.oreilly.com/library/view/data-contracts/9781098157623/

**対象章：** 第5〜8章（Implementation）  
**学習日：** 2026-03-30 | 読了目安：~30分

---

## 主要スペック

| 指標 | 数値 |
|---|---|
| 対象章 | 第5〜8章（4章） |
| Adevinta Spainの1日あたりデータ量 | 4 テラバイト |
| CI/CDエンドツーエンドワークフローのステップ数 | 13ステップ |
| 本番ケーススタディ企業数 | 3社（Convoy / Glassdoor / Adevinta Spain） |

---

## 目次

1. [Part II 概要](#00-part-ii-概要)
2. [Ch.5: データアセットとコントラクト定義](#01-ch5-データアセットとコントラクト定義)
3. [Ch.6: 検出と予防](#02-ch6-検出と予防)
4. [Ch.7: データコントラクトの実装](#03-ch7-データコントラクトの実装)
5. [Ch.8: 本番環境のケーススタディ](#04-ch8-本番環境のケーススタディ)
6. [Part II まとめ](#05-part-ii-まとめ)
7. [参考文献](#06-参考文献)

---

## 00: Part II 概要

Part II（第5〜8章）は、データコントラクトの「**どうやって**」を実践的に確立する。Part I が「なぜ」と「何か」を問いかけたとすれば、Part II は設計図を具体的なツール・コード・本番事例へと変換する。

> **Part II の中心的テーマ：**  
> データコントラクト・アーキテクチャは**4つのコンポーネント**で構成される：データアセット（基盤）、コントラクト定義（仕様）、検出（Detection）、予防（Prevention）。Part II はこれら4コンポーネントを詳細に解説し、実装サンドボックスと3社のケーススタディで「机上の理論」を「動くシステム」へとつなぐ。

| 章 | テーマ |
|---|---|
| **Chapter 5** | 4データアセットカテゴリ + コントラクト定義の3サブコンポーネント |
| **Chapter 6** | 7つの検出手法 + WAP / CI/CD / 5W1Hアラートによる予防 |
| **Chapter 7** | ミュージアムアプリのモックシナリオ：13ステップCI/CDワークフロー実装 |
| **Chapter 8** | 3社のケーススタディ：Convoy・Glassdoor・Adevinta Spain |

---

## 01: Ch.5 — データアセットとコントラクト定義

*出典: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ch05.html*

第5章は理論から実践への最初の橋渡しだ。4コンポーネント構成のうち最初の2つ——**データアセット**と**コントラクト定義**——を詳解する。

### 4つのデータアセットカテゴリ

データコントラクト・アーキテクチャの基盤はデータベースだ。組織内のあらゆるデータソースは以下の4つに分類される：

#### 1. 分析データベース (Analytics Database)

- **特徴：** レスポンスタイムよりスループットを優先（広範なSQLクエリを分で処理）
- **代表例：** Snowflake・BigQuery・Databricks・Amazon Redshift
- **主要パターン：** ELT（Extract-Load-Transform）
- **課題：** 多くのチームはdbtによる変換レイヤーで問題をパッチするが、根本原因（ブレイキングチェンジ）を解決しない

#### 2. トランザクションデータベース (Transactional Database)

- **特徴：** CRUD操作・ACID準拠・第3正規形（3NF）
- **役割：** 最新の運用データの**真実ソース**——データが最初に生まれる場所
- **重要性：** データコントラクトの理想的な配置場所
- **追加用途：** ML予測データのパイプライン、サードパーティデータの取り込み口

**ACIDと不変条件（ACID and invariants）について：**  
Martine Kleppmann（*Designing Data-Intensive Applications*, O'Reilly）より：
> 「一貫性はデータベースの特性ではなく、アプリケーションの特性だ。」

データコントラクトはこれらの制約を、ビジネスとデータが進化するにつれて維持する。

#### 3. イベントソーシング & ストリーム (Event Sourcing & Streams)

**イベントソーシング vs イベントストリームの比較：**

| | イベントソーシング | イベントストリーム |
|---|---|---|
| **ユースケース** | ユーザー人口統計情報の更新、オンラインショッピングカート、サプライチェーンの配送ステータス | デバイスのテレメトリ、高ボリューム金融取引（株式市場）、高トラフィックサイトのクリック追跡 |
| **特徴** | すべてのイベントの**不変ログ**を作成（任意時点の再現可能） | 高ボリュームの集計・リアルタイム処理 |

- **推奨パターン：** Publish-Subscribe（Apache Kafka）
- 多くの組織が両方を使用し、リアルタイムまたはマイクロバッチで分析DBに供給する

#### 4. ファーストパーティデータ / サードパーティプラットフォーム

- **典型例：** Salesforceデータ → Fivetran → BigQuery
- **制約：** サードパーティへのコントラクト強制は通常不可能（レバレッジ不足）
- **CI/CD：** 通常利用できないため、継続的モニタリングとバッチレプリケーション管理が中心

**第三者データのコントラクトワークフロー（7ステップ）：**
1. データコンシューマーがデータ制約を特定
2. コンシューマーがデータアセットのコントラクトを要求
3. データプロデューサーがコントラクトの実現可能性を確認
4. コントラクトがコードとして確定
5. データとコントラクトの継続的モニタリング
6. バッチレプリケーションジョブのキュー管理
7a. 違反時：データアセットオーナーへ通知し失敗プロトコルを実行  
7b. パス時：下流プロセスのためにデータアセットを更新

---

### コントラクト定義の3サブコンポーネント

#### A. データコントラクト スペック (Data Contract Spec)

本書はJSON Schema + YAMLの組み合わせを推奨する。

**スペックの5つの要件：**
1. **技術非依存** — データライフサイクル全体にわたる統合を可能にする
2. **人間が読める形式** — YAMLが一般的な選択
3. **スキーマとセマンティクスの両方をカバー** — 型チェックだけでなくビジネス知識も
4. **セマンティクスバージョニング** — コントラクトは時間とともに進化する必要がある
5. **技術的異質性への対応** — 様々な技術でのデータ定義・制約方法の差異に対応

**コントラクトスペック例（JSON形式）：**

```json
{
  "spec-version": "1.0.0",
  "name": "object_images_contract_spec",
  "namespace": "museum_application",
  "dataAssetResourceName": "postgres://db/museum_database",
  "doc": "metadata for museum objects, including primary images, additional images, and creation timestamps.",
  "owner": {
    "name": "data_engineering_team",
    "contact": "data-eng@museum-app.io"
  },
  "schema": {
    "type": "object",
    "properties": {
      "objectID": {
        "type": "integer",
        "examples": [437133],
        "constraints": {
          "required": true,
          "unique": false,
          "primaryKey": true
        }
      },
      "primary_image": {
        "type": "string",
        "constraints": {
          "required": false,
          "nullable": true
        }
      },
      "additional_images": {
        "type": "array",
        "items": { "type": "string" },
        "constraints": {
          "required": false,
          "nullable": true
        }
      }
    }
  }
}
```

**コントラクト管理フィールド：**

| フィールド | 役割 |
|---|---|
| `spec-version` | スペック形式自体のバージョン（データアセットではなくスペックの変更を追跡） |
| `name` | ユーザー定義のコントラクト名（name + namespace = 複合ユニークID） |
| `namespace` | コントラクト集合の上位グループ名（論理的な「フォルダ」） |
| `dataAssetResourceName` | 対象データソースのURLパス（例: `s3://path/file`, `postgres://db/name`） |
| `doc` | コントラクトの文書化（何を表し・何を強制するか） |
| `owner` | 責任者と連絡先（個人またはチーム。違反時の通知先） |

#### B. ビジネスロジック (Business Logic)

ビジネスロジックはデータを処理するルール・条件・操作の定義であり、静的ではなく**常に進化**する。

**ビジネスロジック ライフサイクル（4ステップ）：**

1. **ビジネスドメインがビジネスロジックを確立** — 組織の機能・手続きを定義
2. **上流エンジニアがビジネスロジックを運用化** — コード化し、ステップとタイムスタンプをトランザクションDBに格納
3. **レプリケーションパイプラインがデータを分析DBへ移動** — トランザクションDB等から分析DBへ
4. **下流データチームが推奨と情報提供** — ビジネスロジックの適用方法を組織全体に伝達

#### C. スキーマレジストリ & データカタログ

コントラクトスペックが「**期待される**」スキーマを定義するのに対し、スキーマレジストリとデータカタログは「**実際の**」スキーマを管理する。

**スキーマ検証ワークフロー（CI/CD経由）：**
1. 開発者がコントラクト下のデータアセットを変更するプルリクエストを作成
2. PRがCI/CDワークフローを起動；Dockerイメージがデータベース付きで起動
3. Dockerデータベースが変更するデータアセットの一部をレプリケート；スキーマ情報を抽出
4. 抽出スキーマをコントラクトスペックの真実ソースと比較
5. スキーマ一致でCIテストが通過；不一致でテスト失敗・マージをブロック

**データカタログの高度ユースケース：**
- ガバナンスポリシーに基づくPII保護の強制
- データリネージに基づく上流変更の影響分析

---

## 02: Ch.6 — 検出と予防

*出典: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ch06.html*

第6章は4コンポーネント構成の後半2つ——**検出（Detection）**と**予防（Prevention）**——を解説する。

### 検出（Detection）の定義

データコントラクトの文脈での検出とは、「**データアセットのコントラクトに関連するメタデータを検出し、関連するステークホルダーにその情報を提示する能力**」だ。

3つの検出活動：
1. コード変更がコントラクトのスキーマ・セマンティクスにどう影響するかを検出
2. ソース〜ターゲット間のライブデータを監査してコントラクトスペックとの整合性を確認
3. 新たなコントラクトを遡及的に適用したときの既存違反を検出

### 7つの検出手法

| # | 手法 | 英語名 | 説明 |
|---|---|---|---|
| 1 | **データ隔離** | Data Quarantining | コントラクトに違反するデータを隔離（別名「デッドレタキュー」）。サードパーティの無通知スキーマ変更シナリオで特に有効。 |
| 2 | **変更データキャプチャ** | Change Data Capture (CDC) | 上流DBのCRUD操作を監視し変更をリアルタイムで下流に伝播。 |
| 3 | **ストリーム処理** | Stream Processing | データ生成時点でリアルタイムに処理・分析。 |
| 4 | **ソースコードフローグラフ** | Source Code Flow Graphs | コードベース内でオブジェクト/ファイルがどのように参照し合うかを示す有向グラフ。 |
| 5 | **データリネージ** | Data Lineage | データがデータインフラ内でどのようにソース化・変換・移動されるかを示す有向グラフ。OpenLineage（openlineage.io）がオープンスタンダード。 |
| 6 | **静的コード解析** | Static Code Analysis (SCA) | コードを実行せずに動作を推論。**最重要手法**。MetaのPysa・Inferなどが代表例。 |
| 7 | **ライブデータモニタリング** | Live Data Monitoring | リアルタイムデータのみを観察・分析。データドリフトの検出に特に有効。 |

> **コードレベル vs データインモーション検出：**  
> コードは変更されていないがビジネスロジックが現実と乖離する「**データドリフト（Data Drift）**」には、コードレベル検出では対応できない——データインモーション検出が必要になる。

### DAMAデータ品質 9次元

DAMA（Data Management Association）の *Data Management Body of Knowledge* より：

| # | 次元（英語） | 日本語 | 例 |
|---|---|---|---|
| 1 | Conformity | 適合性 | タイムスタンプが営業時間中を示しているか |
| 2 | Completeness | 完全性 | 生年月日が日・月・年を含むか |
| 3 | Consistency | 一貫性 | `[1,2,3]`と`["one","two","three"]`が混在しないか |
| 4 | Validity | 妥当性 | サンフランシスコがフランスに関連付けられていないか |
| 5 | Timeliness | 適時性 | データが24時間以内に着地する契約を遵守しているか |
| 6 | Currency | 最新性 | 最終更新が6ヶ月以上前でないか |
| 7 | Cardinality | カーディナリティ | 月は最大12値しか存在しない |
| 8 | Uniqueness | 一意性 | `user_id`はすべてユニーク |
| 9 | Accuracy | 正確性 | アルコール配達サービスが州内で運営されているか |

### WAP（Write-Audit-Publish）パターン

```
[新しいデータ]
      ↓ Write
[ステージングDB] ← データが本番とは隔離
      ↓ Audit（コントラクトスペックに対して品質チェック）
      ↓
  ┌─── 合格 ─────────────────────────────────────┐
  │                                              │
  ↓ Publish                                      ↓ 失敗
[本番DB] ← 下流コンシューマーへ          [デッドレタキュー] ← 隔離
```

### エンドツーエンドリネージ

**ソースコードフローグラフ**（コード内の参照グラフ）と**データリネージ**（データの移動グラフ）を組み合わせたエンドツーエンドリネージにより、データ品質問題の根本原因を自動的にコードまで追跡できる。

従来のプロセス（Mark Freeman の経験）では以下が必要だった：
- ステップ1〜5：データリネージを使ったデータアセットレベルの調査
- ステップ6〜10：ソースコードフローグラフ・`git blame`・PR差分確認・ドキュメント確認・上流エンジニアとの対話

このプロセスは**最善でも数日、最悪では数週間**を要した——これがデータコントラクト開発の直接的動機だ。

### 静的コード解析（SCA）が最重要な理由

1. **シフトレフト：** データ品質問題は分析DBの遥か手前で発生する
2. **可視性：** アプリケーションコードへの可視性を提供する
3. **SDLC統合：** 開発者のワークフロー（IDE・CI/CD）に自然に収まる

---

### 予防（Prevention）の3コンポーネント

#### 1. バージョン管理 (Version Control)

GitがSoftware 1.0の変更管理を変革したように、データコントラクト・アーキテクチャはデータに同じことを行う。

2つの主な利点：
1. コントラクトスペックファイルの変更履歴管理——変更管理の基盤
2. SDLCにおける変更管理の基盤提供

#### 2. CI/CD

最も重要な予防コンポーネント。CI/CDが提供するもの：
- 新しいコードブランチのたびにデータコントラクト違反検出ワークフローを**トリガー**
- 違反検出時にmainブランチへのマージを**ブロック**
- 関連ステークホルダーへの**リアルタイムアラート**

> **重要：** CI/CDはデータチームのニーズをSDLCに後付けではなく**組み込む**——これが最も効果的な予防戦略だ。

#### 3. 違反モニタリングとアラート

**5W1Hアラートフレームワーク：**

| 問い | 内容 |
|---|---|
| **Who（誰）** | 通知すべき最も関連するステークホルダーは誰か |
| **What（何）** | 提示された違反に対して最も関連する情報 |
| **When（いつ）** | 違反アラートをいつ発行すべきか |
| **Where（どこで）** | どのチャネルで通知するか（PRコメント、Slack、メール、ダッシュボード） |
| **Why（なぜ）** | このステークホルダーがなぜ気にすべきか |
| **How（どうやって）** | 違反をどのように解決すべきか |

**⚠️ アラート疲れへの警告：Epic Sepsis Model の教訓**

Epic（米国最大病院EHRベンダー）の敗血症予測MLモデル（ESM）は入院患者の18%でアラートを発したが、陽性予測値はわずか18%（8人中7人はアラートが出ても敗血症を発症しなかった）。*JAMA*の外部検証研究がこれを厳しく批判。

**教訓：** ターゲットステークホルダーにとって意味のないアラートは、アラート疲れとシステムへの信頼の喪失を生む。

---

## 03: Ch.7 — データコントラクトの実装

*出典: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ch07.html*  
*GitHub: https://github.com/data-contract-book/chapter-7-implementing-data-contracts*

第7章は理論を実際のコードへと変換する。**ミュージアムアプリケーション**のモックシナリオを通じてエンドツーエンドのワークフローを実装する。

### モックシナリオ

**あなたの役割：** データプラットフォームエンジニア  
**シナリオ：** 世界中の主要美術館のアートメタデータと画像を集約するアプリを構築するチームを支援する。長期ビジョンはテーマ・ムード・トピックでアートを横断検索できるAIアプリ（例：「青い色調のゴッホ作品を見つけて」）。

**データセット：**
- Metropolitan Museum of Art Open Access — 406,000以上の高解像度画像、470,000以上のアートワーク
- サンドボックス: ~3,000レコードのサンプル（Museum Collection APIより）
- データストア: JSONアレイ → PostgreSQL
- プライマリキー: `object_id`（全テーブル共通）

### プロジェクト構造

```
/workspace
├── .devcontainer.json
├── docker-compose.test.yml      # CI/CD用（テスト実行）
├── docker-compose.yml           # 開発環境用（DBシードあり）
├── requirements.txt
├── .github/
│   └── workflows/
│       └── docker-image.yml     # GitHub Actions ワークフロー
└── data_contract_components/
    ├── contract_definition/
    │   └── object_images_contract_spec.json
    ├── data_assets/
    │   ├── seed_db.py
    │   ├── alembic.ini
    │   └── db_migrations/
    │       └── versions/
    ├── detection/
    │   ├── contract_coverage_detector.py
    │   └── contract_violation_detector.py
    └── prevention/
        └── test_data_contract_violations.py
```

### 4つの実装コンポーネント

#### Component A: データアセット

AlembicによるDBマイグレーションでデータアセットをコードとして管理する。

> **重要な洞察：** データコントラクト実装に必要なのは**メタデータのみ**——実際の本番データへのアクセスは不要。これがセキュリティ要件を大幅に緩和する。

#### Component B: コントラクト定義

`object_images_contract_spec.json` に以下を定義：
- `spec-version`, `name`, `namespace`, `dataAssetResourceName`
- `doc`, `owner`（name + contact）
- `schema`（`table_catalog`, `table_schema`, `object_images`テーブル定義）

**推奨：** 現在のスキーマをベースラインとして使用する。Linux Foundation下のOpen Data Contract Standard（ODCS）も参照可能。

#### Component C: 検出（Detection）

**3ステップの検出フロー：**
1. データカタログから現状スキーマを取得
2. コントラクトスペックからの期待値を読み込み
3. 2者を比較して違反を表面化

**contract_coverage_detector.py — カバレッジ確認（左結合）：**

```python
def detect_coverage_in_data_catalog(self) -> List[str]:
    coverage = self.get_contract_spec_coverage()
    coverage_df = pd.DataFrame(coverage)
    catalog_df = get_data_catalog()

    merged = coverage_df.merge(
        catalog_df,
        on=['table_catalog', 'table_schema', 'table_name'],
        how='left',
        indicator=True
    )
    missing_assets_df = merged[merged['_merge'] == 'left_only']
    missing_table_names = missing_assets_df['table_name'].tolist()
    return missing_table_names  # 空リスト = OK
```

**contract_violation_detector.py — カラムレベル制約検証：**

```python
def detect_constraint_violations(self) -> List[Dict[str, str]]:
    contract_specs_df = pd.DataFrame(
        self.transform_contract_specs_to_catalog_format()
    )
    catalog_df = get_data_catalog()

    merged = contract_specs_df.merge(
        catalog_df,
        on=['table_catalog', 'table_schema', 'table_name', 'column_name'],
        how='left',
        suffixes=('_contract', '_catalog'),
        indicator=True
    )
    # 型の不一致・カラムの欠損などをviolationsとして返す
    ...
```

**実装メモ：** PostgreSQLの `information_schema` テーブルを使用（多くのSQL-basedデータベースで共通）。

#### Component D: 予防（Prevention）

GitHub Actions（`docker-image.yml`）がCI/CDを駆動。`test`ジョブが実行するコマンド：

```bash
python -m unittest \
  data_contract_components/prevention/ \
  test_data_contract_violations.py -v \
  > /workspace/test_output.log 2>&1
```

- 検出スクリプトの出力長がゼロより大きい場合にユニットテストが**失敗**
- 失敗時：`test_output.log`がGitHub PRにコメントとして投稿される
- フィールドレベルの違反詳細を含む（例：型変更 `text → integer`）

---

### エンドツーエンドワークフロー（13ステップ）

| ステップ | アクション |
|---|---|
| 01 | GitHub ActionsがCI/CDワークフローを起動 |
| 02 | `docker-image.yml`の`test`ジョブがDocker Composeを起動 |
| 03 | `docker-compose.test.yml`がコンテナインフラレイヤーとして使用される |
| 04 | コンテナが`postgres`サービスをビルド |
| 05 | `postgres`がPostgreSQL 17 DBを起動し「healthy」シグナルを送信 |
| 06 | 「healthy」受信後、`devcontainer`（Python）サービスをビルド |
| 07 | `devcontainer`がAlembicでDBマイグレーションを実行（提案変更のスキーマを適用） |
| 08 | 依存関係のインストールとDB確認後、`test_data_contract_violations.py`を実行 |
| 09 | Pythonの`unittest`が2つの検出スクリプトを実行 |
| 10 | 検出スクリプトがDBとコントラクトスペックのメタデータを比較 |
| 11 | `object_images_contract_spec.json`を参照してDB実態とスペック期待値を照合 |
| 12 | 違反検出時、`test_output.log`として保存 |
| 13 | テスト失敗の場合、ログをGitHub PRコメントとして投稿 |

---

### コントラクト違反シナリオ：ウォークスルー

**シナリオ：** `object_images`テーブルの正規化リクエスト

```
-- 現在                          -- 要求変更後
object_images                    object_images
────────────────────             ────────────────────
PK: object_id (Integer)          PK: object_id (Integer)
primary_image (Text)             image (Text)          ← 名称変更
additional_images (ARRAY[text])  is_primary_image (Bool) ← 新規追加
created_at (DateTime)            created_at (DateTime)
```

**ステップ3 — ユニットテストで違反検出：**

```bash
python -m unittest data_contract_components/prevention/test_data_contract -v
```

```
test_data_contracts_against_data_catalog ... FAIL

1. Contract: object_images_contract_spec  Table: object_images
   Issue: Column 'primary_image' is defined in contract but missing from data catalog

2. Contract: object_images_contract_spec  Table: object_images
   Issue: Column 'additional_images' is defined in contract but missing from data catalog

2 tests in 0.240s (failures=1)
```

**解決の流れ：**

1. **ステップ5：変更を一時リバート** — `git revert HEAD && git push`
2. **ステップ6：下流チームと協議** — データサイエンスチームが先週から依存開始。移行に最低1ヶ月必要。
3. **ステップ7：並行テーブル戦略の決定**
   - 旧`object_images`を1ヶ月間維持（DSチーム移行期間）
   - 新`object_images_normalized`を作成
   - 新テーブル用の新コントラクトスペックを追加
4. **ステップ9：新テーブルのマイグレーションを追加**

```python
op.create_table(
    'object_images_normalized',
    sa.Column('object_id', sa.Integer, primary_key=True),
    sa.Column('image', sa.Text),
    sa.Column('is_primary_image', sa.Boolean),
    sa.Column('created_at', sa.DateTime, nullable=False),
)
```

5. **ステップ10：全テスト通過 → PRマージ**

```
test_all_contract_assets_present_in_catalog ... ok
test_data_contracts_against_data_catalog ... ok

2 tests in 0.176s
```

> **核心的価値：** コントラクト違反が自動的に検出されたことで、本番前に下流チームとの潜在的な断絶を回避できた。コードベースのコントラクト管理がチーム間の非同期コミュニケーションを置き換えた。

---

## 04: Ch.8 — 本番環境のケーススタディ

*出典: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ch08.html*

第8章はPart IIとPart IIIの橋渡しに位置する。3つの独立した組織が、データ管理の現状維持がもはや機能しないという現実に直面し、データコントラクトへと独自に辿り着いた軌跡を描く。

---

### ケーススタディ 1: Convoy

**事業概要：** 貨物荷主とトラック運送業者の自動マッチングプラットフォーム  
**成長軌跡：** 250名（Series C時）→ 1,500名以上。データチーム 6〜8名 → **80名超**

#### 問題の顕在化

> 「Once we started expanding out the data science team, making it easier for people to write dbt models, we also started to lose visibility into our data pipelines, and understanding what was going on with our data.」  
> — Adrian Kreuziger（データエンジニア）, Convoy

#### 技術的転換点：Kafkaの導入

> 「I think that the actual big driving point was the introduction of Kafka and trying to get our developers and our engineering team to start emitting more semantic events… And what ended up happening was that we would have people trying to reverse engineer real-world events…」  
> — Adrian Kreuziger, Convoy

**根本原因の発見：** Convoyには**データセットのフルライフサイクルを所有するチームが存在しなかった**。そのため、ソースでデータ品質を強制するメカニズムがなかった。

#### 実装と心理的シフト

コントラクトを独立したリポジトリでバージョン管理することで、より安定・予測可能なデータ環境が生まれた。これは技術的改善にとどまらず——**データ所有権と責任についての根本的に異なる考え方**への変革だった。

> 「It basically made data much more like a first-class citizen in developers' minds. By going through this exercise with a team of saying, 'Hey, let's rethink your data model, let's fix it up,' you had buy-in from the developer going forward.」  
> — Adrian Kreuziger, Convoy

#### 教訓

> 「You need to make the data contracts part of the normal developer flow. They need to be something that's part of the standard process… rather than an afterthought that teams get stuck with.」  
> — Adrian Kreuziger, Convoy

---

### ケーススタディ 2: Glassdoor

**事業概要：** 求人・企業レビュープラットフォーム（ビジネスの根幹 = **信頼**）  
**出典：** Zakariah Siyaji「Data Quality at Petabyte Scale: Building Trust in the Data Lifecycle」— Glassdoor Engineering Blog

#### 採用の「完璧な嵐」

3つの条件が揃って採用が起きた：
1. **痛みが存在した**
2. **痛みが有意義な収益インパクトにつながった**
3. **痛みが特定可能な根本原因に帰結した**

#### 実装：5つのコンポーネント

| コンポーネント | 説明 |
|---|---|
| **Write-Audit-Publish (WAP)** | 各バッチデータが本番環境移動前にステージングエリアでレビュー（Blue-Green Deploymentとも説明） |
| **静的コード解析（SCA）** | CI/CDワークフロー内に埋め込み。フロントエンドコード変更がバックエンドに与える影響を自動検出。PRレベルでブロック。 |
| **LLMによる異常検出** | コントラクトとSCAのメタデータにLLMを適用し、ビジネスコンテキストと影響を理解。セマンティック制約に特に強力。 |
| **リアルタイムモニタリング** | DataDog使用。Kafkaとの組み合わせでデータ関連ソフトウェア障害を即座に検出。 |

#### 文化的変化

> 「Proactive data quality isn't about imposing rules at the last minute; it's about instilling confidence in the data… by addressing the psychological dimension of trust through shared responsibility, trust and accountability are distributed across all stakeholders.」  
> — Glassdoor Engineering Blog

プロダクトエンジニアはデータコントラクトスペックを参照してデータ期待値を理解し、他の開発チェックと同様にそれに対して責任を負うようになった。

---

### ケーススタディ 3: Adevinta Spain

**本社：** バルセロナ, スペイン  
**事業：** 欧州向けオンラインマーケットプレイス群  
**データ規模：** 1日約**4テラバイト**  
**出典：** Sergio Couto「Creating Source-Aligned Data Products in Adevinta Spain」

#### 元のアーキテクチャの課題

1. **正規表現によるトピックマッチング：** プレフィックスに合致するすべてのトピックを取り込み → ユースケースのないデータ取り込み → ストレージ・処理コスト増大
2. **スキーマ検証なし：** スキーマはイベント初回到着時にのみ使用。1イベントごとの検証は行われていなかった

> 「We're consuming a lot of events and the typical daily state of engineering is that… something breaks every day.」  
> — Sergio Couto, Adevinta Spain

#### 4つの指導原則

1. **プッシュアーキテクチャへの移行** — データプロデューサーは公開して終わりではなく、製品としてメンテナンスする
2. **アナリストの手動検証に依存しない** — 内部データプロダクトの品質検証を手作業に頼らない
3. **手動ガバナンスよりコードへの組み込み** — 手動ガバナンスはコードへの組み込みより効果が低い
4. **コードと同様にデータを扱う** — 厳密なテスト・バージョニング・継続的改善をデータにも適用

#### 2段階の実装

**Stage 1: 定義時（Definition Time）**

1. データプロデューサーがJSON Schemaを定義し、**GitHub**のスキーマリポジトリへアップロード
2. **生成AI**がスキーマをコントラクトスペック提案に変換（データコンシューマーがレビュー）
3. プロデューサーとコンシューマーが制約に合意 → 自動化ワークフローを起動
4. バリデーション通過時、**Airflow DAGが自動生成**
5. データコンシューマーがDAGをマージ・有効化して運用開始

**Stage 2: ランタイム取り込み（Runtime Ingestion）**

1. 各イベントがAirflow DAGを起動 → Kafkaパイプラインを**Databricks**へ取り込みSparkで処理
2. **S3**からコントラクトスペックを取得・フィルタリング
3. 定義済みスキーマをデータに適用 → 違反はデッドレタキュー（隔離）へ
4. **PII処理：** 非分析用PIIは削除、残りはGDPRに準拠して分離
5. PII/非PIIを分析DB（**Delta Lake** / タイムトラベル機能付き）に格納

**テックスタック全体像：**
```
Segment → Kafka → Airflow DAG（イベント毎トリガー）
        → Databricks（Spark処理）
        → S3（コントラクトスペック格納）
        → スキーマ検証
        → Dead Letter Queue（隔離）または
        → Delta Lake（分析DB / タイムトラベル機能付き）
```

#### 副次的価値：GDPR自動化

データコントラクトの最大の副次的価値は**GDPRコンプライアンスの自動化**だった。以前は手作業に大きく依存していたGDPRプロセスが、コントラクトベースのPIIフィールド識別とルーティングによって自動化された。

---

### 3社に共通する5つのパターン

すべてのケーススタディに流れる横断的パターン：

1. **データチームはリアクティブなトリアージに失われた時間とフォーカスを費やしていた** — プロアクティブな価値創出ではなく、発生した問題への反応に膨大なエネルギーを費やしていた
2. **リアクティブなトリアージの効果は著しく制限されていた** — 憶測・試行錯誤・限られた可視性への依存
3. **データプロデューサーとコンシューマーの上流ミスアライメント** — ほぼすべての問題の根本原因
4. **適切な人（リーダーシップ）が痛みを感じるまで組織は変化しなかった** — 「完璧な嵐」が採用のトリガー
5. **技術ツールだけでは不十分——文化的・組織的変革が同様に重要** — 3社が独立してこの結論に達した

---

## 05: Part II まとめ

Part II（第5〜8章）は、データコントラクトの概念を実際に動くシステムとして具現化するための完全なロードマップを提供した。

### 実装の要点

| # | 要点 | 詳細 |
|---|---|---|
| 01 | **4コンポーネント構成が実装の骨格** | データアセット → コントラクト定義 → 検出 → 予防の4層が相互連携 |
| 02 | **メタデータのみで実装可能** | 実際の本番データへのアクセス不要——セキュリティ要件を大幅に緩和 |
| 03 | **SCA + CI/CDが最強の予防** | コードレベルで問題を捕捉し、「ビルド失敗」として即時フィードバック |
| 04 | **WAPパターンがランタイム保護の中核** | ランタイムデータがダウンストリームに到達する前にコントラクト違反を捕捉 |
| 05 | **技術 + 文化の変革** | Convoy・Glassdoor・Adevintaの3社が独立して確認した普遍的な真実 |
| 06 | **採用のトリガーは「完璧な嵐」** | 痛みの存在 × 明確な収益インパクト × 特定可能な根本原因 |

### Part II の本質的なメッセージ

> データコントラクトは**「契約としてのコード（contract as code）」**だ。GitがSoftware 1.0の変更管理を変革したように、データコントラクトはData Engineering 2.0の変更管理を変革する。技術的なツールキットは今すぐ動かせる——しかし、その真の価値はツールにあるのではなく、**プロデューサーとコンシューマーの間の明示的・強制可能な合意**にある。

---

## 06: 参考文献

| # | 出典 |
|---|---|
| [1] | Chad Sanderson, Mark Freeman, B. E. Schmidt, *Data Contracts: Developing Production-Grade Pipelines at Scale*, O'Reilly Media, 2025. ISBN: 978-1-098-15763-0. Ch.5〜8: https://learning.oreilly.com/library/view/data-contracts/9781098157623/ |
| [2] | Chapter 7 GitHub Repository: https://github.com/data-contract-book/chapter-7-implementing-data-contracts |
| [3] | Zakariah Siyaji, "Data Quality at Petabyte Scale: Building Trust in the Data Lifecycle," Glassdoor Engineering Blog. |
| [4] | Sergio Couto, "Creating Source-Aligned Data Products in Adevinta Spain," Medium. Shift Left Data Conference 2025: https://www.gable.ai/blog/data-contracts-in-the-real-world-the-adevinta-spain-implementation-sergio-catoira-shift-left-data-conference-2025 |
| [5] | Mark Freeman II on LinkedIn, February 19, 2025: https://www.linkedin.com/posts/mafreeman2_glassdoor-has-one-of-the-most-advanced-activity-7298037033761943562-liLj |
| [6] | Chad Sanderson at Convoy, YouTube, December 13, 2022: https://www.youtube.com/watch?v=Gr2GIsTnsSA |
| [7] | DAMA International, *DAMA Data Management Body of Knowledge (DAMA-DMBOK)*. — Nine Dimensions of Data Quality. |
| [8] | Confluent Documentation, "Data Contracts for Schema Registry": https://docs.confluent.io/platform/current/schema-registry/fundamentals/data-contracts.html |
| [9] | Anders Møller and Michael I. Schwartzbach, *Static Program Analysis*. — SCA definition. |
| [10] | Wong et al., "External Validation of a Widely Implemented Proprietary Sepsis Prediction Model," *JAMA Internal Medicine*. — Epic Sepsis Model alert fatigue case study. |
| [11] | OpenLineage — Open Standard for Data Lineage: https://openlineage.io |
| [12] | Open Data Contract Standard (ODCS), Linux Foundation: https://github.com/datacontract/datacontract-specification/blob/main/definition.schema.json |
| [13] | Nabin Debnath, "The 'Shift-Left' Imperative: Implementing Data Contracts in CI/CD Pipeline," DEV Community, October 2025: https://dev.to/nabindebnath/the-shift-left-imperative-implementing-data-contracts-in-cicd-pipeline-40cl |
| [14] | Soda.io, "Preventing Issues with Data Contracts & Testing," June 2025: https://soda.io/blog/preventing-issues-with-data-contracts-and-testing |
| [15] | JSON Schema core definitions: https://json-schema.org/draft-04/draft-zyp-json-schema-04 |

---

*本記事は Chad Sanderson, Mark Freeman, B.E. Schmidt 著『Data Contracts』（O'Reilly Media, 2025）の個人学習ノートです。書籍の内容を要約・解説したものであり、著作権は原著者および O'Reilly Media に帰属します。*
