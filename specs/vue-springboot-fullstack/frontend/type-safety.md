# 类型安全

TypeScript 严格模式约定、OpenAPI 驱动的类型生成、与后端 DTO 类型同步。

## TS 配置基线

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noImplicitAny": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "useUnknownInCatchVariables": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "jsx": "preserve",
    "types": ["vite/client", "element-plus/global"],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*", "src/**/*.vue"]
}
```

## 强制规则

| 规则                                | 原因                                        |
| ----------------------------------- | ------------------------------------------- |
| 禁用 `any`                          | 失去全部类型保护                            |
| 禁用 `@ts-ignore` / `@ts-expect-error` | 隐藏问题                                    |
| 禁用非空断言 `!`                    | 在边界处用 narrowing 替代                   |
| 禁止隐式返回 `void` 与 `undefined`  | 容易写错条件分支                            |
| catch 形参用 `unknown`              | `useUnknownInCatchVariables: true`           |

### narrowing 替代非空断言

```typescript
// 反模式
function getName(user: User | null) {
  return user!.name;
}

// 正确
function getName(user: User | null) {
  if (!user) throw new Error('user is null');
  return user.name;
}
```

## 后端类型自动生成

后端接入 `springdoc-openapi`（详见 [backend/configuration.md](../backend/configuration.md)），暴露 `/v3/api-docs` JSON。前端使用 `openapi-typescript` 生成 TS 类型。

### 安装

```bash
pnpm add -D openapi-typescript
```

### 生成脚本

```json
// package.json
{
  "scripts": {
    "gen:api": "openapi-typescript http://localhost:8080/v3/api-docs -o src/api/generated/schema.d.ts"
  }
}
```

CI 中跑：

```bash
pnpm run gen:api
git diff --exit-code src/api/generated/schema.d.ts
```

差异非 0 即报错（提示开发者重新生成并提交）。

### 使用生成的类型

`openapi-typescript` 输出 `paths` 与 `components.schemas`。封装出友好的工具类型：

```typescript
// src/api/types/index.ts
import type { components, paths } from '@/api/generated/schema';

// 直接拿 DTO/VO
export type OrderVO = components['schemas']['OrderVO'];
export type OrderCreateDTO = components['schemas']['OrderCreateDTO'];

// 从 path 派生 request/response
type Get<P extends keyof paths, M extends keyof paths[P]> = paths[P][M];

export type ListOrdersResponse =
  Get<'/api/orders', 'get'> extends { responses: { 200: { content: { 'application/json': infer R } } } }
    ? R
    : never;
```

实际使用中 API 模块文件直接 import：

```typescript
// src/api/modules/order.ts
import type { OrderVO, OrderCreateDTO, OrderQuery, PageResult } from '@/api/types';

export const orderApi = {
  list(query: OrderQuery): Promise<PageResult<OrderVO>> { /* ... */ },
};
```

## 禁止手抄后端 DTO

```typescript
// 反模式：手写
export interface OrderVO {
  id: string;
  total: number;
  status: string;
}

// 正确：从生成的 schema 导出
export type { OrderVO } from '@/api/types';
```

后端字段一改，前端类型自动报红，配合 CI 拦截。

## 视图模型 vs 后端 DTO

后端 DTO 是"线上格式"，前端视图模型可以更友好（带计算字段、转换枚举等）：

```typescript
// modules/order/types/index.ts
import type { OrderVO } from '@/api/types';

export interface OrderViewModel {
  id: string;
  reference: string;
  totalText: string;            // "￥1,200.00"
  status: 'PENDING' | 'PAID' | 'CANCELLED';
  statusLabel: string;          // "待付款"
  canCancel: boolean;
  rawAmount: number;
}

export function toOrderViewModel(vo: OrderVO): OrderViewModel { /* ... */ }
```

> 视图模型在 `modules/[feature]/types/`，由 composable 或 mapper 函数生成。

## Vue 组件 Props 类型

```vue
<script setup lang="ts">
import type { OrderViewModel } from '@/modules/order';

interface Props {
  order: OrderViewModel;
  editable?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  editable: false,
});
</script>
```

## 全局类型

```typescript
// src/env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_APP_TITLE: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}

declare module '*.vue' {
  import type { DefineComponent } from 'vue';
  const c: DefineComponent;
  export default c;
}
```

## 路由 meta 类型扩展

```typescript
// src/router/types.d.ts
import 'vue-router';

declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean;
    public?: boolean;
    title?: string;
    icon?: string;
    hideInMenu?: boolean;
    roles?: string[];
  }
}
```

## Pinia store 类型

组合式 store 自动类型推断：

```typescript
export const useUserStore = defineStore('user', () => {
  const userInfo = ref<UserVO | null>(null);
  return { userInfo };
});
// useUserStore().userInfo 自动是 Ref<UserVO | null>
```

## 异步函数返回值

显式声明 Promise 返回类型，避免隐式 `Promise<any>`：

```typescript
// 推荐
async function fetchUser(id: string): Promise<UserVO> {
  const res = await userApi.detail(id);
  return res;
}
```

## 工具类型

利用内置工具类型，少造轮子：

```typescript
type OrderInput = Omit<OrderCreateDTO, 'createdAt' | 'id'>;
type PartialOrder = Partial<OrderVO>;
type OrderId = OrderVO['id'];
type OrderStatus = OrderVO['status'];
```

## 常见误区

### `as any` / `as unknown as X`

只能用于：

1. 与第三方库声明缺失对接
2. 测试桩
3. 经过明确注释解释的不可避免场景

主代码路径禁止。

### 数组 reduce 类型推断

```typescript
// 推断出来是 number | OrderItem，不符合预期
items.reduce((acc, i) => acc + i.price, 0);

// 显式指定泛型
items.reduce<number>((acc, i) => acc + i.price, 0);
```

## 最佳实践

1. **strict 模式 + 自动生成类型**
2. **后端类型不手写**
3. **DTO 与视图模型分离**
4. **组件 props 用 interface**
5. **catch 用 `unknown` + 类型守卫**

## 反模式

- 复制后端 Java 类字段到 TS
- `as any` 修饰整个对象
- 用 `Record<string, any>` 替代精确类型
- props 用 `Object as PropType` 老式写法
- 生成类型不入库（每次开发自己生成，CI 跑不通）
