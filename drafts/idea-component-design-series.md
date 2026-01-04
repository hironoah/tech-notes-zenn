# フロントエンド責務分離アーキテクチャ シリーズ

## コンセプト
- Clean Architecture の原則をフロントエンドに適用
- 5万行規模のプロジェクトでの実践経験がベース
- 「なぜこうするのか」を重視
- フレームワーク非依存の設計思想（例は Vue、考え方は React にも適用可能）

## シリーズ構成

### 1. コンポーネントが持つべき責務 ✅ 執筆中
- slug: `component-ui-logic-separation`
- コンポーネントの責務 = 画面表示 + ユースケースの発火点
- Composables による責務の分離
- View と子コンポーネントの役割分担

### 2. Composable の責務分離
- slug: `composable-responsibility-separation`
- Composable が肥大化する問題
- 役割による分類
  - `commands/` - バックエンドへの書き込み（CUD）
  - `loader/` - バックエンドからの読み込み（R）
  - `usecase/` - 複合的なユースケース
  - `selector/` - データ選択・変換（getter）
  - `mutator/` - 状態変更（setter）
  - `ui/` - 純粋な UI 状態
- 依存の方向と禁止事項

### 3. Store を「状態の器」にする
- slug: `store-as-state-container`
- Store の責務 = 状態を保持するだけ
- Entity State パターン（byId + allIds）
- 正規化による O(1) アクセス
- Selector / Mutator への責務移譲
- パフォーマンス比較

### 4. まとめ：フロントエンドの責務分離アーキテクチャ
- slug: `frontend-responsibility-architecture`
- 全体像：Component → Composable → Store
- 依存の方向：Router → Component → Composable → Store/Service → Infrastructure → Utils
- 各レイヤーの禁止事項
- 実務での導入ステップ

## ターゲット
- 中級者以上のフロントエンドエンジニア
- 設計に悩んでいる人
- 「なんとなく動く」から「設計根拠を持って書く」へ移行したい人

## 差別化ポイント
- 5万行規模のプロダクションでの実践経験
- Clean Architecture をフロントエンドに落とし込んだ具体的な体系
- 「よく言われていること」ではなく「実際にやってみてどうだったか」
- Atomic Design への批判的な視点（別記事で扱う可能性）

## 参考ドキュメント
- out/CLAUDE.md（プロジェクトの設計ガイドライン）
- out/docs/state-normalization-patterns.md（状態正規化パターン）
