# Prisma $transaction 完全ガイド

## 📚 $transactionの2つのパターン

Prismaの`$transaction`には2つの使い方があります：

1. **バッチトランザクション** - 複数のPromiseを配列で渡す
2. **インタラクティブトランザクション** - コールバック関数で複雑な処理

## 1. バッチトランザクション（配列形式）

### 基本的な型定義

```typescript
// $transactionに渡せる型
type PrismaPromise<T> = Promise<T> & { [Symbol.toStringTag]: 'PrismaPromise' }

// バッチトランザクションのシグネチャ
$transaction<T extends readonly PrismaPromise<any>[]>(
  operations: T
): Promise<UnwrapTuple<T>>
```

### 実装例

```typescript
// ✅ 正しい使い方：PrismaPromiseの配列を渡す
const [user, profile, post] = await prisma.$transaction([
  prisma.user.create({
    data: { 
      email: "test@example.com",
      name: "Test User"
    }
  }),
  prisma.profile.create({
    data: {
      bio: "Hello World",
      userId: "cuid..." // 事前に生成したID
    }
  }),
  prisma.post.create({
    data: {
      title: "First Post",
      content: "Content",
      authorId: "cuid..." // 同じID
    }
  })
]);

// 型は自動的に推論される
// user: User
// profile: Profile  
// post: Post
```

### IDの依存関係がある場合の問題

```typescript
// ❌ これは動作しない（userIdがまだ存在しない）
const [user, post] = await prisma.$transaction([
  prisma.user.create({ data: { email: "test@example.com" } }),
  prisma.post.create({ 
    data: { 
      title: "Post",
      userId: user.id // エラー：userはまだ定義されていない
    }
  })
]);

// ✅ 解決策1：事前にIDを生成
import { cuid } from '@paralleldrive/cuid2';

const userId = cuid();
const [user, post] = await prisma.$transaction([
  prisma.user.create({ 
    data: { 
      id: userId, // IDを明示的に指定
      email: "test@example.com" 
    }
  }),
  prisma.post.create({ 
    data: { 
      title: "Post",
      userId: userId // 生成したIDを使用
    }
  })
]);

// ✅ 解決策2：ネストされたcreate
const user = await prisma.user.create({
  data: {
    email: "test@example.com",
    posts: {
      create: {
        title: "First Post"
      }
    }
  },
  include: {
    posts: true
  }
});
```

## 2. インタラクティブトランザクション（関数形式）

### 基本的な型定義

```typescript
// インタラクティブトランザクションのシグネチャ
$transaction<T>(
  fn: (prisma: Omit<PrismaClient, '$connect' | '$disconnect' | '$on' | '$transaction' | '$use'>) => Promise<T>,
  options?: {
    maxWait?: number // トランザクション開始までの最大待機時間（デフォルト: 2000ms）
    timeout?: number // トランザクションの最大実行時間（デフォルト: 5000ms）
    isolationLevel?: IsolationLevel // 分離レベル
  }
): Promise<T>
```

### 実装例

```typescript
// ✅ 複雑な依存関係がある処理
const result = await prisma.$transaction(async (tx) => {
  // 1. ユーザー作成
  const user = await tx.user.create({
    data: { email: "test@example.com" }
  });
  
  // 2. 作成したユーザーのIDを使用
  const profile = await tx.profile.create({
    data: {
      bio: "My bio",
      userId: user.id // 先に作成したuserのIDを使用可能
    }
  });
  
  // 3. 条件分岐も可能
  if (user.email.includes("premium")) {
    await tx.subscription.create({
      data: {
        userId: user.id,
        plan: "PREMIUM"
      }
    });
  }
  
  // 4. エラーハンドリング
  const existingEmail = await tx.user.findFirst({
    where: { email: "duplicate@example.com" }
  });
  
  if (existingEmail) {
    throw new Error("Email already exists"); // ロールバックされる
  }
  
  return { user, profile }; // 戻り値の型は自動推論
});
```

## 3. 分離レベル（Isolation Level）

### データベース別のデフォルト分離レベル

