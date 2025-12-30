---
title: "UIとロジックの分離 - コンポーネント設計パターン"
emoji: "🧩"
type: "tech"
topics: ["frontend", "vue", "react", "design"]
published: false
---

## はじめに

500 行超えのコンポーネント。UI の描画、API コール、バリデーション、状態管理が全部一箇所に詰まっている。どこを触れば何が壊れるかわからない。テストも書けない。

動いているから放置される。そして半年後、仕様変更で地獄を見る。

この記事では**UI とロジックの分離パターン**を 3 つ紹介する。「なんとなく分けたほうがいい」ではなく、**なぜ分けるのか**を理解すれば、設計の判断軸ができる。

### この記事の内容

- コンポーネントが肥大化する原因
- 3 つの分離パターンの使い分け
- Before/After のコード例

### 対象読者

- Vue/React での開発経験がある
- 設計の「正解」がわからず手が止まることがある

サンプルは Vue で書くが、考え方は React でも同じだ。

## なぜ UI とロジックを分離するのか

### 分離しないと何が起きるか

典型的な「全部入り」コンポーネントを見てみる。

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const users = ref<User[]>([])
const loading = ref(false)
const error = ref<string | null>(null)
const searchQuery = ref('')
const sortOrder = ref<'asc' | 'desc'>('asc')

// データ取得
onMounted(async () => {
  loading.value = true
  try {
    const res = await fetch('/api/users')
    users.value = await res.json()
  } catch (e) {
    error.value = '取得に失敗しました'
  } finally {
    loading.value = false
  }
})

// フィルタリング
const filteredUsers = computed(() => {
  return users.value
    .filter(u => u.name.includes(searchQuery.value))
    .sort((a, b) =>
      sortOrder.value === 'asc'
        ? a.name.localeCompare(b.name)
        : b.name.localeCompare(a.name)
    )
})

// 削除処理
const deleteUser = async (id: string) => {
  if (!confirm('削除しますか？')) return
  await fetch(`/api/users/${id}`, { method: 'DELETE' })
  users.value = users.value.filter(u => u.id !== id)
}
</script>

<template>
  <div>
    <input v-model="searchQuery" placeholder="検索..." />
    <button @click="sortOrder = sortOrder === 'asc' ? 'desc' : 'asc'">
      {{ sortOrder === 'asc' ? '↑' : '↓' }}
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
- **API通信**: `fetch` によるデータ取得・削除
- **ビジネスロジック**: フィルタリング、ソート
- **UI**: テンプレート部分

一見動いているように見えるが、問題は運用フェーズで顕在化する。

| やりたいこと | 困ること |
|-------------|---------|
| ユーザー一覧を別画面でも使いたい | ロジックがコピペになる |
| フィルタ条件を増やしたい | どこを直せばいいか探しにくい |
| APIのレスポンス形式が変わった | 影響範囲が読めない |
| ロジックだけテストしたい | UIと密結合で書けない |

### 分離するとどうなるか

分離の本質は**関心の分離**です。

- **UI は「どう見せるか」だけを知っている**
- **ロジックは「何をするか」だけを知っている**

この境界が明確なら、変更の影響範囲が限定される。UIを変えてもロジックは無傷、ロジックを変えてもUIは無傷。それぞれ独立してテストできる。

次のセクションでは、この分離を実現する3つのパターンを見ていく。

## 実装パターン

### 1. Presentational / Container

最も古典的なパターンだ。Dan Abramov が React で提唱したもので、Vue でも同じ考え方が使える。

**考え方**
- **Container（コンテナ）**: データ取得・状態管理・ロジックを担当。「何をするか」を知っている
- **Presentational（プレゼンテーショナル）**: 受け取った props を表示するだけ。「どう見せるか」だけを知っている

先ほどの「全部入り」コンポーネントを分離してみる。

