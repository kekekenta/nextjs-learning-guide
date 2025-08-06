# Next.js 大規模アプリケーションのディレクトリ設計

## 📚 複雑なアプリケーションに対応するディレクトリ構成パターン

大規模なNext.jsアプリケーションを構築する際の、スケーラブルで保守しやすいディレクトリ構造を解説します。

## 1. 基本構造（小〜中規模）

```
my-app/
├── app/                      # App Router
│   ├── (auth)/              # ルートグループ：認証関連
│   │   ├── login/
│   │   ├── register/
│   │   └── layout.tsx
│   ├── (dashboard)/         # ルートグループ：ダッシュボード
│   │   ├── dashboard/
│   │   ├── settings/
│   │   └── layout.tsx
│   ├── api/                 # API Routes
│   │   └── v1/
│   ├── components/          # 共通コンポーネント
│   └── layout.tsx
├── components/              # グローバルコンポーネント
├── lib/                     # ユーティリティ
├── hooks/                   # カスタムフック
├── types/                   # TypeScript型定義
└── public/                  # 静的ファイル
```

## 2. エンタープライズ構造（大規模）

```
enterprise-app/
├── apps/                    # マルチアプリケーション
│   ├── web/                # メインWebアプリ
│   │   ├── app/
│   │   ├── components/
│   │   └── package.json
│   ├── admin/              # 管理画面
│   │   ├── app/
│   │   └── package.json
│   └── mobile/             # モバイル向けWeb
│       └── app/
├── packages/               # 共有パッケージ（monorepo）
│   ├── ui/                # UIコンポーネントライブラリ
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   └── styles/
│   │   └── package.json
│   ├── database/          # Prismaスキーマ・モデル
│   │   ├── prisma/
│   │   └── src/
│   ├── auth/              # 認証ロジック
│   ├── api-client/        # API クライアント
│   └── config/            # 共通設定
├── services/              # マイクロサービス
│   ├── notification/      # 通知サービス
│   ├── payment/          # 決済サービス
│   └── analytics/        # 分析サービス
└── infrastructure/       # インフラ設定
    ├── docker/
    ├── kubernetes/
    └── terraform/
```

## 3. ドメイン駆動設計（DDD）構造

```
ddd-app/
├── app/                           # Next.js App Router
│   ├── (routes)/                 # プレゼンテーション層
│   │   ├── products/
│   │   ├── orders/
│   │   └── users/
│   └── api/
├── src/
│   ├── domain/                   # ドメイン層
│   │   ├── product/
│   │   │   ├── entity/          # エンティティ
│   │   │   │   └── Product.ts
│   │   │   ├── value-object/    # 値オブジェクト
│   │   │   │   ├── ProductId.ts
│   │   │   │   └── Price.ts
│   │   │   ├── repository/      # リポジトリインターフェース
│   │   │   │   └── ProductRepository.ts
│   │   │   └── service/         # ドメインサービス
│   │   │       └── ProductService.ts
│   │   ├── order/
│   │   └── user/
│   ├── application/              # アプリケーション層
│   │   ├── product/
│   │   │   ├── use-case/        # ユースケース
│   │   │   │   ├── CreateProduct.ts
│   │   │   │   ├── UpdateProduct.ts
│   │   │   │   └── DeleteProduct.ts
│   │   │   └── dto/             # データ転送オブジェクト
│   │   │       └── ProductDto.ts
│   │   └── order/
│   ├── infrastructure/          # インフラストラクチャ層
│   │   ├── database/
│   │   │   ├── prisma/
│   │   │   └── repository/      # リポジトリ実装
│   │   │       └── ProductRepositoryImpl.ts
│   │   ├── external/            # 外部サービス
│   │   │   ├── payment/
│   │   │   └── email/
│   │   └── cache/
│   └── presentation/            # プレゼンテーション層
│       ├── components/
│       ├── hooks/
│       └── validators/
```

## 4. 機能別モジュール構造

