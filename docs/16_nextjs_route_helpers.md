# Next.js Route Helpers - Rails風URLヘルパー実装

## 📚 RailsのRoute Helperに相当する機能をNext.jsで実装

Railsの`users_path`や`edit_user_path(user)`のような、型安全で保守しやすいルートヘルパーを作成します。

## 1. 基本的なRoute Helper実装

### シンプルな実装
```typescript
// lib/routes.ts
export const routes = {
  home: () => '/',
  login: () => '/login',
  dashboard: () => '/dashboard',
  
  // 動的ルート
  user: (id: string) => `/users/${id}`,
  userEdit: (id: string) => `/users/${id}/edit`,
  
  // クエリパラメータ付き
  users: (params?: { page?: number; filter?: string }) => {
    const query = new URLSearchParams();
    if (params?.page) query.append('page', params.page.toString());
    if (params?.filter) query.append('filter', params.filter);
    return `/users${query.toString() ? `?${query}` : ''}`;
  },
  
  // API Routes
  api: {
    users: () => '/api/users',
    user: (id: string) => `/api/users/${id}`,
    tasks: (workspaceId: string, params?: { projectId?: string; status?: string }) => {
      const query = new URLSearchParams();
      if (params?.projectId) query.append('projectId', params.projectId);
      if (params?.status) query.append('status', params.status);
      return `/api/workspaces/${workspaceId}/tasks${query.toString() ? `?${query}` : ''}`;
    }
  }
} as const;

// 使用例
import { routes } from '@/lib/routes';

// ページ遷移
<Link href={routes.user('123')}>View User</Link>
<Link href={routes.users({ page: 2, filter: 'active' })}>Page 2</Link>

// API呼び出し
const { data } = useSWR(
  routes.api.tasks(workspaceId, { projectId, status: filter.status }),
  fetcher
);
```

## 2. 型安全な高度な実装

### TypeScript型定義付きRoute Helper
```typescript
// lib/routes/types.ts
export interface RouteParams {
  users: {
    page?: number;
    limit?: number;
    filter?: string;
    sort?: 'asc' | 'desc';
  };
  user: {
    id: string;
  };
  workspace: {
    slug: string;
  };
  project: {
    workspaceSlug: string;
    projectId: string;
  };
  task: {
    workspaceSlug: string;
    projectId: string;
    taskId: string;
  };
}

export interface ApiRouteParams {
  users: {
    query?: {
      page?: number;
      limit?: number;
      search?: string;
    };
  };
  user: {
    id: string;
  };
  tasks: {
    workspaceId: string;
    query?: {
      projectId?: string;
      status?: 'TODO' | 'IN_PROGRESS' | 'DONE';
      assigneeId?: string;
    };
  };
}

// lib/routes/index.ts
import { RouteParams, ApiRouteParams } from './types';

class RouteBuilder {
  // ページルート
  home = () => '/' as const;
  login = () => '/login' as const;
  signup = () => '/signup' as const;
  dashboard = () => '/dashboard' as const;
  
  users = (params?: RouteParams['users']) => {
    const query = this.buildQuery(params);
    return `/users${query}` as const;
  };
  
  user = (params: RouteParams['user']) => {
    return `/users/${params.id}` as const;
  };
  
  userEdit = (params: RouteParams['user']) => {
    return `/users/${params.id}/edit` as const;
  };
  
  workspace = (params: RouteParams['workspace']) => {
    return `/workspaces/${params.slug}` as const;
  };
  
  project = (params: RouteParams['project']) => {
    return `/workspaces/${params.workspaceSlug}/projects/${params.projectId}` as const;
  };
  
  task = (params: RouteParams['task']) => {
    return `/workspaces/${params.workspaceSlug}/projects/${params.projectId}/tasks/${params.taskId}` as const;
  };
  
  // APIルート
  api = {
    users: (params?: ApiRouteParams['users']) => {
      const query = this.buildQuery(params?.query);
      return `/api/users${query}` as const;
    },
    
    user: (params: ApiRouteParams['user']) => {
      return `/api/users/${params.id}` as const;
    },
    
    tasks: (params: ApiRouteParams['tasks']) => {
      const query = this.buildQuery(params.query);
      return `/api/workspaces/${params.workspaceId}/tasks${query}` as const;
    },
    
    createTask: (workspaceId: string) => {
      return `/api/workspaces/${workspaceId}/tasks` as const;
    },
    
    updateTask: (workspaceId: string, taskId: string) => {
      return `/api/workspaces/${workspaceId}/tasks/${taskId}` as const;
    }
  };
  
  // ヘルパーメソッド
  private buildQuery(params?: Record<string, any>): string {
    if (!params) return '';
    
    const query = new URLSearchParams();
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        query.append(key, String(value));
      }
    });
    
    const queryString = query.toString();
    return queryString ? `?${queryString}` : '';
  }
  
  // 絶対URLを生成
  absolute(path: string): string {
    const baseUrl = process.env.NEXT_PUBLIC_BASE_URL || 'http://localhost:3000';
    return `${baseUrl}${path}`;
  }
}

export const routes = new RouteBuilder();

// 使用例
import { routes } from '@/lib/routes';

// 型安全な使用
const userUrl = routes.user({ id: '123' }); // /users/123
const tasksUrl = routes.api.tasks({
  workspaceId: 'ws-123',
  query: {
    projectId: 'proj-456',
    status: 'IN_PROGRESS' // 型チェックされる
  }
});
```