```vue
<!-- UserListContainer.vue（Container） -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import UserList from './UserList.vue'

const users = ref<User[]>([])
const loading = ref(false)
const error = ref<string | null>(null)
const searchQuery = ref('')
const sortOrder = ref<'asc' | 'desc'>('asc')

onMounted(async () => {
  loading.value = true
  try {
    const res = await fetch('/api/users')
    users.value = await res.json()
  } catch (e) {
    error.value = '取得に失敗しました'
  } finally {
    loading.value = false
  }
})

const filteredUsers = computed(() => {
  return users.value
    .filter(u => u.name.includes(searchQuery.value))
    .sort((a, b) =>
      sortOrder.value === 'asc'
        ? a.name.localeCompare(b.name)
        : b.name.localeCompare(a.name)
    )
})

const deleteUser = async (id: string) => {
  await fetch(`/api/users/${id}`, { method: 'DELETE' })
  users.value = users.value.filter(u => u.id !== id)
}

const toggleSort = () => {
  sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc'
}
</script>

<template>
  <UserList
    :users="filteredUsers"
    :loading="loading"
    :error="error"
    :search-query="searchQuery"
    :sort-order="sortOrder"
    @update:search-query="searchQuery = $event"
    @toggle-sort="toggleSort"
    @delete="deleteUser"
  />
</template>
```

```vue
<!-- UserList.vue（Presentational） -->
<script setup lang="ts">
defineProps<{
  users: User[]
  loading: boolean
  error: string | null
  searchQuery: string
  sortOrder: 'asc' | 'desc'
}>()

defineEmits<{
  'update:search-query': [value: string]
  'toggle-sort': []
  'delete': [id: string]
}>()
</script>

<template>
  <div>
    <input
      :value="searchQuery"
      @input="$emit('update:search-query', ($event.target as HTMLInputElement).value)"
      placeholder="検索..."
    />
    <button @click="$emit('toggle-sort')">
      {{ sortOrder === 'asc' ? '↑' : '↓' }}
    </button>

    <div v-if="loading">読み込み中...</div>
    <div v-else-if="error">{{ error }}</div>
    <ul v-else>
      <li v-for="user in users" :key="user.id">
        {{ user.name }}
        <button @click="$emit('delete', user.id)">削除</button>
      </li>
    </ul>
  </div>
</template>
```

**メリット**
- Presentational コンポーネントは props と emit だけで完結するため、単体テストしやすい
- UI の見た目だけを変更する場合、Presentational だけを触ればいい
- Storybook との相性が良い

**デメリット**
- ファイル数が増える
- props のバケツリレーが発生しやすい

### 2. Composables / Hooks

Vue 3 の Composition API（React では Hooks）を使って、**ロジックを関数として切り出す**パターンだ。現在の Vue/React 開発では最も一般的な手法だろう。

```ts
// composables/useUsers.ts
import { ref, computed, onMounted } from 'vue'

export function useUsers() {
  const users = ref<User[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)

  const fetchUsers = async () => {
    loading.value = true
    try {
      const res = await fetch('/api/users')
      users.value = await res.json()
    } catch (e) {
      error.value = '取得に失敗しました'
    } finally {
      loading.value = false
    }
  }

  const deleteUser = async (id: string) => {
    await fetch(`/api/users/${id}`, { method: 'DELETE' })
    users.value = users.value.filter(u => u.id !== id)
  }

  onMounted(fetchUsers)

  return {
    users,
    loading,
    error,
    deleteUser,
    refetch: fetchUsers,
  }
}
```

```ts
// composables/useUserFilter.ts
import { ref, computed, type Ref } from 'vue'

export function useUserFilter(users: Ref<User[]>) {
  const searchQuery = ref('')
  const sortOrder = ref<'asc' | 'desc'>('asc')

  const filteredUsers = computed(() => {
    return users.value
      .filter(u => u.name.includes(searchQuery.value))
      .sort((a, b) =>
        sortOrder.value === 'asc'
          ? a.name.localeCompare(b.name)
          : b.name.localeCompare(a.name)
      )
  })

  const toggleSort = () => {
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc'
  }

  return {
    searchQuery,
    sortOrder,
    filteredUsers,
    toggleSort,
  }
}
```

