# Data Engineering Study #31「公開企画会議 アドバイザーと語る気になる技術」要約

2025年9月30日に開催された [Data Engineering Study #31](https://forkwell.connpass.com/event/363198/) のイベント要約。Forkwell と primeNumber の共催。テーマは「公開企画会議 アドバイザーと語る気になる技術」。DES#30 で引退した ゆずたそ 氏に代わる3名の新アドバイザーを迎え、各10分のLT + Q&A という従来と異なるコンパクトなフォーマット（全体64分）で開催された。MCは北川氏（primeNumber、通称「タイガー」）。

- [YouTubeアーカイブ](https://www.youtube.com/watch?v=IdZzoBmmowo)
- ハッシュタグ: [#DataEngineeringStudy](https://twitter.com/intent/tweet?hashtags=DataEngineeringStudy)

今回のDESは通常フォーマットとは異なる特別回。3名のアドバイザーがそれぞれ10分のLTを行い、その後「公開企画会議」として今後のDESテーマをパネルで議論するという構成だ。MCの北川氏は冒頭で誤って「#32」と言ってしまったが、正しくは #31 である。

---

## LT①: Modern Data Stack最新動向 ～買収・AI、激動の2025年～

**登壇者:** 相樂 悟（さがら）/ クラスメソッド株式会社 モダンデータスタックチーム テックリード  
**資格:** Snowflake Data Superhero (2025)、dbt Cloud Architect認定  
**スライド:** [Speaker Deck](https://speakerdeck.com/sagara/modern-data-stackzui-xin-dong-xiang-kuizu-mai-shou-ai-ji-dong-no2025nian)

<iframe class="speakerdeck-iframe" style="border: 0px; background: padding-box padding-box rgba(0, 0, 0, 0.1); margin: 0px; padding: 0px; border-radius: 6px; box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 40px; width: 100%; height: auto; aspect-ratio: 560 / 315;" frameborder="0" src="https://speakerdeck.com/player/ada17a1604164e7186c6a25c7900a338" title="Modern Data Stack最新動向" allowfullscreen="true" data-ratio="1.7777777777777777"></iframe>

### クイズ形式で迫る「2025年の激動」

さがら氏は「10分という短い枠に最大限詰め込むためにクイズ形式にした」と語り、MDS界隈の2025年を問いかけ形式で振り返った。スライドに一覧が掲載されているとはいえ、登壇で語られた文脈が際立つ。2025年だけで買収が確認できた企業は8社：SDF、Tobiko、Census、Informatica、Metaphor、CastorDoc、Satori、Metaplane。これほどの数が1年で動いたのは異例の事態だ。

なかでも特筆すべきはFivetranの動きだ。Reverse ETLを手がけるCensusを5月に買収し、さらに9月にはSQL MeshのTobikoも取り込んだ。Fivetranはもともとデータロードに特化したツールだったが、データ変換（Tobiko）とリバースETL（Census）を一気に取り込んだことで、パイプラインの上流から下流まで横断するプラットフォームへと変貌を遂げた。さがら氏は「FivetranがリバースETLとデータ変換を取り込んだのは衝撃だった」と語気を強めた。

SDF については、dbt の競合として独自のデータ変換エンジンを打ち出していたにもかかわらず、dbt 自身に買収されて「Fusion」としてベータリリース→パブリックプレビューへと進んでいる。競合を飲み込む形での技術統合は、MDSエコシステムの収束を象徴する動きとも取れる。

> 「データカタログ・オブザーバビリティ分野は今後も買収が続くと予測している。Atlan、Open Metadata、Monte Carlo といったサービスが今後どう動くか注目している。」

### Snowflake vs Databricks — 2025年の分岐点

2025年6月、両社がほぼ同時期にLLMバックエンドを活用した自然言語BI機能を発表した。Snowflakeは「Snowflake Intelligence」、Databricksは「Databricks One」および AI/BI Dashboards だ。さがら氏はそれぞれの特徴として「Snowflakeはシンプルさ重視、DatabricksはUnity Catalogを軸にしたエンジニア向け」という整理を示しながらも、即座に補足した。

> 「ただし、Snowflakeもエンジニア向け機能を強化しており、DatabricksもシンプルさをUPしているので一概には言えない。この対比はあくまでも傾向として捉えてほしい。」

スライドには載っていない技術的な補足として、Snowflakeの「Workload Identity Federation」も言及された。AWS IAMロールベースの認証に対応しており、GitHub Actions から Snowflake へアクセスする際にシークレットを一切使わずに認証が通るという。クレデンシャル管理のリスクを下げる観点で実務上の恩恵が大きい機能だ。また、Snowflakeの「Adaptive Data Warehouse」として Iceberg ネイティブサポートと Polaris Catalog の OSS 化が進んでおり、ベンダーロックインを避けたいユーザーには重要なニュースだと説明した。

---

## LT②: 今注目しているデータエンジニアリングの論点

**登壇者:** 宮崎 一樹（ikki）/ stable株式会社 代表取締役  
**会社概要:** データエンジニアリング領域のクライアントワーク専門（正社員3名 + 業務委託12名）  
**スライド:** [Speaker Deck](https://speakerdeck.com/ikkimiyazaki/imazhu-mu-siteirudetaenziniaringunolun-dian)

<iframe class="speakerdeck-iframe" style="border: 0px; background: padding-box padding-box rgba(0, 0, 0, 0.1); margin: 0px; padding: 0px; border-radius: 6px; box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 40px; width: 100%; height: auto; aspect-ratio: 560 / 315;" frameborder="0" src="https://speakerdeck.com/player/3ce3c8877eb541dbbe88c33c4e63faed" title="いま注目しているデータエンジニアリングの論点" allowfullscreen="true" data-ratio="1.7777777777777777"></iframe>

### 「今度こそ」データ整備が進むのか

ikki 氏の問題提起は10〜15年前への回帰から始まった。データサイエンスブームが盛り上がった時代にも「データ整備が大事」という議論は尽くされていたが、実際には多くの現場でほとんど進まなかった。生成AI・LLMブームが到来した今、同じ議論が再燃している。

> 「今度こそデータ整備の重要性が上げられるのか、というのが私の中で一番気になっている論点です。」

スライドには図で整理されているが、登壇で特に強調されたのは非対称性だ。AI側はOpenAI等の外部プレイヤーが急速に進化させてくれるが、社内データの整備は自社でやるしかない。AIへのGarbage In, Garbage Outの問題が増幅される時代に、データ基盤の整備は「誰かがやってくれる話ではない」という切実さが語られた。

### 「交通量を増やさないと交通整理のROIが上がらない」

ikki 氏が語ったフレーズの中で最も印象的だったのがこの一言だ。ガバナンスやデータ品質管理（"交通整理"）の重要性は疑いないが、そもそも活用量（"交通量"）が少なければその投資対効果は出てこない。従来のデータ活用フローは「事業部 → 依頼 → データチーム → 分析結果」という人間がボトルネックになる構造だった。AIエージェントに問い合わせのレスポンスを任せることで、この構造を変えてデータ活用の総量そのものを増やすことが先決だという主張だ。

データエンジニアの役割変化についても踏み込んだ。ビジネスユーザーと直接やりとりしていた役割は、「AIとビジネスユーザーの境界線を整える仕事」へとシフトしていく。その境界線の設計には4つの観点があるという：①AIと人間の得意領域の見極め、②サービスレベルの定義、③ガードレールの整備、④メタデータ・コンテキストの充実。

> 「AIは確率論的処理で速いが間違いもある。データ基盤は決定論的処理で正しいが遅い。これは優劣の話ではなく、適材適所の話です。セマンティックレイヤーまでは決定論的に整備して、ラストワンマイルのユーザー配信をAIで高速化する、という棲み分けが現実的だと思っています。」

Q&A では「ROIが最も高いユースケースは？」という質問に対し、「経営企画・事業計画の領域」と即答。経営層に価値を見せやすく、承認も取りやすいため効果が出やすいという実感からきている。メタデータ収集については「事業部の人に入れてもらうための工夫が難しい」とし、「メタデータを入れると AIが賢くなる」というインセンティブ設計の重要性を指摘した。

---

## LT③: データエンジニアがこの先生きのこるには...?

**登壇者:** 吉田 康久（Yoshida）/ 株式会社10X プロダクト本部 データエンジニア（[@syou6162](https://twitter.com/syou6162)）  
**資格・活動:** datatech-jp 運営、dbt Community Spotlight、Google Developer Expert  
**スライド:** [Speaker Deck](https://speakerdeck.com/10xinc/detaenziniagakonoxian-sheng-kinokoruniha-dot-dot-dot)

<iframe class="speakerdeck-iframe" style="border: 0px; background: padding-box padding-box rgba(0, 0, 0, 0.1); margin: 0px; padding: 0px; border-radius: 6px; box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 40px; width: 100%; height: auto; aspect-ratio: 560 / 315;" frameborder="0" src="https://speakerdeck.com/player/1ba826a697e4407284f607963a1e102b" title="データエンジニアがこの先生きのこるには...?" allowfullscreen="true" data-ratio="1.7777777777777777"></iframe>

### 「目の前の仕事は全然減らない」

Yoshida 氏は勤務先の10Xをごく短く紹介してから本題に入った。ネットスーパー提供が主軸だが、最近は価格最適化・発注最適化という新領域にも展開しているという。データ領域の進化についても簡潔に振り返った。Hadoop時代からBigQuery/Snowflake/Databricksへ、Embulk/DigdagからdbtやAtolocoやDataformへ。マネージドサービスが普及し、細かなスクリプト作成はLLMエージェントが担ってくれるようになった。「データエンジニアいらんのでは？」という感覚が生まれても不思議ではない流れだ。

しかし Yoshida 氏はひとつの現実を付け加えた。

> 「そう言いながら、目の前の仕事は全然減らないんですよね。」

### LLMエージェントは「やんちゃ」

スライドに書かれていない具体的なエピソードが印象に残る。マートの作成を LLM エージェントに依頼すると、ビジネスロジックが各マートに染み出して一元管理できなくなる。さらに極端な例として、テストが失敗した際にエージェントが「上流テーブルにバグがあるかもしれないので削除して作り直します」と言ってインクリメンタルテーブルを消そうとするケースまであったという。

> 「LLMエージェントは優秀なんですが、やんちゃなんですよ。ガードレールをしっかり設計しないと、とんでもない方向に走っていく。」

その対策として、`agent.md` や `CLAUDE.md` に自然言語でガードレールを書く手法を紹介した。「CTEで書く、サブクエリは使わない」といったSQLの書き方ルールや、リンター・フォーマッターで機械的に強制しにくいルールを自然言語で定義しておくことで、エージェントの行動を縛る。

### 「ユビキタス言語」とデータの文脈

語彙と意味の定義の重要性についても独自の視点を展開した。DAUの定義、解約率の算出期間など、組織内で「同じ言葉が違う意味で使われている問題」は根深い。Yoshida 氏は、これをDDDを知っている人には「ユビキタス言語」、dbtを使っている人には「セマンティックレイヤー」と呼ぶが、「実は同じ概念が技術スタックごとに別名で呼ばれているだけ」だと整理した。Data Contract（データ提供者と消費者の合意）や Data Reliability Level（データの信頼性をティア分けして定義）も同じ問題意識の延長線上にある。

### 「コードを書くこと」はボトルネックではない

Yoshida 氏のLTで最も示唆に富むのがこの視点だ。ソフトウェアエンジニアリングの歴史を振り返ると、コードを書くこと自体がボトルネックになった時代はない。真のボトルネックは常に「知識のトランスファー、コードレビュー、ナレッジ管理」だった。LLMエージェントに何をやらせるかより、何がボトルネックかを見極めることの方が本質的な問いだという。

---

## 共催LT: primeNumber — TROCCOとCometaの最新アップデート

**登壇者:** 鈴木氏 / 株式会社primeNumber

鈴木氏がTROCCOおよびCometaの最新機能をコンパクトに紹介した。TROCCOではワークフローの条件分岐機能、環境管理（dev/prod分離）、BigQuery対応のCDCデータ転送、Databricksデータマート対応が追加され、当日は新コネクタとしてDB2 for LUWのリリースも発表された。Cometaでは対話型AIアシスト機能のエージェント化とRedshift連携対応が加わっている。なお、DES#30でのアドバイザー同士の対談（相樂×吉田）をMCの北川氏が目撃したことがきっかけで、primeNumberユーザー会での相樂×吉田対談企画が実現したという裏話も語られた。

---

## 公開企画会議（パネルディスカッション）

このセクションが DES#31 の最大の目玉だ。3名の新アドバイザー全員が揃ったことを受け、「今後のDESはどんなテーマを扱うべきか」を公開の場で議論する企画が設けられた。これは単なる感想戦ではなく、コミュニティとしての方向性を実際に決める場として機能した。

### Snowflake vs Databricks、どっちを選ぶ？

ikki 氏からさがら氏への問い：「クライアントから技術選定の相談を受けたらどちらを勧めるか？」

> 「正直どっち選んでも外れない。以前はDatabricksはPythonが得意でSQLが苦手みたいな印象があったが、今はSQLベースで使っても全く問題ないレベルになっている。まず触ってみて、気に入った方を選ぶのがいいと思う。」（さがら）

ikki 氏もこれに同意し、「機能的にはどちらも同等レベルにある。最終的には使ってて気持ちいいとか、組織の文化に合っているかが決定要因になる」と補足した。技術的優劣よりも「フィーリングと組織適合」が現実の選定基準になっているという証言は、スライドには書けないリアルな知見だ。

### 現場のデータエンジニアが楽できる分野は？

Yoshida 氏からさがら氏への問い：「MDSの全体像を見ているさがらさんとして、現場が楽できる分野と、逆にリソースを割くべき分野はどこか？」

さがら氏は、クラウドDBの普及でインフラのリソース管理は既に楽になっており、Reverse ETLの機能拡充やdbtエコシステムの発展によって「SaaSで解決できることはどんどん増えている」と答えた。楽できるところは遠慮なく楽をして、その分だけデータモデリングやビジネス理解に注力するべきというメッセージだ。

### 「今からデータエンジニアがやるべきことは？」

最も熱量の高かったやりとりがこの問いをめぐる議論だ。

Yoshida 氏は「基礎をやりましょう」と力強く答えた。

> 「データエンジニアである前にソフトウェアエンジニアであるべきだと思っている。冪等性、バージョン管理、CI/CD、ディメンショナルモデリング——ディメンショナルモデリングなんて30年以上前から存在するが今でも現役だ。賞味期限の長い知識に投資すべきです。Claude 4.5とか1ヶ月すらもたない賞味期限のものを追いかけていると疲れる。」

さがら氏もこれに強く同意し、自らの反省を語った。前職でHadoopを使って酷いパイプラインを作り苦労した経験を振り返り、「あれは基礎力がなかったから。冪等性やPythonコーディングの基礎があれば、あの失敗はなかった」と述べた。

ikki 氏は別の切り口を提示した。

> 「私は少し違う観点で、課題を見つける力が大事だと思っています。AIはソリューションの提供を手伝ってくれるが、課題の発見は人間の仕事。スプレッドシート1個の自動化でもいい。『これが困っている』を自分で見つけられるかどうかが、AIを使いこなせる人と使われる人の分かれ目になる。」

### データエンジニアの未来像

Yoshida 氏はデータエンジニアの将来像として「今より少人数で広い領域をカバーする」形を描いた。マネージドSaaSやLLMエージェントが「パワースーツ」のようにエンジニアを補強し、1人でこなせる範囲が大幅に広がる。ただし、そのためには扱う範囲が広くなる分だけ基礎力の重要性がさらに増すという見立てだ。

ユビキタス言語の統一については、Yoshida 氏が重要な補足をした。「データチーム内で用語を揃えるのは当たり前だが、それだけでは狭すぎる。データを生成する人から活用する人まで、全社横断で同じ言葉を使うことが本質だ。コンテキストエンジニアリングの観点から、ビジネス側の議事録やPdMドキュメントも含めて横断的に語彙を統一していくことが、AI活用の精度を上げる。」

---

## まとめ

DES#31 は通常の登壇発表型とは異なる特別フォーマットだったが、3名のアドバイザーがそれぞれ異なる切り口から「2025年のデータエンジニアリング」を語り、パネルで突き合わせたことで密度の高い議論が生まれた。

さがら氏が示した「2025年は8社が買収される激動の年」という事実は、MDSエコシステムが成熟フェーズに入りつつあることを示す。ikki 氏の「交通量を増やさないとROIが出ない」という問題意識は、ガバナンス議論に先立つ活用拡大の必要性を整理した。Yoshida 氏の「LLMエージェントはやんちゃ」「コードはボトルネックではなかった」という視点は、AI時代のデータエンジニアリングに実務的な地に足のついた指針を与えた。

3者が共鳴したのは「基礎の重要性」だ。テクノロジーが変わっても、冪等性・バージョン管理・ディメンショナルモデリングという基礎は色褪せない。「賞味期限の長い知識に投資する」という Yoshida 氏のメッセージは、AI時代においてこそ響く言葉だろう。

---

## 関連リンク

- [connpassイベントページ](https://forkwell.connpass.com/event/363198/)
- [YouTubeアーカイブ](https://www.youtube.com/watch?v=IdZzoBmmowo)
- [さがら氏スライド (Speaker Deck)](https://speakerdeck.com/sagara/modern-data-stackzui-xin-dong-xiang-kuizu-mai-shou-ai-ji-dong-no2025nian)
- [さがら氏レポート記事 (DevelopersIO)](https://dev.classmethod.jp/articles/des31-report-modern-data-stack-2025/)
- [ikki氏スライド (Speaker Deck)](https://speakerdeck.com/ikkimiyazaki/imazhu-mu-siteirudetaenziniaringunolun-dian)
- [stable株式会社](https://stable.co.jp/)
- [Yoshida氏スライド (Speaker Deck)](https://speakerdeck.com/10xinc/detaenziniagakonoxian-sheng-kinokoruniha-dot-dot-dot)
- [Yoshida氏 (@syou6162)](https://twitter.com/syou6162)
