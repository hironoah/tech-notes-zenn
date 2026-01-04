---
title: "Store を「状態の器」にする - Entity State パターン"
emoji: "🏺"
type: "tech"
topics: ["frontend", "vue", "react", "architecture"]
published: true
published_at: "2026-01-05 08:00"
---

## はじめに

前回の記事では、Composable を役割によって分類し、責任を明確にする方法を解説した。

しかし、まだ触れていない重要なテーマがある。**Store の設計**である。

Store をどう設計するかによって、アプリケーション全体のパフォーマンスと保守性が大きく変わる。この記事では、Store を「状態の器」として設計する方法と、Entity State パターン（正規化）について解説する。

### この記事の内容

- Store が「状態の器」であるべき理由
- シンプルな Store の実装例
- Entity State パターン（byId + allIds）の詳細
- Store のディレクトリ構造と責任分離

### 前提

この記事は「[Composable の責任分離](./frontend-composable-responsibility)」の続編である。前回の内容を前提としているため、未読の場合は先にそちらを読むことを推奨する。

## Store が肥大化する問題

### よくある Store の実装

Store にすべてを詰め込んだ実装をよく見かける。

```ts
// stores/user.store.ts
export const useUserStore = defineStore("user", () => {
  const users = ref<User[]>([]);
  const loading = ref(false);
  const error = ref<string | null>(null);
  const selectedUserId = ref<string | null>(null);

  // getter
  const activeUsers = computed(() => users.value.filter((u) => u.isActive));
  const selectedUser = computed(() =>
    users.value.find((u) => u.id === selectedUserId.value)
  );

  // setter
  const setUsers = (newUsers: User[]) => {
    users.value = newUsers;
  };
  const addUser = (user: User) => {
    users.value.push(user);
  };
  const updateUser = (id: string, data: Partial<User>) => {
    const index = users.value.findIndex((u) => u.id === id);
    if (index !== -1) {
      users.value[index] = { ...users.value[index], ...data };
    }
  };
  const removeUser = (id: string) => {
    users.value = users.value.filter((u) => u.id !== id);
  };

  // API 呼び出し
  const fetchUsers = async () => {
    loading.value = true;
    try {
      const res = await fetch("/api/users");
      users.value = await res.json();
    } catch (e) {
      error.value = "取得に失敗しました";
    } finally {
      loading.value = false;
    }
  };

  return {
    users,
    loading,
    error,
    selectedUserId,
    activeUsers,
    selectedUser,
    setUsers,
    addUser,
    updateUser,
    removeUser,
    fetchUsers,
  };
});
```

この実装には"単純に読みづらい"以外に複数の問題がある。

| 問題                                        | 影響                                 |
| ------------------------------------------- | ------------------------------------ |
| 状態 + getter + setter + API 呼び出しが混在 | 責任が不明確で、テストしにくい       |
| 配列検索が O(n)                             | データ量が増えるとパフォーマンス劣化 |
| loading/error が Store に存在               | 複数箇所から呼ぶと状態が衝突する     |

Store は**画面を横断する状態を保持するだけ**であるべきだ。それ以外の責任は Composable に委譲する。

## Store の責任は「状態の器」のみ

### Store に持たせるもの

- `ref<T>` / `shallowRef<T>`：状態の器
- `computed`：派生データの算出

### Store に持たせないもの

| 持たせないもの        | 委譲先                       |
| --------------------- | ---------------------------- |
| Service 呼び出し      | loader composable            |
| loading/error 状態    | loader / commands composable |
| 複数 Store 跨ぎの算出 | ui composable                |

Store は**状態を保持するだけ**。これが「状態の器」の意味である。

## シンプルな Store の実装

すべての Store が複雑な構造を必要とするわけではない。まずは、シンプルな Store から見ていく。

### アプリケーション状態の Store

アプリの初期化状態やテーマ設定など、単一の値を保持するだけの Store。

```ts
// stores/app/store.ts
import { defineStore } from "pinia";
import { ref, computed } from "vue";
import { createSelector } from "./selector";
import { createMutator } from "./mutator";

export const useAppStore = defineStore("app", () => {
  const isInitialized = ref(false);
  // ここではマジックナンバーを用いるが、別途定数定義したほうが良い
  const theme = ref<"light" | "dark">("light");

  // 派生データ(selector)
  const { isDarkMode } = createSelector(theme);

  // mutator
  const { setInitialized, toggleTheme } = createMutator(isInitialized, theme);

  return {
    isInitialized,
    theme,
    isDarkMode,
    setInitialized,
    toggleTheme,
  };
});
```

