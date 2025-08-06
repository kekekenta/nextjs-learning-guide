# Next.js Webサーバーアーキテクチャ完全ガイド

## 📚 学習目標（30分）
RailsのRack/Pumaと比較しながら、Next.jsのサーバー構築を完全理解

## 1. Rails vs Next.js サーバーアーキテクチャ比較

### Rails (Rack + Puma)
```
[Nginx/Apache] → [Puma (アプリサーバー)] → [Rack (インターフェース)] → [Rails App]
                   ├─ ワーカープロセス1
                   ├─ ワーカープロセス2
                   └─ ワーカープロセス3
```

### Next.js
```
[Nginx/CDN] → [Node.js サーバー] → [Next.js Server] → [React App]
               └─ 内蔵HTTPサーバー     ├─ App Router
                  (http module)        ├─ API Routes
                                      └─ Middleware
```

## 2. Next.jsの内蔵サーバー

### 開発環境（next dev）

```javascript
// Next.jsの開発サーバー起動プロセス
// node_modules/next/dist/cli/next-dev.js

const { createServer } = require('http');
const { NextServer } = require('../server/next');

// 内部的な動作
const server = new NextServer({
  dev: true,  // 開発モード
  hostname: 'localhost',
  port: 3000,
  dir: process.cwd(),
});

// HTTPサーバー作成
const httpServer = createServer(server.getRequestHandler());
httpServer.listen(3000);

// 機能:
// - Hot Module Replacement (HMR)
// - TypeScript/JSXのリアルタイムコンパイル
// - エラーオーバーレイ
// - Fast Refresh
```

### 本番環境（next start）

```javascript
// Next.jsの本番サーバー
// ビルド後に起動

// 1. ビルド
// next build → .next/ディレクトリに最適化されたファイル生成

// 2. サーバー起動
const { createServer } = require('http');
const { NextServer } = require('next/dist/server/next');

const server = new NextServer({
  dev: false,  // 本番モード
  hostname: '0.0.0.0',
  port: process.env.PORT || 3000,
  dir: process.cwd(),
  conf: {
    // Next.js設定
    compress: true,  // gzip圧縮
    poweredByHeader: false,
  }
});

const httpServer = createServer(server.getRequestHandler());
httpServer.listen(port);
```

## 3. カスタムサーバー実装

### Express.jsを使用したカスタムサーバー

```javascript
// server.js
const express = require('express');
const next = require('next');

const dev = process.env.NODE_ENV !== 'production';
const hostname = 'localhost';
const port = 3000;

// Next.jsアプリケーションインスタンス
const app = next({ dev, hostname, port });
const handle = app.getRequestHandler();

app.prepare().then(() => {
  const server = express();
  
  // カスタムミドルウェア
  server.use(express.json());
  server.use(express.urlencoded({ extended: true }));
  
  // カスタムルート
  server.get('/custom/:id', (req, res) => {
    const actualPage = '/custom-page';
    const queryParams = { id: req.params.id };
    app.render(req, res, actualPage, queryParams);
  });
  
  // WebSocket対応
  const httpServer = require('http').createServer(server);
  const io = require('socket.io')(httpServer);
  
  io.on('connection', (socket) => {
    console.log('WebSocket connected');
    socket.on('message', (data) => {
      io.emit('broadcast', data);
    });
  });
  
  // Next.jsのデフォルトハンドラー
  server.all('*', (req, res) => {
    return handle(req, res);
  });
  
  httpServer.listen(port, () => {
    console.log(`> Ready on http://${hostname}:${port}`);
  });
});
```

### Fastifyを使用した高性能サーバー

```javascript
// server.js (Fastify版)
const fastify = require('fastify')({ logger: true });
const next = require('next');

const dev = process.env.NODE_ENV !== 'production';
const app = next({ dev });
const handle = app.getRequestHandler();

app.prepare().then(() => {
  // Fastifyプラグイン
  fastify.register(require('@fastify/compress')); // 圧縮
  fastify.register(require('@fastify/helmet'));   // セキュリティ
  fastify.register(require('@fastify/cors'));     // CORS
  
  // カスタムAPI
  fastify.get('/api/health', async (request, reply) => {
    return { status: 'ok', timestamp: Date.now() };
  });
  
  // Next.jsハンドラー
  fastify.all('/*', async (request, reply) => {
    await handle(request.raw, reply.raw);
    reply.sent = true;
  });
  
  fastify.listen({ port: 3000, host: '0.0.0.0' }, (err) => {
    if (err) throw err;
  });
});
```

## 4. Next.jsのデフォルト構成と本番環境の実態

### デフォルト構成（next start）の仕組み

```javascript
// Next.jsデフォルトの動作モデル
// 単一プロセス + イベントループ + 非同期I/O

// 実際の動作：
// 1. メインスレッド（1つ）: JavaScriptコード実行
// 2. ワーカースレッド（4つ）: I/O処理（ファイル読み込み、DB接続など）
// 3. イベントループ: 非同期処理の管理

// 重要：Node.jsは単一プロセスだが、マルチスレッドでI/Oを処理
```

#### Node.jsの実際のスレッドモデル

```bash
# Node.jsプロセスの実際のスレッド数を確認
ps -M <PID> | wc -l
# 結果: 通常7-11スレッド
# - 1 メインスレッド（V8）
# - 4 libuv I/Oスレッド（デフォルト）
# - 2+ V8用スレッド（GC、コンパイル）
# - 1+ DNS解決用スレッド
```

#### I/Oスレッドの本当の役割（Next.jsでの動作）

```javascript
// ❌ よくある誤解
// I/Oスレッド = リクエストを処理するスレッド

// ✅ 実際の役割分担
// メインスレッド：リクエスト処理、レスポンス返却、JavaScriptコード実行
// I/Oスレッド：ファイル読み込み、DB接続、DNS解決などのブロッキング操作のみ
```

##### I/Oスレッドに処理が移譲される具体的なタイミング

```javascript
// app/api/users/route.ts - 実際のNext.jsコード

