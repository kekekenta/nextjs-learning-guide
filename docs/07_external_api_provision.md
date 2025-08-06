# 外部API提供・公開完全ガイド

## 📚 学習目標（45分）
Rails APIモードからNext.js API Routesで外部向けAPIを構築

## 1. API設計の基本比較

### Rails API（Grape/Rails API mode）
```ruby
# Gemfile
gem 'grape'
gem 'grape-entity'
gem 'grape-swagger'
gem 'rack-cors'
gem 'jwt'

# app/api/v1/base.rb
module V1
  class Base < Grape::API
    version 'v1', using: :path
    format :json
    prefix :api
    
    helpers do
      def authenticate!
        error!('401 Unauthorized', 401) unless current_client
      end
      
      def current_client
        token = headers['X-Api-Key']
        @current_client ||= ApiClient.find_by(api_key: token)
      end
    end
    
    mount V1::Users
    mount V1::Products
    
    add_swagger_documentation(
      api_version: 'v1',
      hide_documentation_path: true,
      mount_path: '/swagger',
      info: {
        title: 'My API',
        description: 'API Documentation'
      }
    )
  end
end

# app/api/v1/users.rb
module V1
  class Users < Grape::API
    before { authenticate! }
    
    resource :users do
      desc 'Get all users'
      params do
        optional :page, type: Integer, default: 1
        optional :per_page, type: Integer, default: 20
      end
      get do
        users = User.page(params[:page]).per(params[:per_page])
        present users, with: Entities::User
      end
      
      desc 'Create a user'
      params do
        requires :email, type: String, regexp: /.+@.+/
        requires :name, type: String
        optional :age, type: Integer
      end
      post do
        user = User.create!(declared(params))
        present user, with: Entities::User
      end
    end
  end
end
```

### Next.js 外部API提供
```typescript
// app/api/v1/[[...route]]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { prisma } from '@/lib/prisma';
import { createHash } from 'crypto';

// APIキー認証
async function authenticateApiKey(request: NextRequest) {
  const apiKey = request.headers.get('x-api-key');
  
  if (!apiKey) {
    return null;
  }
  
  // APIキーをハッシュ化して検索（セキュリティ向上）
  const hashedKey = createHash('sha256').update(apiKey).digest('hex');
  
  const client = await prisma.apiClient.findUnique({
    where: { hashedApiKey: hashedKey }
  });
  
  if (client && client.active) {
    // 使用状況を記録
    await prisma.apiUsage.create({
      data: {
        clientId: client.id,
        endpoint: request.url,
        method: request.method,
        statusCode: 200,
        timestamp: new Date()
      }
    });
    
    return client;
  }
  
  return null;
}

// レート制限
const rateLimitMap = new Map<string, { count: number; resetTime: number }>();

function checkRateLimit(clientId: string, limit: number = 100): boolean {
  const now = Date.now();
  const windowMs = 60 * 1000; // 1分
  
  const clientLimit = rateLimitMap.get(clientId);
  
  if (!clientLimit || now > clientLimit.resetTime) {
    rateLimitMap.set(clientId, {
      count: 1,
      resetTime: now + windowMs
    });
    return true;
  }
  
  if (clientLimit.count >= limit) {
    return false;
  }
  
  clientLimit.count++;
  return true;
}

// エラーレスポンス標準化
function errorResponse(message: string, status: number, details?: any) {
  return NextResponse.json(
    {
      error: {
        message,
        status,
        timestamp: new Date().toISOString(),
        details
      }
    },
    { 
      status,
      headers: {
        'Content-Type': 'application/json',
        'X-Content-Type-Options': 'nosniff'
      }
    }
  );
}

// 成功レスポンス標準化
function successResponse(data: any, meta?: any) {
  return NextResponse.json(
    {
      data,
      meta,
      timestamp: new Date().toISOString()
    },
    {
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': 'no-store',
        'X-Content-Type-Options': 'nosniff'
      }
    }
  );
}

// GET /api/v1/users
export async function GET(
  request: NextRequest,
  { params }: { params: { route?: string[] } }
) {
  // 認証
  const client = await authenticateApiKey(request);
  if (!client) {
    return errorResponse('Unauthorized', 401);
  }
  
  // レート制限
  if (!checkRateLimit(client.id, client.rateLimit)) {
    return errorResponse('Too Many Requests', 429, {
      retryAfter: 60
    });
  }
  
  const route = params.route || [];
  const searchParams = request.nextUrl.searchParams;
  
  try {
    // /api/v1/users
    if (route.length === 0 || (route.length === 1 && route[0] === 'users')) {
      const page = parseInt(searchParams.get('page') || '1');
      const limit = Math.min(parseInt(searchParams.get('limit') || '20'), 100);
      
      const [users, total] = await Promise.all([
        prisma.user.findMany({
          skip: (page - 1) * limit,
          take: limit,
          select: {
            id: true,
            email: true,
            name: true,
            createdAt: true
          }
        }),
        prisma.user.count()
      ]);
      
      return successResponse(users, {
        pagination: {
          page,
          limit,
          total,
          pages: Math.ceil(total / limit)
        }
      });
    }
    
    // /api/v1/users/:id
    if (route.length === 2 && route[0] === 'users') {
      const user = await prisma.user.findUnique({
        where: { id: route[1] },
        select: {
          id: true,
          email: true,
          name: true,
          createdAt: true
        }
      });
      
      if (!user) {
        return errorResponse('User not found', 404);
      }
      
      return successResponse(user);
    }
    
    return errorResponse('Endpoint not found', 404);
  } catch (error) {
    console.error('API Error:', error);
    return errorResponse('Internal Server Error', 500);
  }
}

// POST /api/v1/users
const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(255),
  age: z.number().positive().optional()
});

export async function POST(request: NextRequest) {
  const client = await authenticateApiKey(request);
  if (!client) {
    return errorResponse('Unauthorized', 401);
  }
  
  if (!checkRateLimit(client.id, client.rateLimit)) {
    return errorResponse('Too Many Requests', 429);
  }
  
  try {
    const body = await request.json();
    const validatedData = CreateUserSchema.parse(body);
    
    const user = await prisma.user.create({
      data: validatedData,
      select: {
        id: true,
        email: true,
        name: true,
        createdAt: true
      }
    });
    
    return successResponse(user, { created: true });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return errorResponse('Validation failed', 400, error.errors);
    }
    
    return errorResponse('Internal Server Error', 500);
  }
}
```