| データベース | デフォルト分離レベル | 特徴 |
|------------|-------------------|------|
| **MySQL (InnoDB)** | **REPEATABLE READ** | ファントムリード発生（ギャップロックで部分的に防ぐ） |
| **PostgreSQL** | **READ COMMITTED** | Webアプリに最適、REPEATABLE READでファントムリードも防ぐ |
| SQL Server | READ COMMITTED | 標準的な挙動 |
| Oracle | READ COMMITTED | 標準的な挙動 |

```sql
-- MySQLでの確認
SELECT @@transaction_isolation;
-- 結果: REPEATABLE-READ

-- PostgreSQLでの確認
SHOW default_transaction_isolation;
-- 結果: read committed
```

### 各分離レベルの詳細説明

```typescript
// Prismaで使用可能な分離レベル
enum IsolationLevel {
  ReadUncommitted = "ReadUncommitted",  // PostgreSQLでは実質ReadCommitted
  ReadCommitted = "ReadCommitted",      // PostgreSQLのデフォルト
  RepeatableRead = "RepeatableRead",    // MySQLのデフォルト
  Serializable = "Serializable"         // 最も厳密
}
```

#### 1. ReadUncommitted（読み取り未コミット）
```typescript
// PostgreSQLでは実際にはReadCommittedとして動作
await prisma.$transaction(async (tx) => {
  // 他のトランザクションの未コミットデータは読めない（PostgreSQL）
  const data = await tx.user.findMany();
}, {
  isolationLevel: IsolationLevel.ReadUncommitted
});
```

**特徴：**
- ダーティリード：❌ PostgreSQLでは発生しない（MySQLでは発生）
- ノンリピータブルリード：✅ 発生する可能性あり
- ファントムリード：✅ 発生する可能性あり
- パフォーマンス：最も高速
- 使用場面：PostgreSQLでは意味がない（ReadCommittedと同じ）

#### 2. ReadCommitted（読み取りコミット済み）- デフォルト
```typescript
// デフォルトの分離レベル
await prisma.$transaction(async (tx) => {
  // 1回目の読み取り
  const user1 = await tx.user.findUnique({ where: { id: 1 } });
  console.log(user1.balance); // 1000
  
  // この間に他のトランザクションがbalanceを2000に更新してコミット
  
  // 2回目の読み取り（異なる値になる可能性）
  const user2 = await tx.user.findUnique({ where: { id: 1 } });
  console.log(user2.balance); // 2000（変わっている！）
}, {
  isolationLevel: IsolationLevel.ReadCommitted // 省略可（デフォルト）
});
```

**具体的なシナリオ例：ECサイトの在庫表示**

```typescript
// Transaction A: ユーザーAが商品ページを見ている
await prisma.$transaction(async (tx) => {
  // 10:00:00 - 在庫確認
  const product = await tx.product.findUnique({ 
    where: { id: 'iphone' } 
  });
  console.log(`在庫: ${product.stock}`); // 在庫: 10
  
  // 10:00:01 - ユーザーが商品説明を読んでいる間...
  await new Promise(resolve => setTimeout(resolve, 1000));
  
  // 10:00:02 - カートに追加する前に再度在庫確認
  const productAgain = await tx.product.findUnique({ 
    where: { id: 'iphone' } 
  });
  console.log(`在庫: ${productAgain.stock}`); // 在庫: 5（他の人が購入して減った！）
  
  // 最新の在庫情報を元に判断できる
  if (productAgain.stock > 0) {
    await tx.cart.create({
      data: { productId: 'iphone', quantity: 1 }
    });
  }
});

// Transaction B: 同時にユーザーBが購入（10:00:01に実行）
await prisma.$transaction(async (tx) => {
  await tx.product.update({
    where: { id: 'iphone' },
    data: { stock: { decrement: 5 } }
  });
  // このトランザクションがコミットされると、
  // Transaction Aの2回目の読み取りで新しい値が見える
});
```

**なぜ一般的なWebアプリに適しているのか？**

