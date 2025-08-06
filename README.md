# Next.js Learning Guide for Rails Developers

Rails/React/MySQL経験者向けのNext.js/Prisma/PostgreSQL完全学習ガイド

## 📚 概要

このリポジトリは、Rails開発経験者がNext.js、Prisma、PostgreSQLの技術スタックをマスターするための包括的な学習リソースです。各トピックはRailsの概念と比較しながら解説されており、スムーズな移行を支援します。

## 🎯 対象読者

- Rails、React、MySQLの開発経験がある方
- Next.js App Router、Prisma、PostgreSQLを学習したい方
- モダンなフルスタックWeb開発を習得したい方

## 📖 学習コンテンツ

### 基礎編

1. **[アーキテクチャ比較](./docs/01_architecture_comparison.md)**
   - Rails MVC vs Next.js App Router
   - ルーティングシステムの違い
   - レンダリング戦略の比較

2. **[ORM比較：ActiveRecord vs Prisma](./docs/02_activerecord_vs_prisma.md)**
   - 基本的なCRUD操作
   - リレーション定義
   - マイグレーション管理

3. **[API開発パターン](./docs/03_api_development_patterns.md)**
   - Rails Controller vs Next.js API Routes
   - RESTful API設計
   - エラーハンドリング

4. **[データベース設計とマイグレーション](./docs/04_database_migration.md)**
   - スキーマ定義の違い
   - PostgreSQL特有の機能
   - マイグレーション戦略

### 認証・認可

5. **[認証・認可システム](./docs/05_authentication_authorization.md)**
   - Devise vs NextAuth
   - Pundit vs CASL
   - セッション管理

18. **[クライアント・サーバー認証](./docs/18_client_server_auth.md)**
    - サーバーコンポーネントでの認証
    - クライアントコンポーネントでの認証
    - 認証フローパターン

20. **[NextAuth テーブル構造](./docs/20_nextauth_table_structure.md)**
    - 必須テーブルの詳細
    - Deviseとの比較
    - カスタマイズ方法

### バッチ処理・外部連携

6. **[バッチ処理](./docs/06_batch_processing.md)**
   - Sidekiq/Whenever vs Next.js
   - スケジュールジョブ実装
   - バックグラウンド処理

14. **[CLIコマンドでのバッチ処理](./docs/14_cli_commands_for_batch.md)**
    - バッチコマンドの作成方法
    - ECSでの実行パターン
    - エラーハンドリング

7. **[外部API提供](./docs/07_external_api_provision.md)**
   - RESTful API設計
   - 認証・レート制限
   - Webhook実装

### デプロイメント

8. **[ECSデプロイメント](./docs/08_ecs_deployment.md)**
   - Dockerコンテナ化
   - ECSタスク定義
   - CI/CDパイプライン

### Next.js詳細機能

9. **[サーバー/クライアントコンポーネント詳解](./docs/09_server_client_components_detail.md)**
   - レンダリング戦略
   - データフェッチング
   - パフォーマンス最適化

15. **[generateStaticParams詳解](./docs/15_generateStaticParams_detail.md)**
    - 静的生成の仕組み
    - 動的ルートの事前生成
    - ISRとの組み合わせ

### パフォーマンス・最適化

11. **[Prismaトランザクション詳解](./docs/11_prisma_transaction_detail.md)**
    - トランザクション分離レベル
    - MySQL vs PostgreSQLの違い
    - バッチ処理パターン

12. **[Promise.all vs 逐次実行](./docs/12_promise_all_vs_sequential.md)**
    - 並列処理のパフォーマンス
    - 使い分けの指針
    - エラーハンドリング

13. **[LRUキャッシュストレージ](./docs/13_lru_cache_storage.md)**
    - メモリキャッシュの仕組み
    - サーバーレス環境での制限
    - 代替ソリューション

17. **[SWRキャッシュ安全性](./docs/17_swr_cache_safety.md)**
    - グローバルキャッシュの危険性
    - 安全なキー設計
    - 事故防止パターン

### 開発パターン

16. **[Next.js Route Helpers](./docs/16_nextjs_route_helpers.md)**
    - Rails風URLヘルパー実装
    - 型安全なルーティング
    - SWRとの統合

19. **[ディレクトリ構造設計](./docs/19_nextjs_directory_structure.md)**
    - 大規模アプリケーション構造
    - DDD/Vertical Slice Architecture
    - スケーリング戦略

