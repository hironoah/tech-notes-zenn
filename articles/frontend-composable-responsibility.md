---
title: "Composable の責任分離 - 4つのカテゴリで整理する"
emoji: "🔧"
type: "tech"
topics: ["frontend", "vue", "react", "architecture"]
published: true
published_at: "2026-01-05 08:00"
---

## はじめに

前回の記事では、コンポーネントの責任を「画面表示」と「ユースケースの発火点」に限定し、それ以外のロジックは Composable（React では Hooks）に切り出すべきだと述べた。

しかし、これだけでは不十分である。

規模が大きくなると、今度は **Composable/Hooks 自体が肥大化**する。データ取得、状態変更、ビジネスロジック、UI 状態管理——すべてが `useXXX.ts` に詰め込まれ、結局「全部入り Composable」が生まれる。

この記事では、**Composable を役割によって分類**し、責任を明確にする方法を解説する。

### この記事の内容

- Composable が肥大化する原因
- 役割による Composable の分類
- 各カテゴリの具体的な実装パターン

### 前提

この記事は「[コンポーネントの責任](./frontend-component-responsibility)」の続編である。前回の内容を前提としているため、未読の場合は先にそちらを読むことを推奨する。

## Composable が肥大化する原因

### 「とりあえず Composable に切り出す」の罠

前回の原則に従い、ロジックを Composable に切り出したとする。

```ts
// useUserManagement.ts
export const useUserManagement = () => {
  const users = ref<User[]>([]);
  const loading = ref(false);
  const error = ref<string | null>(null);
  const selectedUserId = ref<string | null>(null);

  // データ取得
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

  // ユーザー削除
  const deleteUser = async (id: string) => {
    await fetch(`/api/users/${id}`, { method: "DELETE" });
    users.value = users.value.filter((u) => u.id !== id);
  };

  // ユーザー更新
  const updateUser = async (id: string, data: Partial<User>) => {
    const res = await fetch(`/api/users/${id}`, {
      method: "PUT",
      body: JSON.stringify(data),
    });
    const updated = await res.json();
    users.value = users.value.map((u) => (u.id === id ? updated : u));
  };

  // 選択中のユーザー
  const selectedUser = computed(() =>
    users.value.find((u) => u.id === selectedUserId.value)
  );

  // フィルタリング
  const activeUsers = computed(() => users.value.filter((u) => u.isActive));

  // 選択操作
  const selectUser = (id: string) => {
    selectedUserId.value = id;
  };

  return {
    users,
    loading,
    error,
    selectedUser,
    activeUsers,
    fetchUsers,
    deleteUser,
    updateUser,
    selectUser,
  };
};
```

一見整理されているように見えるが、この Composable には複数の責任が混在している。

- **データ取得**: `fetchUsers`
- **データ更新**: `deleteUser`, `updateUser`
- **状態選択**: `selectedUser`, `activeUsers`
- **UI 状態**: `selectedUserId`, `selectUser`

何か問題があるのか？

| やりたいこと                       | 困ること                                   |
| ---------------------------------- | ------------------------------------------ |
| データ取得ロジックを別画面で使う   | UI 状態（選択）まで付いてくる              |
| フィルタ条件を増やす               | どこに書くべきか迷う                       |
| 別の Composable から状態を参照する | 同じ Composable を複数呼ぶと状態が重複する |
| テストを書く                       | 依存が多すぎてモックが大変                 |

結局、**責任が曖昧なまま切り出しても、肥大化の問題は解決しない**。

## 役割による Composable の分類

では、どう分けるべきか。

Composable を以下の 4 つのカテゴリに分類する。

| カテゴリ     | 役割                                         | 配置場所                |
| ------------ | -------------------------------------------- | ----------------------- |
| **loader**   | バックエンドからのデータ取得                 | `composables/loader/`   |
| **commands** | バックエンドへのデータ送信                   | `composables/commands/` |
| **ui**       | 画面表示に関わるロジック                     | `composables/ui/`       |
| **usecase**  | ユーザー操作に紐づくアプリケーションロジック | `composables/usecase/`  |