```typescript
// ✅ 良い例：ソーシャルメディアのタイムライン
await prisma.$transaction(async (tx) => {
  // 最初のページロード
  const posts = await tx.post.findMany({
    take: 20,
    orderBy: { createdAt: 'desc' }
  });
  
  // ユーザーの既読マーク
  await tx.userReadHistory.create({
    data: { 
      userId: currentUserId,
      lastReadAt: new Date()
    }
  });
  
  // この間に新しい投稿があってもOK（リアルタイム性が重要）
  const notifications = await tx.notification.findMany({
    where: { userId: currentUserId }
  });
  // 最新の通知が取得できる（良いUX）
});

// ✅ 良い例：ショッピングカートの更新
await prisma.$transaction(async (tx) => {
  // カート内容取得
  const cartItems = await tx.cartItem.findMany({
    where: { userId: currentUserId }
  });
  
  // 価格計算（この間に商品価格が変更されてもOK）
  let total = 0;
  for (const item of cartItems) {
    const product = await tx.product.findUnique({
      where: { id: item.productId }
    });
    // 最新の価格を使用（セール開始などに対応）
    total += product.price * item.quantity;
  }
  
  return { items: cartItems, total };
});

// ❌ 悪い例：このケースではReadCommittedは不適切
// 金融取引では RepeatableRead か Serializable を使うべき
await prisma.$transaction(async (tx) => {
  // 残高確認
  const account1 = await tx.account.findUnique({ 
    where: { id: 'acc1' } 
  });
  
  if (account1.balance >= 10000) {
    // この判断の後で他のトランザクションが引き落とし...
    
    // 投資商品を購入（残高不足になる可能性！）
    await tx.investment.create({
      data: { amount: 10000, accountId: 'acc1' }
    });
  }
}, {
  isolationLevel: IsolationLevel.Serializable // 金融取引はこちらを使う
});
```

**ReadCommittedのメリット・デメリット**

```typescript
// メリット1: パフォーマンスが良い
// 他のトランザクションをブロックしにくい
await prisma.$transaction(async (tx) => {
  // 大量のデータを処理してもデッドロックが起きにくい
  const users = await tx.user.findMany();
  
  for (const user of users) {
    // 長時間の処理でも他のトランザクションを妨げない
    await processUser(user);
    
    // 処理中に追加された新規ユーザーも見える
    const newUsers = await tx.user.findMany({
      where: { createdAt: { gt: user.createdAt } }
    });
  }
});

// メリット2: リアルタイム性が高い
await prisma.$transaction(async (tx) => {
  // チャットメッセージの取得
  const messages = await tx.message.findMany({
    where: { roomId }
  });
  
  // 処理中に届いた新着メッセージも取得可能
  await new Promise(r => setTimeout(r, 100));
  
  const newMessages = await tx.message.findMany({
    where: { 
      roomId,
      createdAt: { gt: messages[messages.length - 1]?.createdAt }
    }
  });
  // 最新のメッセージが見える
});

// デメリット: 読み取り一貫性がない
await prisma.$transaction(async (tx) => {
  // レポート生成には不向き
  const count1 = await tx.order.count({
    where: { status: 'COMPLETED' }
  });
  
  // 集計処理...
  
  const count2 = await tx.order.count({
    where: { status: 'COMPLETED' }
  });
  
  // count1 !== count2 の可能性がある（一貫性がない）
  // レポートには RepeatableRead を使うべき
});
```

**特徴：**
- ダーティリード：❌ 発生しない（コミットされていないデータは読めない）
- ノンリピータブルリード：✅ 発生する可能性あり（同じデータを2回読むと違う値になりうる）
- ファントムリード：✅ 発生する可能性あり（新しいレコードが途中で見える）
- パフォーマンス：良好（ロックが少ない）
- 使用場面：一般的なWebアプリケーション、リアルタイム性重視、読み取り一貫性が不要な処理

#### 3. RepeatableRead（反復可能読み取り）- MySQLのデフォルト
```typescript
await prisma.$transaction(async (tx) => {
  // 1回目の読み取り
  const user1 = await tx.user.findUnique({ where: { id: 1 } });
  console.log(user1.balance); // 1000
  
  // この間に他のトランザクションがbalanceを更新してコミット
  
  // 2回目の読み取り（同じ値が保証される）
  const user2 = await tx.user.findUnique({ where: { id: 1 } });
  console.log(user2.balance); // 1000（変わらない！）
  
  // ファントムリードの挙動（DB依存）
  const count1 = await tx.user.count(); // 10件
  // 他のトランザクションが新規ユーザーを追加
  const count2 = await tx.user.count(); 
  // MySQL: 11件（ファントムリード発生！）
  // PostgreSQL: 10件（ファントムリード防止）
}, {
  isolationLevel: IsolationLevel.RepeatableRead
});
```