## 3. SWRとの統合

### カスタムフック with Route Helper
```typescript
// hooks/useApi.ts
import useSWR, { SWRConfiguration } from 'swr';
import { routes } from '@/lib/routes';

const fetcher = (url: string) => fetch(url).then(res => res.json());

// タスク用カスタムフック
export function useTasks(
  workspaceId: string,
  options?: {
    projectId?: string;
    status?: 'TODO' | 'IN_PROGRESS' | 'DONE';
  },
  swrOptions?: SWRConfiguration
) {
  const url = routes.api.tasks({
    workspaceId,
    query: options
  });
  
  return useSWR(url, fetcher, swrOptions);
}

// ユーザー用カスタムフック
export function useUser(userId: string) {
  const url = routes.api.user({ id: userId });
  return useSWR(url, fetcher);
}

// 使用例
function TaskList({ workspaceId, projectId }: Props) {
  const { data, error, mutate } = useTasks(
    workspaceId,
    { projectId, status: 'IN_PROGRESS' },
    { refreshInterval: 5000 }
  );
  
  if (error) return <div>Error loading tasks</div>;
  if (!data) return <div>Loading...</div>;
  
  return <div>{/* タスク表示 */}</div>;
}
```

## 4. Next.js Link統合

### 型安全なLinkコンポーネント
```typescript
// components/TypedLink.tsx
import Link, { LinkProps } from 'next/link';
import { routes } from '@/lib/routes';

type Routes = typeof routes;
type RouteName = keyof Routes;

interface TypedLinkProps extends Omit<LinkProps, 'href'> {
  to: RouteName;
  params?: any;
  children: React.ReactNode;
}

export function TypedLink({ to, params, children, ...props }: TypedLinkProps) {
  const route = routes[to];
  const href = typeof route === 'function' ? route(params) : route;
  
  return (
    <Link href={href} {...props}>
      {children}
    </Link>
  );
}

// 使用例
<TypedLink to="user" params={{ id: '123' }}>
  View User
</TypedLink>

<TypedLink to="workspace" params={{ slug: 'my-workspace' }}>
  Go to Workspace
</TypedLink>
```

## 5. 環境別URL管理

