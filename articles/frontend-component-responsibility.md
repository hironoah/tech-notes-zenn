---
title: "コンポーネントの責任 - UI表示とユースケースの発火点"
emoji: "🧩"
type: "tech"
topics: ["frontend", "vue", "react", "architecture", "component"]
published: true
published_at: "2026-01-05 08:00"
---

## はじめに

コンポーネントに何を書くべきか、明確な基準を持っているだろうか。

多くのプロジェクトでは、コンポーネントに「とりあえず全部」書いてしまう。状態管理、API通信、ビジネスロジック、UI——すべてが1つのファイルに詰め込まれ、気づけば数百行の巨大コンポーネントが誕生する。

この記事では、**コンポーネントの責任を明確に定義**し、肥大化を防ぐ設計パターンを解説する。

### この記事の内容

- コンポーネントが持つべき2つの責任
- Composablesによる責任分離の実装パターン
- ViewとPresentationalコンポーネントの役割分担

### 対象読者

- コンポーネントが肥大化して困っている
- 「どこに何を書くか」の基準が欲しい
- ロジックの再利用やテストに課題を感じている

サンプルはVueで書くが、考え方はReactでも同じ。

## 肥大化するコンポーネントの正体

### 責任が曖昧なコンポーネント

典型的な「全部入り」コンポーネントを見てみる。

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from "vue";

const users = ref<User[]>([]);
const loading = ref(false);
const error = ref<string | null>(null);
const searchQuery = ref("");
const sortOrder = ref<"asc" | "desc">("asc");

// データ取得
onMounted(async () => {
  loading.value = true;
  try {
    const res = await fetch("/api/users");
    users.value = await res.json();
  } catch (e) {
    error.value = "取得に失敗しました";
  } finally {
    loading.value = false;
  }
});

// フィルタリング
const filteredUsers = computed(() => {
  return users.value
    .filter((u) => u.name.includes(searchQuery.value))
    .sort((a, b) =>
      sortOrder.value === "asc"
        ? a.name.localeCompare(b.name)
        : b.name.localeCompare(a.name)
    );
});

// 削除処理
const deleteUser = async (id: string) => {
  if (!confirm("削除しますか？")) return;
  await fetch(`/api/users/${id}`, { method: "DELETE" });
  users.value = users.value.filter((u) => u.id !== id);
};
</script>

<template>
  <div>
    <input v-model="searchQuery" placeholder="検索..." />
    <button @click="sortOrder = sortOrder === 'asc' ? 'desc' : 'asc'">
      {{ sortOrder === "asc" ? "↑" : "↓" }}
    </button>

    <div v-if="loading">読み込み中...</div>
    <div v-else-if="error">{{ error }}</div>
    <ul v-else>
      <li v-for="user in filteredUsers" :key="user.id">
        {{ user.name }}
        <button @click="deleteUser(user.id)">削除</button>
      </li>
    </ul>
  </div>
</template>
```

このコンポーネントには以下が混在している。

- **状態管理**: `users`, `loading`, `error`, `searchQuery`, `sortOrder`
- **API 通信**: `fetch` によるデータ取得・削除
- **ビジネスロジック**: フィルタリング、ソート
- **UI**: テンプレート部分

一見動いているように見えるが、問題は運用フェーズで顕在化する。

| やりたいこと                     | 困ること                     |
| -------------------------------- | ---------------------------- |
| ユーザー一覧を別画面でも使いたい | ロジックがコピペになる       |
| フィルタ条件を増やしたい         | どこを直せばいいか探しにくい |
| API のレスポンス形式が変わった   | 影響範囲が読めない           |
| ロジックだけテストしたい         | UI と密結合で書けない        |

これらの問題は、**コンポーネントの責任が定義されていない**ことに起因する。「コンポーネントに何を書くか」の基準がないから、とりあえず全部書いてしまう。

## コンポーネントが持つべき責任

では、コンポーネントには何をさせるべきか。

結論から言うと、コンポーネントの責任は以下の 2 つである。

1. **画面表示**: データを受け取り、UI として描画する
2. **ユースケースの発火点**: ユーザー操作を起点に、処理を呼び出す

これ以外の処理——データ取得、状態管理、ビジネスロジック——は**コンポーネントの外に出す**。

### なぜこの 2 つなのか

コンポーネントは「画面」と「ユーザー」の接点である。

- 画面に何かを表示する
- ユーザーの操作を受け取る

これがコンポーネントの本質的な役割であり、それ以外は付随的なものに過ぎない。

データをどこから取得するか、どう加工するか、どこに保存するかは、**画面とは独立した関心事**である。これらをコンポーネントに書くと、画面の関心事とデータの関心事が混在し、どちらを変更しても影響範囲が読めなくなる。

### 責任を分けるとどうなるか

責任が明確になると、以下のような構成になる。

```
コンポーネント（責任: 表示 + 発火点）
├── 表示: propsで受け取ったデータをUIに反映
└── 発火点: ユーザー操作に応じて処理を呼び出す