```
modular-app/
├── app/
│   └── [...routes]/              # 動的ルーティング
├── modules/                      # 機能モジュール
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── RegisterForm.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   └── useSession.ts
│   │   ├── services/
│   │   │   └── authService.ts
│   │   ├── stores/              # 状態管理
│   │   │   └── authStore.ts
│   │   ├── types/
│   │   │   └── auth.types.ts
│   │   └── index.ts             # Public API
│   ├── product/
│   │   ├── components/
│   │   │   ├── ProductList/
│   │   │   │   ├── index.tsx
│   │   │   │   ├── ProductList.tsx
│   │   │   │   ├── ProductList.test.tsx
│   │   │   │   └── ProductList.module.css
│   │   │   └── ProductDetail/
│   │   ├── hooks/
│   │   ├── services/
│   │   ├── stores/
│   │   └── types/
│   ├── cart/
│   └── checkout/
├── shared/                       # 共有モジュール
│   ├── components/
│   ├── hooks/
│   ├── utils/
│   └── types/
└── core/                        # コア機能
    ├── api/
    ├── auth/
    ├── database/
    └── config/
```

## 5. Vertical Slice Architecture

```
vertical-slice-app/
├── app/
│   └── [...routes]/
├── features/                    # 機能スライス
│   ├── create-product/         # 1機能 = 1スライス
│   │   ├── api/               # この機能のAPI
│   │   │   └── route.ts
│   │   ├── components/        # この機能のコンポーネント
│   │   │   └── CreateProductForm.tsx
│   │   ├── hooks/            # この機能のフック
│   │   │   └── useCreateProduct.ts
│   │   ├── services/         # この機能のサービス
│   │   │   └── createProduct.ts
│   │   ├── types/            # この機能の型
│   │   │   └── createProduct.types.ts
│   │   └── index.ts          # エントリーポイント
│   ├── list-products/
│   ├── update-product/
│   ├── delete-product/
│   ├── user-authentication/
│   ├── user-profile/
│   └── shopping-cart/
├── shared/                     # 共有リソース
└── infrastructure/            # インフラ
```

## 6. 実践的な大規模構造の例

### SaaSプラットフォーム
```
saas-platform/
├── apps/
│   ├── web/                          # メインアプリ
│   │   ├── app/
│   │   │   ├── (marketing)/         # マーケティングサイト
│   │   │   │   ├── page.tsx
│   │   │   │   ├── pricing/
│   │   │   │   └── about/
│   │   │   ├── (app)/              # アプリケーション
│   │   │   │   ├── [workspace]/    # ワークスペース
│   │   │   │   │   ├── dashboard/
│   │   │   │   │   ├── projects/
│   │   │   │   │   │   └── [projectId]/
│   │   │   │   │   └── settings/
│   │   │   │   └── layout.tsx
│   │   │   └── api/
│   │   │       ├── v1/
│   │   │       └── webhooks/
│   │   └── modules/
│   │       ├── workspace/
│   │       ├── project/
│   │       ├── billing/
│   │       └── analytics/
│   └── admin/                       # 管理画面
│       └── app/
├── packages/                        # 共有パッケージ
│   ├── ui/                         # デザインシステム
│   │   ├── primitives/            # 基本コンポーネント
│   │   ├── compounds/             # 複合コンポーネント
│   │   └── templates/             # テンプレート
│   ├── database/
│   │   ├── prisma/
│   │   └── seeds/
│   ├── auth/
│   ├── email/
│   └── queue/                     # ジョブキュー
├── services/                       # バックエンドサービス
│   ├── api/                      # REST API
│   ├── graphql/                  # GraphQL
│   └── workers/                  # バックグラウンドワーカー
└── tools/                         # 開発ツール
    ├── scripts/
    └── generators/                # コードジェネレーター
```

