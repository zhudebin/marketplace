# API 集成（axios）

axios 实例的封装、请求/响应拦截器、统一响应解包、错误处理与重试。

## 总体原则

1. **唯一入口**：全局只有一个 axios 实例（`http`），所有 API 模块都基于它
2. **类型由后端 OpenAPI 自动生成**（参见 [type-safety.md](./type-safety.md)）
3. **拦截器负责"协议层"**：token、统一响应解包、错误转换
4. **业务层只关心 data**，看不到 `success/code/message` 的包装
5. **组件不直接 import axios**，必须经过模块化的 API 文件

## axios 实例

```typescript
// src/api/http.ts
import axios, {
  type AxiosError,
  type AxiosResponse,
  type InternalAxiosRequestConfig,
} from 'axios';
import { ElMessage } from 'element-plus';
import { useUserStore } from '@/stores/user';
import { router } from '@/router';
import type { Result } from '@/api/types/result';

const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 15000,
  withCredentials: false,
});

// ---------- 请求拦截 ----------
http.interceptors.request.use((config: InternalAxiosRequestConfig) => {
  const userStore = useUserStore();
  if (userStore.token) {
    config.headers.set('Authorization', `Bearer ${userStore.token}`);
  }
  // 给所有请求带上 traceId（可选，用于和后端日志关联）
  config.headers.set('X-Trace-Id', crypto.randomUUID());
  return config;
});

// ---------- 响应拦截 ----------
http.interceptors.response.use(
  // 业务成功路径：解包 data
  (response: AxiosResponse<Result<unknown>>) => {
    const body = response.data;
    if (!body || typeof body.success === 'undefined') {
      // 不符合统一响应（例如下载文件、第三方接口），原样返回
      return response;
    }
    if (body.success) {
      // 统一响应正常：把 body.data 直接作为业务数据返回
      // 让调用方拿到的就是 T 而不是 Result<T>
      response.data = body.data as never;
      return response;
    }
    // success === false，作为错误统一处理
    return Promise.reject(new BusinessError(body.code, body.message, body));
  },
  // 网络层 / HTTP 错误
  (error: AxiosError<Result<unknown>>) => {
    const status = error.response?.status;
    const body = error.response?.data;

    if (status === 401) {
      const userStore = useUserStore();
      userStore.logout();
      router.replace({
        name: 'Login',
        query: { redirect: router.currentRoute.value.fullPath },
      });
      return Promise.reject(new BusinessError('UNAUTHORIZED', '登录已过期'));
    }

    if (status === 403) {
      ElMessage.error('无权限');
      return Promise.reject(new BusinessError('FORBIDDEN', '无权限'));
    }

    const message = body?.message ?? error.message ?? '网络异常';
    return Promise.reject(new BusinessError(body?.code ?? 'NETWORK_ERROR', message, body));
  },
);

export class BusinessError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly raw?: unknown,
  ) {
    super(message);
    this.name = 'BusinessError';
  }
}

export default http;
```

> **核心要点**：
>
> - 拦截器把 `Result<T>.data` 解包成 `T`，调用方代码非常干净
> - 业务失败统一抛 `BusinessError`，便于在组件用 `try/catch` 区分
> - 401 自动跳登录、403 提示无权限，不需要每个调用方处理
> - 5xx / 网络错误也统一为 `BusinessError`

## API 模块封装

每个业务领域一个文件，**只暴露纯 Promise 函数**，不写副作用、不写 toast：

```typescript
// src/api/modules/order.ts
import http from '@/api/http';
import type {
  OrderCreateDTO,
  OrderQuery,
  OrderVO,
  PageResult,
} from '@/api/generated/schema';

export const orderApi = {
  list(query: OrderQuery): Promise<PageResult<OrderVO>> {
    return http.get('/api/orders', { params: query }).then((r) => r.data);
  },
  detail(id: string): Promise<OrderVO> {
    return http.get(`/api/orders/${id}`).then((r) => r.data);
  },
  create(payload: OrderCreateDTO): Promise<OrderVO> {
    return http.post('/api/orders', payload).then((r) => r.data);
  },
  cancel(id: string): Promise<void> {
    return http.post(`/api/orders/${id}/cancel`).then(() => undefined);
  },
};
```