### 認証状態の Store

ログインユーザーの情報を保持する Store。

```ts
// stores/auth/store.ts
import { defineStore } from "pinia";
import { ref, computed } from "vue";
import type { User } from "@/types/user";
import { createSelector } from "./selector";
import { createMutator } from "./mutator";

export const useAuthStore = defineStore("auth", () => {
  const currentUser = ref<User | null>(null);

  // selector
  const { isAuthenticated, userId } = createSelector(currentUser);
  // mutator
  const { setCurrentUser, logout } = createMutator(currentUser);

  return {
    currentUser,
    isAuthenticated,
    userId,
    setCurrentUser,
    logout,
  };
});
```

これらの Store は、**単一の値や少数のプロパティ**を管理するだけなので、シンプルな `ref` + `computed` で十分である。
また、selector と mutator を肥大化回避のため別で定義しているが、簡素な store であれば無理に別定義する必要はない。

### シンプルな Store で良いケース

- アプリの初期化状態
- 認証ユーザー情報
- UI の設定（テーマ、言語など）

これらは **Entity State パターンを使う必要がない**。次のセクションで説明する Entity State パターンは、**複数のエンティティをコレクションとして管理する場合**に使う。

## Entity State パターン（byId + allIds）

ユーザー一覧、商品リスト、タスク管理など、**複数のエンティティをコレクションとして管理する場合**は、Entity State パターンを使う。

### 配列 vs 正規化

多くのプロジェクトでは、データを配列で管理している。

```ts
// 配列パターン
const users = ref<User[]>([]);

// ID で検索 → O(n)
const user = users.value.find((u) => u.id === id);

// 更新 → O(n) + 配列再生成
const index = users.value.findIndex((u) => u.id === id);
users.value = [
  ...users.value.slice(0, index),
  updated,
  ...users.value.slice(index + 1),
];
```

データ量が増えると、検索や更新のたびにパフォーマンスが劣化する。

### Entity State パターンの構造

Entity State パターンでは、データを `byId`（オブジェクト）と `allIds`（配列）で管理する。

```ts
type EntityState<T> = {
  byId: Record<string, T>; // ID → エンティティのマップ
  allIds: string[]; // ID の順序を保持
};
```

```ts
// Entity State パターン
const byId = shallowRef<Record<string, User>>({});
const allIds = ref<string[]>([]);

// ID で検索 → O(1)
const user = byId.value[id];

// 更新 → O(1)
byId.value = { ...byId.value, [id]: updated };
```

### パフォーマンス比較

| 操作    | 配列パターン      | Entity State パターン |
| ------- | ----------------- | --------------------- |
| ID 検索 | O(n)              | O(1)                  |
| 更新    | O(n) + 配列再生成 | O(1)                  |
| 追加    | O(1)              | O(1)                  |
| 削除    | O(n)              | O(1)                  |

1,000 件のデータで ID 検索する場合：

- 配列パターン：平均 500 回の比較
- Entity State パターン：1 回のハッシュ検索

**データ量が増えるほど、差は顕著になる。**

### なぜ allIds が必要か

`byId` だけではオブジェクトのプロパティ順序が保証されない。`allIds` で順序を明示的に管理する。

```ts
// 順序を保持した配列を取得
const users = computed(() =>
  allIds.value.map((id) => byId.value[id]).filter(Boolean)
);
```

## Store のディレクトリ構造

Store を「状態の器」にするために、責任をファイルで分離する。

```
stores/
  user/
    store.ts      ← 状態の器 + DI
    selector.ts   ← データ取得・加工（computed + getter 関数）
    mutator.ts    ← 状態変更（upsert, remove, clear）
    index.ts      ← カプセル化（byId/allIds を隠蔽）
```

### 命名規則

selector と mutator の返り値には、明確な命名規則を設ける。

| 分類     | 命名 | 呼び出し方 | 例             |
| -------- | ---- | ---------- | -------------- |
| selector | 名詞 | `.value`   | `users.value`  |
| selector | 動詞 | `()`       | `getById(id)`  |
| mutator  | 動詞 | `()`       | `upsert(user)` |