### 環境に応じたベースURL
```typescript
// lib/routes/config.ts
export class RouteConfig {
  private baseUrl: string;
  private apiBaseUrl: string;
  
  constructor() {
    this.baseUrl = this.getBaseUrl();
    this.apiBaseUrl = this.getApiBaseUrl();
  }
  
  private getBaseUrl(): string {
    if (typeof window !== 'undefined') {
      // クライアントサイド
      return window.location.origin;
    }
    
    // サーバーサイド
    return process.env.NEXT_PUBLIC_BASE_URL || 'http://localhost:3000';
  }
  
  private getApiBaseUrl(): string {
    // 環境別API URL
    switch (process.env.NEXT_PUBLIC_ENV) {
      case 'production':
        return 'https://api.example.com';
      case 'staging':
        return 'https://staging-api.example.com';
      default:
        return this.getBaseUrl();
    }
  }
  
  // 外部API用
  external = {
    payment: (orderId: string) => 
      `${this.apiBaseUrl}/v1/payments/${orderId}`,
    
    webhook: (event: string) => 
      `${this.apiBaseUrl}/webhooks/${event}`
  };
  
  // 絶対URL生成
  absolute(path: string): string {
    return `${this.baseUrl}${path}`;
  }
}

export const routeConfig = new RouteConfig();
```

## 6. Railsスタイル実装

### Rails風の命名規則
```typescript
// lib/rails-routes.ts
class RailsStyleRoutes {
  // Rails: users_path
  usersPath = (params?: { page?: number }) => {
    const query = params?.page ? `?page=${params.page}` : '';
    return `/users${query}`;
  };
  
  // Rails: user_path(user)
  userPath = (user: { id: string } | string) => {
    const id = typeof user === 'string' ? user : user.id;
    return `/users/${id}`;
  };
  
  // Rails: edit_user_path(user)
  editUserPath = (user: { id: string } | string) => {
    const id = typeof user === 'string' ? user : user.id;
    return `/users/${id}/edit`;
  };
  
  // Rails: new_user_path
  newUserPath = () => '/users/new';
  
  // Rails: workspace_project_tasks_path(workspace, project)
  workspaceProjectTasksPath = (
    workspace: { slug: string } | string,
    project: { id: string } | string,
    params?: { filter?: string }
  ) => {
    const workspaceSlug = typeof workspace === 'string' ? workspace : workspace.slug;
    const projectId = typeof project === 'string' ? project : project.id;
    const query = params?.filter ? `?filter=${params.filter}` : '';
    
    return `/workspaces/${workspaceSlug}/projects/${projectId}/tasks${query}`;
  };
  
  // Rails: api_v1_users_url
  apiV1UsersUrl = () => {
    const baseUrl = process.env.NEXT_PUBLIC_API_URL || '';
    return `${baseUrl}/api/v1/users`;
  };
}

export const r = new RailsStyleRoutes();

// 使用例（Rails風）
import { r } from '@/lib/rails-routes';

<Link href={r.userPath(user)}>View User</Link>
<Link href={r.editUserPath('123')}>Edit</Link>
<Link href={r.workspaceProjectTasksPath(workspace, project, { filter: 'active' })}>
  Tasks
</Link>

// API呼び出し
const { data } = useSWR(r.apiV1UsersUrl(), fetcher);
```

## 7. テスト

```typescript
// __tests__/routes.test.ts
import { routes } from '@/lib/routes';

describe('Route Helpers', () => {
  test('generates user path', () => {
    expect(routes.user({ id: '123' })).toBe('/users/123');
  });
  
  test('generates API path with query params', () => {
    const url = routes.api.tasks({
      workspaceId: 'ws-123',
      query: {
        projectId: 'proj-456',
        status: 'TODO'
      }
    });
    
    expect(url).toBe('/api/workspaces/ws-123/tasks?projectId=proj-456&status=TODO');
  });
  
  test('handles optional params', () => {
    expect(routes.users()).toBe('/users');
    expect(routes.users({ page: 2 })).toBe('/users?page=2');
  });
});
```

## 💡 Rails vs Next.js Route Helper比較

| Rails | Next.js実装 | 説明 |
|-------|------------|------|
| `users_path` | `routes.users()` | 一覧ページ |
| `user_path(@user)` | `routes.user({ id: user.id })` | 詳細ページ |
| `edit_user_path(@user)` | `routes.userEdit({ id: user.id })` | 編集ページ |
| `users_path(page: 2)` | `routes.users({ page: 2 })` | クエリパラメータ |
| `api_v1_users_path` | `routes.api.users()` | API Route |
| `users_url` | `routes.absolute(routes.users())` | 絶対URL |

**メリット：**
- 型安全性
- URLの一元管理
- リファクタリングが容易
- 自動補完サポート
- テスト可能