export async function GET() {
  // ✅ メインスレッドで実行される処理
  console.log('開始');                    // メインスレッド
  const startTime = Date.now();          // メインスレッド
  const userId = 123;                    // メインスレッド
  
  // ⚡ I/Oスレッドに移譲される処理（自動判定）
  
  // 1. ファイルシステム操作 → I/Oスレッドへ
  const fs = require('fs').promises;
  const data = await fs.readFile('./data.json', 'utf-8');
  // ↑ Node.jsが自動でI/Oスレッドに移譲
  
  // 2. データベース操作 → I/Oスレッドへ  
  const users = await prisma.user.findMany({
    where: { active: true }
  });
  // ↑ TCP/IPソケット通信なのでI/Oスレッドへ
  
  // 3. 外部API呼び出し → I/Oスレッドへ
  const response = await fetch('https://api.example.com/data');
  // ↑ ネットワークI/OなのでI/Oスレッドへ
  
  // 4. DNS解決 → I/Oスレッドへ
  const dns = require('dns').promises;
  const addresses = await dns.resolve4('google.com');
  // ↑ DNS解決はI/Oスレッドで実行
  
  // 5. 暗号化処理 → I/Oスレッドへ
  const crypto = require('crypto');
  const hash = crypto.pbkdf2Sync('password', 'salt', 100000, 64, 'sha512');
  // ↑ pbkdf2Syncの場合はメインスレッド（同期処理）
  
  // 非同期版ならI/Oスレッドへ
  const hashAsync = await new Promise((resolve, reject) => {
    crypto.pbkdf2('password', 'salt', 100000, 64, 'sha512', (err, key) => {
      if (err) reject(err);
      else resolve(key);
    });
  });
  // ↑ pbkdf2（非同期）はI/Oスレッドで実行
  
  // ❌ メインスレッドでしか実行されない処理
  
  // JSONパース/シリアライズ
  const parsed = JSON.parse(data);       // メインスレッド
  const stringified = JSON.stringify(users); // メインスレッド
  
  // 配列操作・計算
  const filtered = users.filter(u => u.age > 20); // メインスレッド
  const sum = users.reduce((a, b) => a + b.score, 0); // メインスレッド
  
  // 正規表現
  const isValid = /^[a-z]+$/.test('hello'); // メインスレッド
  
  return Response.json({ users, data });
}
```

##### ⚠️ awaitでもI/Oスレッドに移譲されないケース

```javascript
// app/api/examples/route.ts

export async function GET() {
  // ❌ awaitを使ってもメインスレッドで実行される例
  
  // 1. Promise.resolve（即座に解決するPromise）
  const result1 = await Promise.resolve(42);
  // メインスレッドで即座に解決（I/Oスレッド不使用）
  
  // 2. async関数の計算処理
  async function heavyCalculation() {
    let sum = 0;
    for (let i = 0; i < 1000000; i++) {
      sum += i;
    }
    return sum;
  }
  const result2 = await heavyCalculation();
  // ↑ asyncでもただの計算はメインスレッド！
  
  // 3. setTimeout/setTimeoutのPromise化
  const result3 = await new Promise(resolve => {
    setTimeout(() => resolve('done'), 100);
  });
  // ↑ タイマーはメインスレッドのイベントループで管理
  
  // 4. Next.jsのキャッシュ（メモリキャッシュ）
  const cachedData = await unstable_cache(
    async () => {
      return { data: 'cached' };
    },
    ['cache-key'],
    { revalidate: 60 }
  )();
  // ↑ メモリから読むだけならメインスレッド
  
  // 5. React Server Componentsの処理
  async function ServerComponent() {
    const data = await getStaticData(); // 静的データ取得
    return <div>{data}</div>;
  }
  // ↑ レンダリング処理はメインスレッド
  
  // 6. JavaScriptのネイティブasync処理
  const mapped = await Promise.all(
    [1, 2, 3].map(async (n) => n * 2)
  );
  // ↑ 単純な計算の並列化でもメインスレッド
  
  // 7. カスタムPromise（I/O操作なし）
  const customPromise = await new Promise((resolve) => {
    // 重い計算処理
    const result = fibonacci(45); // CPU集約的
    resolve(result);
  });
  // ↑ Promise内でもCPU処理はメインスレッド！
}

// ✅ 同じ処理をI/Oスレッドで実行する方法

export async function POST() {
  // 1. Worker Threadsを使う
  const { Worker } = require('worker_threads');
  
  const result = await new Promise((resolve, reject) => {
    const worker = new Worker(`
      const { parentPort } = require('worker_threads');
      let sum = 0;
      for (let i = 0; i < 1000000; i++) {
        sum += i;
      }
      parentPort.postMessage(sum);
    `, { eval: true });
    
    worker.on('message', resolve);
    worker.on('error', reject);
  });
  // ↑ これならI/Oスレッドプール使用！
  
  // 2. child_processを使う
  const { exec } = require('child_process').promises;
  const output = await exec('node heavy-calculation.js');
  // ↑ 別プロセスで実行（I/O扱い）
  
  return Response.json({ result });
}
```

##### awaitとI/Oスレッドの判定基準

```javascript
// Node.jsの内部判定ロジック（概念的な説明）

async function determineThreadExecution(operation) {
  // awaitがあってもI/Oスレッドに行かない条件
  
  if (operation.type === 'computation') {
    // 純粋な計算処理
    return 'MAIN_THREAD';
  }
  
  if (operation.type === 'memory_access') {
    // メモリアクセスのみ
    return 'MAIN_THREAD';
  }
  
  if (operation.type === 'timer') {
    // setTimeout, setInterval
    return 'MAIN_THREAD_EVENT_LOOP';
  }
  
  if (operation.type === 'promise_resolve') {
    // 即座に解決されるPromise
    return 'MAIN_THREAD';
  }
  
  // I/Oスレッドに移譲される条件
  if (operation.type === 'file_system') {
    return 'IO_THREAD_POOL';
  }
  
  if (operation.type === 'network') {
    return 'IO_THREAD_POOL';
  }
  
  if (operation.type === 'crypto_async') {
    return 'IO_THREAD_POOL';
  }
  
  if (operation.type === 'dns_lookup') {
    return 'IO_THREAD_POOL';
  }
  
  return 'MAIN_THREAD'; // デフォルト
}
```

##### Next.jsでの実践的な見分け方

```javascript
// app/api/check-thread/route.ts

export async function GET() {
  console.time('Performance Test');
  
  // 🔴 メインスレッドをブロックする（awaitでも）
  const blocking = await (async () => {
    const start = Date.now();
    while (Date.now() - start < 1000) {
      // 1秒間CPUを占有
    }
    return 'blocked';
  })();
  // 他のリクエストが1秒間処理できない！
  
  // 🟢 I/Oスレッドで実行（ブロックしない）
  const nonBlocking = await fs.readFile('./large-file.txt');
  // 他のリクエストは並行処理可能
  
  console.timeEnd('Performance Test');
  
  // 見分け方のコツ：
  // 1. OSのシステムコールが必要か？ → I/Oスレッド
  // 2. ファイル/ネットワーク/DB操作か？ → I/Oスレッド
  // 3. 純粋なJavaScript処理か？ → メインスレッド
  // 4. Node.js APIのドキュメントで「asynchronous」と書かれているか？
  
  return Response.json({ blocking, nonBlocking });
}
```

##### Node.js内部でのI/Oスレッド判定ロジック

```javascript
// Node.jsが内部で行っている判定（簡略化）