## 2. APIキー管理システム

### データベーススキーマ
```prisma
// prisma/schema.prisma
model ApiClient {
  id            String   @id @default(cuid())
  name          String
  email         String
  hashedApiKey  String   @unique
  active        Boolean  @default(true)
  rateLimit     Int      @default(100) // requests per minute
  allowedIps    String[] // IP制限
  allowedScopes String[] // 権限スコープ
  expiresAt     DateTime?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  
  usage         ApiUsage[]
  webhooks      ApiWebhook[]
  
  @@index([hashedApiKey])
}

model ApiUsage {
  id         String   @id @default(cuid())
  clientId   String
  endpoint   String
  method     String
  statusCode Int
  timestamp  DateTime @default(now())
  
  client     ApiClient @relation(fields: [clientId], references: [id])
  
  @@index([clientId, timestamp])
}

model ApiWebhook {
  id        String   @id @default(cuid())
  clientId  String
  url       String
  events    String[] // ['user.created', 'user.updated']
  secret    String
  active    Boolean  @default(true)
  
  client    ApiClient @relation(fields: [clientId], references: [id])
}
```

### APIキー生成・管理画面
```typescript
// app/api/admin/api-keys/route.ts
import { randomBytes } from 'crypto';
import { createHash } from 'crypto';

export async function POST(request: NextRequest) {
  // 管理者認証チェック
  const session = await getServerSession();
  if (!session?.user?.role === 'ADMIN') {
    return errorResponse('Forbidden', 403);
  }
  
  const { name, email, rateLimit, allowedScopes } = await request.json();
  
  // APIキー生成（プレフィックス付き）
  const apiKey = `sk_live_${randomBytes(32).toString('hex')}`;
  const hashedApiKey = createHash('sha256').update(apiKey).digest('hex');
  
  const client = await prisma.apiClient.create({
    data: {
      name,
      email,
      hashedApiKey,
      rateLimit: rateLimit || 100,
      allowedScopes: allowedScopes || ['read']
    }
  });
  
  // APIキーは初回のみ表示（保存しない）
  return NextResponse.json({
    client: {
      id: client.id,
      name: client.name,
      email: client.email
    },
    apiKey // この後は二度と取得できない
  });
}
```

