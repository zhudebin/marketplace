# Vue 前端开发规范

> Vue 3 + Vite + TypeScript + Element Plus 全栈应用通用前端开发规范。

## 技术栈

- **框架**：Vue 3（`<script setup>` + Composition API）
- **构建**：Vite 5+
- **语言**：TypeScript（严格模式）
- **UI 库**：Element Plus（含其内置表单校验）
- **状态管理**：Pinia
- **路由**：Vue Router 4
- **HTTP**：axios
- **AI**：原生 `EventSource` / `fetch + ReadableStream` 消费 SSE

---

## 关联规范

| 规范           | 位置         | 何时阅读             |
| -------------- | ------------ | -------------------- |
| **共享代码标准** | `../shared/` | 始终适用             |
| **思维指南**     | `../guides/` | 实现新特性前         |

---

## 文档列表

| 文件                                                 | 描述                                              | 优先级    |
| ---------------------------------------------------- | ------------------------------------------------- | --------- |
| [directory-structure.md](./directory-structure.md) | 项目结构、模块约定                                | **必读**  |
| [components.md](./components.md)                   | SFC 写法、Props/Emits、组合式 API、语义化 HTML    | **必读**  |
| [routing.md](./routing.md)                         | Vue Router 4 配置、嵌套路由、路由守卫             | **必读**  |
| [authentication.md](./authentication.md)           | Token 存储、登录态恢复、路由级与接口级鉴权        | **必读**  |
| [api-integration.md](./api-integration.md)         | axios 实例、拦截器、统一响应解包、错误处理        | **必读**  |
| [state-management.md](./state-management.md)       | Pinia store 拆分、组合式 store、持久化            | 参考      |
| [composables.md](./composables.md)                 | 自定义 composable 编写规范                        | 参考      |
| [form-validation.md](./form-validation.md)         | Element Plus el-form 校验规则、异步校验、submit   | 参考      |
| [ai-sdk-integration.md](./ai-sdk-integration.md)   | SSE 流式消费、消息累积、工具调用渲染              | 参考      |
| [css-layout.md](./css-layout.md)                   | 主题、CSS 变量、响应式布局、Element Plus 主题定制 | 参考      |
| [type-safety.md](./type-safety.md)                 | OpenAPI 生成类型、严格模式约束                    | 参考      |
| [quality.md](./quality.md)                         | 提交前清单                                        | 参考      |

---

## 按任务快速导航

### 开始开发前

| 任务                       | 文档                                                |
| -------------------------- | --------------------------------------------------- |
| 理解项目结构               | [directory-structure.md](./directory-structure.md) |
| 学习 SFC 与组合式 API      | [components.md](./components.md)                   |
| 配置路由与鉴权             | [routing.md](./routing.md) / [authentication.md](./authentication.md) |

### 开发中

| 任务                       | 文档                                              |
| -------------------------- | ------------------------------------------------- |
| 调用后端接口               | [api-integration.md](./api-integration.md)        |
| 抽取可复用逻辑             | [composables.md](./composables.md)                |
| 管理跨页面状态             | [state-management.md](./state-management.md)      |
| 编写表单与校验             | [form-validation.md](./form-validation.md)        |
| 集成 AI 流式接口           | [ai-sdk-integration.md](./ai-sdk-integration.md)  |
| 处理样式与响应式           | [css-layout.md](./css-layout.md)                  |
| 同步后端类型               | [type-safety.md](./type-safety.md)                |

### 提交前

| 任务                  | 文档                            |
| --------------------- | ------------------------------- |
| 跑质量清单            | [quality.md](./quality.md)      |
| 类型检查              | [type-safety.md](./type-safety.md) |

---

## 核心规则汇总

| 规则                                                          | 参考                                                |
| ------------------------------------------------------------- | --------------------------------------------------- |
| **统一使用 `<script setup lang="ts">`**                       | [components.md](./components.md)                   |
| **不直接调 axios，全部走封装好的 request 实例**               | [api-integration.md](./api-integration.md)         |
| **类型从 OpenAPI 自动生成，禁止手抄后端 DTO**                 | [type-safety.md](./type-safety.md)                 |
| **Pinia store 不存服务端原始数据**（短缓存除外）              | [state-management.md](./state-management.md)       |
| **新代码禁止 `any`、`@ts-ignore`、`!` 非空断言**              | [type-safety.md](./type-safety.md)                 |
| **Element Plus 按需导入** 配合 `unplugin-vue-components`      | [css-layout.md](./css-layout.md)                   |
| **路由守卫只判断登录态，权限粒度交给后端**                    | [authentication.md](./authentication.md)           |
| **业务请求一律通过 composable 暴露**，组件不直接 import API   | [composables.md](./composables.md)                 |
| **表单提交前必须 `formRef.validate()`**                       | [form-validation.md](./form-validation.md)         |

---

## 架构概览

```
+--------------------------------------------------------------+
|                       Vue 3 应用                              |
|                                                               |
|  src/                                                         |
|  ├── views/             ├── modules/                          |
|  │   └── [route].vue    │   └── [feature]/                    |
|  ├── router/            │       ├── components/               |
|  ├── stores/            │       ├── composables/              |
|  └── api/               │       ├── api/                      |
|                         │       └── types/                    |
+-------------------------+------------------------------------+
                          |
              axios + 拦截器 (统一响应解包)
                          |
+-------------------------+------------------------------------+
|              Spring Boot 后端 (REST + JWT)                    |
+--------------------------------------------------------------+
```

---

## 入门步骤

1. **阅读必读文档** - components / api-integration / authentication
2. **搭建项目结构** - 参照 [directory-structure.md](./directory-structure.md)
3. **配置路径别名与类型** - 参照 [type-safety.md](./type-safety.md)
4. **配置 axios 实例** - 参照 [api-integration.md](./api-integration.md)
5. **构建首个特性模块** - 参照 [components.md](./components.md) + [composables.md](./composables.md)
6. **提交前** - 完成 [quality.md](./quality.md) 清单

---

**语言**：所有文档使用**简体中文**编写。
