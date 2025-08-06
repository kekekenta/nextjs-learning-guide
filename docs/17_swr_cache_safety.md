# SWR グローバルキャッシュの安全な使い方

## 📚 SWRのキャッシュ共有による事故を防ぐ完全ガイド

SWRは便利な反面、**グローバルにキャッシュを共有する**ため、使い方を間違えるとセキュリティインシデントにつながります。

## 1. SWRのキャッシュの仕組み

### 基本的な動作
```typescript
// 同じキーを使うと、アプリケーション全体で同じキャッシュを参照
// ComponentA.tsx
const { data } = useSWR('/api/user', fetcher);

// ComponentB.tsx（別のコンポーネント）
const { data } = useSWR('/api/user', fetcher); // 同じキャッシュを参照！
```

### キャッシュのスコープ
```
アプリケーション全体
    ↓
SWRグローバルキャッシュ（Map）
    ↓
キー: '/api/user' → 値: { name: 'John', email: 'john@example.com' }
    ↓
すべてのコンポーネントが参照
```

## 2. よくある事故パターンと対策

### 🔥 事故パターン1: ユーザーデータの混在

#### ❌ 危険な実装
```typescript
// 異なるユーザーでも同じキーを使ってしまう
function UserDashboard() {
  // ユーザーAがログイン → データをキャッシュ
  // ユーザーBがログイン → ユーザーAのキャッシュが表示される！
  const { data } = useSWR('/api/dashboard', fetcher);
  
  return (
    <div>
      <h1>Welcome, {data?.name}</h1> {/* 別のユーザーの名前が表示される可能性 */}
      <p>Email: {data?.email}</p>
    </div>
  );
}
```

#### ✅ 安全な実装
```typescript
// ユーザーIDをキーに含める
function UserDashboard({ userId }: { userId: string }) {
  const { data } = useSWR(
    userId ? `/api/users/${userId}/dashboard` : null,
    fetcher
  );
  
  return (
    <div>
      <h1>Welcome, {data?.name}</h1>
      <p>Email: {data?.email}</p>
    </div>
  );
}

// またはセッション情報を使用
import { useSession } from 'next-auth/react';

function UserDashboard() {
  const { data: session } = useSession();
  const { data } = useSWR(
    session ? `/api/users/${session.user.id}/dashboard` : null,
    fetcher
  );
  
  return <div>{/* ... */}</div>;
}
```

### 🔥 事故パターン2: フィルター条件の無視

#### ❌ 危険な実装
```typescript
function TaskList({ status, priority }: { status: string; priority: string }) {
  // フィルターが変わってもキーが同じ
  const { data } = useSWR('/api/tasks', async () => {
    // キーとは異なるURLでフェッチ（危険！）
    const res = await fetch(`/api/tasks?status=${status}&priority=${priority}`);
    return res.json();
  });
  
  // status="done"のデータがstatus="pending"でも表示される
  return <div>{data?.tasks.map(/* ... */)}</div>;
}
```

#### ✅ 安全な実装
```typescript
// 方法1: URLにパラメータを含める
function TaskList({ status, priority }: { status: string; priority: string }) {
  const url = `/api/tasks?status=${status}&priority=${priority}`;
  const { data } = useSWR(url, fetcher);
  
  return <div>{data?.tasks.map(/* ... */)}</div>;
}

// 方法2: 配列形式のキー（推奨）
function TaskList({ status, priority, userId }: Props) {
  const { data } = useSWR(
    ['tasks', userId, status, priority], // すべての条件をキーに含める
    async ([_, userId, status, priority]) => {
      const params = new URLSearchParams({ status, priority });
      const res = await fetch(`/api/users/${userId}/tasks?${params}`);
      return res.json();
    }
  );
  
  return <div>{data?.tasks.map(/* ... */)}</div>;
}

// 方法3: オブジェクト形式のキー
function TaskList({ filters }: { filters: TaskFilters }) {
  const { data } = useSWR(
    { type: 'tasks', ...filters }, // オブジェクトをキーとして使用
    async (key) => {
      const { type, ...params } = key;
      const query = new URLSearchParams(params);
      return fetch(`/api/${type}?${query}`).then(r => r.json());
    }
  );
}
```

### 🔥 事故パターン3: ログアウト後のデータ残存