**特徴：**
- ダーティリード：❌ 発生しない
- ノンリピータブルリード：❌ 発生しない
- ファントムリード：
  - **MySQL**: ✅ 発生する（ギャップロックで部分的に防ぐ）
  - **PostgreSQL**: ❌ 発生しない（より厳密な実装）
- パフォーマンス：やや低下
- 使用場面：レポート生成、バッチ処理、読み取り一貫性が必要な処理

**⚠️ MySQL使用時の注意点：**
```typescript
// MySQLのREPEATABLE READはデッドロックが起きやすい
// Webアプリケーションでは READ COMMITTED への変更を検討

// MySQL設定変更（my.cnf）
// [mysqld]
// transaction-isolation = READ-COMMITTED

// またはPrismaで明示的に指定
await prisma.$transaction(async (tx) => {
  // 処理
}, {
  isolationLevel: IsolationLevel.ReadCommitted // MySQLでも明示的にREAD COMMITTEDを使用
});
```

#### 4. Serializable（直列化可能）
```typescript
// 最も厳密な分離レベル
await prisma.$transaction(async (tx) => {
  // 在庫確認
  const product = await tx.product.findUnique({
    where: { id: productId }
  });
  
  if (product.stock < quantity) {
    throw new Error("在庫不足");
  }
  
  // 在庫更新（他のトランザクションと競合した場合はエラー）
  const updated = await tx.product.update({
    where: { id: productId },
    data: { stock: { decrement: quantity } }
  });
  
  // 注文作成
  const order = await tx.order.create({
    data: { productId, quantity, userId }
  });
  
  return { order, updated };
}, {
  isolationLevel: IsolationLevel.Serializable,
  timeout: 10000,
  maxWait: 5000
});
```

**特徴：**
- ダーティリード：❌ 発生しない
- ノンリピータブルリード：❌ 発生しない
- ファントムリード：❌ 発生しない
- パフォーマンス：最も低い（シリアライゼーションエラーの可能性）
- 使用場面：金融取引、在庫管理、クリティカルなデータ整合性が必要な処理

### 実践例：各分離レベルの使い分け

```typescript
// 1. ReadCommitted：一般的なCRUD操作
async function updateUserProfile(userId: string, data: any) {
  return await prisma.$transaction(async (tx) => {
    const user = await tx.user.update({
      where: { id: userId },
      data: data
    });
    
    await tx.activityLog.create({
      data: { userId, action: 'PROFILE_UPDATE' }
    });
    
    return user;
  }); // デフォルトのReadCommitted
}

// 2. RepeatableRead：集計・レポート
async function generateMonthlyReport(month: Date) {
  return await prisma.$transaction(async (tx) => {
    // 同じデータを複数回読む必要がある
    const orders = await tx.order.findMany({
      where: { createdAt: { gte: month } }
    });
    
    const totalRevenue = orders.reduce((sum, o) => sum + o.amount, 0);
    
    // 再度同じ条件で読んでも結果が変わらないことが保証される
    const verifyOrders = await tx.order.findMany({
      where: { createdAt: { gte: month } }
    });
    
    return {
      orderCount: orders.length,
      totalRevenue,
      verified: orders.length === verifyOrders.length
    };
  }, {
    isolationLevel: IsolationLevel.RepeatableRead
  });
}

// 3. Serializable：在庫管理・決済処理
async function processPayment(orderId: string, amount: number) {
  const maxRetries = 3;
  let attempt = 0;
  
  while (attempt < maxRetries) {
    try {
      return await prisma.$transaction(async (tx) => {
        // 残高確認
        const account = await tx.account.findUnique({
          where: { userId }
        });
        
        if (account.balance < amount) {
          throw new Error("残高不足");
        }
        
        // 残高更新
        await tx.account.update({
          where: { userId },
          data: { balance: { decrement: amount } }
        });
        
        // 支払い記録
        await tx.payment.create({
          data: { orderId, amount, status: 'COMPLETED' }
        });
        
        return { success: true };
      }, {
        isolationLevel: IsolationLevel.Serializable
      });
    } catch (error) {
      if (error.code === 'P2034') { // シリアライゼーションエラー
        attempt++;
        await new Promise(r => setTimeout(r, 100 * attempt)); // リトライ待機
      } else {
        throw error;
      }
    }
  }
}
```

