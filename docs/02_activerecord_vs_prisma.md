# ActiveRecord vs Prisma 完全比較ガイド

## 📚 学習目標（1.5時間）
ActiveRecordの知識を活かしてPrismaをマスターする

## 1. 基本概念の比較

### ActiveRecord（Rails）
- ORMパターン: Active Recordパターン
- モデル = テーブル = クラス
- マイグレーション駆動

### Prisma
- ORMパターン: Data Mapperパターン
- スキーマファースト
- 型安全性重視

## 2. スキーマ定義

### ActiveRecord
```ruby
# db/migrate/20240101000000_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.string :name
      t.integer :age
      t.references :company, foreign_key: true
      t.timestamps
    end
    
    add_index :users, :email, unique: true
  end
end

# app/models/user.rb
class User < ApplicationRecord
  belongs_to :company
  has_many :posts, dependent: :destroy
  has_and_belongs_to_many :tags
  
  validates :email, presence: true, uniqueness: true
  validates :age, numericality: { greater_than: 0 }
end
```

### Prisma
```prisma
// prisma/schema.prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  age       Int?
  companyId String
  company   Company  @relation(fields: [companyId], references: [id])
  posts     Post[]
  tags      Tag[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@index([email])
}

model Company {
  id    String @id @default(cuid())
  name  String
  users User[]
}

model Post {
  id       String   @id @default(cuid())
  title    String
  content  String?
  userId   String
  user     User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  tags     Tag[]
  createdAt DateTime @default(now())
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]
  users User[]
}
```

## 3. CRUD操作の比較

### 作成（Create）

```ruby
# ActiveRecord
user = User.create!(
  email: "test@example.com",
  name: "Test User",
  company: company
)

# または
user = User.new(email: "test@example.com")
user.name = "Test User"
user.save!

# 関連付けと同時に作成
user = User.create!(
  email: "test@example.com",
  posts_attributes: [
    { title: "First Post" }
  ]
)
```

```typescript
// Prisma
const user = await prisma.user.create({
  data: {
    email: "test@example.com",
    name: "Test User",
    company: {
      connect: { id: companyId }
    }
  }
});

// 関連付けと同時に作成
const user = await prisma.user.create({
  data: {
    email: "test@example.com",
    posts: {
      create: [
        { title: "First Post" }
      ]
    }
  },
  include: {
    posts: true
  }
});
```

### 読み取り（Read）

```ruby
# ActiveRecord
# 単一レコード
user = User.find(1)
user = User.find_by(email: "test@example.com")
user = User.find_by!(email: "test@example.com") # 見つからない場合は例外

# 複数レコード
users = User.all
users = User.where(age: 18..65)
users = User.where("age > ?", 18)
users = User.order(created_at: :desc).limit(10)

# 関連データの取得
user = User.includes(:posts, :company).find(1)
users = User.joins(:posts).where(posts: { published: true })

# 集計
User.count
User.average(:age)
User.group(:company_id).count
```

```typescript
// Prisma
// 単一レコード
const user = await prisma.user.findUnique({
  where: { id: "1" }
});

const user = await prisma.user.findFirst({
  where: { email: "test@example.com" }
});

const user = await prisma.user.findUniqueOrThrow({
  where: { email: "test@example.com" }
});

// 複数レコード
const users = await prisma.user.findMany();

const users = await prisma.user.findMany({
  where: {
    age: {
      gte: 18,
      lte: 65
    }
  }
});

const users = await prisma.user.findMany({
  orderBy: { createdAt: 'desc' },
  take: 10
});

// 関連データの取得
const user = await prisma.user.findUnique({
  where: { id: "1" },
  include: {
    posts: true,
    company: true
  }
});

// 集計
const count = await prisma.user.count();
const avgAge = await prisma.user.aggregate({
  _avg: { age: true }
});
const groupCount = await prisma.user.groupBy({
  by: ['companyId'],
  _count: true
});
```

### 更新（Update）

```ruby
# ActiveRecord
user = User.find(1)
user.update!(name: "New Name")

# または
user.name = "New Name"
user.save!

# 一括更新
User.where(company_id: 1).update_all(active: true)

# 条件付き更新
user.update!(name: "New Name") if user.age > 18
```

```typescript
// Prisma
const user = await prisma.user.update({
  where: { id: "1" },
  data: { name: "New Name" }
});

// 一括更新
const result = await prisma.user.updateMany({
  where: { companyId: "1" },
  data: { active: true }
});

// upsert（存在しなければ作成、存在すれば更新）
const user = await prisma.user.upsert({
  where: { email: "test@example.com" },
  update: { name: "Updated Name" },
  create: { 
    email: "test@example.com",
    name: "New User"
  }
});
```

### 削除（Delete）

```ruby
# ActiveRecord
user = User.find(1)
user.destroy!

# 一括削除
User.where(active: false).destroy_all
User.delete_all # callbackをスキップ

# 関連レコードも削除（dependent: :destroy）
user.destroy! # postsも自動削除
```

```typescript
// Prisma
const user = await prisma.user.delete({
  where: { id: "1" }
});

// 一括削除
const result = await prisma.user.deleteMany({
  where: { active: false }
});

// 関連レコードも削除（onDelete: Cascade）
const user = await prisma.user.delete({
  where: { id: "1" }
}); // postsも自動削除（スキーマで設定）
```

