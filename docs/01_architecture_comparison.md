# Rails vs Next.js アーキテクチャ比較学習ガイド

## 📚 学習目標（1時間）
RailsのMVCパターンからNext.jsのフルスタックアーキテクチャへの移行を理解する

## 1. 基本アーキテクチャの違い

### Rails（MVC）
```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  def index
    @users = User.all
    render json: @users
  end
  
  def show
    @user = User.find(params[:id])
    render json: @user
  end
end

# app/models/user.rb
class User < ApplicationRecord
  validates :email, presence: true
  has_many :posts
end

# config/routes.rb
Rails.application.routes.draw do
  resources :users
end
```

### Next.js（App Router）
```typescript
// app/api/users/route.ts
import { NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';

export async function GET() {
  const users = await prisma.user.findMany();
  return NextResponse.json(users);
}

// app/api/users/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await prisma.user.findUnique({
    where: { id: params.id },
    include: { posts: true }
  });
  return NextResponse.json(user);
}

// app/users/page.tsx (サーバーコンポーネント)
async function UsersPage() {
  const users = await prisma.user.findMany();
  return <UserList users={users} />;
}
```

## 2. ルーティングの違い

### Rails
- `config/routes.rb`で集中管理
- RESTfulルーティング（resources）
- ネストされたルーティング

### Next.js
- ファイルベースルーティング
- `app`ディレクトリ構造がそのままルート
- 動的ルート: `[param]`
- ルートグループ: `(group)`

```
app/
├── (auth)/
│   ├── login/page.tsx     → /login
│   └── register/page.tsx  → /register
├── users/
│   ├── page.tsx           → /users
│   └── [id]/page.tsx      → /users/123
└── api/
    └── users/
        ├── route.ts       → /api/users
        └── [id]/route.ts  → /api/users/123
```

## 3. サーバーサイドレンダリング（SSR）

### Rails
```ruby
# app/controllers/users_controller.rb
def index
  @users = User.all
  # ERBテンプレートでレンダリング
end
```

### Next.js
```typescript
// サーバーコンポーネント（デフォルト）
export default async function UsersPage() {
  const users = await prisma.user.findMany();
  return <UserList users={users} />;
}

// クライアントコンポーネント
'use client';
export function UserInteraction() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

## 4. ミドルウェア

### Rails
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  before_action :authenticate_user!
  
  private
  def authenticate_user!
    # 認証ロジック
  end
end
```

### Next.js
```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token');
  
  if (!token && request.nextUrl.pathname.startsWith('/admin')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}

export const config = {
  matcher: ['/admin/:path*', '/api/admin/:path*']
};
```

## 5. 環境変数の管理

### Rails
```ruby
# config/database.yml
production:
  database: <%= ENV['DATABASE_URL'] %>

# config/credentials.yml.enc (暗号化)
secret_key_base: xxx
```

### Next.js
```bash
# .env.local
DATABASE_URL=postgresql://...
NEXT_PUBLIC_API_URL=https://... # ブラウザで使用可能

# 使用方法
process.env.DATABASE_URL // サーバーサイドのみ
process.env.NEXT_PUBLIC_API_URL // クライアントサイドでも使用可能
```

## 🎯 実践演習

### 課題1: 基本的なCRUD APIの実装
Railsの`resources :users`相当の機能をNext.jsで実装

```typescript
// app/api/users/route.ts
export async function GET() { /* 一覧取得 */ }
export async function POST() { /* 作成 */ }

// app/api/users/[id]/route.ts
export async function GET() { /* 詳細取得 */ }
export async function PUT() { /* 更新 */ }
export async function DELETE() { /* 削除 */ }
```

### 課題2: サーバーコンポーネントとクライアントコンポーネントの使い分け
- データ取得はサーバーコンポーネント
- インタラクティブな要素はクライアントコンポーネント

## 💡 重要な違いのまとめ

| 項目 | Rails | Next.js |
|------|-------|---------|
| アーキテクチャ | MVC | コンポーネントベース |
| ルーティング | routes.rb | ファイルベース |
| API | Controller | Route Handler |
| ビュー | ERB/API JSON | React Components |
| SSR | デフォルト | サーバーコンポーネント |
| CSR | Turbo/別途SPA | クライアントコンポーネント |
| ミドルウェア | before_action | middleware.ts |
| 静的生成 | なし | generateStaticParams |

## 📖 必読リソース
- [Next.js App Router Documentation](https://nextjs.org/docs/app)
- [Server Components vs Client Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)
- [Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)