### 分離レベル比較表

| 分離レベル | ダーティリード | ノンリピータブルリード | ファントムリード | パフォーマンス | 使用場面 |
|-----------|--------------|---------------------|----------------|-------------|---------|
| ReadUncommitted | ❌ (PG) / ✅ (MySQL) | ✅ | ✅ | 最高 | PostgreSQLでは使用しない |
| ReadCommitted<br>**PGデフォルト** | ❌ | ✅ | ✅ | 高 | 一般的なWebアプリ |
| RepeatableRead<br>**MySQLデフォルト** | ❌ | ❌ | ❌ (PG) / ✅ (MySQL) | 中 | レポート、バッチ処理 |
| Serializable | ❌ | ❌ | ❌ | 低 | 金融取引、在庫管理 |

### MySQLからPostgreSQLへの移行時の注意点

```typescript
// MySQL（デフォルト: REPEATABLE READ）
// → PostgreSQL（デフォルト: READ COMMITTED）への移行

// 1. MySQLで動作していたコード
await prisma.$transaction(async (tx) => {
  const order = await tx.order.findFirst();
  // MySQLではこの時点のスナップショットが固定される
  
  await new Promise(r => setTimeout(r, 1000));
  
  const orderAgain = await tx.order.findFirst();
  // MySQL: 最初と同じ結果（REPEATABLE READ）
  // PostgreSQL: 最新の結果（READ COMMITTED）
});

// 2. 移行時の対応方法

// 方法A: PostgreSQLでも明示的にREPEATABLE READを指定
await prisma.$transaction(async (tx) => {
  // MySQLと同じ挙動を維持
}, {
  isolationLevel: IsolationLevel.RepeatableRead
});

// 方法B: アプリケーションロジックをREAD COMMITTEDに最適化
// （推奨：パフォーマンスが良い）
```

