# 组件规范

涵盖 Vue 3 组件的写法、Props/Emits 定义、组合式 API 模式与语义化 HTML。

## SFC 基本结构

**强制**：统一使用 `<script setup lang="ts">`。

```vue
<script setup lang="ts">
import { computed, ref } from 'vue';
import type { OrderViewModel } from '@/modules/order';

interface Props {
  order: OrderViewModel;
  editable?: boolean;
}

interface Emits {
  (e: 'update', order: OrderViewModel): void;
  (e: 'cancel'): void;
}

const props = withDefaults(defineProps<Props>(), {
  editable: false,
});

const emit = defineEmits<Emits>();

const total = computed(() =>
  props.order.items.reduce((s, i) => s + i.price * i.quantity, 0),
);

const submitting = ref(false);

async function handleSave() {
  submitting.value = true;
  try {
    emit('update', props.order);
  } finally {
    submitting.value = false;
  }
}
</script>

<template>
  <el-card class="order-card">
    <h3>{{ order.reference }}</h3>
    <p>合计：{{ total }}</p>
    <el-button
      v-if="editable"
      type="primary"
      :loading="submitting"
      @click="handleSave"
    >
      保存
    </el-button>
  </el-card>
</template>

<style scoped lang="scss">
.order-card {
  margin-bottom: 16px;
}
</style>
```

### 三段顺序

1. `<script setup lang="ts">`（在最上）
2. `<template>`
3. `<style scoped>`

把 `script` 放最上是为了便于阅读：理解组件意图前先看到接口与状态。

## Props 与 Emits

### 必须用类型定义，禁止 Options 写法

```typescript
// 推荐
interface Props {
  userId: string;
  showAvatar?: boolean;
}
const props = withDefaults(defineProps<Props>(), {
  showAvatar: true,
});

// 不推荐
const props = defineProps({
  userId: { type: String, required: true },
  showAvatar: { type: Boolean, default: true },
});
```

### Emits 事件命名

- 使用 kebab-case：`update:model-value`、`row-click`
- 与原生事件区分：业务事件用动词，如 `submit`、`cancel`、`select`
- 双向绑定遵循 `update:propName` 约定

```typescript
interface Emits {
  (e: 'update:modelValue', value: string): void;
  (e: 'select', id: string): void;
  (e: 'cancel'): void;
}
```

### 不要用 Props 暗中传递业务依赖

```typescript
// 反模式：把整个 store / API 客户端当 prop 传
defineProps<{ orderApi: OrderApi }>();

// 正确：组件内部通过 composable 拿
const { fetchOrder } = useOrderActions();
```

## 组件分层

| 层级       | 位置                            | 特征                                    |
| ---------- | ------------------------------- | --------------------------------------- |
| 页面级     | `src/views/XxxView.vue`         | 仅做布局拼装与 composable 调用，禁止业务 |
| 业务组件   | `modules/[feature]/components/` | 包含业务语义（如 `OrderStatusTag`）     |
| 通用组件   | `src/components/`               | 与业务无关、跨模块复用                  |
| UI 基础    | Element Plus / 二次封装         | 表单、表格、按钮等                      |

### 页面级组件示例（薄壳）

```vue
<script setup lang="ts">
import { OrderList, useOrderList } from '@/modules/order';
import PageContainer from '@/components/PageContainer.vue';

const { orders, loading, filter, refresh } = useOrderList();
</script>

<template>
  <PageContainer title="订单管理">
    <OrderList
      :orders="orders"
      :loading="loading"
      :filter="filter"
      @refresh="refresh"
    />
  </PageContainer>
</template>
```

## 语义化与可访问性

### 优先使用语义化标签

```vue
<!-- 不推荐 -->
<div @click="goDetail">详情</div>

<!-- 推荐：可点击就用 button / a -->
<el-button text @click="goDetail">详情</el-button>
<router-link :to="`/orders/${id}`">详情</router-link>
```

### 表单必有 label

```vue
<el-form-item label="邮箱" prop="email">
  <el-input v-model="form.email" placeholder="请输入邮箱" />
</el-form-item>
```

### 图片必有 alt

```vue
<el-image :src="user.avatar" :alt="`${user.name} 的头像`" fit="cover" />
```

## v-for 必带 key

key 必须是稳定唯一标识（业务 id），不要用 index：

```vue
<!-- 反模式 -->
<el-card v-for="(order, i) in orders" :key="i" />

<!-- 正确 -->
<el-card v-for="order in orders" :key="order.id" />
```

## 条件渲染：v-if vs v-show

| 场景                       | 选择     |
| -------------------------- | -------- |
| 切换频繁（tab、折叠）      | `v-show` |
| 一次性判定（权限、空态）   | `v-if`   |
| 未登录时不渲染受保护内容   | `v-if`   |

## ref 与 reactive

- 默认使用 `ref`（基本类型、对象都可以）
- 仅当对象需要"深度可观察且不解包"时才用 `reactive`
- 模板里直接用变量名，不写 `.value`

```typescript
// 推荐
const user = ref<User | null>(null);
const orders = ref<Order[]>([]);

// 仅当需要返回对象给外部使用且不希望解包时
const state = reactive({ count: 0, list: [] as Order[] });
```

## 异步组件与 Suspense

只在真正需要按需加载的页面级组件使用：

```typescript
// router/routes.ts
{
  path: '/orders',
  component: () => import('@/views/OrderListView.vue'),
}
```

## defineExpose

默认子组件不对外暴露任何东西。仅在父组件确实需要调子组件方法时使用：

```typescript
const formRef = ref<FormInstance>();

function reset() {
  formRef.value?.resetFields();
}

defineExpose({ reset });
```

## 最佳实践

1. **`<script setup lang="ts">` 一统天下**
2. **Props/Emits 用类型定义**
3. **页面级组件薄壳化**
4. **业务逻辑放进 composable**，组件只负责渲染与事件转发
5. **样式统一 `scoped`**，全局样式集中在 `src/styles/`

## 反模式

- 用 `<div @click>` 替代 `<button>`
- 在组件 `setup` 里直接 import API 调用（应通过 composable）
- v-for 的 key 用 index
- 给组件传一个 store 实例做 prop
- mixin 复用逻辑（应该用 composable）
- 在 `<template>` 中写复杂表达式（提取为 `computed`）