## 4. 高度なクエリパターン

### スコープ / フィルタリング

```ruby
# ActiveRecord
class User < ApplicationRecord
  scope :active, -> { where(active: true) }
  scope :adults, -> { where("age >= ?", 18) }
  scope :with_posts, -> { joins(:posts).distinct }
end

User.active.adults.with_posts
```

```typescript
// Prisma（ヘルパー関数として実装）
const activeUsers = (where = {}) => ({
  ...where,
  active: true
});

const adultUsers = (where = {}) => ({
  ...where,
  age: { gte: 18 }
});

const users = await prisma.user.findMany({
  where: {
    ...activeUsers(),
    ...adultUsers()
  },
  include: { posts: true }
});
```

### トランザクション

```ruby
# ActiveRecord
ActiveRecord::Base.transaction do
  user = User.create!(email: "test@example.com")
  post = user.posts.create!(title: "First Post")
  # エラーが発生したら自動ロールバック
end
```

```typescript
// Prisma
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: { email: "test@example.com" }
  });
  
  const post = await tx.post.create({
    data: {
      title: "First Post",
      userId: user.id
    }
  });
  
  return { user, post };
});

// または複数の操作を一括実行
const [user, post] = await prisma.$transaction([
  prisma.user.create({ data: { email: "test@example.com" } }),
  prisma.post.create({ data: { title: "First Post", userId: "..." } })
]);
```

### Raw SQL

```ruby
# ActiveRecord
results = ActiveRecord::Base.connection.execute(
  "SELECT * FROM users WHERE age > 18"
)

User.find_by_sql("SELECT * FROM users WHERE age > ?", 18)
```

```typescript
// Prisma
const users = await prisma.$queryRaw`
  SELECT * FROM "User" WHERE age > ${18}
`;

// 型安全なRaw SQL
interface UserResult {
  id: string;
  email: string;
  age: number;
}

const users = await prisma.$queryRaw<UserResult[]>`
  SELECT * FROM "User" WHERE age > ${18}
`;
```

## 5. マイグレーション

### ActiveRecord
```bash
# マイグレーション作成
rails generate migration AddAgeToUsers age:integer

# マイグレーション実行
rails db:migrate

# ロールバック
rails db:rollback

# スキーマ確認
rails db:schema:dump
```

### Prisma
```bash
# スキーマ変更後、マイグレーション作成
npx prisma migrate dev --name add_age_to_users

# 本番環境へのマイグレーション
npx prisma migrate deploy

# スキーマのリセット（開発環境）
npx prisma migrate reset

# データベースとの同期確認
npx prisma db push
```

## 6. バリデーション

### ActiveRecord（モデル内）
```ruby
class User < ApplicationRecord
  validates :email, presence: true, uniqueness: true
  validates :age, numericality: { greater_than: 0 }
  validate :custom_validation
  
  private
  def custom_validation
    errors.add(:email, "Invalid domain") unless email.ends_with?("@example.com")
  end
end
```

### Prisma（アプリケーション層）
```typescript
// Zodなどのバリデーションライブラリと組み合わせる
import { z } from 'zod';

const UserSchema = z.object({
  email: z.string().email(),
  age: z.number().positive(),
  name: z.string().optional()
});

// 使用例
const validatedData = UserSchema.parse(inputData);
const user = await prisma.user.create({
  data: validatedData
});
```

## 7. パフォーマンス最適化

### N+1問題の解決

```ruby
# ActiveRecord
# N+1問題あり
users = User.all
users.each { |user| puts user.posts.count }

# 解決策
users = User.includes(:posts)
users = User.joins(:posts).group("users.id")
```

```typescript
// Prisma
// N+1問題あり
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { userId: user.id }
  });
}

// 解決策
const users = await prisma.user.findMany({
  include: {
    posts: true,
    _count: {
      select: { posts: true }
    }
  }
});
```

## 🎯 実践演習

### 課題1: 複雑なクエリの実装
以下のActiveRecordクエリをPrismaで実装:
```ruby
User.joins(:posts)
    .where(posts: { published: true })
    .where("users.age > ?", 18)
    .group("users.id")
    .having("COUNT(posts.id) > ?", 5)
    .order(created_at: :desc)
```

### 課題2: トランザクション処理
ユーザー登録と同時にプロフィール、初期投稿を作成するトランザクション処理を実装

## 💡 重要な違いのまとめ

| 機能 | ActiveRecord | Prisma |
|------|--------------|--------|
| パターン | Active Record | Data Mapper |
| スキーマ定義 | マイグレーション | schema.prisma |
| 型安全性 | 弱い | 強い（TypeScript） |
| バリデーション | モデル内蔵 | 外部ライブラリ |
| クエリビルダー | メソッドチェーン | オブジェクト形式 |
| Raw SQL | find_by_sql | $queryRaw |
| N+1対策 | includes/joins | include |
| トランザクション | ブロック | $transaction |

## 📖 必読リソース
- [Prisma Documentation](https://www.prisma.io/docs)
- [Prisma vs ActiveRecord](https://www.prisma.io/docs/concepts/more/comparisons/prisma-and-activerecord)
- [Prisma Client API Reference](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference)