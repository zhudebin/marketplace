# 前端目录结构

定义前端工程的目录组织与文件命名约定。

## 整体结构

```
frontend/
├── public/                      # 静态资源（不参与构建处理）
├── src/
│   ├── api/                     # 全局 API 入口（按模块聚合）
│   │   ├── http.ts              # axios 实例与拦截器
│   │   ├── generated/           # OpenAPI 自动生成的类型
│   │   │   └── schema.d.ts
│   │   └── modules/             # 按业务领域拆分的 API 调用
│   │       ├── user.ts
│   │       └── order.ts
│   ├── assets/                  # 参与构建的静态资源（图片、字体）
│   ├── components/              # 全局通用组件
│   │   ├── AppHeader.vue
│   │   ├── AppSidebar.vue
│   │   └── PageContainer.vue
│   ├── composables/             # 全局 composable
│   │   ├── useUser.ts
│   │   └── usePermission.ts
│   ├── directives/              # 自定义指令
│   ├── layouts/                 # 页面布局组件
│   │   ├── DefaultLayout.vue
│   │   └── BlankLayout.vue
│   ├── modules/                 # 业务功能模块（按领域）
│   │   ├── user/
│   │   ├── order/
│   │   └── dashboard/
│   ├── plugins/                 # Vue 插件初始化
│   │   ├── element-plus.ts
│   │   └── pinia.ts
│   ├── router/
│   │   ├── index.ts             # Router 实例
│   │   ├── routes.ts            # 路由表
│   │   └── guards.ts            # 路由守卫
│   ├── stores/                  # 全局 Pinia store
│   │   ├── user.ts
│   │   └── app.ts
│   ├── styles/                  # 全局样式
│   │   ├── index.scss
│   │   ├── variables.scss
│   │   └── element-overrides.scss
│   ├── utils/                   # 通用工具函数（无业务）
│   ├── views/                   # 路由级页面（薄壳，调用 modules）
│   ├── App.vue
│   ├── main.ts
│   └── env.d.ts
├── index.html
├── vite.config.ts
├── tsconfig.json
├── package.json
└── .env.development             # 环境变量
```

## 业务模块结构

每个业务模块（`modules/[feature]/`）应当封闭、自洽：

```
modules/order/
├── components/                  # 模块内组件（不对外暴露）
│   ├── OrderList.vue
│   ├── OrderDetailDrawer.vue
│   └── OrderStatusTag.vue
├── composables/                 # 业务 composable
│   ├── useOrderList.ts          # 列表查询 + 筛选
│   └── useOrderActions.ts       # 创建 / 更新 / 取消
├── api/                         # 模块 API（薄封装，调用 src/api/http）
│   └── index.ts
├── types/                       # 视图模型类型（区别于后端 DTO）
│   └── index.ts
├── constants.ts                 # 模块内常量（状态枚举映射等）
└── index.ts                     # 对外暴露的入口（barrel）
```

> **原则**：路由对应的 `views/OrderListView.vue` 应当只是薄壳，业务逻辑全部从 `modules/order` 引入。

## 路由 → 模块映射

```
src/views/
├── DashboardView.vue       -> modules/dashboard/
├── OrderListView.vue       -> modules/order/
├── OrderDetailView.vue     -> modules/order/
└── UserSettingsView.vue    -> modules/user/
```

## 命名约定

### 文件命名

| 类型             | 约定                          | 示例                          |
| ---------------- | ----------------------------- | ----------------------------- |
| Vue 组件         | PascalCase                    | `OrderList.vue`               |
| Vue 页面（views） | `{Name}View.vue`              | `OrderListView.vue`           |
| Composable       | `useXxx.ts`                   | `useOrderList.ts`             |
| Pinia Store      | camelCase                     | `user.ts`、`order.ts`         |
| 工具函数         | camelCase                     | `formatDate.ts`               |
| 类型文件         | `types.ts` 或 `index.ts`      | `types/index.ts`              |
| 常量             | camelCase 文件、常量 SCREAMING_SNAKE_CASE | `constants.ts`     |
| 样式             | kebab-case                    | `element-overrides.scss`      |

### 导出规范

业务模块通过 `index.ts` 桶式导出，禁止外部直接 import 模块内部文件：

```typescript
// modules/order/index.ts
export { default as OrderList } from './components/OrderList.vue';
export { default as OrderDetailDrawer } from './components/OrderDetailDrawer.vue';
export { useOrderList } from './composables/useOrderList';
export { useOrderActions } from './composables/useOrderActions';
export type { OrderViewModel, OrderFilter } from './types';
```

```typescript
// 使用方
import { OrderList, useOrderList } from '@/modules/order';
```

## 路径别名

`tsconfig.json` 与 `vite.config.ts` 同步配置：

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@/api/*": ["./src/api/*"],
      "@/components/*": ["./src/components/*"],
      "@/composables/*": ["./src/composables/*"],
      "@/modules/*": ["./src/modules/*"],
      "@/stores/*": ["./src/stores/*"],
      "@/utils/*": ["./src/utils/*"]
    }
  }
}
```

## 最佳实践

1. **就近放置**：模块内组件、composable、API 三件套放一起
2. **单一职责**：每个模块解决一个业务领域
3. **桶式导出**：模块只暴露 `index.ts` 中的内容
4. **私有优先**：默认不导出，需要时再放进 `index.ts`
5. **禁止环依赖**：模块之间不允许相互 import；公共逻辑下沉到 `src/composables` 或 `src/utils`

## 反模式

- 把所有 API 调用堆在 `src/api/`，业务模块直接依赖 axios 实例
- `views/` 中写大量业务逻辑（应当薄壳化）
- 跨模块 import 内部文件（绕过 `index.ts`）
- `utils/` 沦为各种业务杂物的垃圾桶
- 嵌套层级超过 4 层
