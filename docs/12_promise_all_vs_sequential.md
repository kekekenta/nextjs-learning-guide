# Promise.all vs 順次実行 - 詳細比較ガイド

## 📚 なぜPromise.allを使うのか？

データベースクエリを並列実行することで、レスポンス時間を大幅に短縮できます。

## 1. 基本的な違い

### 順次実行（遅い）
```typescript
// ❌ 非効率的：クエリが順番に実行される
export async function GET(request: NextRequest) {
  // 1つ目のクエリ: 200ms
  const users = await prisma.user.findMany({
    skip: 0,
    take: 10,
    include: { posts: true }
  });
  
  // 1つ目が完了してから2つ目を実行: 100ms
  const total = await prisma.user.count();
  
  // 合計時間: 200ms + 100ms = 300ms
  return NextResponse.json({ users, total });
}
```

### 並列実行（速い）
```typescript
// ✅ 効率的：クエリが同時に実行される
export async function GET(request: NextRequest) {
  // 両方のクエリを同時に開始
  const [users, total] = await Promise.all([
    prisma.user.findMany({
      skip: 0,
      take: 10,
      include: { posts: true }
    }), // 200ms
    prisma.user.count() // 100ms
  ]);
  
  // 合計時間: max(200ms, 100ms) = 200ms
  return NextResponse.json({ users, total });
}
```

## 2. 実際のパフォーマンス比較

### 測定コード
```typescript
// lib/performance-test.ts
export async function measurePerformance() {
  // 順次実行のテスト
  const sequentialStart = Date.now();
  
  const users1 = await prisma.user.findMany({ take: 100 });
  const posts1 = await prisma.post.findMany({ take: 100 });
  const comments1 = await prisma.comment.findMany({ take: 100 });
  
  const sequentialTime = Date.now() - sequentialStart;
  
  // 並列実行のテスト
  const parallelStart = Date.now();
  
  const [users2, posts2, comments2] = await Promise.all([
    prisma.user.findMany({ take: 100 }),
    prisma.post.findMany({ take: 100 }),
    prisma.comment.findMany({ take: 100 })
  ]);
  
  const parallelTime = Date.now() - parallelStart;
  
  console.log(`順次実行: ${sequentialTime}ms`);
  console.log(`並列実行: ${parallelTime}ms`);
  console.log(`速度向上: ${Math.round((sequentialTime / parallelTime - 1) * 100)}%`);
  
  // 典型的な結果:
  // 順次実行: 600ms (200ms + 200ms + 200ms)
  // 並列実行: 220ms (最も遅いクエリの時間)
  // 速度向上: 173%
}
```

## 3. Promise.allを使うべき場面

### ✅ 使うべき場合

#### 1. 独立したデータ取得
```typescript
// ダッシュボードデータの取得
export async function getDashboardData(userId: string) {
  const [
    userStats,
    recentOrders,
    notifications,
    recommendations
  ] = await Promise.all([
    getUserStatistics(userId),      // 150ms
    getRecentOrders(userId),         // 200ms
    getUnreadNotifications(userId),  // 100ms
    getRecommendations(userId)       // 300ms
  ]);
  
  // 合計時間: 300ms（最も遅いクエリ）
  // 順次実行なら: 750ms
  
  return {
    userStats,
    recentOrders,
    notifications,
    recommendations
  };
}
```

#### 2. ページネーション
```typescript
// ページネーションでデータと総数を取得
export async function getPaginatedData(page: number, limit: number) {
  const [items, totalCount] = await Promise.all([
    prisma.product.findMany({
      skip: (page - 1) * limit,
      take: limit,
      include: {
        category: true,
        reviews: { take: 5 }
      }
    }),
    prisma.product.count()
  ]);
  
  return {
    items,
    pagination: {
      page,
      limit,
      total: totalCount,
      pages: Math.ceil(totalCount / limit)
    }
  };
}
```

#### 3. 複数リソースの集計
```typescript
// 管理画面の統計情報
export async function getAdminStats() {
  const [
    userCount,
    orderCount,
    revenue,
    activeUsers,
    pendingOrders
  ] = await Promise.all([
    prisma.user.count(),
    prisma.order.count(),
    prisma.order.aggregate({
      _sum: { amount: true }
    }),
    prisma.user.count({
      where: {
        lastActiveAt: {
          gte: new Date(Date.now() - 24 * 60 * 60 * 1000)
        }
      }
    }),
    prisma.order.count({
      where: { status: 'PENDING' }
    })
  ]);
  
  return {
    userCount,
    orderCount,
    totalRevenue: revenue._sum.amount || 0,
    activeUsers,
    pendingOrders
  };
}
```

### ❌ 使うべきでない場合

#### 1. 依存関係がある場合
```typescript
// ❌ 悪い例：2つ目のクエリが1つ目の結果に依存
const [user, posts] = await Promise.all([
  prisma.user.findUnique({ where: { id: userId } }),
  prisma.post.findMany({ 
    where: { authorId: user.id } // エラー！userはまだ存在しない
  })
]);

// ✅ 正しい例：順次実行が必要
const user = await prisma.user.findUnique({ 
  where: { id: userId } 
});

if (!user) {
  throw new Error('User not found');
}

const posts = await prisma.post.findMany({ 
  where: { authorId: user.id }
});
```

