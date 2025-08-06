# Devise/Pundit vs NextAuth/CASL 認証・認可完全ガイド

## 📚 学習目標（1時間）
DeviseとPunditの経験を活かしてNext.jsの認証・認可をマスター

## 1. 認証（Authentication）の比較

### Devise（Rails）
```ruby
# Gemfile
gem 'devise'
gem 'devise-jwt' # JWT対応

# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :lockable, :trackable,
         :jwt_authenticatable, jwt_revocation_strategy: JwtDenylist
  
  def jwt_payload
    { 'email' => email, 'role' => role }
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  before_action :authenticate_user!
  
  private
  
  def authenticate_user!
    if request.headers['Authorization'].present?
      jwt_authenticate
    else
      render json: { error: 'Unauthorized' }, status: :unauthorized
    end
  end
  
  def jwt_authenticate
    token = request.headers['Authorization'].split(' ').last
    payload = JWT.decode(token, Rails.application.credentials.secret_key_base).first
    @current_user = User.find(payload['sub'])
  rescue JWT::DecodeError
    render json: { error: 'Invalid token' }, status: :unauthorized
  end
end

# config/routes.rb
Rails.application.routes.draw do
  devise_for :users,
    path: '',
    path_names: {
      sign_in: 'login',
      sign_out: 'logout',
      registration: 'signup'
    },
    controllers: {
      sessions: 'users/sessions',
      registrations: 'users/registrations'
    }
end
```

### NextAuth.js（Next.js）
```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import GoogleProvider from 'next-auth/providers/google';
import { compare } from 'bcryptjs';
import { prisma } from '@/lib/prisma';

const handler = NextAuth({
  providers: [
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
        
        if (!user || !await compare(credentials.password, user.password)) {
          return null;
        }
        
        return {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role
        };
      }
    }),
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!
    })
  ],
  
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;
        token.role = user.role;
      }
      return token;
    },
    
    async session({ session, token }) {
      if (session?.user) {
        session.user.id = token.id as string;
        session.user.role = token.role as string;
      }
      return session;
    }
  },
  
  pages: {
    signIn: '/login',
    signOut: '/logout',
    error: '/auth/error'
  },
  
  session: {
    strategy: 'jwt',
    maxAge: 30 * 24 * 60 * 60 // 30日
  }
});

export { handler as GET, handler as POST };

// middleware.ts - 認証が必要なルートの保護
import { withAuth } from 'next-auth/middleware';

export default withAuth({
  pages: {
    signIn: '/login'
  }
});

export const config = {
  matcher: ['/admin/:path*', '/api/admin/:path*']
};
```

## 2. 認可（Authorization）の比較

### Pundit（Rails）
```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def index?
    false
  end

  def show?
    false
  end

  def create?
    false
  end

  def update?
    false
  end

  def destroy?
    false
  end

  class Scope
    def initialize(user, scope)
      @user = user
      @scope = scope
    end

    def resolve
      scope.all
    end
  end
end

# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def index?
    true
  end

  def show?
    true
  end

  def create?
    user.present?
  end

  def update?
    user_is_owner? || user_is_admin?
  end

  def destroy?
    user_is_owner? || user_is_admin?
  end

  def publish?
    user_is_admin? || (user_is_owner? && record.ready_to_publish?)
  end

  private

  def user_is_owner?
    user.present? && user == record.user
  end

  def user_is_admin?
    user.present? && user.admin?
  end

  class Scope < Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user.present?
        scope.where(user: user).or(scope.published)
      else
        scope.published
      end
    end
  end
end

# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :set_post, only: [:show, :update, :destroy, :publish]
  
  def index
    @posts = policy_scope(Post)
    render json: @posts
  end
  
  def show
    authorize @post
    render json: @post
  end
  
  def create
    @post = Post.new(post_params)
    @post.user = current_user
    authorize @post
    
    if @post.save
      render json: @post, status: :created
    else
      render json: @post.errors, status: :unprocessable_entity
    end
  end
  
  def update
    authorize @post
    
    if @post.update(post_params)
      render json: @post
    else
      render json: @post.errors, status: :unprocessable_entity
    end
  end
  
  def destroy
    authorize @post
    @post.destroy
    head :no_content
  end
  
  def publish
    authorize @post, :publish?
    @post.publish!
    render json: @post
  end
  
  private
  
  def set_post
    @post = Post.find(params[:id])
  end
  
  def post_params
    params.require(:post).permit(:title, :content)
  end
end
```