### 分類の基準

```
ユーザー操作や画面ライフサイクルに紐づくか？
├── Yes → usecase/（ユーザー登録、画面初期表示など）
└── No  → バックエンドと通信するか？
          ├── Yes → 読み取り？ → loader/
          │         書き込み？ → commands/
          └── No  → ui/（表示ロジック、UI状態）
```

この分類は、**責任の方向性**と**何と通信するか**に基づいている。

- `usecase` はユーザー操作に紐づくアプリケーション固有のビジネスロジック
- `loader` / `commands` は外部（バックエンド/localStorage/IndexedDB など）との連携
- `ui` は画面表示のためのロジック

## 各カテゴリの実装パターン

### loader/ - バックエンドからのデータ取得

バックエンドからデータを取得する。**Store に格納する**か、**値をそのまま返却する**かの 2 パターンがある。

#### パターン 1: Store に格納する

複数画面で共有するデータの場合、loader が Store に格納する。

```ts
// composables/loader/useFetchUsersLoader.ts
export const useFetchUsersLoader = () => {
  const { upsert } = useUserStore();
  const loading = ref(false);
  const error = ref<string | null>(null);

  const execute = async () => {
    loading.value = true;
    error.value = null;
    try {
      const result = await userService.getAll();
      result.forEach((user) => upsert(user)); // loader が Store に格納
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

#### パターン 2: 値を返却する

画面専用のデータなど、Store に保存しない場合は値を返却する。

```ts
// composables/loader/useFetchPrefectureLoader.ts
export const useFetchPrefectureLoader = () => {
  const loading = ref(false);
  const error = ref<string | null>(null);

  const execute = async (postalCode: string) => {
    loading.value = true;
    error.value = null;
    try {
      return await addressService.getPrefecture(postalCode); // 値を返却
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

**特徴**:

- Service を呼び出してバックエンドと通信
- **Store に格納する**または**値を返却する**
- loading/error 状態を持つ
- 二重取得の防止などの制御を担当
- **命名規則**: `use{Action}{Entity}Loader`（例: `useFetchUsersLoader`）

### commands/ - バックエンドへのデータ送信

ユーザー削除、登録、更新など、バックエンドの状態を変更する処理。**処理の実行のみを担当**し、Store の更新は usecase に委ねる。

```ts
// composables/commands/useDeleteUserCommand.ts
export const useDeleteUserCommand = () => {
  const loading = ref(false);
  const error = ref<string | null>(null);

  const execute = async (id: string) => {
    loading.value = true;
    error.value = null;
    try {
      await userService.delete(id);
      // Store の更新は usecase が行う
    } catch (e) {
      error.value = "削除に失敗しました";
      throw e;
    } finally {
      loading.value = false;
    }
  };

  return { execute, loading, error };
};
```

**特徴**:

- Service を呼び出してバックエンドと通信（POST/PUT/DELETE）
- **処理の実行のみを担当**（Store の更新は usecase が行う）
- loading/error 状態を持つ
- エラーハンドリングを内包
- **命名規則**: `use{Action}{Entity}Command`（例: `useDeleteUserCommand`）

### ui/ - 画面表示に関わるロジック

Store の getter を使った表示用データ加工、およびモーダル開閉などの UI 状態。

```ts
// composables/ui/useUserDisplay.ts
export const useUserDisplay = () => {
  const { users, getById } = useUserStore();

  // 表示用に加工
  const userOptions = computed(() =>
    users.value.map((u) => ({
      label: `${u.lastName} ${u.firstName}`,
      value: u.id,
    }))
  );

  const getUserLabel = (id: string) => {
    const user = getById(id);
    return user ? `${user.lastName} ${user.firstName}` : "";
  };

  return { userOptions, getUserLabel };
};
```

```ts
// composables/ui/useDisclosure.ts
export const useDisclosure = (initialState = false) => {
  const _isOpen = ref(initialState);

  const isOpen = computed(() => _isOpen.value);

  const open = () => {
    _isOpen.value = true;
  };

  const close = () => {
    _isOpen.value = false;
  };

  const toggle = () => {
    _isOpen.value = !_isOpen.value;
  };

  return { isOpen, open, close, toggle };
};
```

```ts
// composables/ui/useUserSelection.ts
export const useUserSelection = () => {
  const selectedId = ref<string | null>(null);

  const select = (id: string) => {
    selectedId.value = id;
  };

  const clear = () => {
    selectedId.value = null;
  };

  return { selectedId, select, clear };
};
```

**特徴**:

- 画面表示に関わるものはすべてここ
- Store の getter を使う表示用加工
- モーダル開閉、選択状態などの UI 状態
- 再利用性が高い

### usecase/ - ユーザー操作に紐づくアプリケーションロジック

「ユーザー登録」「商品購入」「画面初期表示」など、ユーザー操作や画面ライフサイクルに紐づくビジネスロジックを担当する。Clean Architecture の Use Case 層に対応する。

**特徴**:

- ユーザー操作（ボタン押下、フォーム送信など）に紐づく一連のフロー
- loader / commands を呼び出し、必要に応じて Store の状態を更新
- バリデーション → API 呼び出し → 状態更新 → UI 更新などを統合
- 「バリデーションが通らなければ登録できない」などのビジネスルールを持つ
- **View 専用**の場合は `views/{viewName}/` に配置

### 画面専用データのパターン

Store に保存せず、その画面でしか使わないデータの場合は、**loader のパターン 2（値を返却）**を使い、usecase がローカル ref に格納する。

```ts
// views/addressForm/useAddressForm.ts
export const useAddressForm = () => {
  // 画面専用の状態（Store には保存しない）
  const prefecture = ref<string | null>(null);

  // loader（値を返却する形式）
  const { execute: fetchPrefecture, loading } = useFetchPrefectureLoader();

  // 郵便番号から都道府県を取得
  const lookupAddress = async (postalCode: string) => {
    const result = await fetchPrefecture(postalCode);
    prefecture.value = result; // usecase がローカル ref に格納
  };

  return {
    prefecture,
    loading,
    lookupAddress,
  };
};
```

**ポイント**: 画面専用データの場合は loader が値を返却し、usecase がローカル ref に格納する。Store を経由しないため、他の画面には影響しない。

### View 専用 Composable の配置

usecase や ui のうち、特定の View でしか使わないものは `views/` 配下に置く。

```
views/
  userList/
    UserListView.vue
    useInitializeUserList.ts  ← この View 専用の usecase
    useDeleteUserUsecase.ts   ← この View 専用の usecase
    useUserList.ts            ← この View 専用の ui（フィルタ等）
  userDetail/
    UserDetailView.vue
    useUserDetailPage.ts      ← この View 専用の usecase
```

**判断基準**:

- 複数画面で使う → `composables/usecase/` または `composables/ui/`
- 1 つの画面でしか使わない → `views/{viewName}/`

### usecase の実装例

View 専用の usecase は、それぞれ単一のユーザー操作に対応する。

```ts
// views/userList/useInitializeUserList.ts
export const useInitializeUserList = () => {
  const { execute: fetchUsers, loading, error } = useFetchUsersLoader();

  const initialize = async () => {
    await fetchUsers(); // loader が Store に格納する
  };

  return { initialize, loading, error };
};
```

```ts
// views/userList/useDeleteUserUsecase.ts
export const useDeleteUserUsecase = () => {
  const { remove } = useUserStore();
  const { execute: deleteUserApi } = useDeleteUserCommand();

  const confirmDelete = async (
    selectedId: Ref<string | null>,
    clearSelection: () => void,
    closeModal: () => void
  ) => {
    if (!selectedId.value) return;
    await deleteUserApi(selectedId.value);
    remove(selectedId.value); // commands は Store を更新しないので、usecase が行う
    clearSelection();
    closeModal();
  };

  return { confirmDelete };
};
```

**ポイント**: usecase は単一のユーザー操作に対応する。「初期表示」「削除」など、操作ごとに分けることで再利用性とテスタビリティが向上する。

### View 専用 ui の実装例

View 専用の ui は、その画面固有のフィルタリングやソートなどを担当する。

```ts
// views/userList/useUserList.ts
export const useUserList = () => {
  const { users } = useUserStore();

  // フィルタ条件
  const searchQuery = ref("");
  const statusFilter = ref<"all" | "active" | "inactive">("all");

  // フィルタリング済みユーザー
  const filteredUsers = computed(() => {
    let result = users.value;

    // 検索条件
    if (searchQuery.value) {
      const query = searchQuery.value.toLowerCase();
      result = result.filter((u) => u.name.toLowerCase().includes(query));
    }

    // ステータスフィルタ
    if (statusFilter.value !== "all") {
      const isActive = statusFilter.value === "active";
      result = result.filter((u) => u.isActive === isActive);
    }

    return result;
  });

  return {
    searchQuery,
    statusFilter,
    filteredUsers,
  };
};
```

**ポイント**: View 専用の ui は Store getter を使ってフィルタリングやソートを行う。この画面でしか使わないロジックなので `views/` 配下に置く。

## 肥大化した Composable の分割

冒頭で示した `useUserManagement` を、この分類に従って分割するとどうなるか。

### Before: 肥大化した Composable

```ts
// useUserManagement.ts（再掲）
export const useUserManagement = () => {
  const users = ref<User[]>([]);
  const loading = ref(false);
  const error = ref<string | null>(null);
  const selectedUserId = ref<string | null>(null);

  const fetchUsers = async () => { /* ... */ };      // ← loader
  const deleteUser = async (id: string) => { /* ... */ }; // ← commands + usecase
  const updateUser = async (id: string, data: Partial<User>) => { /* ... */ }; // ← commands + usecase

  const selectedUser = computed(() => /* ... */);    // ← ui
  const activeUsers = computed(() => /* ... */);     // ← ui
  const selectUser = (id: string) => { /* ... */ };  // ← ui

  return { users, loading, error, selectedUser, activeUsers, fetchUsers, deleteUser, updateUser, selectUser };
};
```

### After: 責任ごとに分離

| Before（useUserManagement 内）   | After                                  | カテゴリ |
| -------------------------------- | -------------------------------------- | -------- |
| `users`, `loading`, `error`      | `useFetchUsersLoader`                  | loader   |
| `fetchUsers`                     | `useFetchUsersLoader`                  | loader   |
| `deleteUser`（API 呼び出し部分） | `useDeleteUserCommand`                 | commands |
| `updateUser`（API 呼び出し部分） | `useUpdateUserCommand`                 | commands |
| `deleteUser`（状態更新部分）     | `useDeleteUserUsecase`                 | usecase  |
| `updateUser`（状態更新部分）     | `useUpdateUserUsecase`                 | usecase  |
| `selectedUserId`, `selectUser`   | `useUserSelection`                     | ui       |
| `selectedUser`                   | `useUserSelection` または Store getter | ui       |
| `activeUsers`                    | `useUserList`（View 専用 ui）          | ui       |

### 分割後のファイル構成

```
composables/
  loader/
    useFetchUsersLoader.ts      # fetchUsers の API 呼び出し + Store 格納
  commands/
    useDeleteUserCommand.ts     # deleteUser の API 呼び出し
    useUpdateUserCommand.ts     # updateUser の API 呼び出し
  ui/
    useUserSelection.ts         # selectedUserId, selectUser, selectedUser

views/
  userList/
    useInitializeUserList.ts    # 初期表示 usecase（loader を呼ぶ）
    useDeleteUserUsecase.ts     # 削除 usecase（commands + Store 更新）
    useUserList.ts              # activeUsers（フィルタリング）
```

このように分割することで、各 Composable が単一の責任を持ち、再利用性とテスタビリティが向上する。

## 分離後のコンポーネント

分類された Composable を使うと、Component はシンプルになる。

```vue
<!-- views/userList/UserListView.vue -->
<script setup lang="ts">
import { onMounted, reactive } from "vue";
// usecase（複数呼び出し可）
import { useInitializeUserList } from "./useInitializeUserList";
import { useDeleteUserUsecase } from "./useDeleteUserUsecase";
// ui（View専用）
import { useUserList } from "./useUserList";
// ui（共通）
import { useUserSelection } from "@/composables/ui/useUserSelection";
import { useDisclosure } from "@/composables/ui/useDisclosure";

// usecase: 初期表示
const { initialize, loading, error } = useInitializeUserList();

// usecase: ユーザー削除
const { confirmDelete } = useDeleteUserUsecase();

// ui: フィルタリング（View専用）
const { searchQuery, statusFilter, filteredUsers } = useUserList();

// ui: 選択状態
const { selectedId, select, clear } = useUserSelection();

// ui: モーダル開閉
const deleteModal = reactive(useDisclosure());

onMounted(initialize);
</script>

<template>
  <div>
    <!-- フィルタUI -->
    <input v-model="searchQuery" placeholder="検索..." />
    <select v-model="statusFilter">
      <option value="all">すべて</option>
      <option value="active">有効</option>
      <option value="inactive">無効</option>
    </select>

    <div v-if="loading">読み込み中...</div>
    <div v-else-if="error">{{ error }}</div>
    <ul v-else>
      <li
        v-for="user in filteredUsers"
        :key="user.id"
        :class="{ selected: selectedId === user.id }"
        @click="select(user.id)"
      >
        {{ user.name }}
        <button @click.stop="deleteModal.open()">削除</button>
      </li>
    </ul>

    <ConfirmModal
      :is-open="deleteModal.isOpen"
      @confirm="() => confirmDelete(selectedId, clear, deleteModal.close)"
      @cancel="deleteModal.close()"
    >
      削除しますか？
    </ConfirmModal>
  </div>
</template>
```

Component の責任は「表示」と「発火点」だけ。ロジックは Composable に委譲されている。

**ポイント**:

- Component は**複数の usecase**と**ui**を直接呼び出せる
- **View 専用の ui**（`useUserList`）と**共通の ui**（`useDisclosure`）を組み合わせられる

## 依存の方向

この分類には、明確な依存の方向がある。

```
Component
    ↓
┌─────────────────────────────────┐
│         usecase                 │
└─────────────────────────────────┘
    ↓           ↓           ↓
┌────────┐  ┌────────┐  ┌────────┐
│ loader │  │commands│  │   ui   │
└────────┘  └────────┘  └────────┘
    ↓           ↓           ↓
    └─────┬─────┘           │
          ↓                 ↓
       Service           Store
      (API通信)      (selector/mutator)
```

※ Component は usecase だけでなく ui も直接呼べる

**依存ルール**:

- **Component** → usecase または ui を直接呼べる
- **usecase** → loader / commands / ui / Store mutator を使う
- **loader** → Service を使う（Store 格納または値返却）
- **commands** → Service を使う（Store の更新は usecase が行う）
- **ui** → Store の getter を使う
- **Store** → Service を呼び出さない（状態の器のみ）

## まとめ

Composable を役割で分類することで、責任が明確になる。

| カテゴリ     | 責任                                         |
| ------------ | -------------------------------------------- |
| **loader**   | バックエンドからのデータ取得                 |
| **commands** | バックエンドへのデータ送信                   |
| **ui**       | 画面表示に関わるロジック                     |
| **usecase**  | ユーザー操作に紐づくアプリケーションロジック |

この分類を徹底することで：

- 各 Composable の責任が単一になる
- 依存関係が明確になる
- テストが書きやすくなる
- 再利用性が向上する

---

## 関連記事

フロントエンド責任分離アーキテクチャシリーズ：

- [全体像](./frontend-clean-architecture) - 責任分離のレイヤードアーキテクチャ
- [コンポーネントの責任](./frontend-component-responsibility) - 画面表示とユースケースの発火点
- **Composable の責任分離**（本記事）
- [Store の設計](./frontend-store-entity-state) - Entity State パターンと責任分離
