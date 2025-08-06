# API開発パターン完全比較ガイド

## 📚 学習目標（1時間）
Rails APIモードとNext.js Route Handlersの開発パターンをマスター

## 1. API設計の基本比較

### Rails API
```ruby
# config/application.rb
class Application < Rails::Application
  config.api_only = true
end

# app/controllers/api/v1/users_controller.rb
module Api
  module V1
    class UsersController < ApplicationController
      before_action :set_user, only: [:show, :update, :destroy]
      
      # GET /api/v1/users
      def index
        @users = User.includes(:posts).page(params[:page])
        render json: @users, include: :posts, meta: pagination_meta(@users)
      end
      
      # POST /api/v1/users
      def create
        @user = User.new(user_params)
        
        if @user.save
          render json: @user, status: :created
        else
          render json: { errors: @user.errors }, status: :unprocessable_entity
        end
      end
      
      private
      
      def user_params
        params.require(:user).permit(:email, :name, :age)
      end
      
      def set_user
        @user = User.find(params[:id])
      rescue ActiveRecord::RecordNotFound
        render json: { error: 'User not found' }, status: :not_found
      end
    end
  end
end
```

### Next.js Route Handlers
```typescript
// app/api/v1/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';
import { z } from 'zod';

const UserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
  age: z.number().positive().optional()
});

// GET /api/v1/users
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = parseInt(searchParams.get('page') || '1');
  const limit = 10;
  
  try {
    // ✅ 推奨: Promise.allで並列実行（高速）
    const [users, total] = await Promise.all([
      prisma.user.findMany({
        skip: (page - 1) * limit,
        take: limit,
        include: { posts: true }
      }),
      prisma.user.count()
    ]);
    
    // ⚠️ 動作するが遅い: 順次実行
    // const users = await prisma.user.findMany({
    //   skip: (page - 1) * limit,
    //   take: limit,
    //   include: { posts: true }
    // });
    // const total = await prisma.user.count();
    // → 2つのクエリが順番に実行される（待ち時間が2倍）
    
    // 📊 パフォーマンス比較:
    // Promise.all: 200ms (両方のクエリが同時実行)
    // 順次実行: 400ms (200ms + 200ms)
    
    return NextResponse.json({
      data: users,
      meta: {
        page,
        total,
        pages: Math.ceil(total / limit)
      }
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch users' },
      { status: 500 }
    );
  }
}

// POST /api/v1/users
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const validatedData = UserSchema.parse(body);
    
    const user = await prisma.user.create({
      data: validatedData
    });
    
    return NextResponse.json(user, { status: 201 });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { errors: error.errors },
        { status: 400 }
      );
    }
    
    return NextResponse.json(
      { error: 'Failed to create user' },
      { status: 500 }
    );
  }
}
```

## 2. エラーハンドリング

### Rails
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
  rescue_from ActionController::ParameterMissing, with: :bad_request
  
  private
  
  def not_found(exception)
    render json: { error: exception.message }, status: :not_found
  end
  
  def unprocessable_entity(exception)
    render json: { 
      error: 'Validation failed',
      details: exception.record.errors
    }, status: :unprocessable_entity
  end
  
  def bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end
end
```

### Next.js
```typescript
// lib/api-errors.ts
export class ApiError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public details?: any
  ) {
    super(message);
  }
}

export function handleApiError(error: unknown) {
  console.error('API Error:', error);
  
  if (error instanceof ApiError) {
    return NextResponse.json(
      { error: error.message, details: error.details },
      { status: error.statusCode }
    );
  }
  
  if (error instanceof z.ZodError) {
    return NextResponse.json(
      { error: 'Validation failed', details: error.errors },
      { status: 400 }
    );
  }
  
  return NextResponse.json(
    { error: 'Internal server error' },
    { status: 500 }
  );
}

// 使用例
export async function GET() {
  try {
    const user = await prisma.user.findUniqueOrThrow({
      where: { id: 'invalid' }
    });
    return NextResponse.json(user);
  } catch (error) {
    return handleApiError(error);
  }
}
```

## 3. ページネーション

### Rails (Kaminari/Pagy)
```ruby
# Gemfile
gem 'kaminari' # または gem 'pagy'

# Controller
def index
  @users = User.page(params[:page]).per(params[:per_page] || 10)
  
  render json: {
    data: @users,
    meta: {
      current_page: @users.current_page,
      total_pages: @users.total_pages,
      total_count: @users.total_count
    }
  }
end
```

### Next.js
```typescript
// lib/pagination.ts
interface PaginationParams {
  page?: number;
  limit?: number;
}

export async function paginate<T>(
  model: any,
  { page = 1, limit = 10 }: PaginationParams,
  options = {}
) {
  const skip = (page - 1) * limit;
  
  const [data, total] = await Promise.all([
    model.findMany({
      ...options,
      skip,
      take: limit
    }),
    model.count(options.where ? { where: options.where } : {})
  ]);
  
  return {
    data,
    meta: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit),
      hasNext: page < Math.ceil(total / limit),
      hasPrev: page > 1
    }
  };
}

// 使用例
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '10');
  
  const result = await paginate(prisma.user, { page, limit }, {
    include: { posts: true }
  });
  
  return NextResponse.json(result);
}
```

## 4. フィルタリングとソート

### Rails
```ruby
# app/controllers/api/v1/users_controller.rb
def index
  @users = User.all
  
  # フィルタリング
  @users = @users.where(active: params[:active]) if params[:active].present?
  @users = @users.where("age >= ?", params[:min_age]) if params[:min_age].present?
  @users = @users.joins(:posts).distinct if params[:with_posts] == 'true'
  
  # ソート
  @users = @users.order(params[:sort] || 'created_at DESC')
  
  # ページネーション
  @users = @users.page(params[:page])
  
  render json: @users
