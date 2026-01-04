---
title: "フロントエンド アーキテクチャ実践 - 6万行を一人で書いて辿り着いた形"
emoji: "🏛️"
type: "tech"
topics: ["frontend", "vue", "react", "architecture", "cleanarchitecture"]
published: true
published_at: "2026-01-05 08:00"
---

## はじめに

中〜大規模のフロントエンド開発では、**堅牢で壊れづらく、変更に強い設計**が求められる。要件変更への対応、複数人での開発、長期的な保守——これらを見据えたアーキテクチャが必要だ。

Clean Architecture の原則は参考になるが、オブジェクト指向の世界で生まれた概念をフロントエンドにそのまま適用することはできない。フロントエンドの特性に合わせて、どのような形に落とし込むか——それが設計上の課題となる。

本記事では、試行錯誤の末にたどり着いた責任分離のレイヤードアーキテクチャを紹介する。

### この記事の内容

- アーキテクチャの全体像と依存ルール
- 各レイヤー（Utils / Infrastructure / Service / Store / Composable / Component）の責任
- やってはいけないこと

### 対象読者

- 中〜大規模プロジェクトでアーキテクチャ設計に悩んでいる
- テックリードやアーキテクトとして、チームの設計指針を定めたい
- 影響範囲が読めず、修正するとテスト工数が雪だるま式に膨らむコードを抱えている

サンプルは Vue で書くが、考え方は React でも同じ。

## アーキテクチャ全体像

### レイヤー構成

```
┌─────────────────────────────────────────────────────┐
│                    Component                        │
│              （UI表示 + 発火点）                    │
└─────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│                    Composable                       │
│         loader / commands / ui / usecase            │
└─────────────────────────────────────────────────────┘
        ↓           ↓           ↓
┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐
│   Service   │ │    Store    │ │        Utils        │
│ (API抽象化) │ │ (状態の器)  │ │  (ビジネスルール)   │
└─────────────┘ └─────────────┘ └─────────────────────┘
        ↓
┌─────────────────────────────────────────────────────┐
│                  Infrastructure                     │
│            （axios, 認証, LocalStorage）            │
└─────────────────────────────────────────────────────┘
```

### 依存の方向

**依存は常に上から下へ流れる。逆方向の依存は禁止。**

| レイヤー       | 呼び出し可能          |
| -------------- | --------------------- |
| Component      | Composable            |
| Composable     | Service, Store, Utils |
| Service        | Infrastructure, Utils |
| Store          | Utils                 |
| Infrastructure | Utils                 |
| Utils          | なし（最下層）        |

### 各レイヤーの責任（サマリー）

| レイヤー           | 責任                               |
| ------------------ | ---------------------------------- |
| **Utils**          | ビジネスルール（計算、変換、検証） |
| **Infrastructure** | 外部システム連携（axios、認証）    |
| **Service**        | API 通信の抽象化                   |
| **Store**          | 状態の器（データ保持のみ）         |
| **Composable**     | ユースケース（処理の組み合わせ）   |
| **Component**      | UI 表示 + ユースケースの発火点     |

## Utils - ビジネスルール

### 責任

- 純粋関数（副作用なし）
- バリデーション、計算、変換
- 外部依存は引数で受け取る

### 特徴

Utils は**最下層**にある。また、**どのレイヤーからでも呼び出せる**。

```ts
// utils/validation.ts
export const isValidEmail = (email: string): boolean => {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
};

export const isAdult = (birthDate: Date): boolean => {
  const today = new Date();
  const age = today.getFullYear() - birthDate.getFullYear();
  return age >= 18;
};

// utils/format.ts
export const formatDate = (date: Date): string => {
  return date.toLocaleDateString("ja-JP");
};

export const formatCurrency = (amount: number): string => {
  return new Intl.NumberFormat("ja-JP", {
    style: "currency",
    currency: "JPY",
  }).format(amount);
};
```

### なぜ Utils にビジネスルールを置くのか

- **テスト可能性の高さ**: 純粋関数なので、入力と出力だけをテストすればいい
- **状態に依存しない**: ビジネスルールは普遍であり、アプリケーションの状態に左右されない
- **どこからでも使える**: Composable、Service、どこからでも同じバリデーションを使える

## Infrastructure - 外部システム連携

### 責任

- 外部システムとの連携（HTTP、認証、LocalStorage 等）
- 技術的実装の詳細を隠蔽

