# 状态管理（Pinia）

Pinia store 的拆分原则、组合式 store、持久化与跨 store 协作。

## 状态分类与选型

| 类型             | 工具                  | 何时使用                                  |
| ---------------- | --------------------- | ----------------------------------------- |
| 服务端状态（缓存） | composable + 局部 ref | 列表/详情等业务数据，每次进入页面拉取    |
| 全局长生命周期   | Pinia store           | 当前用户、应用配置、菜单、主题            |
| URL 状态         | Vue Router query      | 列表筛选、分页、tab                       |
| 局部 UI 状态     | `ref` / `reactive`    | 弹窗开关、表单临时值                      |

> **重要**：**Pinia store 不是数据缓存层**。除"当前用户、菜单、主题、未读消息计数"等真正全局且长期持有的状态外，业务数据应通过 composable 在组件局部管理。

## 组合式 Store（推荐）

```typescript
// src/stores/user.ts
import { computed, ref } from 'vue';
import { defineStore } from 'pinia';
import { authApi, userApi } from '@/api/modules';
import type { UserInfo } from '@/api/generated/schema';

export const useUserStore = defineStore('user', () => {
  // state
  const token = ref<string | null>(localStorage.getItem('token'));
  const userInfo = ref<UserInfo | null>(null);

  // getters
  const isLoggedIn = computed(() => Boolean(token.value));
  const roles = computed(() => userInfo.value?.roles ?? []);
  const hasRole = (role: string) => roles.value.includes(role);

  // actions
  async function login(payload: { username: string; password: string }) {
    const res = await authApi.login(payload);
    setToken(res.token);
    await fetchCurrentUser();
  }

  async function fetchCurrentUser() {
    userInfo.value = await userApi.me();
  }

  function setToken(t: string) {
    token.value = t;
    localStorage.setItem('token', t);
  }

  function logout() {
    token.value = null;
    userInfo.value = null;
    localStorage.removeItem('token');
  }

  return {
    token,
    userInfo,
    isLoggedIn,
    roles,
    hasRole,
    login,
    fetchCurrentUser,
    setToken,
    logout,
  };
});
```

### 为何选组合式而非 Options

- 与 `<script setup>` 风格一致
- TypeScript 类型推断更友好
- 复用 composable 更容易（store 内可直接调用其它 composable）

## 在组件中使用

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia';
import { useUserStore } from '@/stores/user';

const userStore = useUserStore();

// 解构响应式 state/getter 必须用 storeToRefs
const { userInfo, isLoggedIn, roles } = storeToRefs(userStore);

// action 可以直接解构
const { logout } = userStore;
</script>
```

> **关键**：`storeToRefs` 仅对 state 和 getter 生效；action 直接解构即可。

## Store 拆分原则

按"主体"拆，不要按"功能动词"拆：

- `user` ← 当前用户、token、角色
- `app` ← 主题、语言、侧边栏折叠、全局 loading
- `menu` ← 菜单树（如果是后端动态下发）
- `notification` ← 全局未读消息计数

不应当出现的 store：

- `orderStore`：订单是业务数据，不是全局状态
- `userListStore`：列表数据应在组件内管理

## 跨 Store 协作

直接在 store 内 import 另一个 store：

```typescript
import { useAppStore } from './app';

export const useUserStore = defineStore('user', () => {
  function logout() {
    useAppStore().resetWorkspace();
    token.value = null;
    // ...
  }
});
```

> 注意：在另一个 store 的函数体外（顶层）调用 `useXxxStore()` 会因 Pinia 未初始化而报错，**必须在函数体内调用**。

## 持久化

仅持久化必要项（如 token、主题）。**不要**持久化业务数据。

```typescript
// src/stores/user.ts
const token = ref<string | null>(localStorage.getItem('token'));

watch(token, (val) => {
  if (val) localStorage.setItem('token', val);
  else localStorage.removeItem('token');
});
```

如使用 `pinia-plugin-persistedstate`：

```typescript
export const useAppStore = defineStore('app', () => {
  const theme = ref<'light' | 'dark'>('light');
  const sidebarCollapsed = ref(false);
  return { theme, sidebarCollapsed };
}, {
  persist: {
    pick: ['theme'], // 仅持久化 theme，不持久化 sidebarCollapsed
  },
});
```

## 异步与错误

action 内部不要 try/catch 后吞掉错误，让调用方决定如何反馈：

```typescript
// 不推荐：吞错
async function login(payload) {
  try {
    await authApi.login(payload);
  } catch (e) {
    ElMessage.error('登录失败');
  }
}

// 推荐：抛上去，组件统一在 onSubmit 处处理
async function login(payload) {
  const res = await authApi.login(payload);
  setToken(res.token);
}
```

```vue
<!-- LoginView.vue -->
<script setup lang="ts">
async function handleLogin() {
  try {
    await userStore.login(form.value);
    router.push('/dashboard');
  } catch (e) {
    ElMessage.error(e?.message ?? '登录失败');
  }
}
</script>
```

## 重置

在登出或租户切换时一键清空：

```typescript
import { getActivePinia } from 'pinia';

export function resetAllStores() {
  const pinia = getActivePinia();
  if (!pinia) return;
  // @ts-expect-error: pinia internal
  pinia._s.forEach((store) => store.$reset?.());
}
```

注意：组合式 store 默认没有 `$reset`，需要手动暴露：

```typescript
export const useUserStore = defineStore('user', () => {
  const token = ref<string | null>(null);
  const userInfo = ref<UserInfo | null>(null);

  function $reset() {
    token.value = null;
    userInfo.value = null;
  }

  return { token, userInfo, $reset };
});
```

## 调试

在开发环境启用 Vue Devtools 的 Pinia 面板，可看到所有 store 的状态与变更历史。生产环境关闭：

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
export default defineConfig({
  define: {
    __VUE_PROD_DEVTOOLS__: false,
  },
});
```

## 最佳实践

1. **组合式 store** 是首选写法
2. **Store 不存业务数据**，只放真正全局长期状态
3. **仅持久化必要字段**
4. **Action 不吞错**
5. **解构 state/getter 用 `storeToRefs`**

## 反模式

- 把 axios 响应直接塞 store 当缓存
- 在组件外（顶层模块作用域）调用 `useXxxStore()`
- store 之间通过 watch 互相同步（应当合并 store）
- store 直接 import 路由器并 push（动作交还给组件）
- 大量小 store（每个特性一个 store）