end
```

### Next.js
```typescript
// app/api/v1/users/route.ts
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  
  // フィルタリング構築
  const where: any = {};
  
  if (searchParams.get('active') !== null) {
    where.active = searchParams.get('active') === 'true';
  }
  
  if (searchParams.get('min_age')) {
    where.age = { gte: parseInt(searchParams.get('min_age')!) };
  }
  
  const include: any = {};
  if (searchParams.get('with_posts') === 'true') {
    include.posts = true;
  }
  
  // ソート
  const sortField = searchParams.get('sort_by') || 'createdAt';
  const sortOrder = searchParams.get('sort_order') || 'desc';
  
  const users = await prisma.user.findMany({
    where,
    include,
    orderBy: { [sortField]: sortOrder }
  });
  
  return NextResponse.json(users);
}
```

## 5. ファイルアップロード

### Rails (Active Storage)
```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :documents
end

# app/controllers/api/v1/users_controller.rb
def upload_avatar
  @user = User.find(params[:id])
  @user.avatar.attach(params[:avatar])
  
  render json: {
    url: url_for(@user.avatar),
    filename: @user.avatar.filename
  }
end
```

### Next.js
```typescript
// app/api/upload/route.ts
import { writeFile } from 'fs/promises';
import { NextRequest, NextResponse } from 'next/server';
import path from 'path';

export async function POST(request: NextRequest) {
  const formData = await request.formData();
  const file = formData.get('file') as File;
  
  if (!file) {
    return NextResponse.json(
      { error: 'No file provided' },
      { status: 400 }
    );
  }
  
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);
  
  // ファイル保存（実際はS3などを使用）
  const filename = `${Date.now()}-${file.name}`;
  const filepath = path.join(process.cwd(), 'public/uploads', filename);
  
  await writeFile(filepath, buffer);
  
  // データベースに保存
  const upload = await prisma.upload.create({
    data: {
      filename,
      url: `/uploads/${filename}`,
      size: file.size,
      mimeType: file.type
    }
  });
  
  return NextResponse.json(upload);
}
```

## 6. レート制限

### Rails (rack-attack)
```ruby
# config/initializers/rack_attack.rb
Rack::Attack.throttle('api/ip', limit: 100, period: 1.minute) do |req|
  req.ip if req.path.start_with?('/api')
end

Rack::Attack.throttle('api/user', limit: 1000, period: 1.hour) do |req|
  req.env['warden'].user&.id if req.path.start_with?('/api')
end
```

### Next.js (カスタム実装)
```typescript
// lib/rate-limit.ts
import { LRUCache } from 'lru-cache';

type Options = {
  uniqueTokenPerInterval?: number;
  interval?: number;
};

export function rateLimit(options?: Options) {
  const tokenCache = new LRUCache({
    max: options?.uniqueTokenPerInterval || 500,
    ttl: options?.interval || 60000,
  });

  return {
    check: async (token: string, limit: number) => {
      const tokenCount = (tokenCache.get(token) as number[]) || [0];
      const currentUsage = tokenCount[0];
      
      if (currentUsage >= limit) {
        return { success: false, remaining: 0 };
      }
      
      tokenCount[0] = currentUsage + 1;
      tokenCache.set(token, tokenCount);
      
      return { success: true, remaining: limit - currentUsage - 1 };
    },
  };
}

// middleware.ts
import { rateLimit } from '@/lib/rate-limit';

const limiter = rateLimit({
  interval: 60 * 1000, // 1分
  uniqueTokenPerInterval: 500
});

export async function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith('/api')) {
    const ip = request.ip ?? '127.0.0.1';
    const { success, remaining } = await limiter.check(ip, 100);
    
    if (!success) {
      return NextResponse.json(
        { error: 'Too many requests' },
        { status: 429, headers: { 'X-RateLimit-Remaining': remaining.toString() } }
      );
    }
  }
}
```

## 7. WebSocket / リアルタイム通信

### Rails (Action Cable)
```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat_#{params[:room_id]}"
  end
  
  def speak(data)
    message = Message.create!(
      content: data['message'],
      user: current_user,
      room_id: params[:room_id]
    )
    
    ActionCable.server.broadcast(
      "chat_#{params[:room_id]}",
      message: render_message(message)
    )
  end
end
```

### Next.js (Socket.io / Pusher)
```typescript
// app/api/socket/route.ts
import { Server } from 'socket.io';

const io = new Server();

io.on('connection', (socket) => {
  socket.on('join-room', (roomId) => {
    socket.join(roomId);
  });
  
  socket.on('send-message', async (data) => {
    const message = await prisma.message.create({
      data: {
        content: data.message,
        userId: data.userId,
        roomId: data.roomId
      }
    });
    
    io.to(data.roomId).emit('new-message', message);
  });
});

// クライアントサイド
'use client';
import { useEffect } from 'react';
import io from 'socket.io-client';

export function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const socket = io();
    
    socket.emit('join-room', roomId);
    
    socket.on('new-message', (message) => {
      // メッセージ表示処理
    });
    
    return () => {
      socket.disconnect();
    };
  }, [roomId]);
}
```

## 🎯 実践演習

### 課題: REST API から GraphQL への移行
Rails GraphQLとNext.js Apollo Serverでの実装比較

## 💡 まとめ

| 機能 | Rails | Next.js |
|------|-------|---------|
| ルーティング | routes.rb | ファイルベース |
| バリデーション | Strong Parameters | Zod/Yup |
| エラーハンドリング | rescue_from | try-catch |
| ページネーション | Kaminari/Pagy | カスタム実装 |
| ファイルアップロード | Active Storage | カスタム/S3 |
| WebSocket | Action Cable | Socket.io |
| レート制限 | rack-attack | カスタム実装 |