# 鉴权（前端）

Token 存储、登录态恢复、路由级与接口级鉴权。

## 鉴权方案概要

- 后端使用 **Spring Security + JWT**（详见 [backend/authentication.md](../backend/authentication.md)）
- 前端只持有 JWT，不接触明文密码（除登录瞬间）
- 每个受保护请求在 `Authorization: Bearer <token>` 头里携带 JWT

## Token 存储

### 推荐：localStorage（适用绝大多数 To-B / 内部系统）

```typescript
const TOKEN_KEY = 'token';

export const tokenStorage = {
  get: () => localStorage.getItem(TOKEN_KEY),
  set: (t: string) => localStorage.setItem(TOKEN_KEY, t),
  clear: () => localStorage.removeItem(TOKEN_KEY),
};
```

**优点**：实现简单、跨标签页可读、刷新不丢

**缺点**：理论上 XSS 可读 → 必须配合 CSP、严格转义、内容安全审计

### 不推荐：sessionStorage

刷新还在但新标签页就丢，体验差。

### 更安全：HttpOnly Cookie

如果业务对安全要求极高，可让后端把 token 放 `HttpOnly`、`Secure`、`SameSite=Strict` 的 cookie。前端无需手动管理，但需要后端开 CORS 凭证 + 加 CSRF 防护。详见 [big-question/cors-samesite.md](../big-question/cors-samesite.md)。

## 用户 Store

参见 [state-management.md](./state-management.md) 的完整 `useUserStore`。关键责任：

- 持有 `token` / `userInfo` / `roles`
- 提供 `login` / `fetchCurrentUser` / `logout`
- 刷新页面后从 localStorage 恢复 token，并主动拉一次 `/me`

## 登录流程

```vue
<!-- views/LoginView.vue -->
<script setup lang="ts">
import { ref } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { ElMessage } from 'element-plus';
import { useUserStore } from '@/stores/user';

const router = useRouter();
const route = useRoute();
const userStore = useUserStore();

const form = ref({ username: '', password: '' });
const loading = ref(false);

async function onSubmit() {
  loading.value = true;
  try {
    await userStore.login(form.value);
    const redirect = (route.query.redirect as string) || '/dashboard';
    router.replace(redirect);
  } catch (e) {
    ElMessage.error(e instanceof Error ? e.message : '登录失败');
  } finally {
    loading.value = false;
  }
}
</script>
```

## 登录态恢复（刷新页面）

刷新后 Pinia 状态丢失，但 token 还在 localStorage。在路由守卫里恢复：

```typescript
// router/guards.ts （摘自 routing.md）
if (!userStore.token) {
  return { name: 'Login', query: { redirect: to.fullPath } };
}

if (!userStore.userInfo) {
  try {
    await userStore.fetchCurrentUser();
  } catch {
    userStore.logout();
    return { name: 'Login', query: { redirect: to.fullPath } };
  }
}
```

## 接口级鉴权

由 axios 请求拦截器统一注入 `Authorization` 头（见 [api-integration.md](./api-integration.md)）。401 由响应拦截器统一处理：

- 清空 token
- 跳 `/login` 并带 `redirect`
- 提示"登录已过期"

## 路由级鉴权

```typescript
{
  path: '/admin',
  meta: { requiresAuth: true, roles: ['ADMIN'] },
  // ...
}
```

> **重申**：前端的 `roles` 检查只是 UX 兜底（不显示菜单/按钮），**真正的安全边界在后端**。任何受保护的资源 / 操作必须由后端 `@PreAuthorize` 强制校验。

## 按钮级鉴权

提供 composable 与指令两种用法：

```typescript
// composables/usePermission.ts
import { computed } from 'vue';
import { storeToRefs } from 'pinia';
import { useUserStore } from '@/stores/user';

export function usePermission() {
  const { roles } = storeToRefs(useUserStore());

  const hasRole = (role: string | string[]) => {
    const required = Array.isArray(role) ? role : [role];
    return required.some((r) => roles.value.includes(r));
  };

  return { hasRole };
}
```

```vue
<script setup lang="ts">
import { usePermission } from '@/composables/usePermission';
const { hasRole } = usePermission();
</script>

<template>
  <el-button v-if="hasRole('ORDER_DELETE')" type="danger">删除订单</el-button>
</template>
```

## 登出

```typescript
function handleLogout() {
  // 1. 可选：先调后端注销接口（让 Redis 黑名单 token，详见后端文档）
  authApi.logout().catch(() => undefined);
  // 2. 清本地状态
  userStore.logout();
  resetAllStores();
  // 3. 跳登录
  router.replace({ name: 'Login' });
}
```

## Token 过期与刷新

详细策略见 [big-question/jwt-refresh-strategy.md](../big-question/jwt-refresh-strategy.md)。摘要：

- 短期 access token（15-30min）+ 长期 refresh token（7-30 天）
- 401 触发**单飞 + 静默刷新**：拦截器并发请求都等同一个 refresh promise
- refresh 失败再跳登录

简化版（仅 access token，无 refresh）：

- 后端 token 1-3 天有效期
- 401 直接跳登录
- 适合后台管理类系统

## 多标签页同步

监听 `storage` 事件，A 标签页登出时 B 标签页同步：

```typescript
window.addEventListener('storage', (e) => {
  if (e.key === 'token' && e.newValue === null) {
    // 别的标签页登出了
    location.reload();
  }
});
```

## 最佳实践

1. **Token 集中管理**（仅 `useUserStore` + axios 拦截器读写）
2. **路由守卫只判登录**，业务权限交后端
3. **401 全局处理**，业务代码不感知
4. **登出清理三件套**：token、stores、跳登录
5. **多标签页同步登出**

## 反模式

- 把 token 放 `Vuex/Pinia` 又放 `localStorage` 又放 `cookie`，三处不一致
- 在每个 API 调用前手动注入 `Authorization`
- 前端用 `roles` 做唯一权限校验
- 401 时只在某个组件处理，其他组件还能继续报错
- 把 JWT 解码后的 payload 当用户信息（应当从 `/me` 接口拉取）
