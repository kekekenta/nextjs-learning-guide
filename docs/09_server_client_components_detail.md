# サーバーコンポーネント vs クライアントコンポーネント詳細解説

## 📚 学習目標（30分）
Next.jsのレンダリング処理フローを完全理解

## 1. リクエスト処理の全体フロー

### ユーザーがページにアクセスした時の流れ

```
[ブラウザ] → [Next.jsサーバー] → [レンダリング] → [レスポンス]
```

### 初回アクセス時の詳細フロー

```typescript
// 1. ブラウザがGETリクエストを送信
// GET https://example.com/users/123

// 2. Next.jsサーバーがリクエストを受信
// 3. ルーティング解決: app/users/[id]/page.tsx

// 4. サーバーコンポーネントの実行（サーバー側）
// app/users/[id]/page.tsx
export default async function UserPage({ params }: { params: { id: string } }) {
  // ① データベースに直接アクセス（サーバー側で実行）
  const user = await prisma.user.findUnique({
    where: { id: params.id },
    include: { posts: true }
  });
  
  // ② APIキーなどのシークレットも安全に使用可能
  const externalData = await fetch('https://api.example.com/data', {
    headers: {
      'Authorization': `Bearer ${process.env.SECRET_API_KEY}` // サーバーのみ
    }
  });
  
  // ③ React Server Component Payload (RSC Payload) として返却
  // HTMLではなく、シリアライズされたReactコンポーネントツリー
  return (
    <div>
      <h1>{user.name}</h1>
      <UserPosts posts={user.posts} />
      {/* クライアントコンポーネントを含む場合 */}
      <UserInteractions userId={user.id} />
    </div>
  );
}

// 5. クライアントコンポーネントの処理
// components/UserInteractions.tsx
'use client'; // この宣言でクライアントコンポーネントになる

export function UserInteractions({ userId }: { userId: string }) {
  // このコードはブラウザで実行される
  const [liked, setLiked] = useState(false);
  
  const handleLike = () => {
    setLiked(!liked);
    // ブラウザからAPIを呼び出し
    fetch(`/api/users/${userId}/like`, { method: 'POST' });
  };
  
  return (
    <button onClick={handleLike}>
      {liked ? '❤️' : '🤍'}
    </button>
  );
}
```

## 2. 実際のレスポンスの違い

### サーバーコンポーネントのレスポンス

```javascript
// Next.jsサーバーが返すレスポンス（初回アクセス時）
{
  // HTMLドキュメント
  html: `
    <!DOCTYPE html>
    <html>
      <head>
        <title>User Profile</title>
        <script src="/_next/static/chunks/main.js"></script>
      </head>
      <body>
        <div id="__next">
          <!-- サーバーでレンダリング済みのHTML -->
          <div>
            <h1>John Doe</h1>
            <div class="posts">
              <article>Post 1</article>
              <article>Post 2</article>
            </div>
            <!-- クライアントコンポーネントのプレースホルダー -->
            <div data-client-component="UserInteractions" data-props='{"userId":"123"}'>
              <button>🤍</button> <!-- 初期状態 -->
            </div>
          </div>
        </div>
        
        <!-- RSC Payload（React Server Component データ） -->
        <script id="__NEXT_DATA__" type="application/json">
          {
            "props": {
              "pageProps": {
                "user": {
                  "id": "123",
                  "name": "John Doe",
                  "posts": [...]
                }
              }
            },
            "page": "/users/[id]",
            "query": {"id": "123"},
            "buildId": "abc123",
            "runtimeConfig": {}
          }
        </script>
      </body>
    </html>
  `,
  
  // HTTPヘッダー
  headers: {
    'Content-Type': 'text/html; charset=utf-8',
    'Cache-Control': 'private, no-cache, no-store, max-age=0, must-revalidate'
  }
}
```

### クライアントサイドナビゲーション時のレスポンス

