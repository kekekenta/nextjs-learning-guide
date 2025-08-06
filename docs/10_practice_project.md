# 実践演習プロジェクト - タスク管理SaaS構築

## 📚 プロジェクト概要
Rails経験を活かしてNext.js/Prisma/PostgreSQLでタスク管理SaaSを6時間で構築

## プロジェクト構成

```
kizna-task-manager/
├── app/                      # Next.js App Router
│   ├── (auth)/              # 認証関連ページ
│   ├── (dashboard)/         # ダッシュボード
│   ├── api/                 # API Routes
│   └── components/          # 共通コンポーネント
├── prisma/
│   ├── schema.prisma        # データベーススキーマ
│   └── migrations/          # マイグレーション
├── lib/                     # ユーティリティ
├── hooks/                   # カスタムフック
└── docker-compose.yml       # 開発環境
```

## Phase 1: 環境構築とDB設計（30分）

### 1.1 プロジェクトセットアップ

```bash
# プロジェクト作成
npx create-next-app@latest kizna-task-manager \
  --typescript --tailwind --app --src-dir=false

cd kizna-task-manager

# 必要なパッケージインストール
pnpm add @prisma/client prisma
pnpm add next-auth @next-auth/prisma-adapter
pnpm add @casl/ability @casl/prisma @casl/react
pnpm add zod react-hook-form @hookform/resolvers
pnpm add axios swr
pnpm add bcryptjs
pnpm add -D @types/bcryptjs

# Prisma初期化
npx prisma init
```

### 1.2 データベーススキーマ設計

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  password      String
  name          String?
  role          Role      @default(USER)
  emailVerified DateTime?
  image         String?
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  accounts      Account[]
  sessions      Session[]
  workspaces    WorkspaceMember[]
  createdTasks  Task[]    @relation("TaskCreator")
  assignedTasks Task[]    @relation("TaskAssignee")
  comments      Comment[]
  activities    Activity[]
  
  @@index([email])
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model Workspace {
  id          String   @id @default(cuid())
  name        String
  slug        String   @unique
  description String?
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  members     WorkspaceMember[]
  projects    Project[]
  tags        Tag[]
  webhooks    Webhook[]
  
  @@index([slug])
}

model WorkspaceMember {
  id          String          @id @default(cuid())
  userId      String
  workspaceId String
  role        WorkspaceRole   @default(MEMBER)
  joinedAt    DateTime        @default(now())
  
  user        User            @relation(fields: [userId], references: [id], onDelete: Cascade)
  workspace   Workspace       @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  
  @@unique([userId, workspaceId])
  @@index([workspaceId])
}

model Project {
  id          String    @id @default(cuid())
  name        String
  description String?
  workspaceId String
  status      ProjectStatus @default(ACTIVE)
  startDate   DateTime?
  endDate     DateTime?
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  
  workspace   Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  tasks       Task[]
  
  @@index([workspaceId])
}

model Task {
  id          String     @id @default(cuid())
  title       String
  description String?
  projectId   String
  creatorId   String
  assigneeId  String?
  priority    Priority   @default(MEDIUM)
  status      TaskStatus @default(TODO)
  dueDate     DateTime?
  
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  completedAt DateTime?
  
  project     Project    @relation(fields: [projectId], references: [id], onDelete: Cascade)
  creator     User       @relation("TaskCreator", fields: [creatorId], references: [id])
  assignee    User?      @relation("TaskAssignee", fields: [assigneeId], references: [id])
  tags        Tag[]
  comments    Comment[]
  activities  Activity[]
  attachments Attachment[]
  
  @@index([projectId])
  @@index([assigneeId])
  @@index([status])
}

model Tag {
  id          String    @id @default(cuid())
  name        String
  color       String
  workspaceId String
  
  workspace   Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  tasks       Task[]
  
  @@unique([name, workspaceId])
}

model Comment {
  id        String   @id @default(cuid())
  content   String
  taskId    String
  userId    String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  task      Task     @relation(fields: [taskId], references: [id], onDelete: Cascade)
  user      User     @relation(fields: [userId], references: [id])
  
  @@index([taskId])
}

model Activity {
  id          String       @id @default(cuid())
  type        ActivityType
  description String
  taskId      String
  userId      String
  metadata    Json?
  createdAt   DateTime     @default(now())
  
  task        Task         @relation(fields: [taskId], references: [id], onDelete: Cascade)
  user        User         @relation(fields: [userId], references: [id])
  
  @@index([taskId])
  @@index([createdAt])
}

model Attachment {
  id        String   @id @default(cuid())
  filename  String
  url       String
  size      Int
  mimeType  String
  taskId    String
  createdAt DateTime @default(now())
  
  task      Task     @relation(fields: [taskId], references: [id], onDelete: Cascade)
}

