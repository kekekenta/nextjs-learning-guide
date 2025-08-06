# NextAuth データベーステーブル構造完全ガイド

## 📚 NextAuthが使用するテーブル設計

NextAuthは認証システムで必要な4つの主要テーブルを使用します。Deviseと比較しながら詳しく解説します。

## 1. NextAuthの基本テーブル構造

### 必須テーブル一覧
```prisma
// prisma/schema.prisma

model User {
  id            String    @id @default(cuid())
  email         String?   @unique
  emailVerified DateTime?
  name          String?
  image         String?
  
  // リレーション
  accounts      Account[]
  sessions      Session[]
  
  // カスタムフィールド（追加可能）
  role          String    @default("USER")
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@unique([provider, providerAccountId])
  @@index([userId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId])
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime
  
  @@unique([identifier, token])
}
```

## 2. 各テーブルの詳細説明

### User テーブル
**役割**: ユーザーの基本情報を保存

| カラム | 型 | 説明 | 必須 |
|--------|-----|------|------|
| id | String | ユーザーの一意識別子（CUID） | ✅ |
| email | String | メールアドレス | ❌ |
| emailVerified | DateTime | メール確認日時 | ❌ |
| name | String | 表示名 | ❌ |
| image | String | アバター画像URL | ❌ |

```typescript
// 使用例
const user = await prisma.user.create({
  data: {
    email: "user@example.com",
    name: "John Doe",
    emailVerified: new Date()
  }
});
```

### Account テーブル
**役割**: OAuth/外部プロバイダー情報を保存

| カラム | 型 | 説明 | 用途 |
|--------|-----|------|------|
| userId | String | Userテーブルへの外部キー | リレーション |
| provider | String | プロバイダー名（google, github等） | OAuth識別 |
| providerAccountId | String | プロバイダー側のユーザーID | OAuth識別 |
| access_token | String | アクセストークン | API呼び出し |
| refresh_token | String | リフレッシュトークン | トークン更新 |
| expires_at | Int | トークン有効期限（Unix時間） | 有効期限管理 |

```typescript
// Googleログイン時の例
{
  userId: "cuid...",
  type: "oauth",
  provider: "google",
  providerAccountId: "115711234567890",
  access_token: "ya29.a0ARrdaM...",
  refresh_token: "1//0eZGp...",
  expires_at: 1681234567,
  scope: "openid email profile"
}
```

### Session テーブル
**役割**: ユーザーセッション管理

| カラム | 型 | 説明 | 用途 |
|--------|-----|------|------|
| sessionToken | String | セッショントークン（Cookie値） | セッション識別 |
| userId | String | ユーザーID | ユーザー紐付け |
| expires | DateTime | セッション有効期限 | 自動削除 |

```typescript
// セッション例
{
  id: "cuid...",
  sessionToken: "4b8f3c2a-...", // httpOnly Cookieに保存
  userId: "user_cuid...",
  expires: "2024-12-31T23:59:59.999Z"
}
```

### VerificationToken テーブル
**役割**: メール確認・パスワードリセット用トークン

| カラム | 型 | 説明 | 用途 |
|--------|-----|------|------|
| identifier | String | メールアドレス等の識別子 | トークン送信先 |
| token | String | 検証用トークン | URL内で使用 |
| expires | DateTime | 有効期限 | セキュリティ |

```typescript
// メール確認トークンの例
{
  identifier: "user@example.com",
  token: "abc123xyz...",
  expires: new Date(Date.now() + 24 * 60 * 60 * 1000) // 24時間
}
```

## 3. Devise（Rails）との比較

### テーブル構造の違い

| 機能 | Devise (Rails) | NextAuth | 備考 |
|------|---------------|----------|------|
| **ユーザー基本情報** | users テーブル | User テーブル | ほぼ同じ |
| **パスワード** | users.encrypted_password | なし（またはカスタム） | NextAuthはOAuth優先 |
| **セッション** | セッションCookie | Session テーブル | NextAuthはDB保存 |
| **OAuth情報** | 別gem必要 | Account テーブル | NextAuth標準装備 |
| **メール確認** | users.confirmation_token | VerificationToken | 別テーブル管理 |
| **ログイン履歴** | users.sign_in_count等 | カスタム実装必要 | NextAuthは最小限 |