```javascript
// ユーザーが別のページへ遷移した時（SPA的な動作）
// GET https://example.com/_next/data/abc123/users/456.json

{
  // RSC Payload のみ（HTMLではない）
  "pageProps": {
    "user": {
      "id": "456",
      "name": "Jane Smith",
      "posts": [
        {"id": "1", "title": "Hello World"},
        {"id": "2", "title": "Next.js Guide"}
      ]
    }
  },
  "__N_SSP": true // Server-Side Propsの存在を示す
}
```

## 3. データフェッチングの詳細

### サーバーコンポーネントでのデータ取得

```typescript
// app/dashboard/page.tsx
export default async function Dashboard() {
  // パラレルでデータ取得（サーバー側で実行）
  const [user, stats, notifications] = await Promise.all([
    // データベース直接アクセス
    prisma.user.findUnique({ where: { id: getCurrentUserId() } }),
    
    // 内部APIコール（同じサーバー内）
    getStatistics(),
    
    // 外部API（秘密鍵を使用）
    fetch('https://api.service.com/notifications', {
      headers: { 'X-API-Key': process.env.SECRET_KEY }
    }).then(res => res.json())
  ]);
  
  // レンダリング時点でデータは全て揃っている
  return (
    <div>
      <UserInfo user={user} />
      <StatsChart data={stats} />
      <NotificationList items={notifications} />
      {/* クライアントコンポーネントにデータを渡す */}
      <InteractiveChart initialData={stats} />
    </div>
  );
}
```

### クライアントコンポーネントでのデータ取得

```typescript
'use client';

export function InteractiveChart({ initialData }: { initialData: Stats }) {
  const [data, setData] = useState(initialData);
  const [loading, setLoading] = useState(false);
  
  // ユーザーインタラクション後のデータ取得
  const refreshData = async () => {
    setLoading(true);
    
    // ブラウザからAPIエンドポイントを呼び出す
    const response = await fetch('/api/stats', {
      headers: {
        'Content-Type': 'application/json',
        // クライアントからはpublicな値のみアクセス可能
        'X-Client-Version': process.env.NEXT_PUBLIC_APP_VERSION
      }
    });
    
    const newData = await response.json();
    setData(newData);
    setLoading(false);
  };
  
  return (
    <div>
      <Chart data={data} />
      <button onClick={refreshData} disabled={loading}>
        {loading ? 'Loading...' : 'Refresh'}
      </button>
    </div>
  );
}
```

## 4. レンダリング戦略の詳細

### Static Rendering（ビルド時生成）

```typescript
// app/blog/[slug]/page.tsx

// generateStaticParams: ビルド時に実行され、生成すべきページのリストを返す
export async function generateStaticParams() {
  // データベースから全記事を取得
  const posts = await prisma.post.findMany({
    where: { published: true },
    select: { slug: true }
  });
  
  // 各slugに対して静的ページが生成される
  // 例: ['hello-world', 'next-guide'] → /blog/hello-world.html, /blog/next-guide.html
  return posts.map((post) => ({
    slug: post.slug,
  }));
}

// ビルド時の処理フロー:
// 1. generateStaticParams()実行 → ['hello-world', 'next-guide', 'prisma-tips']
// 2. 各slugでBlogPostコンポーネント実行
// 3. 静的HTMLファイル生成: .next/server/app/blog/[slug].html

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await prisma.post.findUnique({
    where: { slug: params.slug },
    include: { author: true, tags: true }
  });
  
  if (!post) notFound(); // 404ページ表示
  
  // ビルド時にHTMLが生成される
  // レスポンス: 事前生成された静的HTMLファイル（超高速）
  return <Article post={post} />;
}

// オプション: 事前生成されていないパスの扱い
export const dynamicParams = true; // デフォルト: true
// true: リクエスト時に動的生成（ISR的動作）
// false: 404エラー

// オプション: 再検証間隔（ISR）
export const revalidate = 3600; // 1時間ごとに再生成
```

詳細は[generateStaticParams完全ガイド](./15_generateStaticParams_detail.md)を参照

### Dynamic Rendering（リクエスト時生成）

```typescript
// app/realtime/page.tsx
export const dynamic = 'force-dynamic'; // 常に動的レンダリング

export default async function RealtimePage() {
  // リクエスト毎に実行される
  const liveData = await getLiveData();
  
  return <LiveDashboard data={liveData} />;
}
```

### Streaming（段階的レンダリング）

```typescript
// app/feed/page.tsx
import { Suspense } from 'react';

export default function Feed() {
  return (
    <div>
      {/* 即座にレンダリング */}
      <Header />
      
      {/* 非同期でストリーミング */}
      <Suspense fallback={<PostsSkeleton />}>
        <PostList /> {/* async component */}
      </Suspense>
      
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar /> {/* async component */}
      </Suspense>
    </div>
  );
}

async function PostList() {
  // データ取得に時間がかかる
  const posts = await fetchPosts();
  return <div>{/* posts rendering */}</div>;
}

// レスポンスの流れ:
// 1. 初回: Header + Skeleton
// 2. PostList準備完了: PostListのHTMLをストリーミング
// 3. Sidebar準備完了: SidebarのHTMLをストリーミング
```

## 5. 実際のネットワークトラフィック

### 初回ページロード

```http
GET /users/123 HTTP/1.1
Host: example.com

Response:
HTTP/1.1 200 OK
Content-Type: text/html
Transfer-Encoding: chunked

<!-- Chunk 1: 初期HTML -->
<!DOCTYPE html>
<html><head>...</head><body>
<div id="__next">

<!-- Chunk 2: サーバーコンポーネントの結果 -->
<div><h1>John Doe</h1>
<div class="posts">...

<!-- Chunk 3: クライアントコンポーネントのハイドレーション情報 -->
<script>self.__next_f.push([1,"...RSC Payload..."])</script>

<!-- Chunk 4: 追加のストリーミングデータ -->
<template id="B:0">...</template>
```

### クライアントサイドナビゲーション

```http
GET /_next/data/buildId/users/456.json HTTP/1.1
Host: example.com
Next-Router-Prefetch: 1

Response:
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: s-maxage=31536000, stale-while-revalidate

{
  "pageProps": {
    "user": {...},
    "__N_SSP": true
  }
}
```

## 6. パフォーマンスへの影響

### サーバーコンポーネントの利点

```typescript
// ❌ 従来のクライアントサイドフェッチ
'use client';
function OldWay() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    // 1. ページロード
    // 2. JavaScriptダウンロード・実行
    // 3. APIコール（ウォーターフォール）
    fetch('/api/data').then(res => res.json()).then(setData);
  }, []);
  
  if (!data) return <Spinner />;
  return <div>{data}</div>;
}

// ✅ サーバーコンポーネント
async function NewWay() {
  // サーバー側で全て完了
  const data = await getData();
  
  // 完成したHTMLを送信
  return <div>{data}</div>;
}
```

### バンドルサイズの削減

```javascript
// サーバーコンポーネント内で使用したライブラリは
// クライアントバンドルに含まれない

// サーバーのみ（バンドルサイズ: 0KB）
import { parseMarkdown } from 'heavy-markdown-library'; // 500KB
import { analyzeData } from 'data-analysis-lib'; // 300KB

// クライアント（バンドルに含まれる）
'use client';
import { useState } from 'react'; // 必要最小限
```

## 7. Suspenseとクライアントコンポーネントの使い分け

### Suspenseとは
- **Reactの機能**で、非同期コンポーネントの読み込み中にフォールバックUIを表示
- **サーバーコンポーネントでもクライアントコンポーネントでも使用可能**
- データフェッチングやコンポーネントの遅延読み込みの待機状態を管理

### クライアントコンポーネントとは
- `'use client'`宣言で指定されるコンポーネント
- **ブラウザで実行される**
- useState、useEffect、onClickなどのインタラクティブ機能が使える

### 使い分けの基準

#### Suspenseを使う場面
- **データフェッチングを含むサーバーコンポーネントをラップ**
- ページの一部を段階的に表示したい時（Streaming）
- 重いコンポーネントの遅延読み込み