### 具体例

```ts
// infrastructure/http.ts
import axios from "axios";

const createHttpClient = (getToken?: () => string) => {
  const client = axios.create({
    baseURL: import.meta.env.VITE_API_BASE_URL,
    timeout: 10000,
  });

  // トークン認証
  if (getToken) {
    client.interceptors.request.use((config) => {
      const token = getToken();
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
  }

  return {
    get: <T>(url: string, config?: Record<string, string>) =>
      client.get<T>(url, config),
    post: <T>(url: string, data?: unknown, config?: Record<string, string>) =>
      client.post<T>(url, data, config),
    put: <T>(url: string, data?: unknown, config?: Record<string, string>) =>
      client.put<T>(url, data, config),
    patch: <T>(url: string, data?: unknown, config?: Record<string, string>) =>
      client.patch<T>(url, data, config),
    delete: <T>(url: string, config?: Record<string, string>) =>
      client.delete<T>(url, config),
  };
};

// 実用：認証トークン取得用の外部関数を注入
export const httpClient = createHttpClient(
  () => sessionStorage.getItem("token") ?? ""
);
```

HTTP メソッドをそのままインターフェースとして公開する。ファクトリ関数で認証トークン取得を注入できるため、テスト時はモックを渡せばよい。
これで、axios から fetch に変更になったとしても、この部分を変更するだけで別レイヤーに変更が波及しない。

### 制約

- Infrastructure 層同士の一方向依存のみ許可
- 循環依存は禁止

## Service - API 通信の抽象化

### 責任

- API エンドポイントの定義
- リクエスト/レスポンスの整形
- Infrastructure 層を使用して API 通信

### 具体例

```ts
// infrastructure/http.ts
export type HttpClient = {
  get: <T>(
    url: string,
    config?: Record<string, string>
  ) => Promise<{ data: T }>;
  post: <T>(
    url: string,
    data?: unknown,
    config?: Record<string, string>
  ) => Promise<{ data: T }>;
  put: <T>(
    url: string,
    data?: unknown,
    config?: Record<string, string>
  ) => Promise<{ data: T }>;
  patch: <T>(
    url: string,
    data?: unknown,
    config?: Record<string, string>
  ) => Promise<{ data: T }>;
  delete: <T>(
    url: string,
    config?: Record<string, string>
  ) => Promise<{ data: T }>;
};
```

```ts
// services/user.ts
import { httpClient, type HttpClient } from "@/infrastructure/http";
import type { User } from "@/types/user";

export const createUserService = (client: HttpClient) => {
  const getAll = async (): Promise<User[]> => {
    const { data } = await client.get<User[]>("/users");
    return data;
  };

  const getById = async (id: string): Promise<User> => {
    const { data } = await client.get<User>(`/users/${id}`);
    return data;
  };

  const create = async (user: Omit<User, "id">): Promise<User> => {
    const { data } = await client.post<User>("/users", user);
    return data;
  };

  const remove = async (id: string): Promise<void> => {
    await client.delete(`/users/${id}`);
  };

  return { getAll, getById, create, remove };
};

// 実用：実際のhttpClientを注入
export const userService = createUserService(httpClient);
```

Service は`HttpClient`型に依存する。実装の詳細（axios や fetch）は知らない。テスト時はモックを注入すればよい。

### 注意点

- Service はビジネスロジックを**定義しない**（それは Utils の責任）。ただし Utils を**呼び出す**ことは可能で、バリデーションなどに利用する。
- エラーを検知したら上位層に throw

### DTO の変換をどこで行うか

Service の後はフロントエンドの世界である。DTO をフロントエンドのドメインモデルに変換するタイミングには 2 つのパターンがある。

**パターン A: Service で変換する**

```ts
// Service でDTO → ドメインモデルに変換
const getById = async (id: string): Promise<User> => {
  const { data } = await client.get<UserDTO>(`/users/${id}`);
  return toUser(data); // 不要なプロパティは捨てる
};
```

Service から先はドメインモデルだけが流通する。シンプルで扱いやすい。

**パターン B: Store で DTO を保持し、selector で変換する**

API の仕様上、更新時に「受け取ったのと同じ形式」で送り返す必要がある場合、DTO をそのまま保持する。

```ts
// Store: DTOをそのまま保持
const byId = shallowRef<Record<string, UserDTO>>({});

// selector: 表示用にドメインモデルへ変換
const users = computed(() => allIds.value.map((id) => toUser(byId.value[id])));
```

どちらを選ぶかは API の仕様次第。
API が「本来フロントで使う必要のないものも多く必要とする」仕様の場合はパターン A、
その必要がない場合はパターン B など、どちらかを選択すればよい。
基本的には A の service で変換するものをとっておくのがベターではある。

## Store - 状態の器

### 責任

- リアクティブな状態の保持（Single Source of Truth）
- **状態を保持するだけ**

### Entity State パターン

データは `byId`（オブジェクト）と `allIds`（配列）で管理する。

```ts
// stores/user/store.ts
import { defineStore } from "pinia";
import { shallowRef, ref, computed } from "vue";
import type { User } from "@/types/user";
import { createSelector } from "./selector";

export const useUserStore = defineStore("user", () => {
  // 状態の器
  const byId = shallowRef<Record<string, User>>({});
  const allIds = ref<string[]>([]);

  // 正規化computed（byId + allIds → 配列）
  const { users } = createSelector(byId, allIds);

  return { byId, allIds, users };
});

// stores/user/selector.ts
import { type Ref, type ShallowRef, computed } from "vue";
import type { User } from "@/types/user";
export const createSelector = (
  byId: ShallowRef<Record<string, User>>,
  allIds: Ref<string[]>
) => {
  const users = computed(() =>
    allIds.value.map((id) => byId.value[id]).filter(Boolean)
  );
  return {
    users,
  };
};
```

### パフォーマンス比較

| 操作    | 配列パターン | Entity State |
| ------- | ------------ | ------------ |
| ID 検索 | O(n)         | O(1)         |
| 更新    | O(n)         | O(1)         |
| 削除    | O(n)         | O(1)         |

### Store に持たせないもの

| 持たせないもの     | 委譲先     |
| ------------------ | ---------- |
| Service 呼び出し   | Composable |
| loading/error 状態 | Composable |

※ Store 内部に selector（データ取得・加工）と mutator（状態変更）を定義する。詳細は [Store の設計](./store-as-state-container) を参照。

## Composable - ロジックと状態管理

### 責任

Composable は以下の 4 つに分類される。

| カテゴリ     | 責任                                         |
| ------------ | -------------------------------------------- |
| **loader**   | バックエンドからのデータ取得 → Store 格納    |
| **commands** | バックエンドへのデータ送信                   |
| **ui**       | 画面表示に関わるロジック（モーダル開閉等）   |
| **usecase**  | ユーザー操作に紐づくアプリケーションロジック |

### 具体例

```ts
// composables/loader/useFetchUsers.ts
export const useFetchUsers = () => {
  const { upsert } = useUserStore();
  const loading = ref(false);
  const error = ref<string | null>(null);

  const execute = async () => {
    loading.value = true;
    error.value = null;
    try {
      const users = await userService.getAll();
      users.forEach((user) => upsert(user));
    } catch (e) {
      error.value = "取得に失敗しました";
    } finally {
      loading.value = false;
    }
  };

  return { execute, loading, error };
};
```

## Component - UI

### 責任

Component の責任は**2 つだけ**。

1. **UI 表示**: データを受け取り、描画する
2. **発火点**: ユーザー操作を起点に、Composable を呼び出す

### 具体例

```vue
<script setup lang="ts">
import { onMounted } from "vue";
import { useFetchUsers } from "@/composables/loader/useFetchUsers";
import { useUserStore } from "@/stores/user";

const { execute: fetchUsers, loading, error } = useFetchUsers();
const { users } = useUserStore();

onMounted(fetchUsers);
</script>

<template>
  <div v-if="loading">読み込み中...</div>
  <div v-else-if="error">{{ error }}</div>
  <ul v-else>
    <li v-for="user in users" :key="user.id">
      {{ user.name }}
    </li>
  </ul>
</template>
```

### Component でやってはいけないこと

| 禁止事項         | 理由                       | 代わりにやること  |
| ---------------- | -------------------------- | ----------------- |
| Store を直接呼ぶ | 状態管理を知るべきではない | Composable 経由   |
| `watch`を書く    | ロジックは責任外           | Composable に抽出 |
| 関数を定義       | ロジックは責任外           | Composable に抽出 |
| `computed`を書く | ロジックは責任外           | Composable に抽出 |

## 依存ルールと禁止事項

