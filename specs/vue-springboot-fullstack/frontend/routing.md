# 路由与守卫

Vue Router 4 的配置规范、嵌套路由、路由元信息与守卫。

## Router 实例

```typescript
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router';
import { routes } from './routes';
import { setupGuards } from './guards';

export const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(_to, _from, savedPosition) {
    return savedPosition ?? { top: 0 };
  },
});

setupGuards(router);

export default router;
```

## 路由表

按"布局 + 业务"组织：

```typescript
// src/router/routes.ts
import type { RouteRecordRaw } from 'vue-router';
import DefaultLayout from '@/layouts/DefaultLayout.vue';
import BlankLayout from '@/layouts/BlankLayout.vue';

export const routes: RouteRecordRaw[] = [
  {
    path: '/login',
    component: BlankLayout,
    meta: { public: true },
    children: [
      {
        path: '',
        name: 'Login',
        component: () => import('@/views/LoginView.vue'),
      },
    ],
  },
  {
    path: '/',
    component: DefaultLayout,
    redirect: '/dashboard',
    meta: { requiresAuth: true },
    children: [
      {
        path: 'dashboard',
        name: 'Dashboard',
        component: () => import('@/views/DashboardView.vue'),
        meta: { title: '仪表盘', icon: 'DataLine' },
      },
      {
        path: 'orders',
        name: 'OrderList',
        component: () => import('@/views/OrderListView.vue'),
        meta: { title: '订单列表' },
      },
      {
        path: 'orders/:id',
        name: 'OrderDetail',
        component: () => import('@/views/OrderDetailView.vue'),
        meta: { title: '订单详情', hideInMenu: true },
        props: true,
      },
    ],
  },
  {
    path: '/:pathMatch(.*)*',
    name: 'NotFound',
    component: () => import('@/views/NotFoundView.vue'),
    meta: { public: true },
  },
];
```

### 路由元信息（meta）约定

| 字段           | 类型      | 含义                                        |
| -------------- | --------- | ------------------------------------------- |
| `requiresAuth` | boolean   | 需要登录才能访问                            |
| `public`       | boolean   | 显式公开路由（白名单），免登录              |
| `title`        | string    | 页面标题（用于 document.title 和面包屑）    |
| `icon`         | string    | 菜单图标（Element Plus icon 名）            |
| `hideInMenu`   | boolean   | 不在侧边栏显示                              |
| `roles`        | string[]  | 可选：要求的角色列表（仅前端兜底，权威在后端）|

> **原则**：`requiresAuth` 用作通用拦截；细粒度权限以 `roles` 兜底，但**不是安全边界**——真正的权限校验必须在后端。

## 路由守卫

```typescript
// src/router/guards.ts
import type { Router } from 'vue-router';
import { useUserStore } from '@/stores/user';
import { ElMessage } from 'element-plus';

const WHITE_LIST = new Set(['/login', '/404']);

export function setupGuards(router: Router) {
  router.beforeEach(async (to) => {
    const userStore = useUserStore();

    // 1. 标题
    if (to.meta.title) {
      document.title = `${to.meta.title} - MyApp`;
    }

    // 2. 白名单 / 公开页
    if (to.meta.public || WHITE_LIST.has(to.path)) {
      return true;
    }

    // 3. 未登录
    if (!userStore.token) {
      return { name: 'Login', query: { redirect: to.fullPath } };
    }

    // 4. 已有 token 但用户信息未恢复（刷新场景）
    if (!userStore.userInfo) {
      try {
        await userStore.fetchCurrentUser();
      } catch {
        userStore.logout();
        return { name: 'Login', query: { redirect: to.fullPath } };
      }
    }

    // 5. 角色校验（前端兜底）
    if (to.meta.roles && to.meta.roles.length > 0) {
      const hasRole = to.meta.roles.some((r) => userStore.roles.includes(r));
      if (!hasRole) {
        ElMessage.warning('无权限访问');
        return { name: 'Dashboard' };
      }
    }

    return true;
  });
}
```

## 编程式导航

通过 `useRouter` / `useRoute`，**不要**手动操作 `window.location`：

```typescript
import { useRouter, useRoute } from 'vue-router';

const router = useRouter();
const route = useRoute();

function goDetail(id: string) {
  router.push({ name: 'OrderDetail', params: { id } });
}

function goBack() {
  router.back();
}

const orderId = computed(() => route.params.id as string);
```

## 路由参数与查询

### params vs query

- `params`：资源标识（必填，进入路由的语义一部分）`/orders/:id`
- `query`：可选筛选/分页/排序 `/orders?status=PAID&page=2`

### 列表页查询参数与状态同步

URL query 应当作为列表页筛选/分页的**真理来源**，便于刷新与分享：

```typescript
// composables/useOrderListFilter.ts
import { computed } from 'vue';
import { useRoute, useRouter } from 'vue-router';

export function useOrderListFilter() {
  const route = useRoute();
  const router = useRouter();

  const filter = computed(() => ({
    status: (route.query.status as string) ?? '',
    keyword: (route.query.keyword as string) ?? '',
    page: Number(route.query.page ?? 1),
    pageSize: Number(route.query.pageSize ?? 20),
  }));

  function setFilter(patch: Partial<typeof filter.value>) {
    router.replace({
      query: { ...route.query, ...patch, page: patch.page ?? 1 },
    });
  }

  return { filter, setFilter };
}
```

## 嵌套路由

```typescript
{
  path: 'settings',
  component: () => import('@/views/SettingsLayout.vue'),
  redirect: '/settings/profile',
  children: [
    { path: 'profile', component: () => import('@/views/SettingsProfile.vue') },
    { path: 'security', component: () => import('@/views/SettingsSecurity.vue') },
  ],
}
```

`SettingsLayout.vue` 中放置 `<router-view />` 与公共侧边栏。

## 路由懒加载

**所有页面级组件必须懒加载**，控制首屏体积：

```typescript
component: () => import('@/views/OrderListView.vue');
```

可使用 Vite 的魔法注释做分组：

```typescript
component: () => import(/* webpackChunkName: "order" */ '@/views/OrderListView.vue');
```

## 路由级数据获取

**不要**在 `beforeEach` 里拉业务数据。业务数据应该在组件 `setup` 中通过 composable 拉取，配合 `<Suspense>` 或 loading 状态。

```vue
<!-- OrderDetailView.vue -->
<script setup lang="ts">
import { useRoute } from 'vue-router';
import { useOrderDetail } from '@/modules/order';

const route = useRoute();
const { order, loading, error } = useOrderDetail(route.params.id as string);
</script>
```

## 最佳实践

1. **路由懒加载** 是默认行为
2. **守卫只做登录态判定**，业务权限交后端
3. **URL 是列表页状态的真理来源**
4. **页面级组件命名 `XxxView`**，与业务组件区分
5. **404 用 catch-all** `/:pathMatch(.*)*`

## 反模式

- 在守卫里调业务接口（拖慢路由切换）
- 把业务参数放 params 而非 query（如分页、筛选）
- 路由表与菜单两份维护（应当统一从 routes 派生菜单）
- 用 `window.location.href` 跳转（除非外站）
- 不写懒加载，导致首屏 chunk 过大