```vue
<!-- UserListPage.vue -->
<script setup lang="ts">
import { useUsers } from '@/composables/useUsers'
import { useUserFilter } from '@/composables/useUserFilter'

const { users, loading, error, deleteUser } = useUsers()
const { searchQuery, sortOrder, filteredUsers, toggleSort } = useUserFilter(users)
</script>

<template>
  <div>
    <input v-model="searchQuery" placeholder="検索..." />
    <button @click="toggleSort">
      {{ sortOrder === 'asc' ? '↑' : '↓' }}
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

**デメリット**
- 適切な分割粒度の判断が必要
- composable 間の依存関係が複雑になる可能性

### 3. Headless UI

**ロジックはライブラリが提供し、UI だけを自分で実装する**パターンだ。[Headless UI](https://headlessui.com/)、[Radix UI](https://www.radix-ui.com/)、[Tanstack Table](https://tanstack.com/table) などが代表例。

考え方としては、ライブラリが「振る舞い」を提供し、開発者は「見た目」だけを担当する。

```vue
<!-- Headless UI（Vue版）を使ったドロップダウン -->
<script setup lang="ts">
import {
  Listbox,
  ListboxButton,
  ListboxOptions,
  ListboxOption,
} from '@headlessui/vue'
import { ref } from 'vue'

const people = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
  { id: 3, name: 'Charlie' },
]
const selected = ref(people[0])
</script>

<template>
  <Listbox v-model="selected">
    <!-- 見た目は完全に自由 -->
    <ListboxButton class="my-custom-button">
      {{ selected.name }}
    </ListboxButton>
    <ListboxOptions class="my-custom-dropdown">
      <ListboxOption
        v-for="person in people"
        :key="person.id"
        :value="person"
        v-slot="{ active, selected }"
      >
        <li :class="{ 'bg-blue-500': active, 'font-bold': selected }">
          {{ person.name }}
        </li>
      </ListboxOption>
    </ListboxOptions>
  </Listbox>
</template>
```

**メリット**
- アクセシビリティ、キーボード操作、フォーカス管理などの複雑なロジックを自前で書かなくていい
- デザインの自由度が高い（CSS フレームワークに縛られない）
- 動作の信頼性が高い（ライブラリがテスト済み）

**デメリット**
- ライブラリへの依存が生まれる
- カスタマイズの限界がある場合も

### パターンの使い分け

| パターン | 向いているケース |
|---------|----------------|
| Presentational/Container | 画面固有のロジックを分離したい |
| Composables/Hooks | ロジックを複数画面で再利用したい |
| Headless UI | 複雑な UI パターン（モーダル、ドロップダウン等） |

実際のプロジェクトでは、これらを組み合わせて使うことがほとんどだ。

### 組み合わせの実例

3つのパターンは独立しているわけではない。実務では以下のような構成になる。

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

**View コンポーネントの責務**
- Composable を呼び出し、UI 用のデータを取得
- 子コンポーネントを配置し、画面レイアウトを決める
- 初期表示や子からのイベントに応じて Composable のメソッドを呼ぶ

**子コンポーネントの責務**
- props で親から情報を受け取り、UI に反映する
- イベント発生時（ボタン押下など）は emit で親に伝達する
- **自分ではロジックを持たない**

この構成のメリットは、**子コンポーネントが「親から渡されたものを表示する」だけに徹する**ことだ。ロジックの所在が明確になり、新規参画者も迷わない。

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

### 分割の判断基準

「どこで分けるか」に迷ったら、以下を基準にする。

- **再利用するか？** → するなら Composable に切り出す
- **テストしたいか？** → したいならロジックを分離する
- **変更頻度は？** → UI とロジックで変更タイミングが違うなら分ける

逆に言えば、再利用もテストも不要で、変更も一緒に起きるなら、無理に分ける必要はない。

## まとめ

UI とロジックの分離は、コンポーネントの肥大化を防ぎ、保守性を高めるための基本的な設計手法だ。

**3 つのパターン**
- **Presentational / Container**: 画面固有のロジックを分離
- **Composables / Hooks**: ロジックを再利用可能な関数に切り出す
- **Headless UI**: 複雑な振る舞いをライブラリに任せる

**分離の本質**
- UI は「どう見せるか」だけを知っている
- ロジックは「何をするか」だけを知っている
- この境界が明確なら、変更・テスト・再利用が容易になる

**始め方**
- 新規コードから意識する
- 既存コードは触るついでに少しずつ改善
- 無理に分けない判断も重要

完璧な設計を最初から目指す必要はない。「このコンポーネント、ちょっと大きくなってきたな」と思ったときに、この記事を思い出してもらえれば幸いだ。
