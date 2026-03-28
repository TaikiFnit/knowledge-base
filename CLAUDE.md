# Knowledge Base - Agent Guide

このファイルは、複数のエージェントが並列でこのリポジトリを操作するための仕様書です。

## プロジェクト概要

個人ナレッジベースサイト。AtCoder競プロに限らず、データエンジニアリング・インフラ・各種学習資料など幅広いトピックのレポートを蓄積する静的HTMLサイト。

## デプロイ構成

- **ホスティング**: Vercel（GitHub mainブランチ連携、pushで自動デプロイ）
- **本番URL**: https://knowledge-base-seven-swart.vercel.app
- **ビルド**: なし（静的HTMLのみ）
- **Output Directory**: `.`（ルート直下）

### GitHub Actions

- **Discord通知** (`.github/workflows/notify-discord.yml`): mainブランチへのpushで `articles/*.html` に変更があった場合、Discord Webhookで記事公開通知を送信
  - 必要なSecrets: `DISCORD_WEBHOOK_URL` — Discord Webhook URL
  - 必要なVariables: `SITE_BASE_URL` — サイトのベースURL（未設定時は `https://knowledge-base-seven-swart.vercel.app`）

## ディレクトリ構成

```
knowledge-base/
├── CLAUDE.md              # エージェント向け仕様（正本）
├── agents.md              # CLAUDE.md へのシンボリックリンク
├── README.md              # プロジェクト説明
├── index.html             # トップページ（記事一覧 + タグフィルター）
├── articles/              # 各記事（1記事 = 1 HTML + 任意で .md）
│   ├── ruby-atcoder-guide.html
│   ├── ruby-atcoder-guide.md
│   └── atcoder-practice-analysis.html
└── vercel.json            # Vercel設定
```

## エージェント仕様ファイルの運用

- **正本**: `CLAUDE.md` — 仕様の編集は必ずこちらに対して行う
- **シンボリックリンク**: `agents.md -> CLAUDE.md` — Claude以外のエージェント（Gemini、GPT、Cursor、Devin等）が参照するためのエイリアス
- 今後特定エージェント用のファイル名規約が増えた場合は、同様にCLAUDE.mdへのシンボリックリンクを追加する（例: `AGENTS.md`, `COPILOT.md` 等）
- **絶対に内容をコピーして複数ファイルで管理しないこと**（内容が乖離する原因になる）

## デザイン仕様

- **テーマ**: ダークテーマベース
- **フォント**: JetBrains Mono（コード・ラベル）+ Noto Sans JP（本文） — Google Fontsから読み込み
- **アクセントカラー**: Ruby Red (#CC342D) を基調。記事のテーマに応じて変更可
- **既存記事のデザインを踏襲すること**

### CSS変数（index.html）

```css
--bg: #08080C;
--bg-card: #0F0F16;
--bg-card-hover: #16161F;
--text: #E0DED8;
--text-dim: #6E6E80;
--text-muted: #3E3E50;
--accent: #CC342D;
--accent-light: #E8554E;
--border: #1A1A28;
--green: #34D399;
--cyan: #67E8F9;
--orange: #FB923C;
--purple: #A78BFA;
```

## タグ体系

| タグ名 | data-tags値 | CSSクラス | 用途 |
|--------|-------------|-----------|------|
| Ruby | ruby | tag-ruby | Ruby言語関連 |
| AtCoder | atcoder | tag-atcoder | AtCoder競プロ関連 |
| ガイド | guide | tag-guide | ハウツー・チュートリアル |
| 分析 | analysis | tag-analysis | データ分析・比較 |
| リサーチ | research | tag-research | 調査レポート |
| ツール | tool | tag-tool | ツール・環境関連 |
| データエンジニアリング | data-engineering | tag-data-engineering | DE学習・パイプライン等 |
| 学習 | learning | tag-learning | 一般的な学習資料 |
| Python | python | tag-python | Python関連 |
| SQL | sql | tag-sql | SQL関連 |
| インフラ | infra | tag-infra | インフラ・クラウド関連 |

**新しいタグを追加する場合**: index.html の `<style>` 内に `.tag-xxx` のCSSルールを追加し、`.filter-bar` にボタンを追加すること。

## 記事追加手順（エージェント向け）

### 1. HTMLファイルを作成

- `articles/` ディレクトリに配置
- **1記事 = 1つの自己完結したHTMLファイル**（CSS・JSインライン）
- 外部依存はGoogle Fontsのみ許容
- トップへ戻るリンクを含める: `<a href="../index.html">← トップに戻る</a>`
- Markdown版（.md）も同ディレクトリに配置（推奨）

### 2. index.html に記事カードを追加（必須）

**記事ファイルを追加したら、必ずindex.htmlにもカードを追加すること。index.htmlが未更新の記事はサイト上で発見できない。**

`<!-- ===== ここに新しい記事を追加 ===== -->` コメントの**直後**に追加（新着順）。

```html
<a class="article-card" href="articles/ファイル名.html" data-tags="タグ1,タグ2">
  <div class="article-meta">
    <span class="article-date">YYYY-MM-DD</span>
    <span class="article-tag tag-xxx">タグ名</span>
  </div>
  <div class="article-title">記事タイトル</div>
  <div class="article-desc">記事の概要（1〜2文）</div>
  <div class="article-arrow">→ 読む</div>
</a>
```

### 3. コミット & プッシュ

```bash
git add articles/新記事.html articles/新記事.md index.html
git commit -m "add: 記事タイトル"
git push origin main
```

## 並列作業時のルール

- **index.html の編集**: 記事カード追加はコメントマーカー直後に挿入。コンフリクト防止のため、既存カードは変更しない
- **articles/ への追加**: ファイル名が被らなければ並列追加可能
- **タグ追加**: index.html の CSS とフィルターバーの両方を更新すること
- **コミットメッセージ規約**: `add: 記事タイトル` / `update: 変更内容` / `fix: 修正内容`
- **ブランチ戦略**: mainブランチに直接push（個人プロジェクトのため）

## 今後の拡張候補（未実装）

- ページネーション
- 検索機能
- カテゴリ別ページ
- RSSフィード
- ダークモード/ライトモード切り替え
- OGP画像の自動生成