function executeOperation(operation) {
  // I/Oスレッドに移譲される操作のリスト
  const ioOperations = [
    'fs.readFile',      // ファイル読み込み
    'fs.writeFile',     // ファイル書き込み
    'fs.stat',          // ファイル情報取得
    'dns.lookup',       // DNS解決
    'crypto.pbkdf2',    // 暗号化（非同期版）
    'zlib.gzip',        // 圧縮（非同期版）
  ];
  
  // ネットワーク操作（別の仕組みだがI/O扱い）
  const networkOperations = [
    'net.connect',      // TCP接続
    'http.request',     // HTTP通信
    'https.request',    // HTTPS通信
    'dgram.send',       // UDP通信
  ];
  
  if (ioOperations.includes(operation)) {
    // libuvのスレッドプールで実行
    threadPool.execute(operation);
  } else if (networkOperations.includes(operation)) {
    // epoll/kqueue（OS依存）で非同期I/O
    networkEventLoop.register(operation);
  } else {
    // メインスレッドで実行
    mainThread.execute(operation);
  }
}
```

##### 実際のNext.jsアプリケーションでの例

```javascript
// app/products/[id]/page.tsx
export default async function ProductPage({ params }) {
  // 🔄 処理の流れを可視化
  
  console.log('1. ページ処理開始'); // [メイン]
  
  // 並列I/O処理
  const promises = [
    // [I/Oスレッド#1] DBから商品情報取得
    prisma.product.findUnique({ 
      where: { id: params.id } 
    }),
    
    // [I/Oスレッド#2] 画像メタデータ読み込み  
    fs.readFile(`./images/${params.id}.json`, 'utf-8'),
    
    // [I/Oスレッド#3] 在庫API呼び出し
    fetch(`https://inventory-api.com/stock/${params.id}`),
    
    // [I/Oスレッド#4] Redis キャッシュ確認
    redis.get(`product:${params.id}:views`),
    
    // もし5つ目があれば...
    // [待機] I/Oスレッドが空くまでキューで待機
    // fs.readFile('./extra-data.json')
  ];
  
  console.log('2. I/O処理を開始（ブロックしない）'); // [メイン]
  
  // await で全I/O完了を待つ
  const [product, imageData, stockData, views] = await Promise.all(promises);
  
  console.log('3. 全I/O完了、レンダリング開始'); // [メイン]
  
  // これ以降はメインスレッドで処理
  const processedData = {
    ...product,
    images: JSON.parse(imageData), // [メイン] JSONパース
    stock: stockData.json(),        // [メイン] レスポンス処理
    views: parseInt(views) || 0     // [メイン] 数値変換
  };
  
  return (
    <div>
      {/* JSXレンダリング（メインスレッド） */}
      <h1>{processedData.name}</h1>
      <p>在庫: {processedData.stock}</p>
      <p>閲覧数: {processedData.views}</p>
    </div>
  );
}
```

##### I/Oスレッド処理中のメインスレッドの動作（重要）

```javascript
// ✅ I/Oスレッドが処理している間、メインスレッドは他のリクエストを処理できます！

// app/api/example/route.ts
export async function GET() {
  console.log('1. リクエストA開始');
  
  // DBクエリをI/Oスレッドに委託
  const users = await prisma.user.findMany();  // 100ms かかるとする
  
  // ↑ この100ms の間のタイムライン：
  // [0ms] メインスレッド: I/OスレッドにDB処理を委託して即座に解放
  // [1ms] メインスレッド: リクエストBを受信・処理開始 ✅
  // [5ms] メインスレッド: リクエストCを受信・処理開始 ✅
  // [10ms] メインスレッド: リクエストDのJSONパース処理 ✅
  // [20ms] メインスレッド: リクエストEのルーティング処理 ✅
  // [50ms] メインスレッド: リクエストFのレスポンス送信 ✅
  // ...
  // [100ms] I/Oスレッド: DB処理完了、コールバック登録
  // [101ms] メインスレッド: リクエストAの処理再開
  
  console.log('2. リクエストA処理再開');
  return Response.json({ users });
}
```

##### 実際の並行処理フロー（3人同時アクセス）

```javascript
// 時系列で見る非同期I/Oの威力

// === 時刻 0ms ===
// リクエストA到着
const usersA = await db.query('SELECT * FROM users');
// メインスレッド: I/Oスレッド#1に委託 → 即座に次へ ✅

// === 時刻 1ms ===
// リクエストB到着
const fileB = await fs.readFile('./data.json');
// メインスレッド: I/Oスレッド#2に委託 → 即座に次へ ✅

// === 時刻 2ms ===
// リクエストC到着
const apiC = await fetch('https://api.example.com');
// メインスレッド: I/Oスレッド#3に委託 → 即座に次へ ✅

// === 時刻 50ms ===
// I/Oスレッド#2: ファイル読み込み完了
// メインスレッド: リクエストBの続きを処理
// - JSONパース
// - レスポンス生成
// - クライアントへ送信

// === 時刻 100ms ===
// I/Oスレッド#1: DB処理完了
// メインスレッド: リクエストAの続きを処理

// === 時刻 150ms ===
// I/Oスレッド#3: API通信完了
// メインスレッド: リクエストCの続きを処理

// 結果：3つのリクエストが並行処理された！
```

##### なぜこれが重要なのか

```javascript
// Node.jsのイベントループモデルの本質

// ✅ I/O処理の場合（非ブロッキング）
export async function handleRequest() {
  // 1000人が同時にアクセスしても問題なし
  await prisma.user.findMany();    // メインスレッド解放
  await fs.readFile();              // メインスレッド解放
  await fetch();                    // メインスレッド解放
  await redis.get();                // メインスレッド解放
  
  // メインスレッドは常に空いているので、
  // 新しいリクエストを即座に受け付けられる
}

// ❌ CPU処理の場合（ブロッキング）
export async function handleHeavyRequest() {
  // 1人でもこれをやると全員が待たされる
  const result = heavyCalculation();     // メインスレッドを占有
  const parsed = JSON.parse(hugeData);   // メインスレッドを占有
  const sorted = bigArray.sort();        // メインスレッドを占有
  
  // この間、新しいリクエストは一切処理できない
}

// これがNode.jsがI/O処理に強くCPU処理に弱い理由！
```

##### 並行処理の実例（100人同時アクセス）

```javascript
// 100人が同時にアクセスした場合

// メインスレッド（1つ）の動き：
for (let i = 0; i < 100; i++) {
  // リクエスト#1: DB問い合わせ開始 → I/Oスレッドに委託
  // リクエスト#2: DB問い合わせ開始 → I/Oスレッドに委託
  // リクエスト#3: ファイル読み込み開始 → I/Oスレッドに委託
  // ...
  // リクエスト#100: 処理開始
  
  // I/O完了待ちせず、次々に処理を開始（非同期）
}

