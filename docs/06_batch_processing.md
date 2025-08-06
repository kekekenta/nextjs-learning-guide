# バッチ処理・非同期処理完全ガイド

## 📚 学習目標（45分）
Sidekiq/Whenever からNext.jsのバッチ処理への移行

## 1. バッチ処理アーキテクチャの比較

### Rails（Sidekiq + Whenever）
```ruby
# Gemfile
gem 'sidekiq'
gem 'sidekiq-cron'
gem 'whenever', require: false

# app/workers/data_sync_worker.rb
class DataSyncWorker
  include Sidekiq::Worker
  sidekiq_options queue: 'critical', retry: 3
  
  def perform(user_id)
    user = User.find(user_id)
    
    # 外部APIからデータ取得
    external_data = ExternalApiClient.fetch_user_data(user.external_id)
    
    # データ同期
    user.update!(
      synced_data: external_data,
      last_synced_at: Time.current
    )
    
    # 完了通知
    UserMailer.sync_completed(user).deliver_later
  rescue StandardError => e
    Rails.logger.error "Sync failed for user #{user_id}: #{e.message}"
    raise # Sidekiqにリトライさせる
  end
end

# app/workers/daily_report_worker.rb
class DailyReportWorker
  include Sidekiq::Worker
  
  def perform
    Report.generate_daily_report
    AdminMailer.daily_report(report).deliver_later
  end
end

# config/schedule.rb (Whenever)
every 1.day, at: '9:00 am' do
  runner "DailyReportWorker.perform_async"
end

every 1.hour do
  runner "DataCleanupWorker.perform_async"
end

every 5.minutes do
  runner "HealthCheckWorker.perform_async"
end

# config/sidekiq.yml
:concurrency: 5
:queues:
  - critical
  - default
  - low

:schedule:
  hourly_sync:
    cron: "0 * * * *"
    class: HourlySyncWorker
  daily_cleanup:
    cron: "0 2 * * *"
    class: DailyCleanupWorker
```

### Next.js バッチ処理パターン

#### 1. Vercel Cron Jobs
```typescript
// app/api/cron/daily-report/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { headers } from 'next/headers';

export async function GET(request: NextRequest) {
  // Vercel Cron Jobからの呼び出しを検証
  const authHeader = headers().get('authorization');
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  try {
    // レポート生成
    const report = await generateDailyReport();
    
    // メール送信
    await sendEmail({
      to: process.env.ADMIN_EMAIL!,
      subject: 'Daily Report',
      body: report
    });
    
    // ログ記録
    await prisma.cronLog.create({
      data: {
        job: 'daily-report',
        status: 'success',
        executedAt: new Date()
      }
    });
    
    return NextResponse.json({ success: true });
  } catch (error) {
    console.error('Daily report failed:', error);
    
    await prisma.cronLog.create({
      data: {
        job: 'daily-report',
        status: 'failed',
        error: String(error),
        executedAt: new Date()
      }
    });
    
    return NextResponse.json({ error: 'Failed' }, { status: 500 });
  }
}

// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/daily-report",
      "schedule": "0 9 * * *"
    },
    {
      "path": "/api/cron/hourly-sync",
      "schedule": "0 * * * *"
    }
  ]
}
```

#### 2. BullMQ（Redis Queue）
```typescript
// lib/queue.ts
import { Queue, Worker, QueueScheduler } from 'bullmq';
import Redis from 'ioredis';

const connection = new Redis(process.env.REDIS_URL!);

// キュー定義
export const syncQueue = new Queue('sync', { connection });
export const reportQueue = new Queue('report', { connection });
export const emailQueue = new Queue('email', { connection });

// ジョブ追加
export async function addSyncJob(userId: string) {
  await syncQueue.add('sync-user', { userId }, {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000
    },
    removeOnComplete: true,
    removeOnFail: false
  });
}

// 定期ジョブ設定
export async function setupRecurringJobs() {
  await reportQueue.add('daily-report', {}, {
    repeat: {
      cron: '0 9 * * *',
      tz: 'Asia/Tokyo'
    }
  });
  
  await syncQueue.add('hourly-sync', {}, {
    repeat: {
      every: 60 * 60 * 1000 // 1時間ごと
    }
  });
}

// workers/sync-worker.ts
const syncWorker = new Worker('sync', async (job) => {
  const { userId } = job.data;
  
  console.log(`Processing sync for user ${userId}`);
  
  const user = await prisma.user.findUnique({
    where: { id: userId }
  });
  
  if (!user) {
    throw new Error(`User ${userId} not found`);
  }
  
  // 外部API呼び出し
  const externalData = await fetchExternalData(user.externalId);
  
  // データ更新
  await prisma.user.update({
    where: { id: userId },
    data: {
      syncedData: externalData,
      lastSyncedAt: new Date()
    }
  });
  
  // 通知送信
  await emailQueue.add('send-email', {
    to: user.email,
    subject: 'Sync Completed',
    template: 'sync-completed'
  });
  
  return { success: true, userId };
}, { connection });

// ワーカーのエラーハンドリング
syncWorker.on('failed', (job, err) => {
  console.error(`Job ${job?.id} failed:`, err);
});

syncWorker.on('completed', (job) => {
  console.log(`Job ${job.id} completed`);
});
```