### CASL（Next.js）
```typescript
// lib/abilities.ts
import { AbilityBuilder, PureAbility, AbilityClass } from '@casl/ability';
import { PrismaQuery, Subjects, createPrismaAbility } from '@casl/prisma';

type Actions = 'create' | 'read' | 'update' | 'delete' | 'publish';
type AppSubjects = 'Post' | 'User' | 'all';

export type AppAbility = PureAbility<[Actions, AppSubjects]>;

export function defineAbilitiesFor(user?: any) {
  const { can, cannot, build } = new AbilityBuilder<AppAbility>(
    PureAbility as AbilityClass<AppAbility>
  );

  // ゲストユーザー
  can('read', 'Post', { published: true });

  if (user) {
    // ログインユーザー
    can('create', 'Post');
    can('read', 'Post');
    can('update', 'Post', { userId: user.id });
    can('delete', 'Post', { userId: user.id });

    if (user.role === 'admin') {
      // 管理者
      can('manage', 'all');
    } else if (user.role === 'editor') {
      // 編集者
      can('publish', 'Post');
      can('update', 'Post');
    }
  }

  return build();
}

// hooks/useAbility.ts
import { useSession } from 'next-auth/react';
import { useMemo } from 'react';
import { defineAbilitiesFor } from '@/lib/abilities';

export function useAbility() {
  const { data: session } = useSession();
  
  return useMemo(
    () => defineAbilitiesFor(session?.user),
    [session?.user]
  );
}

// components/Can.tsx
'use client';
import { createContext, useContext, ReactNode } from 'react';
import { createContextualCan } from '@casl/react';

const AbilityContext = createContext<any>(undefined);
export const Can = createContextualCan(AbilityContext.Consumer);

export function AbilityProvider({ 
  children, 
  ability 
}: { 
  children: ReactNode;
  ability: any;
}) {
  return (
    <AbilityContext.Provider value={ability}>
      {children}
    </AbilityContext.Provider>
  );
}

// 使用例：コンポーネント
export function PostActions({ post }: { post: Post }) {
  const ability = useAbility();
  
  return (
    <div>
      <Can I="update" this={post}>
        <button>編集</button>
      </Can>
      
      <Can I="delete" this={post}>
        <button>削除</button>
      </Can>
      
      <Can I="publish" a="Post">
        <button>公開</button>
      </Can>
    </div>
  );
}

// API Route での認可チェック
// app/api/posts/[id]/route.ts
import { getServerSession } from 'next-auth';
import { defineAbilitiesFor } from '@/lib/abilities';
import { ForbiddenError } from '@casl/ability';

export async function PUT(
  request: Request,
  { params }: { params: { id: string } }
) {
  const session = await getServerSession();
  const ability = defineAbilitiesFor(session?.user);
  
  const post = await prisma.post.findUnique({
    where: { id: params.id }
  });
  
  if (!post) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }
  
  try {
    ForbiddenError.from(ability).throwUnlessCan('update', post);
  } catch (error) {
    return NextResponse.json(
      { error: 'Forbidden' },
      { status: 403 }
    );
  }
  
  // 更新処理
  const updatedPost = await prisma.post.update({
    where: { id: params.id },
    data: await request.json()
  });
  
  return NextResponse.json(updatedPost);
}
```

## 3. セッション管理

### Rails（Devise）
```ruby
# セッション設定
Rails.application.config.session_store :cookie_store, 
  key: '_myapp_session',
  expire_after: 30.days

# JWT設定
class User < ApplicationRecord
  include Devise::JWT::RevocationStrategies::JTIMatcher
  
  devise :jwt_authenticatable, jwt_revocation_strategy: self
end

# ログアウト時のトークン無効化
class JwtDenylist < ApplicationRecord
  include Devise::JWT::RevocationStrategies::Denylist
  self.table_name = 'jwt_denylists'
end
```