// I/Oスレッドプール（デフォルト4つ）の動き：
// I/Oスレッド#1: リクエスト#1のDB接続処理中...
// I/Oスレッド#2: リクエスト#2のDB接続処理中...
// I/Oスレッド#3: リクエスト#3のファイル読み込み中...
// I/Oスレッド#4: リクエスト#4のAPI通信中...
// ↑ 5番目以降のI/O操作はキューで待機

// でもメインスレッドは自由！
// → 新規リクエスト受付 ✅
// → レスポンス送信 ✅
// → ルーティング処理 ✅
// → 軽いJavaScript実行 ✅
```

#### メインスレッドのボトルネック問題と解決策

##### ⚠️ 重要：メインスレッドのブロッキングは全リクエストに影響します

```javascript
// ❌ 実際の問題：1人の重い処理が全員に影響

// app/api/process-image/route.ts
export async function POST(request) {
  const image = await request.formData();
  
  // 画像処理で3秒かかるとする
  const result = processLargeImage(image); // 3秒間メインスレッドを占有
  
  return Response.json({ processed: true });
}

// この3秒間に起きること：
// ユーザーA: 画像処理リクエスト → 処理中...
// ユーザーB: トップページアクセス → 3秒待機 ❌
// ユーザーC: ログイン試行 → 3秒待機 ❌
// ユーザーD: 商品検索 → 3秒待機 ❌
// ヘルスチェック: /api/health → タイムアウト ❌
// → サービス全体がダウンしたように見える！

// 実際の症状：
// - 全ページが応答しない
// - APIが全てタイムアウト
// - ロードバランサーがサーバーをunhealthyと判定
// - 自動スケーリングが誤作動
```

##### 🔴 これがNode.js最大の弱点

```javascript
// Rails (Puma) の場合
// config/puma.rb
workers 4  // 4プロセス
threads 5, 5  // 各プロセス5スレッド

// 1つのスレッドが重い処理をしても：
// ✅ 残り19スレッド（4×5-1）は正常に動作
// ✅ 他のユーザーは影響を受けない

// Node.js (デフォルト) の場合
// 単一プロセス・単一メインスレッド

// 1つの重い処理で：
// ❌ 全てのリクエストがブロック
// ❌ サービス全体が停止状態
```

##### 解決策1：Worker Threads（メインスレッドを増やす代替案）

```javascript
// ✅ Worker Threadsで別スレッドで実行
import { Worker } from 'worker_threads';

export async function POST(request) {
  const image = await request.formData();
  
  // 重い処理を別スレッドに委託
  return new Promise((resolve, reject) => {
    const worker = new Worker('./image-processor.js', {
      workerData: { image }
    });
    
    worker.on('message', (result) => {
      resolve(Response.json({ processed: result }));
    });
    
    // メインスレッドはブロックされない！
  });
}

// image-processor.js（ワーカースレッド）
const { parentPort, workerData } = require('worker_threads');
// CPU集約的な処理をここで実行
const result = processImage(workerData.image);
parentPort.postMessage(result);
```

##### 解決策2：マルチプロセス化（最も一般的な本番対策）

```javascript
// ⭐ PM2でマルチプロセス化が本番環境の標準解

// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'nextjs-app',
    script: 'next',
    args: 'start',
    instances: 4,  // 4プロセス = メインスレッド × 4
    exec_mode: 'cluster'
  }]
};

// 実際の動作例：
// 10:00:00 - 通常時（4プロセス稼働）
// プロセス1: ユーザーA,B,Cのリクエスト処理
// プロセス2: ユーザーD,E,Fのリクエスト処理
// プロセス3: ユーザーG,H,Iのリクエスト処理
// プロセス4: ユーザーJ,K,Lのリクエスト処理

// 10:00:01 - ユーザーMが重い画像処理リクエスト
// プロセス1: 画像処理で3秒ブロック ❌
// プロセス2: 通常リクエスト処理継続 ✅
// プロセス3: 通常リクエスト処理継続 ✅
// プロセス4: 通常リクエスト処理継続 ✅
// → 75%のキャパシティで稼働継続！
```

##### 解決策3：処理の分離（アーキテクチャ改善）

```javascript
// ✅ 重い処理は別サービスに分離
export async function POST(request) {
  const image = await request.formData();
  
  // 処理をキューに投入
  await queue.add('process-image', {
    imageId: image.id,
    userId: request.userId
  });
  
  // 即座にレスポンス（非同期処理）
  return Response.json({ 
    status: 'processing',
    jobId: jobId 
  });
}

// 別プロセス/サーバーで処理（Bull Queue等）
queue.process('process-image', async (job) => {
  // 重い処理を専用ワーカーで実行
  const result = await processImage(job.data);
  await notifyUser(job.data.userId, result);
});
```

##### まとめ：本番環境での現実的な判断基準

```javascript
// 実務での判断フローチャート
const productionDecision = {
  '小規模サービス（〜100req/s）': {
    問題: 'ほぼ発生しない',
    対策: 'デフォルトでOK',
    理由: 'CPU処理が少なければ問題なし'
  },
  
  '中規模サービス（100-1000req/s）': {
    問題: '時々発生する',
    対策: 'PM2は必須（instances: 2-4）',
    理由: '1つの重い処理で全体停止は致命的'
  },
  
  '大規模サービス（1000req/s〜）': {
    問題: '頻繁に発生',
    対策: 'PM2 + Worker Threads + Queue',
    理由: 'あらゆる対策を組み合わせる'
  },
  
  'CPU処理が多いサービス': {
    問題: 'Node.js自体が不適切',
    対策: 'Go, Rust, Java等を検討',
    理由: 'そもそもNode.jsの設計思想と合わない'
  }
};

// ⚠️ 重要な事実
// これがRailsのPumaがデフォルトでマルチプロセス・
// マルチスレッドな理由でもある
// → RubyもGILでシングルスレッド的だが、
//   最初から並行処理を前提に設計されている

// Node.jsの制約
// - メインスレッドは1プロセスに1つ（変更不可）
// - デフォルトは単一プロセスで本番には不適切な場合が多い
// - PM2やClusterでのマルチプロセス化がほぼ必須
```

#### I/Oスレッドプールの詳細と調整

```javascript
// デフォルトは4スレッドだが、調整可能！

// 1. 環境変数で増やす（最大1024まで）
process.env.UV_THREADPOOL_SIZE = '16';  // 16スレッドに増加

// 2. Next.jsの起動スクリプトで設定
// package.json
{
  "scripts": {
    "start": "UV_THREADPOOL_SIZE=16 next start",
    "start:optimized": "UV_THREADPOOL_SIZE=128 next start"
  }
}

