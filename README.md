# Knowledge Base

個人ナレッジベース — 競プロ、データエンジニアリング、技術、ツールの調査・分析レポートを蓄積する場所。

## 構成

```
index.html          # トップページ（タグフィルター付き記事一覧）
articles/           # 各レポート（HTML + MD）
vercel.json         # Vercel設定
CLAUDE.md           # エージェント向け仕様書（並列作業ガイド含む）
```

## デプロイ

GitHub mainブランチへのpushで Vercel に自動デプロイ。

## 記事追加

エージェント向けの詳細手順は [CLAUDE.md](CLAUDE.md) を参照。

1. `articles/` に HTMLファイルを作成
2. `index.html` のコメントマーカー直後に記事カードを追加
3. `git add`, `commit`, `push` → 自動デプロイ

## タグ

ALL / Ruby / AtCoder / データエンジニアリング / 学習 / ガイド / 分析 / リサーチ / ツール / Python / SQL / インフラ
