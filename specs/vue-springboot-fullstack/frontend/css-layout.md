# 样式与布局

主题、CSS 变量、响应式布局、Element Plus 主题定制与按需导入。

## CSS 方案

- **作用域**：组件内样式一律 `scoped`
- **预处理器**：SCSS（Element Plus 主题定制需要）
- **全局样式**：集中在 `src/styles/`
- **CSS 变量**：用于主题色、间距等运行时可切换的值
- **不引入 Tailwind**（Element Plus 体系下不必要，避免风格混用）

## 全局样式入口

```scss
// src/styles/index.scss
@use './variables.scss';
@use './reset.scss';
@use './element-overrides.scss';

html, body {
  margin: 0;
  padding: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', 'Microsoft YaHei', sans-serif;
  font-size: 14px;
  color: var(--app-text-primary);
  background-color: var(--app-bg);
}

#app {
  min-height: 100vh;
}
```

```typescript
// src/main.ts
import './styles/index.scss';
```

## CSS 变量与主题

```scss
// src/styles/variables.scss
:root {
  // 间距
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;

  // 文字
  --app-text-primary: #1f2329;
  --app-text-secondary: #4e5969;
  --app-text-disabled: #c9cdd4;

  // 背景
  --app-bg: #f7f8fa;
  --app-bg-card: #ffffff;
  --app-border: #e5e6eb;

  // 圆角
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
}

[data-theme='dark'] {
  --app-text-primary: #e5e6eb;
  --app-text-secondary: #a9aeb8;
  --app-bg: #17171a;
  --app-bg-card: #232324;
  --app-border: #2e2e30;
}
```

切换：

```typescript
document.documentElement.setAttribute('data-theme', 'dark');
```

## Element Plus 按需导入

使用 `unplugin-vue-components` + `unplugin-auto-import`，避免手动 `import { ElButton }`：

```bash
pnpm add -D unplugin-vue-components unplugin-auto-import
```

```typescript
// vite.config.ts
import AutoImport from 'unplugin-auto-import/vite';
import Components from 'unplugin-vue-components/vite';
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers';

export default defineConfig({
  plugins: [
    AutoImport({
      resolvers: [ElementPlusResolver({ importStyle: 'sass' })],
    }),
    Components({
      resolvers: [ElementPlusResolver({ importStyle: 'sass' })],
    }),
  ],
});
```

> **注意**：`importStyle: 'sass'` 才能让主题定制 `@use` 起作用。详细问题见 [big-question/element-plus-on-demand.md](../big-question/element-plus-on-demand.md)。

## Element Plus 主题定制

```scss
// src/styles/element-overrides.scss
@forward 'element-plus/theme-chalk/src/common/var.scss' with (
  $colors: (
    'primary': (
      'base': #4080ff,
    ),
    'success': (
      'base': #00b42a,
    ),
  ),
  $border-radius: (
    'base': 6px,
  )
);

@use 'element-plus/theme-chalk/src/index.scss' as *;
```

```typescript
// vite.config.ts
export default defineConfig({
  css: {
    preprocessorOptions: {
      scss: {
        additionalData: `@use "@/styles/element-overrides.scss" as *;`,
      },
    },
  },
});
```

## 图标

使用 `@element-plus/icons-vue`：

```bash
pnpm add @element-plus/icons-vue
```

按需引用：

```vue
<script setup lang="ts">
import { Edit, Delete } from '@element-plus/icons-vue';
</script>

<template>
  <el-button :icon="Edit">编辑</el-button>
  <el-icon><Delete /></el-icon>
</template>
```

## 响应式布局

### 断点约定（与 Element Plus 一致）

```scss
// src/styles/breakpoints.scss
$breakpoints: (
  'xs': 0,
  'sm': 768px,
  'md': 992px,
  'lg': 1200px,
  'xl': 1920px,
);

@mixin respond-to($name) {
  $w: map-get($breakpoints, $name);
  @media (min-width: $w) { @content; }
}
```

```scss
.card {
  padding: var(--space-3);
  @include respond-to('md') {
    padding: var(--space-6);
  }
}
```

### Element Plus Row/Col

```vue
<el-row :gutter="16">
  <el-col :xs="24" :sm="12" :md="8" :lg="6">
    <DataCard />
  </el-col>
</el-row>
```

## 布局组件骨架

```vue
<!-- layouts/DefaultLayout.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia';
import { useAppStore } from '@/stores/app';
import AppSidebar from '@/components/AppSidebar.vue';
import AppHeader from '@/components/AppHeader.vue';

const { sidebarCollapsed } = storeToRefs(useAppStore());
</script>

<template>
  <el-container class="layout">
    <el-aside :width="sidebarCollapsed ? '64px' : '220px'">
      <AppSidebar />
    </el-aside>
    <el-container>
      <el-header>
        <AppHeader />
      </el-header>
      <el-main>
        <router-view v-slot="{ Component }">
          <transition name="fade" mode="out-in">
            <component :is="Component" />
          </transition>
        </router-view>
      </el-main>
    </el-container>
  </el-container>
</template>

<style scoped lang="scss">
.layout {
  height: 100vh;
}
.el-aside {
  transition: width 0.2s;
  background: var(--app-bg-card);
  border-right: 1px solid var(--app-border);
}
.el-header {
  background: var(--app-bg-card);
  border-bottom: 1px solid var(--app-border);
}
.fade-enter-active, .fade-leave-active { transition: opacity 0.15s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
```

## 滚动与高度

`el-main` 默认有 `overflow: auto`，子内容若需要 100% 高度，请用 flex：

```scss
.page {
  display: flex;
  flex-direction: column;
  height: 100%;

  .toolbar { flex: none; }
  .content { flex: 1; overflow: auto; }
}
```

## 表格行高与拥挤

不要在 `el-table` 上使用 `size="large"` 默认值。中后台高密度表格使用 `size="small"`。

## 移动端

如需支持移动端，在 `index.html` 加 viewport：

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
```

并使用 `@media (max-width: 768px)` 写专门样式或切换到 Vant / Mobile UI。

## 最佳实践

1. **scoped 默认**，全局样式集中在 `src/styles`
2. **CSS 变量管理主题**
3. **Element Plus 按需导入** + `importStyle: 'sass'`
4. **图标按需 import**
5. **统一断点**

## 反模式

- 全量引入 Element Plus 样式
- 在每个组件 `<style>` 写 `:root { ... }`
- `!important` 满天飞（应通过 SCSS 主题变量覆盖）
- 用 px 写颜色与间距（应用 CSS 变量）
- 多套 UI 库混用（如同时引 Element Plus + Ant Design Vue）