// 3. PM2での設定
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'nextjs-app',
    script: 'node_modules/next/dist/bin/next',
    args: 'start',
    env: {
      UV_THREADPOOL_SIZE: '32'  // I/Oスレッドを32に
    }
  }]
};
```

#### なぜI/Oスレッドを増やすと性能が向上するのか

```javascript
// 具体例：ECサイトの商品ページ

export default async function ProductPage({ params }) {
  // 並列でI/O処理を実行
  const [product, reviews, recommendations, inventory] = await Promise.all([
    db.product.findById(params.id),      // I/Oスレッド#1
    db.reviews.findByProduct(params.id),  // I/Oスレッド#2  
    api.getRecommendations(params.id),    // I/Oスレッド#3
    redis.get(`inventory:${params.id}`)   // I/Oスレッド#4
  ]);
  
  // デフォルト4スレッドの場合：
  // ✅ 4つ全て並列実行可能
  
  // もし5つ目のI/O操作があったら：
  const shipping = await api.getShipping(); // 待機！他のI/Oが終わるまで
  
  // UV_THREADPOOL_SIZE=8 なら：
  // ✅ 5つ全て同時実行可能 → レスポンス高速化
}
```

#### I/Oスレッド数の選び方

```javascript
// I/O操作の種類と推奨スレッド数

// 1. ファイルシステム操作が多い場合
// - 静的ファイル配信
// - ログ書き込み
// 推奨: UV_THREADPOOL_SIZE = 16-32

// 2. データベース接続が多い場合
// - 複数のDB同時クエリ
// - 大量のコネクションプール
// 推奨: UV_THREADPOOL_SIZE = 32-64

// 3. 暗号化処理が多い場合
// - JWT検証
// - HTTPS/TLS処理
// 推奨: UV_THREADPOOL_SIZE = 64-128

// 計算式の目安
const recommendedThreads = Math.min(
  128,  // 最大値
  Math.max(
    4,   // 最小値
    CPUコア数 * 4,  // 基本値
    同時I/O操作数の予測値
  )
);
```

#### パフォーマンステスト例

```bash
# デフォルト（4スレッド）でのテスト
UV_THREADPOOL_SIZE=4 npm start &
ab -n 10000 -c 100 http://localhost:3000/api/heavy-io
# 結果: 500 req/s

# 16スレッドでのテスト
UV_THREADPOOL_SIZE=16 npm start &
ab -n 10000 -c 100 http://localhost:3000/api/heavy-io
# 結果: 1500 req/s（3倍改善）

# 128スレッドでのテスト
UV_THREADPOOL_SIZE=128 npm start &
ab -n 10000 -c 100 http://localhost:3000/api/heavy-io
# 結果: 2000 req/s（これ以上増やしても改善なし）
```

#### 注意点とトレードオフ

```javascript
// ⚠️ スレッドを増やしすぎるデメリット
// - メモリ使用量増加（1スレッド約1MB）
// - コンテキストスイッチのオーバーヘッド
// - OSのリソース制限に到達

// 最適値の見つけ方
// 1. デフォルトの4から開始
// 2. 負荷テストでボトルネック確認
// 3. 2倍ずつ増やして測定（4→8→16→32）
// 4. 改善が止まった値を採用

// モニタリングポイント
const metrics = {
  'Event Loop Lag': '< 50ms',      // イベントループの遅延
  'I/O Wait Time': '< 100ms',       // I/O待機時間
  'Thread Pool Queue': '< 10',      // キュー待機数
  'CPU Usage': '< 80%',            // CPU使用率
};
```

### デフォルト構成の本番運用可否

| トラフィック規模 | デフォルト構成の適性 | 理由 |
|----------------|-------------------|------|
| 小規模（〜100 req/s） | ✅ 推奨 | Node.jsの非同期I/Oで十分対応可能 |
| 中規模（100-1000 req/s） | ⚠️ 条件付き | CPU使用率とレスポンスタイムを監視必要 |
| 大規模（1000+ req/s） | ❌ 非推奨 | マルチプロセス化が必須 |

#### デフォルト構成が適している場合

```javascript
// ✅ I/Oバウンドなアプリケーション（推奨）
export default async function Page() {
  // データベースアクセス（I/O待機）
  const data = await db.query('SELECT * FROM users');
  
  // 外部API呼び出し（I/O待機）
  const apiData = await fetch('https://api.example.com');
  
  // これらはイベントループで効率的に処理される
  return <div>{/* ... */}</div>;
}

// パフォーマンス特性：
// - 同時接続数: 数千〜数万（メモリ次第）
// - レスポンスタイム: 安定
// - CPU使用率: 低〜中
```

#### デフォルト構成が適さない場合

```javascript
// ❌ CPUバウンドな処理（非推奨）
export default function Page() {
  // CPU集約的な処理でメインスレッドがブロック
  const result = heavyComputation(); // 画像処理、暗号化など
  
  // この間、他のリクエストが処理できない！
  return <div>{result}</div>;
}

// 症状：
// - レスポンスタイムの劣化
// - タイムアウトエラー
// - 503エラー（Service Unavailable）
```

## 5. 本番環境の推奨構成（課題解決）

### 構成選択のフローチャート

```
アプリケーションの特性は？
├─ I/Oバウンド（DB、API中心）
│  └─ トラフィック量は？
│     ├─ 小規模 → デフォルト（next start）で十分
│     ├─ 中規模 → PM2（2-4プロセス）
│     └─ 大規模 → PM2（CPUコア数）+ ロードバランサー
└─ CPUバウンド（計算処理多い）
   └─ 必ずマルチプロセス化（PM2/Cluster）
```

### A. 小規模サービス（スタートアップ、社内ツール）

```javascript
// package.json - デフォルト構成で十分
{
  "scripts": {
    "start": "next start",
    "start:prod": "NODE_ENV=production next start -p 3000"
  }
}

// 想定スペック
// - EC2 t3.small (2 vCPU, 2GB RAM)
// - 同時接続: 〜500
// - リクエスト: 〜100 req/s
// - 月間コスト: $15-20
```

### B. 中規模サービス（一般的なWebサービス）

```javascript
// PM2でマルチプロセス化
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'nextjs-app',
    script: 'node_modules/next/dist/bin/next',
    args: 'start',
    instances: 2,  // 控えめに2プロセスから開始
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    max_memory_restart: '500M',  // メモリリーク対策
  }]
};

