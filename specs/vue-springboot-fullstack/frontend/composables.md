# Composables

Vue 3 自定义 composable 的编写规范、命名约定与常见模式。

## 什么时候写 composable

需要"在多个组件之间共享有响应性的逻辑"或"封装一段较重的副作用 + 状态"时。典型场景：

- 数据请求 + loading + error 的封装
- 表单提交流程
- 防抖/节流的搜索框
- 监听 resize/scroll 等浏览器事件
- 业务状态机（如订单的状态流转）

## 命名

- 文件名 `useXxx.ts`，函数名 `useXxx`
- 业务 composable 放 `modules/[feature]/composables/`
- 通用 composable 放 `src/composables/`

## 基础模板：数据请求

```typescript
// modules/order/composables/useOrderList.ts
import { ref, watch } from 'vue';
import type { Ref } from 'vue';
import { orderApi } from '@/api/modules';
import type { OrderQuery, OrderVO, PageResult } from '@/api/generated/schema';

export interface UseOrderListReturn {
  data: Ref<OrderVO[]>;
  total: Ref<number>;
  loading: Ref<boolean>;
  error: Ref<Error | null>;
  refresh: () => Promise<void>;
}

export function useOrderList(query: Ref<OrderQuery>): UseOrderListReturn {
  const data = ref<OrderVO[]>([]);
  const total = ref(0);
  const loading = ref(false);
  const error = ref<Error | null>(null);

  async function refresh() {
    loading.value = true;
    error.value = null;
    try {
      const res: PageResult<OrderVO> = await orderApi.list(query.value);
      data.value = res.records;
      total.value = res.total;
    } catch (e) {
      error.value = e as Error;
      data.value = [];
      total.value = 0;
    } finally {
      loading.value = false;
    }
  }

  watch(query, refresh, { immediate: true, deep: true });

  return { data, total, loading, error, refresh };
}
```

> **关键约定**：
>
> - 入参中**响应式数据**统一用 `Ref<T>`，避免组件传值后丢失响应性
> - 返回值中暴露的状态用 `Ref`，方法用普通函数
> - 副作用（watch、interval、event listener）必须在 composable 内自我清理

## 业务动作封装

```typescript
// modules/order/composables/useOrderActions.ts
import { ref } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import { orderApi } from '@/api/modules';
import type { OrderCreateDTO } from '@/api/generated/schema';

export function useOrderActions() {
  const submitting = ref(false);

  async function createOrder(payload: OrderCreateDTO) {
    submitting.value = true;
    try {
      const order = await orderApi.create(payload);
      ElMessage.success('订单创建成功');
      return order;
    } finally {
      submitting.value = false;
    }
  }

  async function cancelOrder(id: string) {
    await ElMessageBox.confirm('确认取消该订单？', '提示', { type: 'warning' });
    await orderApi.cancel(id);
    ElMessage.success('已取消');
  }

  return { submitting, createOrder, cancelOrder };
}
```

## 副作用必须清理

监听全局事件、定时器、ResizeObserver 等都必须在 `onBeforeUnmount` 清理：

```typescript
import { onBeforeUnmount, onMounted, ref } from 'vue';

export function useWindowSize() {
  const width = ref(window.innerWidth);
  const height = ref(window.innerHeight);

  function onResize() {
    width.value = window.innerWidth;
    height.value = window.innerHeight;
  }

  onMounted(() => window.addEventListener('resize', onResize));
  onBeforeUnmount(() => window.removeEventListener('resize', onResize));

  return { width, height };
}
```

## 接收响应式参数

支持调用方传 `Ref` 或普通值，统一用 `MaybeRefOrGetter` + `toValue`（Vue 3.3+）：

```typescript
import { toValue, watch } from 'vue';
import type { MaybeRefOrGetter } from 'vue';

export function useUserDetail(userId: MaybeRefOrGetter<string>) {
  const user = ref<UserVO | null>(null);

  watch(
    () => toValue(userId),
    async (id) => {
      if (!id) return;
      user.value = await userApi.detail(id);
    },
    { immediate: true },
  );

  return { user };
}
```

调用方：

```typescript
const route = useRoute();
const { user } = useUserDetail(() => route.params.id as string);
```

## 防抖搜索

```typescript
// composables/useDebouncedSearch.ts
import { ref, watch } from 'vue';
import type { Ref } from 'vue';

export function useDebouncedSearch<T>(
  keyword: Ref<string>,
  fetcher: (kw: string) => Promise<T[]>,
  delay = 300,
) {
  const results = ref<T[]>([]) as Ref<T[]>;
  const loading = ref(false);
  let timer: number | undefined;

  watch(keyword, (kw) => {
    clearTimeout(timer);
    if (!kw.trim()) {
      results.value = [];
      return;
    }
    timer = window.setTimeout(async () => {
      loading.value = true;
      try {
        results.value = await fetcher(kw);
      } finally {
        loading.value = false;
      }
    }, delay);
  });

  return { results, loading };
}
```

## 使用第三方 composable

强烈推荐 [VueUse](https://vueuse.org/) 处理常见场景（`useLocalStorage`、`useDebounceFn`、`useEventListener`、`useIntersectionObserver` 等），不要自己造轮子。

```typescript
import { useLocalStorage, useDebounceFn } from '@vueuse/core';

const draft = useLocalStorage('order-draft', { items: [] });
const debouncedSave = useDebounceFn(save, 500);
```

## 测试性

composable 应当尽量是"纯函数 + 注入依赖"，方便测试：

```typescript
// 推荐：依赖注入
export function useOrderList(query: Ref<OrderQuery>, api = orderApi) {
  // ...
}

// 测试时
const fakeApi = { list: vi.fn() };
const { data } = useOrderList(ref({ page: 1, pageSize: 10 }), fakeApi);
```

## 最佳实践

1. **命名 `useXxx`**，文件同名
2. **响应式入参用 `Ref` 或 `MaybeRefOrGetter`**
3. **副作用自我清理**
4. **业务请求统一封装**，组件不直接 import API
5. **优先用 VueUse**，不重复造轮子

## 反模式

- composable 里直接读 `route` 而不接收参数（耦合路由）
- 在 composable 顶层做副作用（必须在 setup 阶段执行的副作用要包在 `onMounted`）
- 把 watch 写得依赖边界不清，导致请求风暴
- composable 返回普通对象（应保持响应性）
- 在循环中调用 composable
