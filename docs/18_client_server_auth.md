# Next.js クライアント・サーバー認証完全ガイド

## 📚 クライアントとサーバーでの認証の仕組み

Next.js App Routerでは、サーバーコンポーネントとクライアントコンポーネントで認証の扱い方が異なります。

## 1. 認証フローの全体像

```mermaid
[ブラウザ] → [ログイン] → [NextAuthサーバー] → [JWT/Session生成]
    ↓                                              ↓
[Cookie保存] ← [セッショントークン返却] ← [データベース保存]
    ↓
[後続リクエスト] → [Cookie送信] → [サーバー認証] → [データ取得]
    ↓
[クライアント認証] → [API呼び出し] → [認証付きリクエスト]
```

## 2. サーバーサイド認証

### サーバーコンポーネントでの認証
```typescript
// app/dashboard/page.tsx（サーバーコンポーネント）
import { getServerSession } from 'next-auth';
import { authOptions } from '@/app/api/auth/[...nextauth]/route';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  // サーバーサイドでセッション取得
  const session = await getServerSession(authOptions);
  
  // 未認証の場合はリダイレクト
  if (!session) {
    redirect('/login');
  }
  
  // データベース直接アクセス（サーバーサイドのみ）
  const userData = await prisma.user.findUnique({
    where: { id: session.user.id },
    include: { 
      profile: true,
      subscription: true 
    }
  });
  
  return (
    <div>
      <h1>Welcome, {userData.name}</h1>
      {/* このデータは既にサーバーで取得済み */}
      <UserProfile data={userData.profile} />
    </div>
  );
}
```

### API Routeでの認証
```typescript
// app/api/protected/route.ts
import { getServerSession } from 'next-auth';
import { authOptions } from '@/app/api/auth/[...nextauth]/route';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  // APIルートでの認証チェック
  const session = await getServerSession(authOptions);
  
  if (!session) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    );
  }
  
  // 認証済みユーザーのデータ取得
  const data = await prisma.userData.findMany({
    where: { userId: session.user.id }
  });
  
  return NextResponse.json(data);
}
```

### Middlewareでの認証
```typescript
// middleware.ts
import { withAuth } from 'next-auth/middleware';
import { NextResponse } from 'next/server';

export default withAuth(
  function middleware(req) {
    const token = req.nextauth.token;
    const isAdmin = token?.role === 'admin';
    
    // 管理者以外のアクセスを制限
    if (req.nextUrl.pathname.startsWith('/admin') && !isAdmin) {
      return NextResponse.redirect(new URL('/unauthorized', req.url));
    }
  },
  {
    callbacks: {
      authorized: ({ token }) => !!token
    }
  }
);

export const config = {
  matcher: [
    '/dashboard/:path*',
    '/admin/:path*',
    '/api/protected/:path*'
  ]
};
```

## 3. クライアントサイド認証

### クライアントコンポーネントでの認証
```typescript
// app/components/UserMenu.tsx（クライアントコンポーネント）
'use client';

import { useSession, signIn, signOut } from 'next-auth/react';
import { useRouter } from 'next/navigation';

export function UserMenu() {
  // クライアントサイドでセッション取得
  const { data: session, status } = useSession();
  const router = useRouter();
  
  // ローディング状態
  if (status === 'loading') {
    return <div>Loading...</div>;
  }
  
  // 未認証
  if (status === 'unauthenticated') {
    return (
      <button onClick={() => signIn()}>
        Sign In
      </button>
    );
  }
  
  // 認証済み
  return (
    <div>
      <span>{session?.user?.email}</span>
      <button onClick={() => signOut()}>
        Sign Out
      </button>
    </div>
  );
}
```

### 認証付きAPI呼び出し
```typescript
// hooks/useAuthFetch.ts
'use client';

import { useSession } from 'next-auth/react';
import { useCallback } from 'react';

export function useAuthFetch() {
  const { data: session } = useSession();
  
  const authFetch = useCallback(async (url: string, options?: RequestInit) => {
    if (!session) {
      throw new Error('Not authenticated');
    }
    
    const response = await fetch(url, {
      ...options,
      headers: {
        ...options?.headers,
        'Content-Type': 'application/json',
        // セッションCookieは自動的に送信される
      }
    });
    
    if (response.status === 401) {
      // トークンが無効な場合は再ログイン
      signIn();
      throw new Error('Session expired');
    }
    
    return response;
  }, [session]);
  
  return authFetch;
}

// 使用例
function MyComponent() {
  const authFetch = useAuthFetch();
  
  const handleSubmit = async (data: any) => {
    const response = await authFetch('/api/protected', {
      method: 'POST',
      body: JSON.stringify(data)
    });
    
    const result = await response.json();
  };
}
```

## 4. 認証情報の共有パターン

### SessionProviderによる共有
```typescript
// app/providers.tsx
'use client';

import { SessionProvider } from 'next-auth/react';

export function Providers({ 
  children,
  session 
}: { 
  children: React.ReactNode;
  session?: any;
}) {
  return (
    <SessionProvider session={session}>
      {children}
    </SessionProvider>
  );
}

// app/layout.tsx
import { getServerSession } from 'next-auth';
import { Providers } from './providers';

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  // サーバーサイドでセッション取得
  const session = await getServerSession();
  
  return (
    <html>
      <body>
        {/* クライアントに渡す */}
        <Providers session={session}>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

### カスタムコンテキストでの認証状態管理
```typescript
// contexts/AuthContext.tsx
'use client';

