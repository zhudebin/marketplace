# 共享开发规范

> 适用于所有使用 Vue + Spring Boot 架构的全栈应用。

---

## 文档列表

| 文件                                       | 描述                          | 何时阅读                  |
| ------------------------------------------ | ----------------------------- | ------------------------- |
| [api-contract.md](./api-contract.md)       | 统一 API 响应结构与错误约定   | 始终                      |
| [types-sync.md](./types-sync.md)           | OpenAPI 驱动的前端类型生成    | 接口变更时                |
| [code-quality.md](./code-quality.md)       | 命名、注释、禁用项强制规则    | 始终                      |
| [dependencies.md](./dependencies.md)       | 推荐依赖版本与升级注意事项    | 添加/升级依赖时           |

---

## 快速导航

| 任务                  | 文件                                   |
| --------------------- | -------------------------------------- |
| 设计 API 返回格式     | [api-contract.md](./api-contract.md)   |
| 同步前后端类型        | [types-sync.md](./types-sync.md)       |
| 命名约定              | [code-quality.md](./code-quality.md)   |
| 选择/升级依赖         | [dependencies.md](./dependencies.md)   |

---

## 核心规则（强制）

| 规则                                                       | 文件                                   |
| ---------------------------------------------------------- | -------------------------------------- |
| 后端响应必须遵循 `{ success, code, message, data }`        | [api-contract.md](./api-contract.md)   |
| 前端类型必须由 OpenAPI 自动生成，不得手写                  | [types-sync.md](./types-sync.md)       |
| 禁用 `any` / `@ts-ignore`（前端）；禁用 `Object` 通用容器（后端） | [code-quality.md](./code-quality.md) |
| 命名：文件 kebab-case；TS 类型/Java 类 PascalCase；变量 camelCase | [code-quality.md](./code-quality.md) |
| 禁止 `console.log` / `System.out.println` 进入主分支       | [code-quality.md](./code-quality.md)   |

---

## 提交前自检（前后端通用）

- [ ] **接口契约一致**：后端改了 DTO/VO，前端类型同步重新生成
- [ ] **错误码登记**：新增错误码已写入错误码常量
- [ ] **无敏感信息**：日志、响应、提交记录中不含 token/密码/PII
- [ ] **命名一致**：同一字段在前后端使用相同名称（驼峰）
- [ ] **本地构建通过**：前端 `pnpm build`，后端 `mvn clean package -DskipTests`

---

## 跨层 Code Review 清单

- [ ] 字段命名前后端一致
- [ ] 后端枚举与前端类型同步
- [ ] 列表接口前后端分页参数一致（`pageNum/pageSize`）
- [ ] 时间字段统一（推荐后端返回毫秒时间戳或 ISO 8601）
- [ ] 接口语义动词正确（GET 查询、POST 创建、PUT/PATCH 更新、DELETE 删除）

---

**语言**：所有文档使用**简体中文**编写。