#### 3. Node-cron（スタンドアロン）
```typescript
// scripts/batch-processor.ts
import cron from 'node-cron';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// 毎日午前2時にデータクリーンアップ
cron.schedule('0 2 * * *', async () => {
  console.log('Starting daily cleanup...');
  
  try {
    // 30日以上前のログを削除
    const result = await prisma.log.deleteMany({
      where: {
        createdAt: {
          lt: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
        }
      }
    });
    
    console.log(`Deleted ${result.count} old logs`);
    
    // 期限切れセッションの削除
    await prisma.session.deleteMany({
      where: {
        expiresAt: {
          lt: new Date()
        }
      }
    });
  } catch (error) {
    console.error('Cleanup failed:', error);
  }
});

// 5分ごとにヘルスチェック
cron.schedule('*/5 * * * *', async () => {
  try {
    await prisma.$queryRaw`SELECT 1`;
    console.log('Database health check: OK');
  } catch (error) {
    console.error('Database health check failed:', error);
    // アラート送信
  }
});

// プロセス起動
console.log('Batch processor started');

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('Shutting down batch processor...');
  await prisma.$disconnect();
  process.exit(0);
});
```

## 2. 大量データ処理

### Rails（バッチ処理）
```ruby
# app/jobs/bulk_import_job.rb
class BulkImportJob < ApplicationJob
  def perform(file_path)
    CSV.foreach(file_path, headers: true).each_slice(1000) do |rows|
      users_data = rows.map do |row|
        {
          email: row['email'],
          name: row['name'],
          created_at: Time.current,
          updated_at: Time.current
        }
      end
      
      User.insert_all(users_data)
    end
  end
end

# find_in_batches
User.find_in_batches(batch_size: 1000) do |users|
  users.each do |user|
    UserProcessorWorker.perform_async(user.id)
  end
end
```

### Next.js（ストリーミング処理）
```typescript
// lib/batch-import.ts
import { Readable } from 'stream';
import { pipeline } from 'stream/promises';
import { parse } from 'csv-parse';

export async function bulkImport(filePath: string) {
  const parser = parse({
    columns: true,
    skip_empty_lines: true
  });
  
  let batch: any[] = [];
  const BATCH_SIZE = 1000;
  
  parser.on('readable', async function() {
    let record;
    while ((record = parser.read()) !== null) {
      batch.push({
        email: record.email,
        name: record.name,
        createdAt: new Date(),
        updatedAt: new Date()
      });
      
      if (batch.length >= BATCH_SIZE) {
        await prisma.user.createMany({
          data: batch,
          skipDuplicates: true
        });
        batch = [];
      }
    }
  });
  
  parser.on('end', async () => {
    if (batch.length > 0) {
      await prisma.user.createMany({
        data: batch,
        skipDuplicates: true
      });
    }
  });
  
  const stream = fs.createReadStream(filePath);
  await pipeline(stream, parser);
}

// カーソルベースのページネーション
async function processUsersInBatches() {
  let cursor: string | undefined;
  const batchSize = 1000;
  
  while (true) {
    const users = await prisma.user.findMany({
      take: batchSize,
      skip: cursor ? 1 : 0,
      cursor: cursor ? { id: cursor } : undefined,
      orderBy: { id: 'asc' }
    });
    
    if (users.length === 0) break;
    
    // バッチ処理
    await Promise.all(
      users.map(user => processUser(user))
    );
    
    cursor = users[users.length - 1].id;
  }
}
```

## 3. 長時間実行タスクの管理