### 使用例
const result = await prisma.$transaction(
  async (tx) => {
    // 在庫確認と更新（同時実行での不整合を防ぐ）
    const product = await tx.product.findUnique({
      where: { id: productId }
    });
    
    if (product.stock < quantity) {
      throw new Error("在庫不足");
    }
    
    const updatedProduct = await tx.product.update({
      where: { id: productId },
      data: { stock: { decrement: quantity } }
    });
    
    const order = await tx.order.create({
      data: {
        productId,
        quantity,
        userId
      }
    });
    
    return { order, updatedProduct };
  },
  {
    isolationLevel: IsolationLevel.Serializable, // 最も安全
    timeout: 10000 // 10秒でタイムアウト
  }
);
```

## 4. 実践的な使用例

### 銀行振込のトランザクション

```typescript
async function transferMoney(
  fromAccountId: string,
  toAccountId: string,
  amount: number
) {
  return await prisma.$transaction(async (tx) => {
    // 1. 送金元の残高確認
    const fromAccount = await tx.account.findUnique({
      where: { id: fromAccountId }
    });
    
    if (!fromAccount || fromAccount.balance < amount) {
      throw new Error("残高不足");
    }
    
    // 2. 送金元から引き落とし
    const updatedFrom = await tx.account.update({
      where: { id: fromAccountId },
      data: { balance: { decrement: amount } }
    });
    
    // 3. 送金先に入金
    const updatedTo = await tx.account.update({
      where: { id: toAccountId },
      data: { balance: { increment: amount } }
    });
    
    // 4. 取引履歴を記録
    const transaction = await tx.transaction.create({
      data: {
        fromAccountId,
        toAccountId,
        amount,
        type: 'TRANSFER',
        status: 'COMPLETED'
      }
    });
    
    // 5. 通知を作成
    await tx.notification.createMany({
      data: [
        {
          userId: fromAccount.userId,
          message: `${amount}円を送金しました`
        },
        {
          userId: updatedTo.userId,
          message: `${amount}円を受け取りました`
        }
      ]
    });
    
    return {
      transaction,
      fromBalance: updatedFrom.balance,
      toBalance: updatedTo.balance
    };
  }, {
    isolationLevel: IsolationLevel.Serializable,
    timeout: 15000
  });
}
```

### 一括インポート処理

```typescript
async function bulkImportUsers(userData: any[]) {
  return await prisma.$transaction(async (tx) => {
    const results = {
      created: 0,
      updated: 0,
      failed: [] as string[]
    };
    
    for (const data of userData) {
      try {
        // upsert: 存在しなければ作成、存在すれば更新
        await tx.user.upsert({
          where: { email: data.email },
          create: {
            email: data.email,
            name: data.name,
            profile: {
              create: {
                bio: data.bio
              }
            }
          },
          update: {
            name: data.name,
            profile: {
              update: {
                bio: data.bio
              }
            }
          }
        });
        
        // 統計を更新
        const existing = await tx.user.findUnique({
          where: { email: data.email }
        });
        
        if (existing) {
          results.updated++;
        } else {
          results.created++;
        }
      } catch (error) {
        results.failed.push(data.email);
      }
    }
    
    // インポートログを記録
    await tx.importLog.create({
      data: {
        totalRecords: userData.length,
        created: results.created,
        updated: results.updated,
        failed: results.failed.length,
        failedEmails: results.failed
      }
    });
    
    return results;
  });
}
```

## 5. エラーハンドリング

```typescript
try {
  const result = await prisma.$transaction(async (tx) => {
    // トランザクション処理
    const user = await tx.user.create({
      data: { email: "test@example.com" }
    });
    
    // 意図的にエラーを発生させる
    if (user.email.includes("test")) {
      throw new Error("テストユーザーは作成できません");
    }
    
    return user;
  });
} catch (error) {
  if (error instanceof Prisma.PrismaClientKnownRequestError) {
    // Prismaの既知のエラー
    if (error.code === 'P2002') {
      console.error('ユニーク制約違反');
    }
  } else if (error instanceof Error) {
    // カスタムエラー
    console.error('トランザクション失敗:', error.message);
  }
  
  // 全ての変更は自動的にロールバックされる
}
```

## 6. パフォーマンス考慮事項

```typescript
// ❌ 非効率：個別のクエリ
for (const item of items) {
  await prisma.item.create({ data: item });
}

// ✅ 効率的：バッチトランザクション
await prisma.$transaction(
  items.map(item => 
    prisma.item.create({ data: item })
  )
);

// ✅ さらに効率的：createMany（ただしトランザクション保証なし）
await prisma.item.createMany({
  data: items,
  skipDuplicates: true
});

// ✅ トランザクション + バルク操作
await prisma.$transaction(async (tx) => {
  // 既存データを削除
  await tx.item.deleteMany({
    where: { categoryId }
  });
  
  // 新しいデータを一括作成
  await tx.item.createMany({
    data: items
  });
});
```

## 🎯 Rails ActiveRecordとの比較

| ActiveRecord | Prisma |
|--------------|--------|
| `ActiveRecord::Base.transaction do ... end` | `prisma.$transaction(async (tx) => { ... })` |
| `User.transaction do ... end` | `prisma.$transaction([...])` |
| `raise ActiveRecord::Rollback` | `throw new Error()` |
| `requires_new: true` | 新しい`$transaction`呼び出し |
| `isolation: :serializable` | `isolationLevel: 'Serializable'` |

## 💡 重要なポイント

1. **バッチ vs インタラクティブ**
   - 単純な並列処理 → バッチ（配列）
   - 依存関係・条件分岐あり → インタラクティブ（関数）

2. **型安全性**
   - 全ての操作は型推論される
   - 戻り値の型も自動推論

3. **自動ロールバック**
   - エラー発生時は全て自動ロールバック
   - 明示的なコミット不要

4. **ネスト不可**
   - `$transaction`内で別の`$transaction`は使用不可
   - 代わりに同じtxオブジェクトを使用