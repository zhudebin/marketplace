# Element Plus 按需导入

| 项       | 值        |
| -------- | --------- |
| 分类     | 前端构建  |
| 严重等级 | Warning   |

## 问题描述

随便引入 Element Plus 后出现：

1. 打包体积比业务代码本身大几倍（dist > 3 MB）
2. 配置了 `unplugin-vue-components` 自动导入，但**样式丢失**：
   - `el-message` 弹出无样式
   - `el-message-box` 弹出无样式
   - `el-notification` / `el-loading` 同样无样式
3. 配置了 `themeChalk` 但本地能跑，生产构建出来按钮没颜色
4. 暗黑模式切换不生效
5. 用 `<el-icon>` 后图标不显示
6. SSR / 静态构建时报 `Cannot find module 'element-plus/es/...'`

## 根因

- **Element Plus 的样式不会随组件自动按需引入**：`unplugin-vue-components` 只解决 JS，CSS 需要单独配
- **以 JS API 方式调用的组件**（`ElMessage`、`ElMessageBox`、`ElNotification`、`ElLoading`）不会被模板解析器扫到，自动导入插件**抓不到**
- 主题变量需要全局引入 SCSS 入口
- Icons 是单独的包 `@element-plus/icons-vue`，需要额外注册

## 解决方案

### 1. 安装

```bash
pnpm add element-plus @element-plus/icons-vue
pnpm add -D unplugin-vue-components unplugin-auto-import
```

### 2. Vite 配置

```typescript
// vite.config.ts
import AutoImport from 'unplugin-auto-import/vite';
import Components from 'unplugin-vue-components/vite';
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers';

export default defineConfig({
  plugins: [
    vue(),
    AutoImport({
      resolvers: [
        ElementPlusResolver({ importStyle: 'sass' }),  // ← 关键
      ],
    }),
    Components({
      resolvers: [
        ElementPlusResolver({ importStyle: 'sass' }),
      ],
    }),
  ],
  css: {
    preprocessorOptions: {
      scss: {
        additionalData: `@use "@/styles/element/index.scss" as *;`,
      },
    },
  },
});
```

### 3. JS API 组件的样式手动引入

```typescript
// src/main.ts
import 'element-plus/theme-chalk/src/message.scss';
import 'element-plus/theme-chalk/src/message-box.scss';
import 'element-plus/theme-chalk/src/notification.scss';
import 'element-plus/theme-chalk/src/loading.scss';
```

> **这是最容易踩的坑**：自动导入只管模板里的标签，`ElMessage()` 这种**函数式调用没有标签**，插件无从识别。

### 4. 主题定制

```scss
/* src/styles/element/index.scss */
@forward 'element-plus/theme-chalk/src/common/var.scss' with (
  $colors: (
    'primary': (
      'base': #2A78F0,
    ),
  )
);
```

```scss
/* src/main.ts 或 App.vue */
import './styles/element/index.scss';
```

### 5. 暗黑模式

```typescript
// src/main.ts
import 'element-plus/theme-chalk/dark/css-vars.css';
```

切换：

```typescript
document.documentElement.classList.toggle('dark');
```

### 6. Icons 注册

```typescript
// src/main.ts
import * as ElIconModules from '@element-plus/icons-vue';

const app = createApp(App);
for (const [key, component] of Object.entries(ElIconModules)) {
  app.component(key, component);
}
```

> 也可以局部按需 import，避免一次性注册所有图标。

### 7. Tree-shaking 验证

构建后用 `rollup-plugin-visualizer` 检查：

```bash
pnpm add -D rollup-plugin-visualizer
```

```typescript
import { visualizer } from 'rollup-plugin-visualizer';

plugins: [
  visualizer({ open: true, gzipSize: true }),
]
```

正常情况下 `element-plus` 在 dist 里**只包含你用到的组件**，不应整包出现。

## 关键启示

- 自动按需导入**只对模板组件有效**，JS API 调用必须手动引样式
- 主题、暗黑、图标三块**单独配置**，缺一不可
- 上线前用 visualizer 看一眼包体，能省很多冤枉时间
- 任何"组件能用但样式不对"的问题，**先想想这是不是 JS API 组件**

## 自检清单

- [ ] vite.config 用了 ElementPlusResolver 且 `importStyle: 'sass'`？
- [ ] `main.ts` 手动 import 了 `message/message-box/notification/loading` 的样式？
- [ ] 主题变量入口已配？
- [ ] 暗黑模式 css-vars 已引入（如需要）？
- [ ] 图标已注册或显式 import？
- [ ] 构建产物里 element-plus 是按需，不是整包？