// 想定スペック
// - EC2 t3.medium (2 vCPU, 4GB RAM)
// - 同時接続: 〜2000
// - リクエスト: 100-500 req/s
// - 月間コスト: $30-40
```

### C. 大規模サービス（エンタープライズ）

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'nextjs-app',
    script: 'node_modules/next/dist/bin/next',
    args: 'start',
    instances: 'max',  // 全CPUコア使用
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
      // パフォーマンスチューニング
      UV_THREADPOOL_SIZE: 8,  // I/Oスレッド増加
      NODE_OPTIONS: '--max-old-space-size=2048'
    },
    max_memory_restart: '1G',
    min_uptime: '10s',
    max_restarts: 10,
    
    // 監視とアラート
    error_file: './logs/err.log',
    out_file: './logs/out.log',
    merge_logs: true,
    time: true,
  }]
};

// Nginxでロードバランシング
// nginx.conf
upstream nextjs {
  least_conn;  // 接続数が少ないサーバーへ
  server app1:3000 weight=1;
  server app2:3000 weight=1;
  server app3:3000 weight=1;
  keepalive 100;
}

// 想定スペック
// - EC2 c5.xlarge (4 vCPU, 8GB RAM) × 3台
// - ALB + Auto Scaling
// - 同時接続: 10,000+
// - リクエスト: 1000+ req/s
// - 月間コスト: $300-500
```

### D. サーバーレス構成（Vercel/AWS Lambda）

```javascript
// vercel.json - 自動スケーリング
{
  "functions": {
    "app/**/*.js": {
      "maxDuration": 10,
      "memory": 1024
    }
  }
}

// メリット：
// - 自動スケーリング（0→∞）
// - 使った分だけ課金
// - インフラ管理不要
// - グローバルCDN

// デメリット：
// - コールドスタート（初回遅延）
// - 実行時間制限（10-30秒）
// - ベンダーロックイン
```

## 6. プロセス管理の実装詳細

### Node.js Clusterモジュール（PM2を使わない実装）

```javascript
// cluster-server.js
const cluster = require('cluster');
const os = require('os');
const next = require('next');

const dev = process.env.NODE_ENV !== 'production';
const port = parseInt(process.env.PORT, 10) || 3000;

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  
  console.log(`Master ${process.pid} is running`);
  
  // ワーカープロセスを起動
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  // ワーカーが死んだら再起動
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    cluster.fork();
  });
  
} else {
  // ワーカープロセス
  const app = next({ dev });
  const handle = app.getRequestHandler();
  
  app.prepare().then(() => {
    const server = require('http').createServer((req, res) => {
      handle(req, res);
    });
    
    server.listen(port, (err) => {
      if (err) throw err;
      console.log(`Worker ${process.pid} ready on port ${port}`);
    });
  });
}
```

## 5. 本番環境のデプロイ構成

### Vercel（推奨）
```javascript
// vercel.json
{
  "builds": [{
    "src": "package.json",
    "use": "@vercel/next"
  }],
  "routes": [
    {
      "src": "/api/(.*)",
      "dest": "/api/$1",
      "headers": {
        "cache-control": "s-maxage=1, stale-while-revalidate"
      }
    }
  ],
  "functions": {
    "app/**/*.js": {
      "maxDuration": 10
    }
  }
}
```

### Docker + Nginx構成

```dockerfile
# Dockerfile
FROM node:18-alpine AS base

# 依存関係インストール
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json yarn.lock* ./
RUN yarn --frozen-lockfile

# ビルド
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN yarn build

# 本番イメージ
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

```nginx
# nginx.conf
upstream nextjs_upstream {
  server nextjs1:3000;
  server nextjs2:3000;
  server nextjs3:3000;
  
  # ヘルスチェック
  keepalive 64;
}

server {
  listen 80;
  server_name example.com;
  
  # gzip圧縮
  gzip on;
  gzip_types text/plain application/json application/javascript text/css;
  
  # 静的ファイルキャッシュ
  location /_next/static {
    proxy_cache STATIC;
    proxy_pass http://nextjs_upstream;
    add_header Cache-Control "public, max-age=31536000, immutable";
  }
  
  # APIルート
  location /api {
    proxy_pass http://nextjs_upstream;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }
  
  # アプリケーション
  location / {
    proxy_pass http://nextjs_upstream;
    proxy_http_version 1.1;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
  }
}
```

## 6. サーバーレス環境

### AWS Lambda + API Gateway

```javascript
// serverless.yml
service: nextjs-app

provider:
  name: aws
  runtime: nodejs18.x
  region: ap-northeast-1

functions:
  nextApp:
    handler: server.handler
    events:
      - http:
          path: /{proxy+}
          method: ANY
      - http:
          path: /
          method: ANY
    environment:
      NODE_ENV: production

plugins:
  - serverless-nextjs-plugin

custom:
  serverless-nextjs:
    buildDir: .next
```

### Edge Runtime（Vercel Edge Functions）

```typescript
// app/api/edge/route.ts
import { NextRequest } from 'next/server';

export const runtime = 'edge'; // Edge Runtimeを使用

export async function GET(request: NextRequest) {
  // CloudflareワーカーやVercel Edge Networkで実行
  // Node.js APIは使用不可、Web標準APIのみ
  
  const country = request.geo?.country || 'unknown';
  
  return new Response(JSON.stringify({
    message: 'Hello from Edge',
    country,
    timestamp: Date.now()
  }), {
    status: 200,
    headers: {
      'content-type': 'application/json',
      'cache-control': 'public, s-maxage=10'
    }
  });
}
```

## 7. まとめ：デフォルト vs スケールアップ

### デフォルト（next start）の真実

```javascript
// Node.jsの実際の動作
// ❌ 誤解：単一スレッドで全て処理
// ✅ 実際：単一プロセス + マルチスレッドI/O

// スレッド構成：
// - メインスレッド × 1（JavaScript実行）
// - I/Oワーカー × 4（ファイル、DB、ネットワーク）
// - その他 × 2-4（GC、DNS解決など）
```

**デフォルトで十分なケース（70%のプロジェクト）:**
- 社内ツール、管理画面
- 月間100万PV以下のWebサイト
- I/O中心のアプリケーション

**スケールアップが必要なケース（30%）:**
- リアルタイムサービス
- CPU集約的な処理
- 秒間100リクエスト以上

### 段階的スケールアップ戦略

```bash
# Step 1: デフォルトで開始（開発〜初期）
npm run start

# Step 2: 負荷増加時（中期）
npm install pm2
pm2 start ecosystem.config.js -i 2

# Step 3: 本格運用（成熟期）
pm2 start ecosystem.config.js -i max
# + Nginx/ALB
# + Redis/CDN
```

## 8. ワーカープロセスとリクエストキューイング（詳細）

### Puma vs Node.js の並行処理モデル

#### Puma（Rails）のモデル
```ruby
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 2 }  # ワーカープロセス数
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

