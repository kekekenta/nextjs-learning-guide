# Next.jsプロジェクトでのCLIコマンド作成とECS実行

## 📚 バッチ処理用コマンドの作成方法

Next.jsプロジェクトでも、独立したNode.jsスクリプトとしてバッチ処理コマンドを作成できます。

## 1. 基本的なコマンド作成

### プロジェクト構造
```
your-nextjs-app/
├── app/                 # Next.jsアプリ
├── prisma/
│   └── schema.prisma
├── scripts/            # CLIコマンド用ディレクトリ
│   ├── daily-report.ts
│   ├── sync-data.ts
│   ├── cleanup.ts
│   └── migrate-data.ts
├── lib/
│   ├── prisma.ts      # Prismaクライアント
│   └── email.ts       # 共通ユーティリティ
└── package.json
```

### package.jsonにスクリプト定義
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    
    // バッチ処理コマンド
    "batch:daily-report": "tsx scripts/daily-report.ts",
    "batch:sync-data": "tsx scripts/sync-data.ts",
    "batch:cleanup": "tsx scripts/cleanup.ts",
    "batch:migrate": "tsx scripts/migrate-data.ts",
    
    // 引数付きコマンド
    "batch:process-user": "tsx scripts/process-user.ts"
  },
  "dependencies": {
    "@prisma/client": "^5.0.0",
    "next": "^14.0.0"
  },
  "devDependencies": {
    "tsx": "^4.0.0",  // TypeScript実行用
    "dotenv": "^16.0.0"
  }
}
```

## 2. バッチ処理スクリプトの実装

### 基本的なバッチスクリプト
```typescript
// scripts/daily-report.ts
import { PrismaClient } from '@prisma/client';
import { sendEmail } from '../lib/email';
import dotenv from 'dotenv';

// 環境変数の読み込み
dotenv.config();

const prisma = new PrismaClient();

async function generateDailyReport() {
  console.log('Starting daily report generation...');
  
  try {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    yesterday.setHours(0, 0, 0, 0);
    
    const today = new Date();
    today.setHours(0, 0, 0, 0);
    
    // データ集計
    const stats = await prisma.order.aggregate({
      where: {
        createdAt: {
          gte: yesterday,
          lt: today
        }
      },
      _sum: {
        amount: true
      },
      _count: true
    });
    
    const newUsers = await prisma.user.count({
      where: {
        createdAt: {
          gte: yesterday,
          lt: today
        }
      }
    });
    
    const report = {
      date: yesterday.toISOString().split('T')[0],
      orders: stats._count,
      revenue: stats._sum.amount || 0,
      newUsers
    };
    
    console.log('Report generated:', report);
    
    // メール送信
    await sendEmail({
      to: process.env.ADMIN_EMAIL!,
      subject: `Daily Report - ${report.date}`,
      body: JSON.stringify(report, null, 2)
    });
    
    console.log('Report sent successfully');
    process.exit(0);
  } catch (error) {
    console.error('Error generating report:', error);
    process.exit(1);
  } finally {
    await prisma.$disconnect();
  }
}

// スクリプト実行
if (require.main === module) {
  generateDailyReport();
}
```

### コマンドライン引数を受け取るスクリプト
```typescript
// scripts/process-user.ts
import { PrismaClient } from '@prisma/client';
import { Command } from 'commander';

const prisma = new PrismaClient();
const program = new Command();

program
  .name('process-user')
  .description('Process user data')
  .version('1.0.0')
  .requiredOption('-u, --user-id <id>', 'User ID to process')
  .option('-d, --dry-run', 'Dry run mode', false)
  .option('-v, --verbose', 'Verbose output', false);

program.parse();

const options = program.opts();

async function processUser(userId: string, dryRun: boolean, verbose: boolean) {
  if (verbose) {
    console.log(`Processing user: ${userId}`);
    console.log(`Dry run: ${dryRun}`);
  }
  
  try {
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: { orders: true }
    });
    
    if (!user) {
      throw new Error(`User ${userId} not found`);
    }
    
    if (verbose) {
      console.log(`Found user: ${user.email}`);
      console.log(`Orders: ${user.orders.length}`);
    }
    
    if (!dryRun) {
      // 実際の処理
      await prisma.user.update({
        where: { id: userId },
        data: {
          processedAt: new Date(),
          status: 'PROCESSED'
        }
      });
      
      console.log(`User ${userId} processed successfully`);
    } else {
      console.log(`[DRY RUN] Would process user ${userId}`);
    }
    
    process.exit(0);
  } catch (error) {
    console.error('Error:', error);
    process.exit(1);
  }
}

// 実行
processUser(options.userId, options.dryRun, options.verbose)
  .finally(() => prisma.$disconnect());
