# 前端提交前清单

## 类型与质量

- [ ] **无 `any`**：`grep -RE '\bany\b' src --include='*.ts' --include='*.vue'` 仅命中第三方声明
- [ ] **无 `@ts-ignore` / `@ts-expect-error`**
- [ ] **无非空断言 `!`**（除少数和第三方互操作的 ref `!.value`）
- [ ] **类型来自 `@/api/types` / `@/api/generated`**，未手抄后端 DTO
- [ ] `pnpm type-check` 通过（`vue-tsc --noEmit`）

## 构建

- [ ] `pnpm lint` 0 错误
- [ ] `pnpm build` 通过
- [ ] dist 体积无大幅膨胀（参考既有水位）

## API 与状态

- [ ] **新接口已通过 `pnpm gen:api` 同步类型**
- [ ] **无组件直接 import `axios`**：仅通过 `src/api/http` 与 `api/modules`
- [ ] **业务调用通过 composable**，组件不直接 import API 模块
- [ ] **错误抛 `BusinessError`**，调用方按 `code` 决定 UX
- [ ] **新增的 Pinia store** 仅放真正全局长期状态，未塞业务列表数据

## 路由

- [ ] **页面级组件懒加载**
- [ ] **新路由有 `meta.title`**
- [ ] **受保护路由 `requiresAuth: true`** 或 `public: true` 二选一
- [ ] **列表筛选写入 URL**，刷新可恢复

## 表单

- [ ] **每个 `el-form-item` 有 `prop`**
- [ ] **提交前调用 `formRef.validate()`**
- [ ] **submitting 在 finally 重置**
- [ ] **关闭弹窗时 `resetFields()`**

## 样式

- [ ] **组件 `<style scoped>`**，未污染全局
- [ ] **颜色/间距用 CSS 变量**，未硬编码
- [ ] **Element Plus 按需导入** 配置正常
- [ ] **无 `!important` 滥用**

## 可访问性 / 可用性

- [ ] **可点击元素是 `<button>` / `<a>` / `el-button`**，不是 `<div>`
- [ ] **图片有 `alt`**
- [ ] **空态有提示**（`el-empty`）
- [ ] **加载态有 skeleton 或 spinner**
- [ ] **错误态有重试入口**

## 安全

- [ ] **无 `v-html` 直接渲染用户/AI 输入**（必经 DOMPurify）
- [ ] **token 仅存 localStorage**，未出现在 URL 或 query
- [ ] **登出清理三件套**：token、stores、跳登录
- [ ] **生产环境 `__VUE_PROD_DEVTOOLS__: false`**

## AI 流式相关（如涉及）

- [ ] **支持 abort**
- [ ] **缓冲区拼接**到 `\n\n` 才解析
- [ ] **单帧错误隔离**，不中断整个流
- [ ] **Markdown 渲染走 DOMPurify**

## Git / 提交

- [ ] **生成的类型文件已提交**（`src/api/generated/schema.d.ts`）
- [ ] **package.json + lock 一起提交**
- [ ] **commit message 遵循团队约定**（如 conventional commits）

---

## 快速命令

```bash
# 类型检查
pnpm type-check

# Lint
pnpm lint

# 重新生成接口类型
pnpm gen:api

# 完整构建
pnpm build
```
