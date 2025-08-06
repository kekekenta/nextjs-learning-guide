# データベース設計とマイグレーション完全ガイド

## 📚 学習目標（45分）
MySQLからPostgreSQLへ、ActiveRecordからPrismaへのマイグレーション戦略

## 1. データベース設計の違い

### MySQL vs PostgreSQL

```sql
-- MySQL
CREATE TABLE users (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  name VARCHAR(100),
  metadata JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_email (email)
);

-- PostgreSQL
CREATE TABLE users (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  name VARCHAR(100),
  metadata JSONB,
  tags TEXT[],
  created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_metadata ON users USING GIN(metadata);
```

### PostgreSQL特有の機能活用

```prisma
// prisma/schema.prisma
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  metadata  Json?    // JSONB型として保存
  tags      String[] // 配列型
  createdAt DateTime @default(now()) @db.Timestamptz
  
  @@index([email])
  @@index([metadata(ops: JsonbPathOps)], type: Gin)
}

// 使用例
const users = await prisma.user.findMany({
  where: {
    metadata: {
      path: ['settings', 'theme'],
      equals: 'dark'
    },
    tags: {
      has: 'developer'
    }
  }
});
```

## 2. マイグレーション戦略

### Rails マイグレーション
```ruby
# db/migrate/20240101000000_create_users.rb
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.string :name
      t.jsonb :metadata
      t.string :tags, array: true, default: []
      t.references :company, foreign_key: true
      
      t.timestamps
    end
    
    add_index :users, :email, unique: true
    add_index :users, :metadata, using: :gin
    add_index :users, :tags, using: :gin
  end
end

# ロールバック可能な変更
class AddStatusToUsers < ActiveRecord::Migration[7.0]
  def up
    add_column :users, :status, :string, default: 'active'
    User.update_all(status: 'active')
  end
  
  def down
    remove_column :users, :status
  end
end
```

### Prisma マイグレーション
```bash
# 初期セットアップ
npx prisma init
npx prisma db pull # 既存DBからスキーマ生成

# マイグレーション作成・実行
npx prisma migrate dev --name create_users

# 本番環境
npx prisma migrate deploy

# マイグレーションのリセット（開発環境）
npx prisma migrate reset
```

```prisma
// カスタムマイグレーション例
// prisma/migrations/20240101000000_add_status/migration.sql
ALTER TABLE "User" ADD COLUMN "status" TEXT DEFAULT 'active';
UPDATE "User" SET "status" = 'active' WHERE "status" IS NULL;
```

## 3. インデックス戦略

### Rails
```ruby
class AddIndexesToUsers < ActiveRecord::Migration[7.0]
  def change
    # 単一カラムインデックス
    add_index :users, :email
    
    # 複合インデックス
    add_index :users, [:company_id, :created_at]
    
    # 部分インデックス
    add_index :users, :email, where: "active = true"
    
    # 全文検索インデックス（PostgreSQL）
    enable_extension 'pg_trgm'
    add_index :users, :name, using: :gin, opclass: :gin_trgm_ops
  end
end
```

### Prisma
```prisma
model User {
  id        String   @id
  email     String
  name      String?
  companyId String
  active    Boolean  @default(true)
  createdAt DateTime @default(now())
  
  // 単一インデックス
  @@index([email])
  
  // 複合インデックス
  @@index([companyId, createdAt])
  
  // ユニーク複合インデックス
  @@unique([email, companyId])
  
  // 全文検索（Raw SQLで追加）
}
```

## 4. リレーション設計

### 1対多関係
```ruby
# Rails
class Company < ApplicationRecord
  has_many :users, dependent: :destroy
end

class User < ApplicationRecord
  belongs_to :company
end
```

```prisma
// Prisma
model Company {
  id    String @id @default(uuid())
  name  String
  users User[]
}

model User {
  id        String  @id @default(uuid())
  companyId String
  company   Company @relation(fields: [companyId], references: [id], onDelete: Cascade)
}
```

### 多対多関係
```ruby
# Rails
class User < ApplicationRecord
  has_and_belongs_to_many :projects
  # または
  has_many :user_projects
  has_many :projects, through: :user_projects
end
```