### 依存ルール

```
Component
    ↓
Composable
    ↓           ↓
Service      Store
    ↓
Infrastructure
```

**逆方向の依存は絶対禁止。**

### 各レイヤーの禁止事項

| レイヤー       | 禁止事項                                    |
| -------------- | ------------------------------------------- |
| Component      | Store 直接参照、watch、async 関数、computed |
| Composable     | Component の参照                            |
| Service        | Store 参照、UI ロジック                     |
| Store          | Service 呼び出し、複雑なロジック            |
| Infrastructure | 上位層への依存                              |

## テスタビリティと DI

### なぜ DI が必要か

レイヤーを分離しても、テスト時に依存を差し替えられなければ意味がない。Composable が直接 Store や Service を呼び出すと、テストでモックに差し替えられない。

### 高階関数による DI

ファクトリ関数で依存を注入し、テスト時に差し替え可能にする。
ここでは、うえで上げた loader を DI でテスト時に差し替え可能に実装を修正する。

```ts
// composables/loader/useFetchUsers.ts

// 依存のインターフェースを定義
type UserService = {
  getAll: () => Promise<User[]>;
};

type UseUserStore = () => {
  upsert: (user: User) => void;
};

type Dependencies = {
  userService: UserService;
  useUserStore: UseUserStore;
};

// ファクトリ関数（依存を注入）
export const createUseFetchUsers = (deps: Dependencies) => {
  const { upsert } = deps.useUserStore();
  const { userService } = deps;
  return () => {
    const loading = ref(false);
    const error = ref<string | null>(null);

    const execute = async () => {
      loading.value = true;
      error.value = null;
      try {
        const users = await userService.getAll();
        users.forEach((user) => upsert(user));
      } catch (e) {
        error.value = "取得に失敗しました";
      } finally {
        loading.value = false;
      }
    };

    return { execute, loading, error };
  };
};

// 実用（デフォルトの依存を注入済み）
export const useFetchUsers = createUseFetchUsers({
  userService,
  useUserStore,
});
```

### テストでの使い方

```ts
// useFetchUsers.test.ts
describe("useFetchUsers", () => {
  it("取得成功時にStoreに格納される", async () => {
    const mockUsers = [{ id: "1", name: "Test" }];
    const mockService = { getAll: vi.fn().mockResolvedValue(mockUsers) };
    const mockUpsert = vi.fn();
    const mockStore = () => ({ upsert: mockUpsert });

    const useFetchUsers = createUseFetchUsers({
      userService: mockService,
      useUserStore: mockStore,
    });

    const { execute } = useFetchUsers();
    await execute();

    expect(mockService.getAll).toHaveBeenCalled();
    expect(mockUpsert).toHaveBeenCalledTimes(mockUsers.length);
  });
});
```

### DI のメリット

- **テスト容易**: 依存をモックに差し替えられる
- **疎結合**: 依存先の実装を知らなくていい

## まとめ

### アーキテクチャの 3 原則

1. **責任の分離**: 各レイヤーは単一の責任を持つ
2. **依存の方向**: 常に上から下へ、逆方向は禁止
3. **ビジネスルールの独立**: Utils に閉じ込め、どこからでも使える

### 各レイヤーの責任（総まとめ）

| レイヤー       | 責任           | 特徴                         |
| -------------- | -------------- | ---------------------------- |
| Utils          | ビジネスルール | 純粋関数、どこからでも呼べる |
| Infrastructure | 外部連携       | axios、認証                  |
| Service        | API 抽象化     | エンドポイント定義           |
| Store          | 状態の器       | データ保持のみ               |
| Composable     | ユースケース   | 処理の組み合わせ             |
| Component      | UI             | 表示 + 発火点                |

### このアーキテクチャの効果

- **責任が明確**: どこに何を書くか迷わない
- **テストしやすい**: 各レイヤーを独立してテスト可能
- **変更に強い**: 影響範囲が限定される
- **チーム開発しやすい**: 指針があれば一貫したコードになる

---

## 関連記事

各レイヤーの詳細については、以下の記事で深掘りしている。

- [コンポーネントの責任](./frontend-component-responsibility) - 画面表示とユースケースの発火点
- [Composable の責任分離](./frontend-composable-responsibility) - loader / commands / ui / usecase への分類
- [Store の設計](./frontend-store-entity-state) - Entity State パターンと責任分離