model Webhook {
  id          String   @id @default(cuid())
  url         String
  events      String[] // ['task.created', 'task.completed']
  workspaceId String
  active      Boolean  @default(true)
  secret      String
  createdAt   DateTime @default(now())
  
  workspace   Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
}

enum Role {
  USER
  ADMIN
}

enum WorkspaceRole {
  OWNER
  ADMIN
  MEMBER
  VIEWER
}

enum ProjectStatus {
  PLANNING
  ACTIVE
  ON_HOLD
  COMPLETED
  ARCHIVED
}

enum TaskStatus {
  TODO
  IN_PROGRESS
  IN_REVIEW
  DONE
  CANCELLED
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}

enum ActivityType {
  TASK_CREATED
  TASK_UPDATED
  TASK_COMPLETED
  TASK_ASSIGNED
  COMMENT_ADDED
  ATTACHMENT_ADDED
}
```

## Phase 2: 認証・認可実装（1時間）

### 2.1 NextAuth設定

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import GoogleProvider from 'next-auth/providers/google';
import { PrismaAdapter } from '@next-auth/prisma-adapter';
import { compare } from 'bcryptjs';
import { prisma } from '@/lib/prisma';

const handler = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" }
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          throw new Error('メールアドレスとパスワードを入力してください');
        }
        
        const user = await prisma.user.findUnique({
          where: { email: credentials.email }
        });
        
        if (!user || !await compare(credentials.password, user.password)) {
          throw new Error('メールアドレスまたはパスワードが正しくありません');
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
    async session({ session, token }) {
      if (session?.user) {
        session.user.id = token.sub!;
        session.user.role = token.role as string;
      }
      return session;
    },
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role;
      }
      return token;
    }
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
  session: {
    strategy: 'jwt'
  }
});

export { handler as GET, handler as POST };
```

### 2.2 CASL認可設定

```typescript
// lib/abilities.ts
import { defineAbility } from '@casl/ability';
import { PrismaQuery, Subjects } from '@casl/prisma';

type Actions = 'create' | 'read' | 'update' | 'delete' | 'manage';
type AppSubjects = 
  | 'Task' 
  | 'Project' 
  | 'Workspace' 
  | 'Comment' 
  | 'all';

export function defineAbilitiesFor(user: any, workspace?: any) {
  return defineAbility((can, cannot) => {
    if (user.role === 'ADMIN') {
      can('manage', 'all');
      return;
    }
    
    if (workspace) {
      const membership = workspace.members.find(
        (m: any) => m.userId === user.id
      );
      
      if (!membership) return;
      
      switch (membership.role) {
        case 'OWNER':
          can('manage', 'Workspace', { id: workspace.id });
          can('manage', 'Project', { workspaceId: workspace.id });
          can('manage', 'Task', { 'project.workspaceId': workspace.id });
          break;
          
        case 'ADMIN':
          can('manage', 'Project', { workspaceId: workspace.id });
          can('manage', 'Task', { 'project.workspaceId': workspace.id });
          cannot('delete', 'Workspace');
          break;
          
        case 'MEMBER':
          can('read', 'Workspace', { id: workspace.id });
          can('read', 'Project', { workspaceId: workspace.id });
          can('create', 'Task');
          can('update', 'Task', { assigneeId: user.id });
          can('update', 'Task', { creatorId: user.id });
          break;
          
        case 'VIEWER':
          can('read', 'Workspace', { id: workspace.id });
          can('read', 'Project', { workspaceId: workspace.id });
          can('read', 'Task');
          break;
      }
    }
    
    // 共通権限
    can('create', 'Comment');
    can('update', 'Comment', { userId: user.id });
    can('delete', 'Comment', { userId: user.id });
  });
}
```

## Phase 3: API実装（1.5時間）

### 3.1 タスクAPI