# リクエスト処理フロー:
# 1. Nginxがリクエスト受信
# 2. Pumaのマスタープロセスが空いているワーカーに振り分け
# 3. ワーカー内のスレッドプールから空きスレッドが処理
# 4. スレッドがビジー → リクエストはキューで待機
# 5. バックログキュー（デフォルト1024）を超えると503エラー

preload_app!
before_fork do
  # ワーカーフォーク前の処理
end
```

#### Node.js（Next.js）のモデル
```javascript
// Node.jsはシングルスレッド・イベントループモデル
// リクエスト処理フロー:
// 1. リクエスト受信 → イベントキューに追加
// 2. イベントループが順次処理（非同期I/O）
// 3. CPU負荷の高い処理 → Worker Threadsで並列化
// 4. I/O待機 → libuv（C++ライブラリ）のスレッドプール

// デフォルトの並行性
// - 同時接続数: 理論上無制限（メモリ依存）
// - I/Oスレッドプール: 4スレッド（UV_THREADPOOL_SIZE）
```

### Next.jsでのワーカープロセス実装

#### PM2とは？

**PM2はデフォルトでは含まれていません。別途インストールが必要です。**

```bash
# PM2のインストール（グローバル）
npm install -g pm2

# またはプロジェクトローカル
npm install --save-dev pm2

# 確認
pm2 --version
```

PM2は本番環境用のNode.jsプロセスマネージャーで、以下の機能を提供：
- 自動クラスタリング（複数ワーカープロセス）
- ゼロダウンタイムリロード
- プロセス監視と自動再起動
- ログ管理
- CPU/メモリ監視

#### 1. PM2によるマルチプロセス管理（Pumaワーカー相当）

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'nextjs-app',
    script: 'node_modules/next/dist/bin/next',
    args: 'start',
    
    // Pumaのworkers相当
    instances: 4,  // または 'max' でCPUコア数分
    exec_mode: 'cluster',
    
    // リクエストの振り分け戦略
    // round-robin: 順番に振り分け（デフォルト）
    // least-connection: 接続数が少ないワーカーへ
    instance_var: 'INSTANCE_ID',
    
    // メモリ制限（Pumaのワーカー再起動相当）
    max_memory_restart: '1G',
    
    // グレースフルリロード
    listen_timeout: 5000,
    kill_timeout: 5000,
    
    // エラー時の再起動
    min_uptime: '10s',
    max_restarts: 10,
  }]
};

// 起動コマンド
// pm2 start ecosystem.config.js
// pm2 reload nextjs-app  # ゼロダウンタイムリロード

// その他のPM2コマンド
// pm2 list              # プロセス一覧
// pm2 monit             # リアルタイム監視
// pm2 logs              # ログ確認
// pm2 stop nextjs-app   # 停止
// pm2 delete nextjs-app # 削除
```

#### 2. Next.jsデフォルトの動作（PM2なし）

```javascript
// Next.jsのデフォルトは単一プロセス
// package.json
{
  "scripts": {
    "dev": "next dev",        // 開発: 単一プロセス
    "build": "next build",    // ビルド
    "start": "next start"     // 本番: 単一プロセス（PM2なし）
  }
}

// next startの内部動作
// - 単一のNode.jsプロセスで動作
// - イベントループで並行処理
// - CPU集約的な処理でブロッキングのリスク
// - スケーラビリティは限定的
```

#### 3. Node.js Clusterによる実装（PM2を使わない場合）

```javascript
// cluster-server.js
const cluster = require('cluster');
const os = require('os');
const next = require('next');
const http = require('http');

// マスタープロセス（Pumaのマスター相当）
if (cluster.isMaster) {
  const numWorkers = process.env.WEB_CONCURRENCY || os.cpus().length;
  
  // ワーカー管理用のマップ
  const workers = new Map();
  
  // ワーカー起動
  for (let i = 0; i < numWorkers; i++) {
    const worker = cluster.fork();
    workers.set(worker.id, {
      id: worker.id,
      pid: worker.process.pid,
      requests: 0,
      connections: 0
    });
  }
  
  // ワーカーへのリクエスト振り分け戦略
  cluster.schedulingPolicy = cluster.SCHED_RR; // Round-robin
  // cluster.SCHED_NONE: OSのスケジューラに任せる（Linux）
  
  // ワーカー監視
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    workers.delete(worker.id);
    
    // 自動再起動
    if (!worker.exitedAfterDisconnect) {
      const newWorker = cluster.fork();
      workers.set(newWorker.id, {
        id: newWorker.id,
        pid: newWorker.process.pid,
        requests: 0,
        connections: 0
      });
    }
  });
  
  // グレースフルシャットダウン
  process.on('SIGTERM', () => {
    console.log('Master received SIGTERM, shutting down gracefully');
    for (const worker of cluster.workers) {
      worker.disconnect();
    }
  });
  
} else {
  // ワーカープロセス
  const app = next({ dev: false });
  const handle = app.getRequestHandler();
  
  app.prepare().then(() => {
    const server = http.createServer((req, res) => {
      // リクエスト処理
      handle(req, res);
    });
    
    // サーバー設定
    server.maxConnections = 1000;  // 最大同時接続数
    server.timeout = 30000;        // タイムアウト（30秒）
    server.keepAliveTimeout = 65000;
    
    server.listen(3000, (err) => {
      if (err) throw err;
      console.log(`Worker ${process.pid} listening on port 3000`);
    });
    
    // グレースフルシャットダウン
    process.on('SIGTERM', () => {
      console.log(`Worker ${process.pid} received SIGTERM`);
      server.close(() => {
        process.exit(0);
      });
    });
  });
}
```

### リクエストキューイングとバックプレッシャー

#### Node.jsのリクエストキューイング

```javascript
// Node.jsの内部キューイング
const http = require('http');
const next = require('next');

const app = next({ dev: false });
const handle = app.getRequestHandler();

app.prepare().then(() => {
  const server = http.createServer(async (req, res) => {
    // Node.jsの内部キュー:
    // 1. TCP接続キュー（OSレベル、backlog）
    // 2. Node.jsイベントキュー
    // 3. libuv I/Oスレッドプール
    
    await handle(req, res);
  });
  
  // TCPバックログ設定（Pumaのqueue_requests相当）
  server.listen(3000, '0.0.0.0', 511, () => {
    // 第3引数の511がバックログサイズ
    // デフォルト: 511（Linux）
    console.log('Server listening with backlog: 511');
  });
  
  // 最大接続数制限
  server.maxConnections = 1000;
  
  // ヘッダータイムアウト（スローロリス攻撃対策）
  server.headersTimeout = 60000;
  server.requestTimeout = 30000;
});
```

#### カスタムキューイング実装