```

### 長時間実行バッチ処理
```typescript
// scripts/sync-all-data.ts
import { PrismaClient } from '@prisma/client';
import pLimit from 'p-limit';

const prisma = new PrismaClient();
const limit = pLimit(5); // 並列度を5に制限

class DataSyncJob {
  private processed = 0;
  private failed = 0;
  private startTime = Date.now();
  
  async run() {
    console.log('Starting data sync job...');
    
    try {
      const users = await prisma.user.findMany({
        where: {
          needsSync: true
        }
      });
      
      console.log(`Found ${users.length} users to sync`);
      
      // プログレスバー表示
      const progressInterval = setInterval(() => {
        this.printProgress(users.length);
      }, 1000);
      
      // バッチ処理（並列実行）
      const results = await Promise.all(
        users.map(user => 
          limit(() => this.syncUser(user))
        )
      );
      
      clearInterval(progressInterval);
      this.printSummary();
      
      process.exit(this.failed > 0 ? 1 : 0);
    } catch (error) {
      console.error('Fatal error:', error);
      process.exit(1);
    }
  }
  
  private async syncUser(user: any) {
    try {
      // 外部APIとの同期処理
      const externalData = await this.fetchExternalData(user.externalId);
      
      await prisma.user.update({
        where: { id: user.id },
        data: {
          syncedData: externalData,
          lastSyncedAt: new Date(),
          needsSync: false
        }
      });
      
      this.processed++;
    } catch (error) {
      console.error(`Failed to sync user ${user.id}:`, error);
      this.failed++;
    }
  }
  
  private async fetchExternalData(externalId: string) {
    // 外部API呼び出しのシミュレーション
    await new Promise(resolve => setTimeout(resolve, 100));
    return { synced: true, timestamp: new Date() };
  }
  
  private printProgress(total: number) {
    const elapsed = Math.floor((Date.now() - this.startTime) / 1000);
    const rate = this.processed / elapsed || 0;
    const eta = Math.floor((total - this.processed) / rate) || 0;
    
    console.log(
      `Progress: ${this.processed}/${total} | ` +
      `Failed: ${this.failed} | ` +
      `Rate: ${rate.toFixed(1)}/s | ` +
      `ETA: ${eta}s`
    );
  }
  
  private printSummary() {
    const elapsed = Math.floor((Date.now() - this.startTime) / 1000);
    console.log('\n=== Sync Job Summary ===');
    console.log(`Total processed: ${this.processed}`);
    console.log(`Failed: ${this.failed}`);
    console.log(`Time elapsed: ${elapsed}s`);
    console.log(`Success rate: ${((this.processed / (this.processed + this.failed)) * 100).toFixed(2)}%`);
  }
}

// 実行
if (require.main === module) {
  const job = new DataSyncJob();
  job.run().finally(() => prisma.$disconnect());
}
```

## 3. ECSでの実行

### Dockerfile（バッチ処理用）
```dockerfile
# マルチステージビルド
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npm run build

# バッチ実行用イメージ
FROM node:20-alpine AS batch
WORKDIR /app

# 必要なファイルのみコピー
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/scripts ./scripts
COPY --from=builder /app/lib ./lib
COPY --from=builder /app/package.json ./
COPY --from=builder /app/.env.production ./

# デフォルトコマンド（ECSタスク定義で上書き）
CMD ["npm", "run", "batch:daily-report"]
```

### ECSタスク定義
```json
{
  "family": "batch-daily-report",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "batch-job",
      "image": "${ECR_URI}:latest",
      "essential": true,
      "command": ["npm", "run", "batch:daily-report"],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:db-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/batch-jobs",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "daily-report"
        }
      }
    }
  ]
}
```

### EventBridge（CloudWatch Events）でのスケジュール実行
```typescript
// terraform/batch-schedules.tf
resource "aws_cloudwatch_event_rule" "daily_report" {
  name                = "daily-report-schedule"
  description         = "Trigger daily report batch job"
  schedule_expression = "cron(0 9 * * ? *)" // 毎日9時（UTC）
}

resource "aws_cloudwatch_event_target" "daily_report_ecs" {
  rule      = aws_cloudwatch_event_rule.daily_report.name
  target_id = "daily-report-ecs-target"
  arn       = aws_ecs_cluster.main.arn
  role_arn  = aws_iam_role.events_role.arn
  
  ecs_target {
    task_definition_arn = aws_ecs_task_definition.batch_daily_report.arn
    task_count          = 1
    launch_type         = "FARGATE"
    
    network_configuration {
      subnets          = aws_subnet.private[*].id
      security_groups  = [aws_security_group.batch.id]
      assign_public_ip = false
    }
  }
}