## 3. API仕様書とSDK提供

### OpenAPI仕様書生成
```typescript
// lib/openapi.ts
export const openApiSpec = {
  openapi: '3.0.0',
  info: {
    title: 'My API',
    version: '1.0.0',
    description: 'External API for partners',
    contact: {
      email: 'api@example.com'
    }
  },
  servers: [
    {
      url: 'https://api.example.com/v1',
      description: 'Production server'
    }
  ],
  components: {
    securitySchemes: {
      ApiKeyAuth: {
        type: 'apiKey',
        in: 'header',
        name: 'X-API-Key'
      }
    },
    schemas: {
      User: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          email: { type: 'string', format: 'email' },
          name: { type: 'string' },
          createdAt: { type: 'string', format: 'date-time' }
        }
      },
      Error: {
        type: 'object',
        properties: {
          error: {
            type: 'object',
            properties: {
              message: { type: 'string' },
              status: { type: 'integer' },
              timestamp: { type: 'string' }
            }
          }
        }
      }
    }
  },
  paths: {
    '/users': {
      get: {
        summary: 'List users',
        security: [{ ApiKeyAuth: [] }],
        parameters: [
          {
            name: 'page',
            in: 'query',
            schema: { type: 'integer', default: 1 }
          },
          {
            name: 'limit',
            in: 'query',
            schema: { type: 'integer', default: 20, maximum: 100 }
          }
        ],
        responses: {
          '200': {
            description: 'Success',
            content: {
              'application/json': {
                schema: {
                  type: 'object',
                  properties: {
                    data: {
                      type: 'array',
                      items: { $ref: '#/components/schemas/User' }
                    }
                  }
                }
              }
            }
          },
          '401': {
            description: 'Unauthorized',
            content: {
              'application/json': {
                schema: { $ref: '#/components/schemas/Error' }
              }
            }
          }
        }
      }
    }
  }
};

// app/api/v1/swagger/route.ts
export async function GET() {
  return NextResponse.json(openApiSpec);
}
```

### TypeScript SDK生成
```typescript
// sdk/typescript/src/index.ts
export class MyApiClient {
  private apiKey: string;
  private baseUrl: string;
  
  constructor(apiKey: string, options?: { baseUrl?: string }) {
    this.apiKey = apiKey;
    this.baseUrl = options?.baseUrl || 'https://api.example.com/v1';
  }
  
  private async request<T>(
    endpoint: string,
    options?: RequestInit
  ): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      ...options,
      headers: {
        'X-API-Key': this.apiKey,
        'Content-Type': 'application/json',
        ...options?.headers
      }
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new ApiError(error.error.message, response.status);
    }
    
    const result = await response.json();
    return result.data as T;
  }
  
  async getUsers(params?: { page?: number; limit?: number }) {
    const query = new URLSearchParams(params as any).toString();
    return this.request<User[]>(`/users?${query}`);
  }
  
  async getUser(id: string) {
    return this.request<User>(`/users/${id}`);
  }
  
  async createUser(data: CreateUserInput) {
    return this.request<User>('/users', {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }
}

// 使用例
const client = new MyApiClient('sk_live_xxxxx');
const users = await client.getUsers({ page: 1, limit: 10 });
```

## 4. Webhook通知システム