- **名詞**（`users`, `activeUsers`）→ `ComputedRef` → `.value` でアクセス
- **動詞**（`getById`, `upsert`, `remove`）→ 関数 → `()` で呼び出し

この規則により、利用側で「computed か関数か」が一目でわかる。

### store.ts - 状態の器

```ts
// stores/user/store.ts
import { defineStore } from "pinia";
import { shallowRef, ref } from "vue";
import type { User } from "@/types/user";
import { createSelector } from "./selector";
import { createMutator } from "./mutator";

export const useStore = defineStore("user", () => {
  // 状態の器
  const byId = shallowRef<Record<string, User>>({});
  const allIds = ref<string[]>([]);

  // selector / mutator を DI で生成
  const { users, activeUsers, getById, searchByName } = createSelector(
    byId,
    allIds
  );
  const { upsert, upsertMany, clear } = createMutator(byId, allIds);

  return {
    byId,
    allIds,
    users,
    activeUsers,
    getById,
    searchByName,
    upsert,
    upsertMany,
    clear,
  };
});
```

**ポイント**:

- `byId` には `shallowRef` を使用（プロパティレベルの追跡を避けてパフォーマンス向上）
- `createSelector` / `createMutator` に状態を DI（依存性注入）

※ DI パターンの詳細については別記事で解説予定。

### selector.ts - データ取得・加工

```ts
// stores/user/selector.ts
import { computed, type ShallowRef, type Ref } from "vue";
import type { User } from "@/types/user";

export const createSelector = (
  byId: ShallowRef<Record<string, User>>,
  allIds: Ref<string[]>
) => {
  // 名詞 → ComputedRef（.value でアクセス）
  const users = computed(() =>
    allIds.value.map((id) => byId.value[id]).filter(Boolean)
  );

  const activeUsers = computed(() => users.value.filter((u) => u.isActive));

  // 動詞 → 関数（() で呼び出し）
  const getById = (id: string): User | undefined => byId.value[id];

  const searchByName = (query: string) => {
    return computed(() =>
      users.value.filter((u) =>
        u.name.toLowerCase().includes(query.toLowerCase())
      )
    );
  };

  return { users, activeUsers, getById, searchByName };
};
```

**ポイント**:

- 名詞（`users`, `activeUsers`）→ `ComputedRef` を返す
- 動詞（`getById`, `searchByName`）→ 関数を返す

### mutator.ts - 状態変更

```ts
// stores/user/mutator.ts
import type { ShallowRef, Ref } from "vue";
import type { User } from "@/types/user";

export const createMutator = (
  byId: ShallowRef<Record<string, User>>,
  allIds: Ref<string[]>
) => {
  // 1 件追加・更新（upsert）
  const upsert = (user: User): void => {
    byId.value = { ...byId.value, [user.id]: user };
    if (!allIds.value.includes(user.id)) {
      allIds.value.push(user.id);
    }
  };

  // 複数件追加・更新（バッチ更新）
  const upsertMany = (users: User[]): void => {
    const newById = { ...byId.value };
    const newIds: string[] = [];

    users.forEach((user) => {
      newById[user.id] = user;
      if (!allIds.value.includes(user.id)) {
        newIds.push(user.id);
      }
    });

    byId.value = newById; // 1 回だけリアクティビティ発火
    if (newIds.length > 0) {
      allIds.value = [...allIds.value, ...newIds];
    }
  };

  // 全削除
  const clear = (): void => {
    byId.value = {};
    allIds.value = [];
  };

  return { upsert, upsertMany, clear };
};
```

**ポイント**:

- `delete` キーワードは使わない（リアクティビティが壊れる）
- spread 演算子で新しいオブジェクトを作成（immutable パターン）

### index.ts - カプセル化

```ts
// stores/user/index.ts
import { useStore } from "./store";

export const useUserStore = () => {
  const store = useStore();

  // byId / allIds は外部に露出しない
  return {
    // selector（名詞 → .value、動詞 → ()）
    users: store.users,
    activeUsers: store.activeUsers,
    getById: store.getById,
    searchByName: store.searchByName,
    // mutator（動詞 → ()）
    upsert: store.upsert,
    upsertMany: store.upsertMany,
    clear: store.clear,
  };
};
```

**ポイント**:

- `byId` / `allIds` は外部に露出しない（カプセル化）
- 利用側は `useUserStore()` のみ使用
- store 自体は全プロパティを保持（SSR 対応・Pinia DevTools 対応）