> 由于响应拦截器已经把 `Result<T>` 解包，这里 `r.data` 就是业务数据 `T`。

### 聚合导出

```typescript
// src/api/modules/index.ts
export { orderApi } from './order';
export { userApi } from './user';
export { authApi } from './auth';
```

```typescript
// 业务调用
import { orderApi } from '@/api/modules';
const list = await orderApi.list({ page: 1, pageSize: 20 });
```

## 统一响应类型

```typescript
// src/api/types/result.ts
export interface Result<T> {
  success: boolean;
  code: string;
  message: string;
  data: T;
  traceId?: string;
}

export interface PageResult<T> {
  records: T[];
  total: number;
  pageNum: number;
  pageSize: number;
}
```

字段名与后端 [api-contract.md](../shared/api-contract.md) 保持完全一致。

## 错误处理三层

| 层级       | 谁处理              | 例子                                  |
| ---------- | ------------------- | ------------------------------------- |
| 协议层     | 拦截器              | 401 → 跳登录、403 → 提示              |
| 通用业务   | 拦截器              | success=false → 抛 BusinessError      |
| 具体业务   | 调用方组件 / store  | "用户名已存在" → 表单字段红字         |

调用方示例：

```typescript
async function handleCreate() {
  try {
    await orderApi.create(form.value);
    ElMessage.success('创建成功');
  } catch (e) {
    if (e instanceof BusinessError && e.code === 'INVENTORY_NOT_ENOUGH') {
      ElMessage.warning('库存不足');
      return;
    }
    ElMessage.error(e instanceof Error ? e.message : '操作失败');
  }
}
```

## 文件下载

下载场景拦截器不能解包，需要使用 `responseType: 'blob'`：

```typescript
export function exportOrders(query: OrderQuery): Promise<Blob> {
  return http
    .get('/api/orders/export', {
      params: query,
      responseType: 'blob',
    })
    .then((r) => r.data);
}

// 调用
const blob = await exportOrders(filter.value);
const url = URL.createObjectURL(blob);
const a = document.createElement('a');
a.href = url;
a.download = 'orders.xlsx';
a.click();
URL.revokeObjectURL(url);
```

> 后端导出接口在响应头 `Content-Type: application/octet-stream` 时不要包 `Result<T>`。

## 文件上传

```typescript
export function uploadAvatar(file: File): Promise<{ url: string }> {
  const fd = new FormData();
  fd.append('file', file);
  return http
    .post('/api/files/avatar', fd, {
      headers: { 'Content-Type': 'multipart/form-data' },
    })
    .then((r) => r.data);
}
```

## 取消请求

使用 `AbortController`：

```typescript
const controller = new AbortController();
http.get('/api/long', { signal: controller.signal });

// 切页面、切 tab
controller.abort();
```

通常封装到 composable 中：

```typescript
import { onBeforeUnmount } from 'vue';

export function useCancelable() {
  const controller = new AbortController();
  onBeforeUnmount(() => controller.abort());
  return controller.signal;
}
```

## 重试（仅幂等接口）

只对 GET / 可幂等接口加重试，写入类操作不要自动重试：

```typescript
import axiosRetry from 'axios-retry';

axiosRetry(http, {
  retries: 2,
  retryDelay: axiosRetry.exponentialDelay,
  retryCondition: (e) =>
    axiosRetry.isNetworkOrIdempotentRequestError(e) ||
    (e.response?.status ?? 0) >= 500,
});
```

## 环境变量

```ini
# .env.development
VITE_API_BASE_URL=http://localhost:8080

# .env.production
VITE_API_BASE_URL=/api-proxy
```

## 最佳实践

1. **唯一 axios 实例**
2. **拦截器解包 `Result<T>`**，业务代码看到的是 `T`
3. **业务错误统一用 `BusinessError`**
4. **API 模块文件零副作用**（不弹 toast、不跳路由）
5. **组件通过 composable 调用 API**，不直接 import 模块

## 反模式

- 在多处 `import axios from 'axios'`
- 业务代码到处写 `if (res.success)` 判断
- API 文件里调用 `ElMessage`
- 用 `try/catch` 把所有错误吞掉
- 把 token 放在请求 body / URL 里