Composable / Hooks（責任: ロジック）
├── データ取得・更新
├── 状態管理
└── ビジネスロジック
```

この構成のメリットは明確である。

- コンポーネントを変更しても、ロジックは無傷
- ロジックを変更しても、コンポーネントは無傷
- ロジックだけを単体テストできる
- ロジックを別の画面で再利用できる

## 責任に基づいた実装

ここからは、責任を分離した具体的な実装パターンを見ていく。

### Composables による責任の分離

Vue 3 の Composables（React では Hooks）を使い、**ロジックを関数として切り出す**。これが現在のフロントエンド開発における標準的な手法である。

先ほどの「全部入り」コンポーネントから、ロジックを Composable に切り出してみる。

```ts
// composables/useUsers.ts
import { ref, computed, onMounted } from "vue";

export function useUsers() {
  const users = ref<User[]>([]);
  const loading = ref(false);
  const error = ref<string | null>(null);

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

  const deleteUser = async (id: string) => {
    await fetch(`/api/users/${id}`, { method: "DELETE" });
    users.value = users.value.filter((u) => u.id !== id);
  };

  onMounted(fetchUsers);

  return {
    users,
    loading,
    error,
    deleteUser,
    refetch: fetchUsers,
  };
}
```

```ts
// composables/useUserFilter.ts
import { ref, computed, type Ref } from "vue";

export function useUserFilter(users: Ref<User[]>) {
  const searchQuery = ref("");
  const sortOrder = ref<"asc" | "desc">("asc");

  const filteredUsers = computed(() => {
    return users.value
      .filter((u) => u.name.includes(searchQuery.value))
      .sort((a, b) =>
        sortOrder.value === "asc"
          ? a.name.localeCompare(b.name)
          : b.name.localeCompare(a.name)
      );
  });

  const toggleSort = () => {
    sortOrder.value = sortOrder.value === "asc" ? "desc" : "asc";
  };

  return {
    searchQuery,
    sortOrder,
    filteredUsers,
    toggleSort,
  };
}
```

```vue
<!-- UserListPage.vue -->
<script setup lang="ts">
import { useUsers } from "@/composables/useUsers";
import { useUserFilter } from "@/composables/useUserFilter";

const { users, loading, error, deleteUser } = useUsers();
const { searchQuery, sortOrder, filteredUsers, toggleSort } =
  useUserFilter(users);
</script>

<template>
  <div>
    <input v-model="searchQuery" placeholder="検索..." />
    <button @click="toggleSort">
      {{ sortOrder === "asc" ? "↑" : "↓" }}
    </button>

    <div v-if="loading">読み込み中...</div>
    <div v-else-if="error">{{ error }}</div>
    <ul v-else>
      <li v-for="user in filteredUsers" :key="user.id">
        {{ user.name }}
        <button @click="deleteUser(user.id)">削除</button>
      </li>
    </ul>
  </div>
</template>
```

**メリット**

- ロジックの再利用が容易（`useUsers` を別画面でも使える）
- 関心ごとに分割できる（データ取得とフィルタリングを別の composable に）
- ロジック単体でテスト可能

Composables を使うことで、コンポーネントは本来の責任——表示と発火点——に集中できる。

### View と子コンポーネントの役割分担

実務では、コンポーネントは階層構造を持つ。ここで重要なのは、**どのコンポーネントが Composable を呼ぶか**である。

```
View（Container役 / routerと紐づく）
├── Composable を呼び出してデータ取得・ロジック実行
├── 子コンポーネントを配置してレイアウトを決める
├── 子からのイベントを受けて Composable を呼ぶ
│
└── 子コンポーネント（Presentational）
    ├── props でデータを受け取って描画
    └── イベント発生時は親に emit