## Entity State パターンの注意点

### 絶対に守るべきルール

| ルール                        | 理由                   | 違反時の結果       |
| ----------------------------- | ---------------------- | ------------------ |
| `shallowRef` を `byId` に使用 | プロパティ追跡を避ける | パフォーマンス劣化 |
| `byId` と `allIds` を同時更新 | 整合性保証             | 幽霊データ         |
| immutable パターン            | リアクティビティ保証   | UI が更新されない  |
| `delete` キーワード禁止       | リアクティビティ破壊   | UI が更新されない  |
| バッチ更新を使用              | 再レンダリング最小化   | UI フリーズ        |

### アンチパターン

```ts
// ❌ delete キーワード（リアクティビティが壊れる）
delete store.byId[id];

// ❌ 片方だけ更新（幽霊データが発生）
store.byId[user.id] = user;
// allIds への追加を忘れた

// ❌ 複数更新のmutatorでループ内で更新（毎回リアクティビティ発火 → 遅い）
users.forEach((u) => {
  store.byId[u.id] = u;
});
```

### 正しいパターン

```ts
// ✅ spread 演算子で削除
const { [id]: _, ...rest } = byId.value;
byId.value = rest;

// ✅ 同時更新で整合性を保つ
byId.value = { ...byId.value, [user.id]: user };
if (!allIds.value.includes(user.id)) {
  allIds.value.push(user.id);
}

// ✅ バッチ更新
const newById = { ...byId.value };
users.forEach((u) => {
  newById[u.id] = u;
});
byId.value = newById; // 1 回だけ発火
```

## Composable との連携

Store の selector/mutator は、Composable から呼び出される。

### loader での使用

```ts
// composables/loader/useFetchUsersLoader.ts
import { useUserStore } from "@/stores/user";

export const useFetchUsersLoader = () => {
  const { upsertMany } = useUserStore();
  const loading = ref(false);
  const error = ref<string | null>(null);

  const execute = async () => {
    loading.value = true;
    error.value = null;
    try {
      const result = await userService.getAll();
      upsertMany(result); // loader が Store に格納
    } catch (e) {
      error.value = "取得に失敗しました";
      throw e;
    } finally {
      loading.value = false;
    }
  };

  return { execute, loading, error };
};
```

### usecase での使用

```ts
// views/userList/useInitializeUserList.ts
import { useUserStore } from "@/stores/user";
import { useFetchUsersLoader } from "@/composables/loader/useFetchUsersLoader";

export const useInitializeUserList = () => {
  const { users } = useUserStore();
  const { execute: fetchUsers, loading, error } = useFetchUsersLoader();

  const initialize = async () => {
    await fetchUsers(); // loader が Store に格納する
  };

  return {
    users, // ComputedRef → .value でアクセス
    loading,
    error,
    initialize,
  };
};
```

### 依存の流れ

```
Component
    ↓
usecase（useInitializeUserList）
    ↓           ↓
loader      Store selector
    ↓
Service + Store mutator
```

- **loader** → Service を呼び出し、結果を Store の mutator に渡す
- **usecase** → loader を呼び出し、Store の selector を使ってデータを取得
- **Component** → usecase を呼び出す

## まとめ

Store を「状態の器」として設計することで、責任が明確になる。

| ファイル    | 責任                                       |
| ----------- | ------------------------------------------ |
| store.ts    | 状態の器 + DI で selector/mutator を生成   |
| selector.ts | データ取得・加工（computed + getter 関数） |
| mutator.ts  | 状態変更（upsert, upsertMany, clear）      |
| index.ts    | カプセル化（byId/allIds を隠蔽）           |

Entity State パターン（byId + allIds）を採用することで：

- O(1) の高速アクセス
- データの一貫性
- パフォーマンスの向上
- テストの容易さ

命名規則により、利用側での使い分けも明確になる：

- **名詞**（`users`）→ `.value` でアクセス
- **動詞**（`getById`, `upsert`）→ `()` で呼び出し

---

## 関連記事

フロントエンド責任分離アーキテクチャシリーズ：

- [全体像](./frontend-clean-architecture) - 責任分離のレイヤードアーキテクチャ
- [コンポーネントの責任](./frontend-component-responsibility) - 画面表示とユースケースの発火点
- [Composable の責任分離](./frontend-composable-responsibility) - loader / commands / ui / usecase への分類
- **Store の設計**（本記事）