### Next.js（NextAuth）
```typescript
// セッション管理
import { getServerSession } from 'next-auth';

// サーバーコンポーネント
export default async function Profile() {
  const session = await getServerSession();
  
  if (!session) {
    redirect('/login');
  }
  
  return <div>Welcome {session.user.email}</div>;
}

// クライアントコンポーネント
'use client';
import { useSession, signIn, signOut } from 'next-auth/react';

export function AuthButton() {
  const { data: session, status } = useSession();
  
  if (status === 'loading') return <div>Loading...</div>;
  
  if (session) {
    return (
      <button onClick={() => signOut()}>
        Sign out ({session.user.email})
      </button>
    );
  }
  
  return <button onClick={() => signIn()}>Sign in</button>;
}
```

## 4. マルチファクタ認証（MFA）

### Rails（Devise）
```ruby
# Gemfile
gem 'devise-two-factor'
gem 'rqrcode'

class User < ApplicationRecord
  devise :two_factor_authenticatable,
         otp_secret_encryption_key: ENV['OTP_SECRET_KEY']
  
  def generate_two_factor_qr_code
    issuer = 'MyApp'
    label = "#{issuer}:#{email}"
    qrcode = RQRCode::QRCode.new(otp_provisioning_uri(label, issuer: issuer))
    qrcode.as_svg(module_size: 4)
  end
end
```

### Next.js
```typescript
// lib/two-factor.ts
import speakeasy from 'speakeasy';
import QRCode from 'qrcode';

export async function generateTwoFactorSecret(email: string) {
  const secret = speakeasy.generateSecret({
    name: `MyApp (${email})`,
    length: 32
  });
  
  const qrCode = await QRCode.toDataURL(secret.otpauth_url!);
  
  return {
    secret: secret.base32,
    qrCode
  };
}

export function verifyTwoFactorToken(token: string, secret: string) {
  return speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token,
    window: 2
  });
}
```

## 5. ソーシャルログイン

### Rails（Devise + OmniAuth）
```ruby
# Gemfile
gem 'omniauth'
gem 'omniauth-google-oauth2'
gem 'omniauth-rails_csrf_protection'

# config/initializers/devise.rb
config.omniauth :google_oauth2, 
  ENV['GOOGLE_CLIENT_ID'],
  ENV['GOOGLE_CLIENT_SECRET'],
  scope: 'email,profile'

# app/models/user.rb
devise :omniauthable, omniauth_providers: [:google_oauth2]

def self.from_omniauth(auth)
  where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
    user.email = auth.info.email
    user.name = auth.info.name
    user.password = Devise.friendly_token[0, 20]
  end
end
```

### Next.js（NextAuth）
```typescript
// 既に上記のNextAuth設定で対応済み
// プロバイダー追加例
providers: [
  GoogleProvider({
    clientId: process.env.GOOGLE_CLIENT_ID!,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET!
  }),
  GitHubProvider({
    clientId: process.env.GITHUB_ID!,
    clientSecret: process.env.GITHUB_SECRET!
  }),
  // カスタムOAuth2プロバイダー
  {
    id: "custom-oauth2",
    name: "Custom OAuth2",
    type: "oauth",
    authorization: "https://provider.com/oauth/authorize",
    token: "https://provider.com/oauth/token",
    userinfo: "https://provider.com/oauth/userinfo",
    clientId: process.env.CUSTOM_CLIENT_ID,
    clientSecret: process.env.CUSTOM_CLIENT_SECRET
  }
]
```

## 🎯 実践演習

### 課題: 完全な認証・認可システムの実装
1. メール認証付きユーザー登録
2. ロールベースアクセス制御
3. API トークン管理
4. 監査ログ

## 💡 重要な対応表

| 機能 | Devise/Pundit | NextAuth/CASL |
|------|---------------|---------------|
| 認証方式 | Cookie/JWT | JWT/Session |
| ユーザー登録 | devise :registerable | カスタム実装 |
| パスワードリセット | devise :recoverable | カスタム実装 |
| メール確認 | devise :confirmable | カスタム実装 |
| ロック機能 | devise :lockable | カスタム実装 |
| ポリシー定義 | Punditクラス | CASL abilities |
| スコープ | policy_scope | ability.can |
| 認可チェック | authorize | ability.can |
| ソーシャルログイン | OmniAuth | NextAuth providers |