### Deviseのusersテーブル
```ruby
# Deviseの標準カラム
create_table :users do |t|
  t.string :email,              null: false
  t.string :encrypted_password, null: false
  
  # Recoverable
  t.string   :reset_password_token
  t.datetime :reset_password_sent_at
  
  # Rememberable
  t.datetime :remember_created_at
  
  # Trackable
  t.integer  :sign_in_count, default: 0
  t.datetime :current_sign_in_at
  t.datetime :last_sign_in_at
  t.string   :current_sign_in_ip
  t.string   :last_sign_in_ip
  
  # Confirmable
  t.string   :confirmation_token
  t.datetime :confirmed_at
  t.datetime :confirmation_sent_at
  
  # Lockable
  t.integer  :failed_attempts, default: 0
  t.string   :unlock_token
  t.datetime :locked_at
end
```

### NextAuthで同等の機能を実装
```prisma
// NextAuth + カスタムフィールドで実装
model User {
  id            String    @id @default(cuid())
  email         String?   @unique
  emailVerified DateTime?
  name          String?
  image         String?
  
  // Devise互換のカスタムフィールド
  hashedPassword       String?   // encrypted_password相当
  signInCount          Int       @default(0)
  currentSignInAt      DateTime?
  lastSignInAt         DateTime?
  currentSignInIp      String?
  lastSignInIp         String?
  failedAttempts       Int       @default(0)
  lockedAt             DateTime?
  
  // リレーション
  accounts      Account[]
  sessions      Session[]
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}
```

## 4. 実装パターン

### 基本的な認証フロー

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/prisma';
import GoogleProvider from 'next-auth/providers/google';
import CredentialsProvider from 'next-auth/providers/credentials';
import bcrypt from 'bcryptjs';

