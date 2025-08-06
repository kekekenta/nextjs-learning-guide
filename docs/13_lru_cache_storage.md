# LRUCache の保存場所と仕組み

## 📚 LRUCacheはどこに保存される？

**答え：メモリ（RAM）内に保存されます**

## 1. LRUCacheの基本的な仕組み

```typescript
import { LRUCache } from 'lru-cache';

// このキャッシュはNode.jsプロセスのメモリ内に作られる
const tokenCache = new LRUCache({
  max: 500,      // 最大500個のアイテムを保存
  ttl: 60000,    // 60秒後に自動削除
});

// データの保存（メモリ内）
tokenCache.set('user-123', { requestCount: 1 });

// データの取得（メモリから）
const data = tokenCache.get('user-123');
```

## 2. メモリ内保存の特徴

### メリット
```typescript
// ✅ 超高速アクセス（マイクロ秒単位）
const cache = new LRUCache({ max: 1000 });

// メモリアクセス: 0.001ms
cache.set('key', 'value');
const value = cache.get('key');

// 比較：データベースアクセス: 10-50ms
const dbValue = await prisma.cache.findUnique({
  where: { key: 'key' }
});
```

### デメリット
```typescript
// ❌ サーバー再起動で消える
const cache = new LRUCache({ max: 500 });
cache.set('important-data', { value: 123 });

// サーバー再起動 or クラッシュ
process.exit(0);

// 再起動後：データは失われている
const value = cache.get('important-data'); // undefined
```

## 3. Next.js環境での注意点

### 開発環境（dev）
```typescript
// ⚠️ 開発環境ではホットリロードでキャッシュがリセットされる
export function rateLimit() {
  // ファイルを保存するたびに新しいインスタンスが作られる
  const tokenCache = new LRUCache({
    max: 500,
    ttl: 60000,
  });
  
  // 結果：レート制限が機能しない可能性
  return tokenCache;
}
```

### 本番環境（production）
```typescript
// ✅ 本番環境：グローバルに定義して永続化
let globalCache: LRUCache<string, number[]> | undefined;

export function getRateLimitCache() {
  if (!globalCache) {
    globalCache = new LRUCache<string, number[]>({
      max: 500,
      ttl: 60000,
    });
  }
  return globalCache;
}

// 使用
const cache = getRateLimitCache();
cache.set('ip-192.168.1.1', [1, 2, 3]);
```

### Vercelなどのサーバーレス環境
```typescript
// ⚠️ サーバーレス環境では各リクエストで新しいインスタンス
// → LRUCacheは機能しない

// ❌ これは動作しない
const cache = new LRUCache({ max: 500 });

export async function middleware(request: NextRequest) {
  const ip = request.ip;
  const count = cache.get(ip) || 0; // 常に0になる
  cache.set(ip, count + 1);
}

// ✅ 解決策：外部ストレージを使用
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

export async function middleware(request: NextRequest) {
  const ip = request.ip;
  const count = await redis.incr(`rate-limit:${ip}`);
  await redis.expire(`rate-limit:${ip}`, 60);
}
```

## 4. 永続化が必要な場合の代替手段

### 1. Redis（推奨）
```typescript
// Redisを使った永続的なレート制限
import Redis from 'ioredis';

class RedisRateLimit {
  private redis: Redis;
  
  constructor(redisUrl: string) {
    this.redis = new Redis(redisUrl);
  }
  
  async check(key: string, limit: number, windowMs: number) {
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      // 初回アクセス時にTTLを設定
      await this.redis.pexpire(key, windowMs);
    }
    
    return {
      success: current <= limit,
      remaining: Math.max(0, limit - current),
      reset: Date.now() + windowMs
    };
  }
}

// 使用例
const rateLimit = new RedisRateLimit(process.env.REDIS_URL!);

export async function middleware(request: NextRequest) {
  const { success, remaining } = await rateLimit.check(
    `rate:${request.ip}`,
    100,  // 100リクエスト
    60000 // 1分間
  );
  
  if (!success) {
    return NextResponse.json(
      { error: 'Too many requests' },
      { 
        status: 429,
        headers: { 'X-RateLimit-Remaining': remaining.toString() }
      }
    );
  }
}
```