import { createContext, useContext, useEffect, useState } from 'react';
import { useSession } from 'next-auth/react';

interface AuthContextType {
  user: any;
  isLoading: boolean;
  isAuthenticated: boolean;
  permissions: string[];
}

const AuthContext = createContext<AuthContextType>({
  user: null,
  isLoading: true,
  isAuthenticated: false,
  permissions: []
});

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const { data: session, status } = useSession();
  const [permissions, setPermissions] = useState<string[]>([]);
  
  useEffect(() => {
    if (session?.user) {
      // 権限情報を取得
      fetch('/api/user/permissions')
        .then(res => res.json())
        .then(data => setPermissions(data.permissions));
    }
  }, [session]);
  
  return (
    <AuthContext.Provider value={{
      user: session?.user || null,
      isLoading: status === 'loading',
      isAuthenticated: status === 'authenticated',
      permissions
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

## 5. 認証トークンの管理

### JWTトークン管理
```typescript
// lib/auth/token-manager.ts
class TokenManager {
  private accessToken: string | null = null;
  private refreshToken: string | null = null;
  
  // トークン保存（セキュアなhttpOnly Cookieを推奨）
  setTokens(access: string, refresh: string) {
    this.accessToken = access;
    this.refreshToken = refresh;
    
    // セキュアストレージに保存
    if (typeof window !== 'undefined') {
      // メモリのみ（最も安全）
      // localStorageは避ける（XSS脆弱性）
    }
  }
  
  // 認証ヘッダー生成
  getAuthHeaders(): HeadersInit {
    if (!this.accessToken) {
      throw new Error('No access token');
    }
    
    return {
      'Authorization': `Bearer ${this.accessToken}`
    };
  }
  
  // トークンリフレッシュ
  async refreshAccessToken() {
    if (!this.refreshToken) {
      throw new Error('No refresh token');
    }
    
    const response = await fetch('/api/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken: this.refreshToken })
    });
    
    const data = await response.json();
    this.accessToken = data.accessToken;
    
    return data.accessToken;
  }
}

export const tokenManager = new TokenManager();
```

## 6. 認証フローパターン

### パターン1: Cookie認証（推奨）
```typescript
// NextAuthのデフォルト
// httpOnly Cookieでセッショントークンを管理

// サーバー側
const session = await getServerSession();

// クライアント側
const { data: session } = useSession();
```

### パターン2: JWT Bearer Token
```typescript
// カスタム実装が必要
// APIキー認証やモバイルアプリ向け

// サーバー側
export async function GET(request: Request) {
  const token = request.headers.get('Authorization')?.split(' ')[1];
  const payload = await verifyJWT(token);
  
  if (!payload) {
    return new Response('Unauthorized', { status: 401 });
  }
}

// クライアント側
fetch('/api/data', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
```

### パターン3: API Key認証
```typescript
// 外部サービス向け

// サーバー側
const apiKey = request.headers.get('X-API-Key');
const client = await prisma.apiClient.findUnique({
  where: { apiKey: hashApiKey(apiKey) }
});

if (!client || !client.active) {
  return new Response('Invalid API Key', { status: 401 });
}
```

## 7. セキュリティベストプラクティス

### CSRF対策
```typescript
// NextAuthは自動的にCSRF保護を提供
// カスタム実装の場合
import { randomBytes } from 'crypto';

export async function generateCSRFToken() {
  const token = randomBytes(32).toString('hex');
  
  // セッションに保存
  await redis.set(`csrf:${sessionId}`, token, 'EX', 3600);
  
  return token;
}

export async function validateCSRFToken(sessionId: string, token: string) {
  const storedToken = await redis.get(`csrf:${sessionId}`);
  return storedToken === token;
}
```

### XSS対策
```typescript
// 1. トークンをlocalStorageに保存しない
// 2. httpOnly Cookieを使用
// 3. Content Security Policy設定

// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline';"
          }
        ]
      }
    ];
  }
};
```

## 8. 認証状態のデバッグ

```typescript
// components/AuthDebug.tsx（開発環境のみ）
'use client';

import { useSession } from 'next-auth/react';

export function AuthDebug() {
  const { data: session, status } = useSession();
  
  if (process.env.NODE_ENV !== 'development') {
    return null;
  }
  
  return (
    <div style={{
      position: 'fixed',
      bottom: 0,
      right: 0,
      background: 'black',
      color: 'white',
      padding: '10px',
      fontSize: '12px'
    }}>
      <div>Status: {status}</div>
      <div>User: {session?.user?.email || 'None'}</div>
      <div>Role: {session?.user?.role || 'None'}</div>
      <details>
        <summary>Session Data</summary>
        <pre>{JSON.stringify(session, null, 2)}</pre>
      </details>
    </div>
  );
}
```

## まとめ

### サーバーサイド認証
- `getServerSession()` でセッション取得
- データベース直接アクセス可能
- SEO対応、初期表示が高速

### クライアントサイド認証
- `useSession()` フックでセッション取得
- リアクティブな UI 更新
- SPA的な動作

### 推奨パターン
1. **初期データ取得**: サーバーコンポーネント
2. **インタラクティブな操作**: クライアントコンポーネント
3. **トークン管理**: httpOnly Cookie（NextAuth）
4. **API認証**: Middleware + API Route