```typescript
// app/api/workspaces/[workspaceId]/tasks/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { z } from 'zod';
import { prisma } from '@/lib/prisma';
import { defineAbilitiesFor } from '@/lib/abilities';

const CreateTaskSchema = z.object({
  title: z.string().min(1).max(255),
  description: z.string().optional(),
  projectId: z.string(),
  assigneeId: z.string().optional(),
  priority: z.enum(['LOW', 'MEDIUM', 'HIGH', 'URGENT']),
  dueDate: z.string().datetime().optional(),
  tags: z.array(z.string()).optional()
});

export async function GET(
  request: NextRequest,
  { params }: { params: { workspaceId: string } }
) {
  const session = await getServerSession();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  const searchParams = request.nextUrl.searchParams;
  const projectId = searchParams.get('projectId');
  const status = searchParams.get('status');
  const assigneeId = searchParams.get('assigneeId');
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '20');
  
  const where: any = {
    project: {
      workspaceId: params.workspaceId
    }
  };
  
  if (projectId) where.projectId = projectId;
  if (status) where.status = status;
  if (assigneeId) where.assigneeId = assigneeId;
  
  const [tasks, total] = await Promise.all([
    prisma.task.findMany({
      where,
      include: {
        creator: { select: { id: true, name: true, email: true } },
        assignee: { select: { id: true, name: true, email: true } },
        tags: true,
        _count: { select: { comments: true, attachments: true } }
      },
      orderBy: [
        { priority: 'desc' },
        { createdAt: 'desc' }
      ],
      skip: (page - 1) * limit,
      take: limit
    }),
    prisma.task.count({ where })
  ]);
  
  return NextResponse.json({
    data: tasks,
    meta: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
}

export async function POST(
  request: NextRequest,
  { params }: { params: { workspaceId: string } }
) {
  const session = await getServerSession();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  try {
    const body = await request.json();
    const validatedData = CreateTaskSchema.parse(body);
    
    // 権限チェック
    const workspace = await prisma.workspace.findUnique({
      where: { id: params.workspaceId },
      include: { members: true }
    });
    
    const ability = defineAbilitiesFor(session.user, workspace);
    if (!ability.can('create', 'Task')) {
      return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
    }
    
    // タスク作成
    const task = await prisma.task.create({
      data: {
        ...validatedData,
        creatorId: session.user.id,
        tags: validatedData.tags ? {
          connect: validatedData.tags.map(id => ({ id }))
        } : undefined
      },
      include: {
        creator: true,
        assignee: true,
        tags: true
      }
    });
    
    // アクティビティ記録
    await prisma.activity.create({
      data: {
        type: 'TASK_CREATED',
        description: `タスク「${task.title}」を作成しました`,
        taskId: task.id,
        userId: session.user.id
      }
    });
    
    // Webhook通知
    const webhooks = await prisma.webhook.findMany({
      where: {
        workspaceId: params.workspaceId,
        active: true,
        events: { has: 'task.created' }
      }
    });
    
    for (const webhook of webhooks) {
      // 非同期でWebhook送信
      sendWebhook(webhook, { event: 'task.created', data: task });
    }
    
    return NextResponse.json(task, { status: 201 });
  } catch (error) {
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
}

async function sendWebhook(webhook: any, payload: any) {
  // Webhook送信処理（非同期）
  fetch(webhook.url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Webhook-Secret': webhook.secret
    },
    body: JSON.stringify(payload)
  }).catch(console.error);
}
```

## Phase 4: フロントエンド実装（1.5時間）

### 4.1 タスク一覧画面

```typescript
// app/(dashboard)/[workspaceSlug]/projects/[projectId]/page.tsx
import { notFound } from 'next/navigation';
import { getServerSession } from 'next-auth';
import { prisma } from '@/lib/prisma';
import TaskList from './components/TaskList';
import TaskFilters from './components/TaskFilters';
import CreateTaskButton from './components/CreateTaskButton';

export default async function ProjectPage({
  params
}: {
  params: { workspaceSlug: string; projectId: string }
}) {
  const session = await getServerSession();
  if (!session) notFound();
  
  const workspace = await prisma.workspace.findUnique({
    where: { slug: params.workspaceSlug },
    include: {
      members: {
        where: { userId: session.user.id }
      }
    }
  });
  
  if (!workspace || workspace.members.length === 0) {
    notFound();
  }
  
  const project = await prisma.project.findUnique({
    where: { id: params.projectId },
    include: {
      tasks: {
        include: {
          assignee: true,
          tags: true,
          _count: {
            select: { comments: true }
          }
        }
      }
    }
  });
  
  if (!project) notFound();
  
  return (
    <div className="container mx-auto px-4 py-8">
      <div className="flex justify-between items-center mb-8">
        <h1 className="text-3xl font-bold">{project.name}</h1>
        <CreateTaskButton 
          projectId={project.id} 
          workspaceId={workspace.id}
        />
      </div>
      
      <TaskFilters />
      
      <TaskList 
        initialTasks={project.tasks}
        projectId={project.id}
        workspaceId={workspace.id}
      />
    </div>
  );
}
```

### 4.2 タスクコンポーネント

