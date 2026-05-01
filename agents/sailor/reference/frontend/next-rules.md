# Next.js 管理后台前端规范

## 技术栈

- **框架**：Next.js 16 (App Router)
- **UI**：React 19 + shadcn/ui + Tailwind CSS 4
- **状态管理**：zustand
- **类型**：TypeScript 5

---

## 一、目录分层规范

```
src/
├── app/                      # Next.js App Router 页面
│   ├── (auth)/               # 路由分组：认证相关
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/          # 路由分组：主后台
│   │   ├── layout.tsx       # 后台布局
│   │   ├── users/
│   │   ├── articles/
│   │   └── settings/
│   ├── layout.tsx            # 根布局
│   ├── page.tsx              # 首页
│   └── globals.css           # 全局样式
├── components/               # 组件层
│   ├── ui/                   # shadcn/ui 原始组件（按需引入）
│   └── business/             # 业务特有组件
├── api/                      # API 封装层
│   ├── client.ts             # fetch 封装 / 请求实例
│   ├── request.ts            # 请求/响应拦截器
│   └── modules/              # 分模块 API
│       ├── auth.ts
│       ├── user.ts
│       └── ...
├── lib/                      # 工具库
├── stores/                   # Zustand 状态管理
├── types/                    # TypeScript 类型定义
│   ├── api.d.ts              # API 响应类型
│   ├── user.d.ts
│   └── ...
├── hooks/                   # 自定义 Hooks
├── utils/                   # 纯函数工具
└── constants/               # 常量定义
```

### 命名约定

| 类型 | 约定 | 示例 |
|------|------|------|
| 页面组件 | PascalCase | `UsersPage.tsx` |
| 业务组件 | PascalCase | `UserForm.tsx` |
| UI 组件 | PascalCase | `Button.tsx` |
| Hooks | camelCase，use 前缀 | `useAuth.ts` |
| 工具函数 | camelCase | `formatDate.ts` |
| 常量 | UPPER_SNAKE_CASE | `MAX_PAGE_SIZE` |
| 类型/接口 | PascalCase | `UserInfo`, `ApiResponse` |

---

## 二、API 规范

### 目录结构

```
api/
├── client.ts      # 请求实例配置
├── request.ts    # 拦截器封装
└── modules/       # 分模块 API
```

### 请求封装原则

| 规则 | 说明 |
|------|------|
| 基础 URL | `NEXT_PUBLIC_API_BASE_URL` 环境变量 |
| 请求方法 | `GET` / `POST` / `PUT` / `DELETE` |
| Content-Type | `application/json` |
| 认证 Token | 存储在 Cookie 或 zustand store，通过拦截器注入 |
| 错误处理 | 统一在拦截器处理，区分业务错误和网络错误 |

### 响应格式

```ts
interface ApiResponse<T = unknown> {
  code: number
  data: T
  message: string
}

interface ResponseList<T> {
  list: T[]
  total: number
  page: number
  pageSize: number
}

interface ResponseDetail<T> {
  item: T
}
```

### API 模块示例

```ts
// api/modules/user.ts
import { request } from '../request'

export const userApi = {
  list: (params: UserQuery) =>
    request.get<ResponseList<User>>('/users', { params }),

  detail: (id: string) =>
    request.get<ResponseDetail<User>>(`/users/${id}`),

  create: (data: UserForm) =>
    request.post<User>('/users', data),

  update: (id: string, data: UserForm) =>
    request.put<User>(`/users/${id}`, data),

  delete: (id: string) =>
    request.delete<void>(`/users/${id}`),
}
```

---

## 三、错误码规范

### 错误码分段

| 区间 | 类别 | 说明 |
|------|------|------|
| `0` | 成功 | 操作成功 |
| `1xxx` | 认证授权 | Token 失效、无权限等 |
| `2xxx` | 参数错误 | 请求参数校验失败 |
| `3xxx` | 业务错误 | 业务逻辑校验失败 |
| `4xxx` | 资源错误 | 资源不存在、被占用等 |
| `5xxx` | 服务器错误 | 数据库、文件系统等 |

### 通用错误码（待约定具体值）

| code | message | 说明 |
|------|---------|------|
| `0` | - | 成功 |
| `1001` | `Token 已过期，请重新登录` | Token 失效 |
| `1002` | `无访问权限` | 权限不足 |
| `1003` | `账户已被禁用` | 账户被封禁 |
| `2001` | `参数不能为空` | 必填参数缺失 |
| `2002` | `参数格式错误` | 类型/格式校验失败 |
| `3001` | `数据已存在` | 唯一性校验失败 |
| `3002` | `数据关联引用` | 删除时有关联数据 |
| `4001` | `资源不存在` | 查询目标不存在 |
| `5001` | `服务器内部错误` | 未捕获异常 |

### 前端错误处理策略

| 场景 | 处理方式 |
|------|----------|
| `code: 0` | 直接返回 `data` |
| `code: 1001~1003` | 跳转登录页，清除 Token |
| `code: 2001~2002` | Toast 提示具体字段错误 |
| `code: 3xxx~4xxx` | Toast 提示 `message` |
| `code: 5xxx` | Toast 提示"服务异常"，上报日志 |
| 网络错误 | Toast 提示"网络异常"，重试机制 |

---

## 四、状态管理规范

### Zustand Store 组织

```
stores/
├── useAuthStore.ts      # 认证状态
├── useUserStore.ts     # 用户信息
└── useUIStore.ts       # UI 状态（侧边栏、主题等）
```

### Store 设计原则

- 按业务域拆分，避免单一 store 膨胀
- 异步操作在 store 内封装，不暴露原始 dispatch
- 敏感数据（Token）存在内存中，页面刷新后重新获取

---

## 五、组件规范

### shadcn/ui 使用原则

- **按需引入**：`components/ui/` 下按需引入组件，避免全量导入
- **不修改源码**：如需定制，继承后扩展，不改原始组件
- **统一包装**：业务组件对 UI 组件进行二次封装，添加默认样式和业务逻辑

### Props 类型化

```tsx
interface UserFormProps {
  mode: 'create' | 'edit'
  userId?: string
  onSuccess: () => void
  onCancel: () => void
}
```