```javascript
// request-queue.js
class RequestQueue {
  constructor(options = {}) {
    this.maxQueueSize = options.maxQueueSize || 100;
    this.maxConcurrent = options.maxConcurrent || 10;
    this.queue = [];
    this.processing = 0;
    this.metrics = {
      totalRequests: 0,
      queuedRequests: 0,
      rejectedRequests: 0,
      averageWaitTime: 0
    };
  }
  
  async add(handler, req, res) {
    // キューサイズチェック（Pumaのqueue_max相当）
    if (this.queue.length >= this.maxQueueSize) {
      this.metrics.rejectedRequests++;
      res.statusCode = 503;
      res.end('Service Unavailable - Queue Full');
      return;
    }
    
    // キューに追加
    const startTime = Date.now();
    this.metrics.queuedRequests++;
    
    return new Promise((resolve, reject) => {
      this.queue.push({
        handler,
        req,
        res,
        startTime,
        resolve,
        reject
      });
      
      this.process();
    });
  }
  
  async process() {
    // 並行処理数チェック
    if (this.processing >= this.maxConcurrent || this.queue.length === 0) {
      return;
    }
    
    const item = this.queue.shift();
    this.processing++;
    
    try {
      // 待機時間記録
      const waitTime = Date.now() - item.startTime;
      this.updateAverageWaitTime(waitTime);
      
      // リクエスト処理
      await item.handler(item.req, item.res);
      item.resolve();
    } catch (error) {
      item.reject(error);
    } finally {
      this.processing--;
      this.metrics.totalRequests++;
      
      // 次のリクエスト処理
      if (this.queue.length > 0) {
        this.process();
      }
    }
  }
  
  updateAverageWaitTime(waitTime) {
    const alpha = 0.1; // 指数移動平均の係数
    this.metrics.averageWaitTime = 
      alpha * waitTime + (1 - alpha) * this.metrics.averageWaitTime;
  }
  
  getMetrics() {
    return {
      ...this.metrics,
      currentQueueSize: this.queue.length,
      currentProcessing: this.processing
    };
  }
}

// 使用例
const queue = new RequestQueue({
  maxQueueSize: 100,
  maxConcurrent: 10
});

const server = http.createServer(async (req, res) => {
  // キューに追加
  await queue.add(handle, req, res);
});

// メトリクスエンドポイント
setInterval(() => {
  console.log('Queue metrics:', queue.getMetrics());
}, 5000);
```

### 負荷制御とレート制限

```javascript
// rate-limiter.js
const rateLimit = require('express-rate-limit');

// Next.jsカスタムサーバーでのレート制限
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // リクエスト数上限
  message: 'Too many requests',
  standardHeaders: true,
  legacyHeaders: false,
  
  // キューイング戦略
  skipSuccessfulRequests: false,
  skipFailedRequests: false,
  
  // カスタムキージェネレーター
  keyGenerator: (req) => {
    return req.ip; // IPアドレスベース
  },
  
  // ストア（Redis使用例）
  store: new RedisStore({
    client: redis,
    prefix: 'rate-limit:'
  })
});

// サーキットブレーカーパターン
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failures = 0;
    this.nextAttempt = Date.now();
  }
  
  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failures = 0;
    if (this.state === 'HALF_OPEN') {
      this.state = 'CLOSED';
    }
  }
  
  onFailure() {
    this.failures++;
    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
    }
  }
}
```

### 比較表：Puma vs Node.js

| 機能 | Puma (Rails) | Node.js (Next.js) |
|-----|-------------|-------------------|
| ワーカープロセス | workers設定 | PM2/Cluster |
| スレッド | threads設定 | Worker Threads |
| リクエストキュー | queue_requests | イベントループ |
| バックログ | backlog設定 | server.listen()の第3引数 |
| 最大キューサイズ | queue_max | カスタム実装必要 |
| 並行処理モデル | マルチプロセス+マルチスレッド | イベントループ+非同期I/O |
| CPU活用 | 自動（複数ワーカー） | 手動（PM2/Cluster） |
| メモリ共有 | プロセス間で独立 | プロセス間で独立 |
| リクエスト振り分け | ラウンドロビン | ラウンドロビン/OS依存 |

## 9. パフォーマンスチューニング

### Node.jsオプション

```bash
# メモリ制限を増やす
NODE_OPTIONS="--max-old-space-size=4096" next start

# V8オプション
NODE_OPTIONS="--max-old-space-size=4096 --optimize-for-size" next start
```

### Next.js設定最適化

```javascript
// next.config.js
module.exports = {
  // 圧縮設定
  compress: true,
  
  // HTTPヘッダー設定
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'X-DNS-Prefetch-Control',
            value: 'on'
          },
          {
            key: 'X-Frame-Options',
            value: 'SAMEORIGIN'
          }
        ]
      }
    ];
  },
  
  // 出力設定
  output: 'standalone', // Dockerデプロイ用
  
  // 実験的機能
  experimental: {
    // ISRメモリキャッシュ
    isrMemoryCacheSize: 0, // MB単位、0で無効
    
    // Worker Threads
    workerThreads: true,
    cpus: 4,
  }
};
```

## 10. モニタリングとロギング

### アプリケーションパフォーマンスモニタリング

```javascript
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    // New Relic
    require('newrelic');
    
    // または Datadog
    const tracer = require('dd-trace').init({
      logInjection: true,
      analytics: true
    });
  }
}

// カスタムメトリクス
import { performance } from 'perf_hooks';

export async function measureServerAction() {
  const start = performance.now();
  
  try {
    // 処理実行
    await someHeavyOperation();
  } finally {
    const duration = performance.now() - start;
    console.log(`Operation took ${duration}ms`);
    
    // メトリクス送信
    metrics.timing('server.action.duration', duration);
  }
}
```

## 🎯 重要な違いのまとめ

| 機能 | Rails (Rack/Puma) | Next.js |
|-----|------------------|---------|
| インターフェース | Rack | Node.js HTTP |
| アプリサーバー | Puma, Unicorn | Node.js内蔵 |
| プロセス管理 | Puma (マルチスレッド/プロセス) | PM2, Cluster |
| 並行処理 | スレッド/プロセス | イベントループ + Worker |
| 静的ファイル配信 | Nginx経由 | CDN or Next.js |
| WebSocket | Action Cable | Socket.io/WS |
| サーバーレス対応 | 限定的 | フルサポート |
| Edge対応 | なし | Edge Runtime |

## まとめ

Next.jsはNode.jsの内蔵HTTPサーバーをベースに構築され、Railsと異なり：

1. **シンプルな構成**: 追加のアプリサーバー不要
2. **イベント駆動**: 非同期I/Oで高い並行性
3. **柔軟なデプロイ**: サーバーフル/サーバーレス両対応
4. **Edge対応**: CDNエッジでの実行可能

本番環境では、PM2やDockerでのクラスタリング、Nginxでのリバースプロキシ構成が一般的です。