#### ❌ 危険な実装
```typescript
function useAuth() {
  const logout = async () => {
    await fetch('/api/logout', { method: 'POST' });
    router.push('/login');
    // キャッシュが残ったまま！次のユーザーに前のユーザーのデータが見える
  };
  
  return { logout };
}
```

#### ✅ 安全な実装
```typescript
import { mutate } from 'swr';

function useAuth() {
  const logout = async () => {
    await fetch('/api/logout', { method: 'POST' });
    
    // 方法1: すべてのキャッシュをクリア
    await mutate(
      () => true, // すべてのキーにマッチ
      undefined,  // データをundefinedに
      { revalidate: false } // 再検証しない
    );
    
    // 方法2: 特定のパターンのキャッシュのみクリア
    await mutate(
      (key) => typeof key === 'string' && key.startsWith('/api/user'),
      undefined,
      { revalidate: false }
    );
    
    // 方法3: 重要なキーを個別にクリア
    const keysToInvalidate = [
      '/api/me',
      '/api/dashboard',
      '/api/settings'
    ];
    
    await Promise.all(
      keysToInvalidate.map(key => 
        mutate(key, undefined, { revalidate: false })
      )
    );
    
    router.push('/login');
  };
  
  return { logout };
}
```

### 🔥 事故パターン4: 権限の異なるデータの混在

#### ❌ 危険な実装
```typescript
// 管理者と一般ユーザーで同じキーを使用
function UserList() {
  const { data: users } = useSWR('/api/users', fetcher);
  
  // 管理者: 全ユーザーのメールアドレスが見える
  // 一般ユーザー: 名前だけ見える
  // → キャッシュ共有により一般ユーザーにメールアドレスが漏洩！
  
  return users.map(user => (
    <div>
      {user.name}
      {user.email} {/* 権限がなくても表示される可能性 */}
    </div>
  ));
}
```

#### ✅ 安全な実装
```typescript
function UserList() {
  const { data: session } = useSession();
  
  // ロールをキーに含める
  const { data: users } = useSWR(
    session ? [`/api/users`, session.user.role] : null,
    async ([url, role]) => {
      const res = await fetch(url, {
        headers: { 'X-User-Role': role }
      });
      return res.json();
    }
  );
  
  return users.map(user => (
    <div>
      {user.name}
      {session?.user.role === 'admin' && user.email}
    </div>
  ));
}
```

## 3. 安全なキー設計パターン

### キー設計の原則
```typescript
// lib/swr-keys.ts
export const swrKeys = {
  // ユーザースコープ（必ずuserIdを含める）
  user: {
    profile: (userId: string) => `/api/users/${userId}/profile`,
    settings: (userId: string) => `/api/users/${userId}/settings`,
    tasks: (userId: string, filters?: any) => 
      [`/api/users/${userId}/tasks`, filters].filter(Boolean),
  },
  
  // グローバルスコープ（全ユーザー共通）
  public: {
    posts: () => '/api/posts/public',
    announcements: () => '/api/announcements',
  },
  
  // 権限スコープ（ロールを含める）
  admin: {
    users: (role: string) => `/api/admin/users?role=${role}`,
    logs: (role: string) => `/api/admin/logs?role=${role}`,
  },
  
  // 複合キー（配列形式）
  complex: {
    filteredData: (userId: string, filters: any, options: any) => 
      ['data', userId, filters, options],
  }
} as const;

// 使用例
import { swrKeys } from '@/lib/swr-keys';

function MyComponent({ userId }: { userId: string }) {
  const { data } = useSWR(
    swrKeys.user.profile(userId),
    fetcher
  );
}
```

## 4. SWRConfigによるスコープ分離

### ユーザーごとのキャッシュ分離
```typescript
// app/providers.tsx
import { SWRConfig } from 'swr';

function UserScopeProvider({ 
  children, 
  userId 
}: { 
  children: React.ReactNode;
  userId: string;
}) {
  // ユーザーごとに異なるキャッシュマップを使用
  const cache = useMemo(() => new Map(), [userId]);
  
  return (
    <SWRConfig
      value={{
        provider: () => cache,
        // キーにユーザーIDを自動付与
        use: [
          (useSWRNext) => (key, fetcher, config) => {
            const serializedKey = Array.isArray(key) 
              ? ['user', userId, ...key]
              : typeof key === 'string'
              ? `/users/${userId}${key}`
              : key;
            
            return useSWRNext(serializedKey, fetcher, config);
          }
        ]
      }}
    >
      {children}
    </SWRConfig>
  );
}
```

### 環境ごとのキャッシュ設定
```typescript
// 開発環境: キャッシュを短く
// 本番環境: 通常のキャッシュ
function AppProvider({ children }: { children: React.ReactNode }) {
  return (
    <SWRConfig
      value={{
        dedupingInterval: process.env.NODE_ENV === 'development' ? 0 : 2000,
        focusThrottleInterval: process.env.NODE_ENV === 'development' ? 0 : 5000,
        shouldRetryOnError: process.env.NODE_ENV === 'production',
      }}
    >
      {children}
    </SWRConfig>
  );
}
```

## 5. センシティブデータの扱い

### 機密データ用の設定
```typescript
// hooks/useSecureData.ts
function useSecureData(key: string) {
  return useSWR(key, fetcher, {
    // キャッシュ共有を無効化
    dedupingInterval: 0,
    
    // リトライしない
    shouldRetryOnError: false,
    
    // フォーカス時の再検証を無効化
    revalidateOnFocus: false,
    
    // 再接続時の再検証を無効化
    revalidateOnReconnect: false,
    
    // キャッシュの有効期限を短く
    refreshInterval: 0,
  });
}

// 使用例
function BankAccount() {
  const { data } = useSecureData('/api/bank/balance');
  return <div>Balance: ${data?.balance}</div>;
}
```

## 6. テストでの考慮事項

```typescript
// __tests__/swr-cache.test.tsx
import { SWRConfig, mutate } from 'swr';
import { render } from '@testing-library/react';

describe('SWR Cache Tests', () => {
  beforeEach(() => {
    // 各テストの前にキャッシュをクリア
    mutate(() => true, undefined, { revalidate: false });
  });
  
  it('should not share cache between users', async () => {
    const cache1 = new Map();
    const cache2 = new Map();
    
    // ユーザー1のコンポーネント
    const { getByText: getByText1 } = render(
      <SWRConfig value={{ provider: () => cache1 }}>
        <UserProfile userId="user1" />
      </SWRConfig>
    );
    
    // ユーザー2のコンポーネント
    const { getByText: getByText2 } = render(
      <SWRConfig value={{ provider: () => cache2 }}>
        <UserProfile userId="user2" />
      </SWRConfig>
    );
    
    // 異なるデータが表示されることを確認
    expect(getByText1('User 1')).toBeInTheDocument();
    expect(getByText2('User 2')).toBeInTheDocument();
  });
});
```

## 7. チェックリスト

### デプロイ前の安全性チェック

- [ ] すべてのユーザー固有データのキーにユーザーIDが含まれているか
- [ ] フィルター条件がキーに反映されているか
- [ ] ログアウト時にキャッシュクリアしているか
- [ ] 権限によって異なるデータは分離されているか
- [ ] センシティブなデータは`dedupingInterval: 0`を設定しているか
- [ ] テストで異なるユーザー間のキャッシュ分離を確認したか

## 8. トラブルシューティング

### デバッグ方法
```typescript
// SWRのキャッシュを確認
import { cache } from 'swr';

function DebugCache() {
  const handleDebug = () => {
    // 現在のキャッシュの中身を確認
    console.log('Current cache:', cache);
    
    // 特定のキーのデータを確認
    const userData = cache.get('/api/user');
    console.log('User data in cache:', userData);
  };
  
  return <button onClick={handleDebug}>Debug Cache</button>;
}

// Chrome DevToolsでSWRをデバッグ
if (process.env.NODE_ENV === 'development') {
  window.__SWR__ = { cache, mutate };
}
```

## まとめ

SWRのグローバルキャッシュは強力な機能ですが、**適切なキー設計なしには危険**です。

### 最重要ポイント

1. **ユーザーIDを必ずキーに含める**
2. **動的パラメータはすべてキーに反映**
3. **ログアウト時はキャッシュクリア**
4. **センシティブデータは特別扱い**

### Rails開発者への注意

Railsの`current_user`のような暗黙的なスコープはSWRにはありません。
常に明示的にユーザースコープを指定する必要があります。

```ruby
# Rails: 自動的にスコープ
current_user.tasks # ユーザーのタスクのみ
```

```typescript
// Next.js + SWR: 明示的にスコープ
useSWR(`/api/users/${userId}/tasks`) // userIdを必ず含める
```