```

**View コンポーネントの責任**

- Composable を呼び出し、UI 用のデータを取得
- 子コンポーネントを配置し、画面レイアウトを決める
- 初期表示や子からのイベントに応じて Composable のメソッドを呼ぶ

**子コンポーネントの責任**

- props で親から情報を受け取り、UI に反映する
- イベント発生時（ボタン押下など）は emit で親に伝達する
- **自分ではロジックを持たない**

具体例を見てみる。

```vue
<!-- views/UserListView.vue（View） -->
<script setup lang="ts">
import { useUsers } from "@/composables/useUsers";
import { useUserFilter } from "@/composables/useUserFilter";
import UserSearchBar from "@/components/UserSearchBar.vue";
import UserList from "@/components/UserList.vue";

// Composable を呼び出してロジックを取得
const { users, loading, error, deleteUser } = useUsers();
const { searchQuery, sortOrder, filteredUsers, toggleSort } =
  useUserFilter(users);

// 子からのイベントに応じて Composable のメソッドを呼ぶ
const handleDelete = (id: string) => {
  if (confirm("削除しますか？")) {
    deleteUser(id);
  }
};
</script>

<template>
  <div class="user-list-view">
    <!-- 子コンポーネントにデータを渡し、レイアウトを決める -->
    <UserSearchBar
      v-model:query="searchQuery"
      :sort-order="sortOrder"
      @toggle-sort="toggleSort"
    />
    <UserList
      :users="filteredUsers"
      :loading="loading"
      :error="error"
      @delete="handleDelete"
    />
  </div>
</template>
```

```vue
<!-- components/UserList.vue（子コンポーネント） -->
<script setup lang="ts">
// props で受け取るだけ
defineProps<{
  users: User[];
  loading: boolean;
  error: string | null;
}>();

// イベントは親に伝達するだけ
defineEmits<{
  delete: [id: string];
}>();
</script>

<template>
  <div v-if="loading">読み込み中...</div>
  <div v-else-if="error">{{ error }}</div>
  <ul v-else>
    <li v-for="user in users" :key="user.id">
      {{ user.name }}
      <button @click="$emit('delete', user.id)">削除</button>
    </li>
  </ul>
</template>
```

この構成のポイントは、**子コンポーネントが「親から渡されたものを表示する」だけに徹する**ことである。

- View が Composable を呼ぶ → ロジックの所在が明確
- 子コンポーネントは props/emit のみ → 再利用しやすく、テストしやすい
- 確認ダイアログなどの UI に関わる判断も View が持つ

## 実践：どこから始めるか

パターンを知っていても、既存コードをどう改善するかは別問題だ。いくつかの指針を示す。

### 新規コードから始める

既存の巨大コンポーネントをいきなりリファクタリングするのはリスクが高い。まずは**新規で書くコードから分離を意識する**のがおすすめだ。

- 新しい画面を作るとき → 最初から Composable でロジックを切り出す
- 新しい UI パーツを作るとき → Presentational として props/emit だけで設計する

### 既存コードは「触るついで」に

既存コードは、**機能追加や修正のタイミングで少しずつ改善**していく。

```
1. バグ修正で該当コンポーネントを触る
2. ついでに一部のロジックを Composable に切り出す
3. テストを書く
4. 次回触るときにさらに改善
```

一気に完璧を目指すより、少しずつ良くしていく方が現実的だ。

## まとめ

この記事で述べてきたことを整理する。

**コンポーネントの責任**

- **画面表示**: データを受け取り、UI として描画する
- **ユースケースの発火点**: ユーザー操作を起点に、処理を呼び出す

これ以外——データ取得、状態管理、ビジネスロジック——は Composable に切り出す。

**実装の構成**

- **View**: Composable を呼び出し、子コンポーネントを配置する
- **子コンポーネント**: props で受け取り、emit で伝達するだけ

---

## 関連記事

フロントエンド責任分離アーキテクチャシリーズ：

- [全体像](./frontend-clean-architecture) - 責任分離のレイヤードアーキテクチャ
- **コンポーネントの責任**（本記事）
- [Composable の責任分離](./frontend-composable-responsibility) - loader / commands / ui / usecase への分類
- [Store の設計](./frontend-store-entity-state) - Entity State パターンと責任分離