21. **[Webサーバーアーキテクチャ](./docs/21_nextjs_web_server_architecture.md)**
    - Rails Puma vs Next.js Node.js
    - ワーカープロセスとスレッドモデル
    - 本番環境のスケーリング戦略

### 実践プロジェクト

10. **[実践プロジェクト](./docs/10_practice_project.md)**
    - タスク管理SaaSの実装
    - 全機能の統合実装
    - ベストプラクティス適用

## 🚀 学習の進め方

### 推奨学習順序

1. **基礎理解（2-3時間）**
   - アーキテクチャ比較（01）
   - ORM比較（02）
   - API開発パターン（03）

2. **実装パターン習得（2-3時間）**
   - 認証・認可（05, 18, 20）
   - サーバー/クライアントコンポーネント（09）
   - ディレクトリ構造（19）

3. **高度な機能（2-3時間）**
   - トランザクション（11）
   - バッチ処理（06, 14）
   - デプロイメント（08）

4. **実践演習（3-4時間）**
   - 実践プロジェクト（10）
   - 各機能の統合実装

## 💡 学習のポイント

### Rails開発者向けの重要な違い

1. **ルーティング**
   - Rails: `routes.rb`で一元管理
   - Next.js: ファイルシステムベース

2. **ORM**
   - ActiveRecord: 規約重視、マジックメソッド多数
   - Prisma: 型安全、明示的な定義

3. **認証**
   - Devise: フル機能の認証システム
   - NextAuth: OAuth中心、カスタマイズ前提

4. **デプロイ**
   - Rails: Capistrano、Heroku
   - Next.js: Vercel、AWS ECS、Docker

## 🛠 環境構築

```bash
# Node.js 18以上が必要
node --version

# プロジェクト作成
npx create-next-app@latest my-app --typescript --tailwind --app

# Prisma セットアップ
npm install prisma @prisma/client
npx prisma init

# NextAuth セットアップ
npm install next-auth @auth/prisma-adapter

# 開発サーバー起動
npm run dev
```

## 📝 各ドキュメントの概要

| ファイル | 内容 | 学習時間目安 |
|---------|------|-------------|
| 01_architecture_comparison.md | アーキテクチャの基本比較 | 30分 |
| 02_activerecord_vs_prisma.md | ORM詳細比較と実装例 | 45分 |
| 03_api_development_patterns.md | API設計パターン | 30分 |
| 04_database_migration.md | DB設計とマイグレーション | 30分 |
| 05_authentication_authorization.md | 認証認可の実装 | 45分 |
| 06_batch_processing.md | バッチ処理パターン | 30分 |
| 07_external_api_provision.md | 外部API提供方法 | 30分 |
| 08_ecs_deployment.md | ECSデプロイ詳細 | 45分 |
| 09_server_client_components_detail.md | コンポーネント詳解 | 45分 |
| 10_practice_project.md | 実践プロジェクト | 3時間 |
| 11_prisma_transaction_detail.md | トランザクション詳解 | 30分 |
| 12_promise_all_vs_sequential.md | 非同期処理パターン | 20分 |
| 13_lru_cache_storage.md | キャッシュ管理 | 20分 |
| 14_cli_commands_for_batch.md | CLIバッチコマンド | 30分 |
| 15_generateStaticParams_detail.md | 静的生成詳解 | 30分 |
| 16_nextjs_route_helpers.md | ルートヘルパー実装 | 30分 |
| 17_swr_cache_safety.md | SWRキャッシュ安全性 | 30分 |
| 18_client_server_auth.md | 認証パターン詳解 | 30分 |
| 19_nextjs_directory_structure.md | ディレクトリ設計 | 30分 |
| 20_nextauth_table_structure.md | NextAuthテーブル構造 | 30分 |
| 21_nextjs_web_server_architecture.md | Webサーバーアーキテクチャ | 45分 |

## 🤝 コントリビューション

このガイドは継続的に改善されています。以下の方法で貢献できます：

- 誤字脱字の修正
- より良い説明の提案
- 新しいトピックの追加
- サンプルコードの改善

## 📄 ライセンス

MIT License

## 🔗 関連リンク

- [Next.js公式ドキュメント](https://nextjs.org/docs)
- [Prisma公式ドキュメント](https://www.prisma.io/docs)
- [NextAuth.js公式ドキュメント](https://next-auth.js.org)
- [PostgreSQL公式ドキュメント](https://www.postgresql.org/docs/)

## ✨ Special Thanks

Rails開発者コミュニティとNext.js開発者コミュニティの皆様に感謝します。
