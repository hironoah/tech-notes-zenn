# tech-notes-zenn

Zenn技術記事リポジトリ

## 構成

```
tech-notes-zenn/
├── articles/       # 記事
├── books/          # 本
├── images/         # 画像
└── drafts/         # 下書き・ネタ帳
```

## セットアップ

```bash
npm install
```

## 使い方

```bash
# プレビュー
npm run preview

# 新規記事作成
npm run new:article

# 新規本作成
npm run new:book
```

## 記事執筆フロー

1. `drafts/idea-xxx.md` にネタを書く
2. `feature/slug名` ブランチを切る
3. 記事を執筆して `articles/` に配置
4. PRを作成してレビュー
5. mainにマージ → 自動デプロイ