```prisma
// Prisma - 暗黙的な中間テーブル
model User {
  id       String    @id
  projects Project[]
}

model Project {
  id    String @id
  users User[]
}

// Prisma - 明示的な中間テーブル
model User {
  id           String        @id
  userProjects UserProject[]
}

model Project {
  id           String        @id
  userProjects UserProject[]
}

model UserProject {
  id        String   @id @default(uuid())
  userId    String
  projectId String
  role      String
  joinedAt  DateTime @default(now())
  
  user    User    @relation(fields: [userId], references: [id])
  project Project @relation(fields: [projectId], references: [id])
  
  @@unique([userId, projectId])
}
```

## 5. データシーディング

### Rails
```ruby
# db/seeds.rb
Company.create!(name: 'Tech Corp')

10.times do |i|
  User.create!(
    email: "user#{i}@example.com",
    name: Faker::Name.name,
    company: Company.first
  )
end

# 実行
rails db:seed
```

### Prisma
```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

async function main() {
  const company = await prisma.company.create({
    data: { name: 'Tech Corp' }
  });
  
  await prisma.user.createMany({
    data: Array.from({ length: 10 }, (_, i) => ({
      email: `user${i}@example.com`,
      name: `User ${i}`,
      companyId: company.id
    }))
  });
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());

// package.json
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}

// 実行
npx prisma db seed
```

## 6. データベース最適化

### クエリ最適化
```ruby
# Rails - N+1問題の解決
users = User.includes(:company, :posts).where(active: true)

# クエリ分析
User.where(active: true).explain
```

```typescript
// Prisma - N+1問題の解決
const users = await prisma.user.findMany({
  where: { active: true },
  include: {
    company: true,
    posts: true
  }
});

// クエリログ有効化
const prisma = new PrismaClient({
  log: ['query', 'info', 'warn', 'error']
});
```

### コネクションプーリング
```ruby
# Rails - database.yml
production:
  pool: 25
  timeout: 5000
  reaping_frequency: 10
```

```typescript
// Prisma - DATABASE_URL
DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=25&pool_timeout=10"

// PrismaClient設定
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL
    }
  }
});
```

## 7. トランザクション管理

### Rails
```ruby
ActiveRecord::Base.transaction do
  user = User.create!(email: 'test@example.com')
  Profile.create!(user: user, bio: 'Test bio')
  
  # ネストされたトランザクション
  ActiveRecord::Base.transaction(requires_new: true) do
    Audit.create!(action: 'user_created', user: user)
  end
end
```

### Prisma
```typescript
// インタラクティブトランザクション
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({
    data: { email: 'test@example.com' }
  });
  
  const profile = await tx.profile.create({
    data: {
      userId: user.id,
      bio: 'Test bio'
    }
  });
  
  return { user, profile };
});

// バッチトランザクション
const [user, profile] = await prisma.$transaction([
  prisma.user.create({ data: { email: 'test@example.com' } }),
  prisma.profile.create({ data: { userId: '...', bio: 'Test bio' } })
]);

// 分離レベル設定
await prisma.$transaction(
  async (tx) => {
    // 処理
  },
  {
    isolationLevel: 'Serializable',
    maxWait: 5000,
    timeout: 10000
  }
);
```

## 8. バックアップとリストア

### PostgreSQL基本コマンド
```bash
# バックアップ
pg_dump -h localhost -U user -d database > backup.sql
pg_dump --format=custom -h localhost -U user -d database > backup.dump

# リストア
psql -h localhost -U user -d database < backup.sql
pg_restore -h localhost -U user -d database backup.dump

# Prismaでのバックアップ前準備
npx prisma migrate deploy # 最新のマイグレーション適用
```

## 🎯 実践演習

### 課題: 既存RailsアプリのPrismaへの移行
1. 既存のRailsスキーマをPrismaスキーマに変換
2. データ移行スクリプトの作成
3. インデックス最適化

## 💡 重要なポイント

| 項目 | Rails/MySQL | Prisma/PostgreSQL |
|------|-------------|-------------------|
| ID生成 | AUTO_INCREMENT | UUID |
| JSON | JSON | JSONB |
| 配列 | シリアライズ | ネイティブ配列 |
| 全文検索 | LIKE/MATCH | GIN/GiST |
| マイグレーション | Railsコマンド | Prismaコマンド |
| シード | seeds.rb | seed.ts |