#### 2. 条件分岐がある場合
```typescript
// ❌ 不必要なクエリを実行してしまう
const [userData, premiumData] = await Promise.all([
  getUserData(userId),
  getPremiumContent(userId) // ユーザーがプレミアムでなくても実行される
]);

// ✅ 条件に応じて実行
const userData = await getUserData(userId);

let premiumData = null;
if (userData.isPremium) {
  premiumData = await getPremiumContent(userId);
}
```

#### 3. トランザクション内
```typescript
// ❌ トランザクション内では意味がない（順次実行される）
await prisma.$transaction(async (tx) => {
  const [user, profile] = await Promise.all([
    tx.user.create({ data: userData }),
    tx.profile.create({ data: profileData })
  ]);
  // Prismaは内部で順次実行する
});

// ✅ トランザクションは素直に書く
await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({ data: userData });
  const profile = await tx.profile.create({ 
    data: { ...profileData, userId: user.id }
  });
});
```

## 4. エラーハンドリング

### Promise.allのエラー処理
```typescript
// ⚠️ 1つでも失敗すると全体が失敗
try {
  const [users, posts, comments] = await Promise.all([
    prisma.user.findMany(),
    prisma.post.findMany(),
    prisma.comment.findMany() // これが失敗すると...
  ]);
} catch (error) {
  // 全体がエラーになる（usersとpostsも取得できない）
}

// ✅ Promise.allSettledで個別にハンドリング
const results = await Promise.allSettled([
  prisma.user.findMany(),
  prisma.post.findMany(),
  prisma.comment.findMany()
]);

const users = results[0].status === 'fulfilled' ? results[0].value : [];
const posts = results[1].status === 'fulfilled' ? results[1].value : [];
const comments = results[2].status === 'fulfilled' ? results[2].value : [];

// または個別にデフォルト値を設定
const [usersResult, postsResult, commentsResult] = results;

return {
  users: usersResult.status === 'fulfilled' ? usersResult.value : [],
  posts: postsResult.status === 'fulfilled' ? postsResult.value : [],
  comments: commentsResult.status === 'fulfilled' ? commentsResult.value : [],
  errors: results
    .filter(r => r.status === 'rejected')
    .map(r => r.reason)
};
```

## 5. 実践的なパターン

### バッチ処理の最適化
```typescript
// 大量のユーザーに通知を送る
async function sendBulkNotifications(userIds: string[], message: string) {
  // ❌ 非効率：1つずつ処理
  for (const userId of userIds) {
    await createNotification(userId, message);
  }
  // 1000ユーザー × 50ms = 50秒
  
  // ✅ バッチで並列処理
  const batchSize = 10;
  for (let i = 0; i < userIds.length; i += batchSize) {
    const batch = userIds.slice(i, i + batchSize);
    await Promise.all(
      batch.map(userId => createNotification(userId, message))
    );
  }
  // (1000ユーザー / 10) × 50ms = 5秒
}
```

### APIコールの並列化
```typescript
// 複数の外部APIを呼び出す
async function aggregateExternalData(productId: string) {
  const [
    inventoryData,
    pricingData,
    reviewsData,
    shippingData
  ] = await Promise.all([
    fetchInventoryAPI(productId),
    fetchPricingAPI(productId),
    fetchReviewsAPI(productId),
    fetchShippingAPI(productId)
  ]);
  
  return {
    stock: inventoryData.quantity,
    price: pricingData.currentPrice,
    rating: reviewsData.averageRating,
    deliveryTime: shippingData.estimatedDays
  };
}
```

## 6. パフォーマンステクニック

### 選択的な並列化
```typescript
export async function getProductDetails(productId: string, options: {
  includeReviews?: boolean;
  includeRelated?: boolean;
  includeInventory?: boolean;
}) {
  // 必須データ
  const product = await prisma.product.findUnique({
    where: { id: productId }
  });
  
  if (!product) throw new Error('Product not found');
  
  // オプショナルデータを並列取得
  const promises: Promise<any>[] = [];
  const keys: string[] = [];
  
  if (options.includeReviews) {
    promises.push(prisma.review.findMany({ 
      where: { productId },
      take: 10 
    }));
    keys.push('reviews');
  }
  
  if (options.includeRelated) {
    promises.push(prisma.product.findMany({
      where: { 
        categoryId: product.categoryId,
        id: { not: productId }
      },
      take: 5
    }));
    keys.push('related');
  }
  
  if (options.includeInventory) {
    promises.push(getInventoryStatus(productId));
    keys.push('inventory');
  }
  
  const results = await Promise.all(promises);
  
  // 結果をオブジェクトにマップ
  const additionalData = keys.reduce((acc, key, index) => {
    acc[key] = results[index];
    return acc;
  }, {} as any);
  
  return {
    ...product,
    ...additionalData
  };
}
```

## 💡 まとめ

| 方法 | 使用場面 | パフォーマンス | 例 |
|------|---------|--------------|-----|
| Promise.all | 独立したクエリ | 最速 | ページネーション、ダッシュボード |
| 順次実行 | 依存関係あり | 遅い | 認証→データ取得 |
| Promise.allSettled | エラー許容 | 速い | 外部API集約 |
| バッチPromise.all | 大量処理 | 効率的 | 一括通知送信 |

**重要な原則：**
- 独立したデータは並列で取得
- 依存関係があれば順次実行
- エラーハンドリングを考慮
- データベース接続プールの制限に注意