```typescript
// lib/webhook.ts
import { createHmac } from 'crypto';

export class WebhookSender {
  async send(webhook: any, event: string, data: any) {
    const payload = {
      event,
      data,
      timestamp: new Date().toISOString()
    };
    
    const signature = createHmac('sha256', webhook.secret)
      .update(JSON.stringify(payload))
      .digest('hex');
    
    try {
      const response = await fetch(webhook.url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,
          'X-Webhook-Event': event
        },
        body: JSON.stringify(payload)
      });
      
      // ログ記録
      await prisma.webhookLog.create({
        data: {
          webhookId: webhook.id,
          event,
          status: response.status,
          response: await response.text()
        }
      });
      
      // 失敗時のリトライ
      if (!response.ok) {
        await this.scheduleRetry(webhook, payload);
      }
    } catch (error) {
      console.error('Webhook failed:', error);
      await this.scheduleRetry(webhook, payload);
    }
  }
  
  private async scheduleRetry(webhook: any, payload: any) {
    // リトライロジック（BullMQなど使用）
  }
}

// 使用例：ユーザー作成時のWebhook
export async function createUser(data: any, clientId: string) {
  const user = await prisma.user.create({ data });
  
  // Webhook送信
  const webhooks = await prisma.apiWebhook.findMany({
    where: {
      clientId,
      events: { has: 'user.created' },
      active: true
    }
  });
  
  const sender = new WebhookSender();
  for (const webhook of webhooks) {
    await sender.send(webhook, 'user.created', user);
  }
  
  return user;
}
```

## 5. API認証方式の比較実装

### Bearer Token認証
```typescript
// JWT Bearer Token
export async function authenticateBearerToken(request: NextRequest) {
  const auth = request.headers.get('authorization');
  if (!auth?.startsWith('Bearer ')) {
    return null;
  }
  
  const token = auth.substring(7);
  
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!);
    return await prisma.apiClient.findUnique({
      where: { id: payload.clientId }
    });
  } catch {
    return null;
  }
}
```

### OAuth 2.0
```typescript
// OAuth 2.0 実装
export async function POST(request: NextRequest) {
  const { grant_type, client_id, client_secret, code } = await request.json();
  
  if (grant_type === 'authorization_code') {
    // 認可コードフロー
    const authCode = await prisma.authCode.findUnique({
      where: { code }
    });
    
    if (!authCode || authCode.clientId !== client_id) {
      return errorResponse('Invalid authorization code', 400);
    }
    
    const accessToken = jwt.sign(
      { clientId: client_id, scope: authCode.scope },
      process.env.JWT_SECRET!,
      { expiresIn: '1h' }
    );
    
    return NextResponse.json({
      access_token: accessToken,
      token_type: 'Bearer',
      expires_in: 3600
    });
  }
}
```

## 6. API監視とメトリクス

```typescript
// middleware.ts
export async function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith('/api/v1')) {
    const start = Date.now();
    
    // リクエストログ
    console.log(JSON.stringify({
      type: 'api_request',
      method: request.method,
      path: request.nextUrl.pathname,
      ip: request.ip,
      userAgent: request.headers.get('user-agent')
    }));
    
    const response = NextResponse.next();
    
    // レスポンスログ
    response.headers.set('X-Response-Time', `${Date.now() - start}ms`);
    
    return response;
  }
}
```

## 7. API バージョニング戦略

```typescript
// app/api/[version]/[[...route]]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: { version: string; route?: string[] } }
) {
  const version = params.version;
  
  // バージョン別の処理
  switch (version) {
    case 'v1':
      return handleV1Request(request, params.route);
    case 'v2':
      return handleV2Request(request, params.route);
    default:
      return errorResponse('API version not supported', 400);
  }
}

// バージョン間の互換性維持
function transformV1ToV2(data: any) {
  return {
    ...data,
    // V2での新フィールド
    version: 2,
    metadata: {}
  };
}
```

## 8. デプロイとCORS設定

```typescript
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Origin', value: '*' },
          { key: 'Access-Control-Allow-Methods', value: 'GET,POST,PUT,DELETE,OPTIONS' },
          { key: 'Access-Control-Allow-Headers', value: 'X-API-Key, Content-Type' }
        ]
      }
    ];
  }
};
```

## 🎯 実践演習

### 課題: パートナー向けAPI構築
1. APIキー管理システム
2. レート制限実装
3. Webhook通知
4. SDK生成とドキュメント

## 💡 Rails vs Next.js API提供比較

| 機能 | Rails | Next.js |
|------|-------|---------|
| APIフレームワーク | Grape/Rails API | API Routes |
| 認証 | Devise Token Auth | カスタム実装 |
| ドキュメント | grape-swagger | OpenAPI手動 |
| レート制限 | rack-attack | カスタム/Redis |
| バージョニング | URLパス/ヘッダー | URLパス |
| SDK生成 | swagger-codegen | openapi-generator |