// 異なるコマンドを実行するタスク
resource "aws_ecs_task_definition" "batch_cleanup" {
  family = "batch-cleanup"
  
  container_definitions = jsonencode([
    {
      name    = "batch-job"
      image   = "${var.ecr_uri}:latest"
      command = ["npm", "run", "batch:cleanup"]
      // ... その他の設定
    }
  ])
}
```

## 4. ローカル開発とテスト

### 開発環境での実行
```bash
# 直接実行
npx tsx scripts/daily-report.ts

# npm script経由
npm run batch:daily-report

# 引数付き実行
npm run batch:process-user -- --user-id user123 --dry-run

# 環境変数を指定して実行
NODE_ENV=production npm run batch:daily-report

# デバッグモード
DEBUG=* npm run batch:daily-report
```

### テストの作成
```typescript
// scripts/__tests__/daily-report.test.ts
import { execSync } from 'child_process';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

describe('Daily Report Batch', () => {
  beforeEach(async () => {
    // テストデータの準備
    await prisma.order.createMany({
      data: [
        { amount: 1000, createdAt: new Date() },
        { amount: 2000, createdAt: new Date() }
      ]
    });
  });
  
  afterEach(async () => {
    await prisma.order.deleteMany();
  });
  
  it('should generate report successfully', () => {
    const output = execSync('npm run batch:daily-report', {
      encoding: 'utf-8',
      env: { ...process.env, NODE_ENV: 'test' }
    });
    
    expect(output).toContain('Report generated');
    expect(output).toContain('orders: 2');
  });
  
  it('should handle errors gracefully', async () => {
    // データベース接続を切断してエラーを発生させる
    await prisma.$disconnect();
    
    expect(() => {
      execSync('npm run batch:daily-report', {
        encoding: 'utf-8'
      });
    }).toThrow();
  });
});
```

## 5. 監視とログ

### CloudWatch Logsでの監視
```typescript
// scripts/utils/logger.ts
export class BatchLogger {
  private jobName: string;
  
  constructor(jobName: string) {
    this.jobName = jobName;
  }
  
  info(message: string, meta?: any) {
    console.log(JSON.stringify({
      level: 'info',
      job: this.jobName,
      message,
      timestamp: new Date().toISOString(),
      ...meta
    }));
  }
  
  error(message: string, error?: any) {
    console.error(JSON.stringify({
      level: 'error',
      job: this.jobName,
      message,
      error: error?.message || error,
      stack: error?.stack,
      timestamp: new Date().toISOString()
    }));
  }
  
  metric(name: string, value: number, unit = 'None') {
    console.log(JSON.stringify({
      _aws: {
        CloudWatchMetrics: [{
          Namespace: 'BatchJobs',
          Dimensions: [['JobName']],
          Metrics: [{
            Name: name,
            Unit: unit
          }]
        }],
        Timestamp: Date.now()
      },
      JobName: this.jobName,
      [name]: value
    }));
  }
}

// 使用例
const logger = new BatchLogger('daily-report');

logger.info('Starting job');
logger.metric('ProcessedRecords', 100, 'Count');
logger.metric('ProcessingTime', 5.2, 'Seconds');
logger.error('Job failed', new Error('Database connection lost'));
```

## 6. エラーハンドリングとリトライ

```typescript
// scripts/utils/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxAttempts?: number;
    delay?: number;
    onRetry?: (attempt: number, error: Error) => void;
  } = {}
): Promise<T> {
  const { maxAttempts = 3, delay = 1000, onRetry } = options;
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      
      onRetry?.(attempt, error as Error);
      await new Promise(resolve => setTimeout(resolve, delay * attempt));
    }
  }
  
  throw new Error('Max attempts reached');
}

// 使用例
await withRetry(
  async () => {
    return await prisma.user.findMany();
  },
  {
    maxAttempts: 5,
    delay: 2000,
    onRetry: (attempt, error) => {
      console.log(`Attempt ${attempt} failed: ${error.message}`);
    }
  }
);
```

## 💡 まとめ

**Next.jsプロジェクトでのバッチ処理：**
1. `scripts/`ディレクトリに独立したTypeScriptファイルを作成
2. `tsx`でTypeScriptを直接実行
3. Prismaや共通ライブラリを再利用
4. ECSタスクとして同じDockerイメージから異なるコマンドを実行
5. EventBridgeでスケジュール実行

**Rails（rake task）との対応：**
- `rake db:migrate` → `npx prisma migrate deploy`
- `rake reports:daily` → `npm run batch:daily-report`
- `rake users:process[id]` → `npm run batch:process-user -- --user-id id`