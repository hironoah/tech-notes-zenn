# tech-notes

Zenn/Qiita技術記事管理リポジトリ

## 構成

```
tech-notes/
├── zenn/           # Zenn記事（中級者以上向け）
│   ├── articles/
│   ├── books/      # 本（フロントエンドアーキテクト戦略など）
│   └── images/
├── qiita/          # Qiita記事（初学者向け）
│   ├── public/
│   └── images/
└── drafts/         # 下書き・ネタ帳
```

## セットアップ

```bash
npm install
```

## 使い方

### Zenn

```bash
# プレビュー
npm run zenn:preview

# 新規記事作成
npm run zenn:new
```

### Qiita

```bash
# プレビュー
npm run qiita:preview

# 記事公開
npm run qiita:publish
```

## 記事執筆フロー

1. `drafts/idea-xxx.md` にネタを書く
2. `feature/slug名` ブランチを切る
3. 記事を執筆して `zenn/articles/` or `qiita/public/` に配置
4. PRを作成してレビュー
5. mainにマージ → 自動デプロイ