### Next.js（Server-Sent Events）
```typescript
// app/api/export/route.ts
export async function GET(request: NextRequest) {
  const encoder = new TextEncoder();
  
  const stream = new ReadableStream({
    async start(controller) {
      try {
        // 進捗状況を送信
        controller.enqueue(
          encoder.encode(`data: ${JSON.stringify({ 
            status: 'started', 
            progress: 0 
          })}\n\n`)
        );
        
        const totalCount = await prisma.user.count();
        let processed = 0;
        let cursor: string | undefined;
        
        while (true) {
          const users = await prisma.user.findMany({
            take: 100,
            skip: cursor ? 1 : 0,
            cursor: cursor ? { id: cursor } : undefined
          });
          
          if (users.length === 0) break;
          
          // データ処理
          for (const user of users) {
            await processUserExport(user);
            processed++;
            
            // 進捗更新
            if (processed % 10 === 0) {
              controller.enqueue(
                encoder.encode(`data: ${JSON.stringify({
                  status: 'processing',
                  progress: Math.round((processed / totalCount) * 100)
                })}\n\n`)
              );
            }
          }
          
          cursor = users[users.length - 1].id;
        }
        
        // 完了通知
        controller.enqueue(
          encoder.encode(`data: ${JSON.stringify({
            status: 'completed',
            progress: 100,
            downloadUrl: '/api/download/export.csv'
          })}\n\n`)
        );
        
        controller.close();
      } catch (error) {
        controller.enqueue(
          encoder.encode(`data: ${JSON.stringify({
            status: 'error',
            error: String(error)
          })}\n\n`)
        );
        controller.close();
      }
    }
  });
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive'
    }
  });
}

// クライアント側
function ExportButton() {
  const [progress, setProgress] = useState(0);
  const [status, setStatus] = useState('idle');
  
  const handleExport = () => {
    const eventSource = new EventSource('/api/export');
    
    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setStatus(data.status);
      setProgress(data.progress || 0);
      
      if (data.status === 'completed') {
        eventSource.close();
        window.location.href = data.downloadUrl;
      }
    };
    
    eventSource.onerror = () => {
      eventSource.close();
      setStatus('error');
    };
  };
  
  return (
    <div>
      <button onClick={handleExport}>Export Data</button>
      {status === 'processing' && (
        <progress value={progress} max={100}>{progress}%</progress>
      )}
    </div>
  );
}
```

## 4. エラーハンドリングとリトライ

```typescript
// lib/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxAttempts?: number;
    delay?: number;
    backoff?: 'linear' | 'exponential';
    onRetry?: (attempt: number, error: Error) => void;
  } = {}
): Promise<T> {
  const {
    maxAttempts = 3,
    delay = 1000,
    backoff = 'exponential',
    onRetry
  } = options;
  
  let lastError: Error;
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      if (attempt < maxAttempts) {
        const waitTime = backoff === 'exponential' 
          ? delay * Math.pow(2, attempt - 1)
          : delay * attempt;
        
        onRetry?.(attempt, lastError);
        await new Promise(resolve => setTimeout(resolve, waitTime));
      }
    }
  }
  
  throw lastError!;
}

// 使用例
const result = await withRetry(
  async () => {
    return await fetchExternalAPI();
  },
  {
    maxAttempts: 5,
    delay: 2000,
    backoff: 'exponential',
    onRetry: (attempt, error) => {
      console.log(`Attempt ${attempt} failed:`, error.message);
    }
  }
);
```

## 5. 監視とログ

```typescript
// lib/batch-monitor.ts
interface BatchJob {
  id: string;
  name: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  startedAt?: Date;
  completedAt?: Date;
  error?: string;
  metadata?: any;
}

export class BatchMonitor {
  async startJob(name: string, metadata?: any): Promise<BatchJob> {
    const job = await prisma.batchJob.create({
      data: {
        name,
        status: 'running',
        startedAt: new Date(),
        metadata
      }
    });
    
    return job;
  }
  
  async completeJob(jobId: string, result?: any) {
    await prisma.batchJob.update({
      where: { id: jobId },
      data: {
        status: 'completed',
        completedAt: new Date(),
        result
      }
    });
  }
  
  async failJob(jobId: string, error: Error) {
    await prisma.batchJob.update({
      where: { id: jobId },
      data: {
        status: 'failed',
        completedAt: new Date(),
        error: error.message
      }
    });
    
    // アラート送信
    await sendAlert({
      type: 'batch-job-failed',
      jobId,
      error: error.message
    });
  }
}

// 使用例
const monitor = new BatchMonitor();

export async function runDailyReport() {
  const job = await monitor.startJob('daily-report');
  
  try {
    const result = await generateReport();
    await monitor.completeJob(job.id, result);
  } catch (error) {
    await monitor.failJob(job.id, error as Error);
    throw error;
  }
}
```

## 🎯 実践演習

### 課題: データ同期システムの実装
1. 外部APIからの定期的なデータ取得
2. 差分検出と更新
3. エラーハンドリングとリトライ
4. 進捗モニタリング

## 💡 重要な比較

| 機能 | Rails | Next.js |
|------|-------|---------|
| ジョブキュー | Sidekiq | BullMQ/Vercel |
| スケジューラー | Whenever | Cron/Vercel Cron |
| バックグラウンド処理 | ActiveJob | Worker/API Routes |
| 長時間タスク | ActionCable | SSE/WebSocket |
| バッチ処理 | find_in_batches | Cursor pagination |
| 監視 | Sidekiq Web | カスタム実装 |