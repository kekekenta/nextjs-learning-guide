# generateStaticParams 完全ガイド

## 📚 generateStaticParamsとは？

Next.js App Routerで**ビルド時に動的ルートの静的ページを事前生成**するための関数です。
Railsでいうと、事前にすべてのルートのHTMLをキャッシュ生成するイメージです。

## 1. 基本的な仕組み

### 動的ルートとは
```
app/
├── blog/
│   └── [slug]/        # 動的ルート: /blog/hello, /blog/world など
│       └── page.tsx
├── products/
│   └── [category]/
│       └── [id]/      # ネストした動的ルート: /products/electronics/iphone-15
│           └── page.tsx
```

### generateStaticParamsの役割
```typescript
// app/blog/[slug]/page.tsx

// ビルド時に実行される
export async function generateStaticParams() {
  // この関数が返すパラメータの数だけ静的ページが生成される
  const posts = await prisma.post.findMany({
    select: { slug: true }
  });
  
  // [
  //   { slug: 'hello-world' },
  //   { slug: 'next-js-guide' },
  //   { slug: 'prisma-tutorial' }
  // ]
  return posts.map((post) => ({
    slug: post.slug
  }));
}

// 各パラメータに対してこのページコンポーネントが実行される
export default async function BlogPost({ 
  params 
}: { 
  params: { slug: string } 
}) {
  const post = await prisma.post.findUnique({
    where: { slug: params.slug }
  });
  
  return <Article post={post} />;
}
```

## 2. ビルド時の処理フロー

```typescript
// ビルド時の処理順序
// 1. generateStaticParams()が実行される
// 2. 返されたパラメータリストを取得
// 3. 各パラメータでページコンポーネントを実行
// 4. HTMLファイルを生成して保存

// 例：3つのブログ記事がある場合
// ビルド後のファイル構造:
// .next/server/app/blog/
// ├── hello-world.html
// ├── next-js-guide.html
// └── prisma-tutorial.html
```

## 3. 実践的な使用例

### 基本例：ブログ記事
```typescript
// app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation';

export async function generateStaticParams() {
  const posts = await prisma.post.findMany({
    where: { published: true },
    select: { slug: true }
  });
  
  console.log(`Generating ${posts.length} blog pages...`);
  
  return posts.map((post) => ({
    slug: post.slug
  }));
}

export default async function BlogPost({ 
  params 
}: { 
  params: { slug: string } 
}) {
  const post = await prisma.post.findUnique({
    where: { 
      slug: params.slug,
      published: true 
    },
    include: {
      author: true,
      tags: true
    }
  });
  
  // 存在しないページへのアクセス
  if (!post) {
    notFound(); // 404ページを表示
  }
  
  return (
    <article>
      <h1>{post.title}</h1>
      <p>By {post.author.name}</p>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

### ネストした動的ルート
```typescript
// app/products/[category]/[id]/page.tsx

export async function generateStaticParams() {
  // すべての商品を取得
  const products = await prisma.product.findMany({
    where: { active: true },
    include: { category: true }
  });
  
  // カテゴリとIDの組み合わせを返す
  return products.map((product) => ({
    category: product.category.slug,
    id: product.id
  }));
}

// 返される値の例:
// [
//   { category: 'electronics', id: 'iphone-15' },
//   { category: 'electronics', id: 'macbook-pro' },
//   { category: 'clothing', id: 'tshirt-blue' }
// ]

export default async function ProductPage({ 
  params 
}: { 
  params: { category: string; id: string } 
}) {
  const product = await prisma.product.findFirst({
    where: {
      id: params.id,
      category: { slug: params.category }
    }
  });
  
  return <ProductDetail product={product} />;
}
```

### 段階的な生成（大規模サイト向け）
```typescript
// app/products/[id]/page.tsx

export async function generateStaticParams() {
  // 大量のデータがある場合、一部だけ事前生成
  const popularProducts = await prisma.product.findMany({
    where: {
      OR: [
        { featured: true },
        { views: { gte: 1000 } },
        { createdAt: { gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } }
      ]
    },
    take: 100, // 上位100件のみ事前生成
    select: { id: true }
  });
  
  return popularProducts.map((product) => ({
    id: product.id
  }));
}

// generateStaticParamsで返されなかったパスの扱い
export const dynamicParams = true; // デフォルト: true
// true: 事前生成されていないパスはリクエスト時に生成（ISR的な動作）
// false: 事前生成されていないパスは404

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await prisma.product.findUnique({
    where: { id: params.id }
  });
  
  if (!product) notFound();
  
  return <ProductDetail product={product} />;
}
```

## 4. 再検証（Revalidation）戦略

### 時間ベースの再検証
```typescript
// app/news/[id]/page.tsx

export async function generateStaticParams() {
  const articles = await prisma.article.findMany({
    where: { published: true },
    orderBy: { publishedAt: 'desc' },
    take: 50 // 最新50件を事前生成
  });
  
  return articles.map((article) => ({
    id: article.id
  }));
}

// 60秒ごとに再検証（ISR: Incremental Static Regeneration）
export const revalidate = 60;

export default async function NewsArticle({ params }: { params: { id: string } }) {
  const article = await prisma.article.findUnique({
    where: { id: params.id }
  });
  
  return <Article data={article} />;
}
```

### オンデマンド再検証
```typescript
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(request: NextRequest) {
  const { path, tag, secret } = await request.json();
  
  // セキュリティチェック
  if (secret !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 });
  }
  
  if (path) {
    // 特定のパスを再検証
    revalidatePath(`/blog/${path}`);
  }
  
  if (tag) {
    // タグベースの再検証
    revalidateTag(tag);
  }
  
  return NextResponse.json({ revalidated: true });
}

// 使用例：ブログ記事更新時
await fetch('https://yoursite.com/api/revalidate', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    path: 'hello-world',
    secret: process.env.REVALIDATE_SECRET
  })
});
```

## 5. パフォーマンス最適化

### 並列生成の制御
```typescript
// next.config.js
module.exports = {
  experimental: {
    // 同時に生成するページ数を制限（メモリ使用量を抑える）
    workerThreads: false,
    cpus: 4
  },
  
  // ビルド時の静的生成を無効化（必要に応じて）
  output: 'standalone'
};
```

### 条件付き生成
```typescript
export async function generateStaticParams() {
  // 環境変数で制御
  if (process.env.SKIP_BUILD_STATIC_GENERATION) {
    return [];
  }
  
  // 開発環境では少なめに生成
  const limit = process.env.NODE_ENV === 'development' ? 10 : 1000;
  
  const posts = await prisma.post.findMany({
    where: { published: true },
    take: limit,
    orderBy: { views: 'desc' }
  });
  
  return posts.map((post) => ({
    slug: post.slug
  }));
}
```

## 6. エラーハンドリング

```typescript
export async function generateStaticParams() {
  try {
    const posts = await prisma.post.findMany();
    return posts.map((post) => ({ slug: post.slug }));
  } catch (error) {
    console.error('Failed to generate static params:', error);
    // エラー時は空配列を返す（動的生成にフォールバック）
    return [];
  }
}

// カスタム404ページ
// app/blog/[slug]/not-found.tsx
export default function NotFound() {
  return (
    <div>
      <h2>記事が見つかりません</h2>
      <p>お探しの記事は存在しないか、削除された可能性があります。</p>
    </div>
  );
}
```

## 7. generateStaticParams vs getStaticPaths（Pages Router）

```typescript
// Pages Router（旧）
export async function getStaticPaths() {
  const posts = await getAllPosts();
  
  return {
    paths: posts.map((post) => ({
      params: { slug: post.slug }
    })),
    fallback: 'blocking' // or false, true
  };
}

// App Router（新）
export async function generateStaticParams() {
  const posts = await getAllPosts();
  
  return posts.map((post) => ({
    slug: post.slug
  }));
}

// fallbackの代わりにdynamicParamsを使用
export const dynamicParams = true; // fallback: true/blocking相当
// export const dynamicParams = false; // fallback: false相当
```

## 8. 実際のビルドログ例

```bash
$ npm run build

Route (app)                              Size     First Load JS
┌ ○ /                                    5.2 kB         92.1 kB
├ ● /blog/[slug]                         1.3 kB         88.2 kB
├   ├ /blog/hello-world
├   ├ /blog/next-js-guide
├   └ /blog/prisma-tutorial
├ ● /products/[category]/[id]            2.1 kB         89.0 kB
├   ├ /products/electronics/iphone-15
├   ├ /products/electronics/macbook-pro
├   └ [+10 more paths]
└ ○ /api/health                          0 B            0 B

○  (Static)   prerendered as static content
●  (SSG)      prerendered as static HTML (uses generateStaticParams)
```

## 💡 重要なポイント

| 概念 | 説明 | Rails相当 |
|------|------|-----------|
| generateStaticParams | ビルド時に動的ルートを静的生成 | 事前キャッシュ生成 |
| dynamicParams | 未生成パスの扱い | キャッシュミス時の動作 |
| revalidate | 再検証間隔 | キャッシュ有効期限 |
| notFound() | 404処理 | raise ActiveRecord::RecordNotFound |

**使用すべき場面：**
- ブログ、ニュースサイト
- 商品カタログ
- ドキュメントサイト
- 更新頻度が低いコンテンツ

**使用を避けるべき場面：**
- リアルタイムデータ
- ユーザー固有のコンテンツ
- 頻繁に更新されるデータ
- 数百万ページ規模のサイト（一部のみ生成推奨）