### 2. データベース（Prisma）
```typescript
// データベースを使ったレート制限
model RateLimit {
  id        String   @id @default(cuid())
  key       String   @unique
  count     Int      @default(1)
  window    DateTime
  createdAt DateTime @default(now())
  
  @@index([window])
}

export async function checkRateLimit(key: string, limit: number) {
  const windowStart = new Date(Date.now() - 60000); // 1分前
  
  // 既存のレート制限を確認
  const existing = await prisma.rateLimit.findFirst({
    where: {
      key,
      window: { gte: windowStart }
    }
  });
  
  if (existing) {
    if (existing.count >= limit) {
      return { success: false, remaining: 0 };
    }
    
    await prisma.rateLimit.update({
      where: { id: existing.id },
      data: { count: { increment: 1 } }
    });
    
    return { 
      success: true, 
      remaining: limit - existing.count - 1 
    };
  }
  
  // 新規作成
  await prisma.rateLimit.create({
    data: {
      key,
      window: new Date()
    }
  });
  
  // 古いレコードを削除
  await prisma.rateLimit.deleteMany({
    where: {
      window: { lt: windowStart }
    }
  });
  
  return { success: true, remaining: limit - 1 };
}
```

### 3. Upstash（サーバーレス向けRedis）
```typescript
// Upstash Redis（Vercel推奨）
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export async function rateLimit(identifier: string) {
  const limit = 10;
  const window = 60; // 秒
  
  const key = `rate_limit:${identifier}`;
  const current = await redis.incr(key);
  
  if (current === 1) {
    await redis.expire(key, window);
  }
  
  return {
    success: current <= limit,
    remaining: Math.max(0, limit - current)
  };
}
```

## 5. 環境別の推奨構成

### 開発環境
```typescript
// development: メモリキャッシュで十分
const cache = new LRUCache({ 
  max: 100, 
  ttl: 60000 
});
```

### 単一サーバー（EC2、ECS）
```typescript
// production (single server): LRUCacheが使える
const cache = global.rateLimitCache || new LRUCache({
  max: 10000,
  ttl: 60000
});

if (!global.rateLimitCache) {
  global.rateLimitCache = cache;
}
```

### マルチサーバー / サーバーレス
```typescript
// production (serverless/multi-server): Redis必須
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  password: process.env.REDIS_PASSWORD,
});
```

## 6. メモリ使用量の計算

```typescript
// LRUCacheのメモリ使用量を見積もる
const cache = new LRUCache({
  max: 1000, // 最大1000アイテム
  maxSize: 5000, // 最大5000バイト
  sizeCalculation: (value, key) => {
    // 各アイテムのサイズを計算
    return JSON.stringify(value).length + key.length;
  },
});

// 見積もり計算
// 1アイテム = 約100バイト
// 1000アイテム = 約100KB
// 10,000アイテム = 約1MB
// 100,000アイテム = 約10MB
```

## 💡 まとめ

| 保存場所 | 永続性 | 速度 | スケーラビリティ | 使用場面 |
|---------|--------|------|----------------|----------|
| LRUCache（メモリ） | ❌ | ⚡超高速 | ❌ | 開発環境、単一サーバー |
| Redis | ✅ | 🚀高速 | ✅ | 本番環境、マルチサーバー |
| Database | ✅ | 🐢遅い | ✅ | 監査ログ、永続的な記録 |
| Upstash | ✅ | 🚀高速 | ✅ | サーバーレス環境 |

**重要：**
- LRUCacheはメモリ内のみ
- サーバー再起動で消える
- サーバーレス環境では使えない
- 本番環境ではRedisを推奨