### Eコマースプラットフォーム
```
ecommerce/
├── apps/
│   ├── storefront/               # 店舗フロント
│   │   ├── app/
│   │   │   ├── (shop)/
│   │   │   │   ├── products/
│   │   │   │   ├── categories/
│   │   │   │   └── cart/
│   │   │   ├── (account)/
│   │   │   │   ├── orders/
│   │   │   │   └── profile/
│   │   │   └── (checkout)/
│   │   │       └── checkout/
│   │   └── features/
│   │       ├── catalog/
│   │       ├── cart/
│   │       ├── checkout/
│   │       └── search/
│   ├── vendor/                  # 販売者管理
│   │   └── app/
│   └── admin/                   # 管理者
│       └── app/
├── domains/                     # ビジネスドメイン
│   ├── catalog/
│   │   ├── product/
│   │   ├── category/
│   │   └── inventory/
│   ├── order/
│   │   ├── cart/
│   │   ├── checkout/
│   │   └── fulfillment/
│   ├── customer/
│   └── payment/
└── shared/
    ├── ui/
    ├── utils/
    └── types/
```

## 7. ファイル命名規則

### コンポーネント
```
components/
├── Button/
│   ├── Button.tsx              # コンポーネント本体
│   ├── Button.test.tsx         # テスト
│   ├── Button.stories.tsx      # Storybook
│   ├── Button.module.css       # スタイル
│   └── index.ts               # エクスポート
```

### 機能モジュール
```
features/
├── user-profile/
│   ├── api/
│   │   └── getUserProfile.ts
│   ├── components/
│   │   └── UserProfile.tsx
│   ├── hooks/
│   │   └── useUserProfile.ts
│   ├── types/
│   │   └── userProfile.types.ts
│   └── index.ts
```

## 8. ベストプラクティス

### 1. コロケーション原則
```typescript
// 関連するファイルは近くに配置
// ❌ 悪い例
src/
├── components/ProductCard.tsx
├── hooks/useProduct.ts
├── styles/productCard.css
└── types/product.ts

// ✅ 良い例
src/
└── features/
    └── product/
        ├── ProductCard.tsx
        ├── useProduct.ts
        ├── productCard.css
        └── product.types.ts
```

### 2. バレルエクスポート
```typescript
// features/auth/index.ts
export { LoginForm } from './components/LoginForm';
export { useAuth } from './hooks/useAuth';
export type { AuthUser } from './types/auth.types';

// 使用側
import { LoginForm, useAuth, AuthUser } from '@/features/auth';
```

### 3. 絶対パスインポート
```typescript
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@/components/*": ["src/components/*"],
      "@/features/*": ["src/features/*"],
      "@/lib/*": ["src/lib/*"]
    }
  }
}

// 使用例
import { Button } from '@/components/Button';
import { useAuth } from '@/features/auth';
```

## 9. スケーリング戦略

### 段階的な成長
```
1. 初期（〜10画面）
   → シンプルな app/ 構造

2. 成長期（10〜50画面）
   → 機能別モジュール構造

3. 成熟期（50画面〜）
   → ドメイン駆動設計 or Vertical Slice

4. エンタープライズ（複数アプリ）
   → Monorepo + マイクロフロントエンド
```

### リファクタリング指針
```typescript
// 3回以上使われたら共通化
// 2箇所で使用 → そのまま
// 3箇所で使用 → shared/ へ移動
// 5箇所以上 → packages/ として独立
```

## まとめ

### 選択基準

| 規模 | 推奨構造 | 特徴 |
|------|---------|------|
| 小規模（〜20画面） | 基本構造 | シンプル、学習コスト低 |
| 中規模（20〜100画面） | 機能別モジュール | 機能単位で整理、チーム開発向き |
| 大規模（100画面〜） | DDD or Vertical Slice | ビジネスロジック重視、独立性高 |
| エンタープライズ | Monorepo | 複数アプリ、コード共有 |

### Rails開発者への注意点

Railsの規約（Convention over Configuration）と異なり、Next.jsは自由度が高いため、プロジェクト開始時にチームで構造を決定することが重要です。

```ruby
# Rails: 決まった構造
app/
├── controllers/
├── models/
└── views/
```

```typescript
// Next.js: 自由に設計可能
// チームで規約を決める必要がある
```