export const authOptions = {
  adapter: PrismaAdapter(prisma),
  providers: [
    // OAuth認証（Account テーブル使用）
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    
    // パスワード認証（カスタム実装）
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" }
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }
        
        const user = await prisma.user.findUnique({
          where: { email: credentials.email }
        });
        
        if (!user || !user.hashedPassword) {
          return null;
        }
        
        const isValid = await bcrypt.compare(
          credentials.password,
          user.hashedPassword
        );
        
        if (!isValid) {
          return null;
        }
        
        return {
          id: user.id,
          email: user.email,
          name: user.name,
          image: user.image
        };
      }
    })
  ],
  callbacks: {
    // セッション作成時の処理
    async session({ session, token, user }) {
      if (session.user) {
        session.user.id = user.id;
        session.user.role = user.role;
      }
      return session;
    }
  },
  session: {
    strategy: 'database', // Sessionテーブル使用
    maxAge: 30 * 24 * 60 * 60, // 30日
  }
};
```

### データの関連性

```mermaid
User (1) ─── (n) Account    : OAuth情報
User (1) ─── (n) Session    : アクティブセッション
VerificationToken           : 独立（メール確認用）
```

### 実際のデータ例

```typescript
// 1人のユーザーが複数の認証方法を持つ場合
const userData = {
  user: {
    id: "cuid_user_123",
    email: "user@example.com",
    name: "John Doe",
    emailVerified: "2024-01-01T00:00:00Z"
  },
  accounts: [
    {
      provider: "google",
      providerAccountId: "google_123456",
      access_token: "ya29..."
    },
    {
      provider: "github",
      providerAccountId: "github_789",
      access_token: "gho_..."
    }
  ],
  sessions: [
    {
      sessionToken: "session_web_abc",
      expires: "2024-02-01T00:00:00Z"
    },
    {
      sessionToken: "session_mobile_xyz",
      expires: "2024-02-15T00:00:00Z"
    }
  ]
};
```

## 5. カスタマイズ例

### ユーザープロフィール追加

```prisma
model User {
  id            String    @id @default(cuid())
  email         String?   @unique
  emailVerified DateTime?
  name          String?
  image         String?
  
  // プロフィール拡張
  bio           String?   @db.Text
  phoneNumber   String?
  dateOfBirth   DateTime?
  country       String?
  language      String    @default("ja")
  timezone      String    @default("Asia/Tokyo")
  
  // 権限管理
  role          Role      @default(USER)
  permissions   String[]  // ["read:posts", "write:posts"]
  
  // 組織管理
  organizationId String?
  organization   Organization? @relation(fields: [organizationId], references: [id])
  
  // 2要素認証
  twoFactorEnabled  Boolean @default(false)
  twoFactorSecret   String?
  
  accounts      Account[]
  sessions      Session[]
  posts         Post[]
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### ログイン履歴テーブル追加

```prisma
model LoginHistory {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  
  ipAddress String
  userAgent String
  loginAt   DateTime @default(now())
  success   Boolean
  
  // 位置情報（オプション）
  country   String?
  city      String?
  
  @@index([userId])
  @@index([loginAt])
}
```

## 6. マイグレーション

### Prismaマイグレーション実行

```bash
# スキーマ作成
npx prisma migrate dev --name init-nextauth

# 本番環境
npx prisma migrate deploy
```

### 生成されるSQL（PostgreSQL例）

```sql
-- User table
CREATE TABLE "User" (
    "id" TEXT NOT NULL,
    "email" TEXT,
    "emailVerified" TIMESTAMP(3),
    "name" TEXT,
    "image" TEXT,
    "role" TEXT NOT NULL DEFAULT 'USER',
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,
    
    CONSTRAINT "User_pkey" PRIMARY KEY ("id")
);

CREATE UNIQUE INDEX "User_email_key" ON "User"("email");

-- Account table
CREATE TABLE "Account" (
    "id" TEXT NOT NULL,
    "userId" TEXT NOT NULL,
    "type" TEXT NOT NULL,
    "provider" TEXT NOT NULL,
    "providerAccountId" TEXT NOT NULL,
    "refresh_token" TEXT,
    "access_token" TEXT,
    "expires_at" INTEGER,
    
    CONSTRAINT "Account_pkey" PRIMARY KEY ("id"),
    CONSTRAINT "Account_userId_fkey" 
        FOREIGN KEY ("userId") 
        REFERENCES "User"("id") 
        ON DELETE CASCADE
);

CREATE UNIQUE INDEX "Account_provider_providerAccountId_key" 
    ON "Account"("provider", "providerAccountId");
```

## 7. セキュリティ考慮事項

### トークン管理
```typescript
// セッショントークンの安全な生成
import { randomBytes } from 'crypto';

function generateSessionToken(): string {
  return randomBytes(32).toString('hex');
}

// CSRFトークン
function generateCSRFToken(): string {
  return randomBytes(24).toString('hex');
}
```

### データ削除ポリシー
```prisma
// カスケード削除設定
model Account {
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model Session {
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

### セッション管理
```typescript
// 期限切れセッションの自動削除
async function cleanupExpiredSessions() {
  await prisma.session.deleteMany({
    where: {
      expires: {
        lt: new Date()
      }
    }
  });
}

// Cronジョブで定期実行
// 0 */6 * * * cleanupExpiredSessions
```

## まとめ

### NextAuthテーブル構造の特徴

1. **最小限の構成**: 4テーブルで認証システム完結
2. **OAuth対応**: Accountテーブルで複数プロバイダー管理
3. **セッション管理**: データベースベースで安全
4. **拡張性**: カスタムフィールド追加が容易

### Deviseとの主な違い

| 項目 | Devise | NextAuth |
|------|--------|----------|
| パスワード管理 | 標準装備 | オプション |
| OAuth | 追加gem必要 | 標準装備 |
| セッション | Cookie | Database |
| カスタマイズ | Rails規約 | 自由度高い |
| テーブル数 | 1つ（users） | 4つ |

### 使い分けの指針

- **NextAuth向き**: OAuth中心、マルチプロバイダー、SPA
- **Devise向き**: パスワード認証中心、Rails標準機能活用、管理画面