```typescript
// app/(dashboard)/[workspaceSlug]/projects/[projectId]/components/TaskList.tsx
'use client';

import { useState, useEffect } from 'react';
import useSWR from 'swr';
import TaskCard from './TaskCard';
import { Task } from '@prisma/client';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export default function TaskList({
  initialTasks,
  projectId,
  workspaceId
}: {
  initialTasks: Task[];
  projectId: string;
  workspaceId: string;
}) {
  const [filter, setFilter] = useState({
    status: 'all',
    assignee: 'all'
  });
  
  const { data, error, mutate } = useSWR(
    `/api/workspaces/${workspaceId}/tasks?projectId=${projectId}&status=${filter.status}`,
    fetcher,
    { fallbackData: { data: initialTasks } }
  );
  
  const tasks = data?.data || [];
  
  const handleStatusChange = async (taskId: string, newStatus: string) => {
    const response = await fetch(`/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ status: newStatus })
    });
    
    if (response.ok) {
      mutate();
    }
  };
  
  const tasksByStatus = {
    TODO: tasks.filter((t: Task) => t.status === 'TODO'),
    IN_PROGRESS: tasks.filter((t: Task) => t.status === 'IN_PROGRESS'),
    IN_REVIEW: tasks.filter((t: Task) => t.status === 'IN_REVIEW'),
    DONE: tasks.filter((t: Task) => t.status === 'DONE')
  };
  
  return (
    <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
      {Object.entries(tasksByStatus).map(([status, statusTasks]) => (
        <div key={status} className="bg-gray-50 rounded-lg p-4">
          <h3 className="font-semibold mb-4">{status.replace('_', ' ')}</h3>
          <div className="space-y-2">
            {statusTasks.map((task: Task) => (
              <TaskCard
                key={task.id}
                task={task}
                onStatusChange={handleStatusChange}
              />
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}
```

## Phase 5: バッチ処理とWebhook（1時間）

### 5.1 定期レポート生成

```typescript
// scripts/daily-report.ts
import { prisma } from '@/lib/prisma';
import { sendEmail } from '@/lib/email';

async function generateDailyReport() {
  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);
  yesterday.setHours(0, 0, 0, 0);
  
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  
  const workspaces = await prisma.workspace.findMany({
    include: {
      members: {
        where: { role: { in: ['OWNER', 'ADMIN'] } },
        include: { user: true }
      }
    }
  });
  
  for (const workspace of workspaces) {
    const stats = await prisma.task.aggregate({
      where: {
        project: { workspaceId: workspace.id },
        updatedAt: { gte: yesterday, lt: today }
      },
      _count: true
    });
    
    const completedTasks = await prisma.task.findMany({
      where: {
        project: { workspaceId: workspace.id },
        completedAt: { gte: yesterday, lt: today }
      },
      include: { assignee: true, project: true }
    });
    
    const report = {
      workspace: workspace.name,
      date: yesterday.toISOString().split('T')[0],
      tasksUpdated: stats._count,
      tasksCompleted: completedTasks.length,
      completedDetails: completedTasks
    };
    
    // レポートをメール送信
    for (const member of workspace.members) {
      await sendEmail({
        to: member.user.email,
        subject: `Daily Report - ${workspace.name}`,
        template: 'daily-report',
        data: report
      });
    }
  }
}

// Vercel Cron または node-cron で実行
if (require.main === module) {
  generateDailyReport()
    .then(() => console.log('Daily report generated'))
    .catch(console.error)
    .finally(() => prisma.$disconnect());
}
```

## Phase 6: デプロイ準備（30分）

### 6.1 Docker設定

```dockerfile
# Dockerfile
FROM node:20-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/prisma ./prisma

USER nextjs
EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

### 6.2 環境変数設定

```bash
# .env.production
DATABASE_URL=postgresql://user:password@host:5432/taskmanager
NEXTAUTH_URL=https://yourdomain.com
NEXTAUTH_SECRET=your-secret-key
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
REDIS_URL=redis://your-redis-url
AWS_ACCESS_KEY_ID=your-aws-key
AWS_SECRET_ACCESS_KEY=your-aws-secret
S3_BUCKET=your-s3-bucket
```

## 🎯 学習チェックリスト

- [ ] Prismaでのリレーション設計を理解
- [ ] NextAuthでの認証実装を習得
- [ ] CASLでの認可制御を実装
- [ ] サーバーコンポーネントでのデータ取得
- [ ] クライアントコンポーネントでの状態管理
- [ ] API Routesでの CRUD 実装
- [ ] リアルタイム更新（SWR）の実装
- [ ] バッチ処理の実装
- [ ] Webhook連携の実装
- [ ] Dockerでのコンテナ化

## 次のステップ

1. **パフォーマンス最適化**
   - React.memo, useMemo の活用
   - 画像最適化（next/image）
   - コード分割

2. **テスト実装**
   - Jest でのユニットテスト
   - Playwright での E2E テスト

3. **監視・ログ**
   - Sentry エラー監視
   - DataDog APM

4. **スケーリング**
   - Redis キャッシング
   - CDN 活用
   - データベース読み書き分離