```typescript
// サーバーコンポーネントでのデータフェッチング
export default function Page() {
  return (
    <div>
      <h1>Dashboard</h1>
      {/* データ取得中はLoadingを表示 */}
      <Suspense fallback={<Loading />}>
        <SlowDataComponent /> {/* DBアクセスなど時間がかかる処理 */}
      </Suspense>
    </div>
  );
}

async function SlowDataComponent() {
  // サーバー側でデータ取得
  const data = await fetchDataFromDB();
  return <DataDisplay data={data} />;
}
```

#### クライアントコンポーネントを使う場面
- **ユーザーインタラクション**（クリック、入力、ホバー）
- **ブラウザAPI使用**（localStorage、window、document）
- **状態管理**（useState、useReducer）
- **リアルタイム更新**、WebSocket接続

```typescript
'use client';
export function InteractiveForm() {
  const [value, setValue] = useState('');
  const [submitted, setSubmitted] = useState(false);
  
  const handleSubmit = () => {
    // ブラウザで実行される処理
    localStorage.setItem('formData', value);
    setSubmitted(true);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input onChange={(e) => setValue(e.target.value)} />
      <button type="submit">送信</button>
    </form>
  );
}
```

### 実践的な組み合わせパターン

```typescript
// app/dashboard/page.tsx
export default function Dashboard() {
  return (
    <div>
      {/* 1. 静的部分：即座に表示 */}
      <Header />
      <Navigation />
      
      {/* 2. 非同期データ：Suspenseで段階表示 */}
      <div className="grid grid-cols-2">
        <Suspense fallback={<ChartSkeleton />}>
          <SalesChart /> {/* サーバーでデータ取得 */}
        </Suspense>
        
        <Suspense fallback={<TableSkeleton />}>
          <UserTable /> {/* サーバーでデータ取得 */}
        </Suspense>
      </div>
      
      {/* 3. インタラクティブ部分：クライアントコンポーネント */}
      <FilterPanel /> {/* 'use client' でユーザー操作 */}
    </div>
  );
}

// サーバーコンポーネント（データ取得）
async function SalesChart() {
  const salesData = await fetchSalesData();
  return (
    <div>
      <h2>売上データ</h2>
      {/* データを表示 */}
      <ChartDisplay data={salesData} />
      {/* インタラクティブな機能を追加 */}
      <ChartInteractions initialData={salesData} />
    </div>
  );
}

// クライアントコンポーネント（インタラクション）
'use client';
function ChartInteractions({ initialData }) {
  const [range, setRange] = useState('week');
  
  return (
    <div>
      <button onClick={() => setRange('day')}>日別</button>
      <button onClick={() => setRange('week')}>週別</button>
      <button onClick={() => setRange('month')}>月別</button>
    </div>
  );
}
```

### 判断フローチャート

```
データ取得が必要？
  ├─ Yes → Suspenseでラップ
  │         └─ ユーザー操作も必要？
  │              ├─ Yes → Suspense内にクライアントコンポーネント配置
  │              └─ No → サーバーコンポーネントのみ
  └─ No → ユーザー操作が必要？
           ├─ Yes → クライアントコンポーネント
           └─ No → 通常のサーバーコンポーネント
```

### よくある間違い

```typescript
// ❌ 間違い：クライアントコンポーネントで直接async/await
'use client';
async function WrongComponent() { // エラー！
  const data = await fetchData();
  return <div>{data}</div>;
}

// ✅ 正解：useEffectまたはサーバーコンポーネントを使用
'use client';
function CorrectComponent() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  
  return <div>{data}</div>;
}

// ✅ または、サーバーコンポーネントでデータ取得
async function ServerComponent() {
  const data = await fetchData();
  return <ClientComponent initialData={data} />;
}
```

## 🎯 重要なポイント

| 側面 | サーバーコンポーネント | クライアントコンポーネント |
|------|------------------------|---------------------------|
| 実行場所 | サーバー（Node.js） | ブラウザ |
| データアクセス | DB直接、秘密鍵OK | Public APIのみ |
| 初期レスポンス | 完成したHTML | JavaScript実行後 |
| SEO | 完璧 | 要対策 |
| インタラクティブ性 | なし | フル対応 |
| バンドルサイズ | 影響なし | 全て含まれる |