# Vue + Spring Boot 全栈开发规范

为生产级 Vue 3 + Spring Boot 3 全栈应用提供通用开发规范。前后端单仓（monorepo），后端使用 MyBatis-Plus + MySQL，前端使用 Vite + Element Plus + Pinia。

## 结构

### [前端](./frontend/index.md)

Vue 3 + Vite + TypeScript + Element Plus 前端开发模式：

- [目录结构](./frontend/directory-structure.md)
- [组件规范](./frontend/components.md)
- [路由与守卫](./frontend/routing.md)
- [状态管理 (Pinia)](./frontend/state-management.md)
- [Composables](./frontend/composables.md)
- [API 集成 (axios)](./frontend/api-integration.md)
- [鉴权](./frontend/authentication.md)
- [表单与校验 (Element Plus)](./frontend/form-validation.md)
- [AI SDK 集成](./frontend/ai-sdk-integration.md)
- [样式与布局](./frontend/css-layout.md)
- [类型安全](./frontend/type-safety.md)
- [质量清单](./frontend/quality.md)

### [后端](./backend/index.md)

Spring Boot 3 + MyBatis-Plus + MySQL 后端开发模式：

- [目录结构](./backend/directory-structure.md)
- [Controller 规范](./backend/controller-patterns.md)
- [Service 规范](./backend/service-patterns.md)
- [MyBatis-Plus 与 Mapper](./backend/mapper-mybatis-plus.md)
- [DTO/VO/Entity 与校验](./backend/dto-and-validation.md)
- [数据库与 Flyway](./backend/database.md)
- [鉴权 (Spring Security + JWT)](./backend/authentication.md)
- [全局异常与统一响应](./backend/exception-handling.md)
- [日志 (Logback JSON)](./backend/logging.md)
- [缓存与性能](./backend/performance.md)
- [配置管理](./backend/configuration.md)
- [AI SDK 集成 (Spring AI)](./backend/ai-sdk-integration.md)
- [质量清单](./backend/quality.md)

### [跨层共享](./shared/index.md)

跨层关注点：

- [API 契约 (统一响应)](./shared/api-contract.md)
- [类型同步 (OpenAPI -> TS)](./shared/types-sync.md)
- [代码质量](./shared/code-quality.md)
- [依赖版本](./shared/dependencies.md)

### [思维指南](./guides/index.md)

开发前的思考框架：

- [实现前检查清单](./guides/pre-implementation-checklist.md)
- [跨层思考指南](./guides/cross-layer-thinking-guide.md)

### [常见问题 / 踩坑](./big-question/index.md)

生产环境踩过的坑及解决方案：

- [MyBatis-Plus 逻辑删除与唯一索引冲突](./big-question/mybatis-plus-logic-delete.md)
- [JWT 过期与刷新策略](./big-question/jwt-refresh-strategy.md)
- [Flyway 与开发期 Schema 漂移](./big-question/flyway-dev-drift.md)
- [Element Plus + Vite 按需导入与样式丢失](./big-question/element-plus-on-demand.md)
- [Spring Boot 3 与 Jakarta EE 迁移陷阱](./big-question/springboot3-jakarta-migration.md)
- [跨域与 Cookie SameSite](./big-question/cors-samesite.md)

## 技术栈

- **前端**：Vue 3 + Vite + TypeScript + Element Plus + Pinia + Vue Router 4 + axios
- **后端**：Spring Boot 3.x（Java 17+）+ MyBatis-Plus + MySQL + Flyway + Redis
- **鉴权**：Spring Security + JWT
- **接口文档**：springdoc-openapi（Swagger UI）
- **对象转换**：MapStruct
- **日志**：Logback + Logstash JSON encoder
- **AI**：Spring AI（后端）+ SSE/EventSource（前端流式消费）
- **构建**：Vite（前端）+ Maven/Gradle（后端），单仓双工程

## 仓库形态

```
project-root/
├── frontend/              # Vue 3 工程
├── backend/               # Spring Boot 工程
├── docs/                  # 项目文档
├── docker-compose.yml     # 本地依赖（MySQL/Redis）
└── README.md
```

## 用法

本规范可作为：

1. **新项目模板** - 复制整体结构创建新项目
2. **参考文档** - 实现具体功能时查阅对应小节
3. **CR 检查清单** - 对照规范评审代码
4. **新人 Onboarding** - 帮助新成员理解项目约定

---

**语言**：所有文档使用**简体中文**编写。
