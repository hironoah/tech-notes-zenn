# CLAUDE.md

## このリポジトリについて
Zenn/Qiita技術記事管理リポジトリ（npm workspacesによるモノレポ構成）

## 目的
技術発信・ポートフォリオ

## コンテンツ戦略

| プラットフォーム | 対象読者 | 内容 |
|------------------|----------|------|
| Zenn | 中級者以上 | 深掘り、設計、実務Tips |
| Qiita | 初学者向け | 入門、チュートリアル |

## 記事スタイル
- 文体: です/ます調
- トーン: フォーマル
- 技術分野: フロントエンド全般（JS/TS, Vue/React）

## 命名規則

### slug
- 形式: `{技術名}-{内容}`（日付なし）
- 例: `vue3-composables-guide`, `react-hooks-performance-tips`
- 12〜50文字、ハイフン区切り、小文字英数字のみ

### 画像
- 配置: `images/{slug}/01-xxx.png`
- 連番 + 内容を示す名前

### drafts
- `idea-xxx.md`: アイデア段階
- `wip-xxx.md`: 執筆中
- `review-xxx.md`: レビュー待ち

## Git運用

### ブランチ戦略
```
main（公開用）
  ↑ PRマージ = 公開
feature/slug名（記事執筆）
```

### フロー
1. mainで `drafts/idea-xxx.md` にネタを書く
2. 記事を書き始めるとき `feature/slug名` ブランチを切る
3. featureブランチで `drafts/` → `zenn/articles/` or `qiita/public/` に移動して執筆
4. PR作成時に `/review-article` でセルフレビュー
5. mainにマージ → 自動デプロイ

## よく使うtopics
javascript, typescript, vue, react, frontend, nextjs, nuxtjs, css, html

## 記事テンプレート

### Zenn用
```markdown
---
title: ""
emoji: ""
type: "tech"
topics: []
published: false
---

## はじめに

## 本題

## まとめ
```

### Qiita用
```markdown
---
title: ""
tags:
  -
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

## 本題

## まとめ
```
