# 推荐依赖

> 列出本规范默认使用的库与版本基线。**升级前必须读完对应文档**，避免破坏性变更。

## 前端（Vue 3）

### 核心

| 包                  | 用途             | 推荐版本                |
| ------------------- | ---------------- | ----------------------- |
| `vue`               | 框架             | `^3.4`                  |
| `vite`              | 构建             | `^5.0`                  |
| `typescript`        | 类型             | `^5.4`                  |
| `vue-router`        | 路由             | `^4.3`                  |
| `pinia`             | 状态             | `^2.1`                  |
| `axios`             | HTTP             | `^1.7`                  |
| `element-plus`      | UI 组件库         | `^2.7`                  |
| `@element-plus/icons-vue` | 图标         | `^2.3`                  |
| `dayjs`             | 时间             | `^1.11`                 |
| `lodash-es`         | 工具函数（按需） | `^4.17`                 |

### 构建/开发

| 包                                      | 用途                       |
| --------------------------------------- | -------------------------- |
| `@vitejs/plugin-vue`                    | Vue SFC 支持               |
| `unplugin-auto-import`                  | API 自动导入               |
| `unplugin-vue-components`               | 组件按需自动导入           |
| `unocss` 或 `tailwindcss`（二选一）     | 原子化 CSS                 |
| `eslint` + `@vue/eslint-config-typescript` | 静态检查                |
| `prettier`                              | 格式化                     |
| `vitest` + `@vue/test-utils`            | 单元测试                   |
| `openapi-typescript`                    | 后端类型生成               |
| `husky` + `lint-staged`                 | 提交钩子                   |

### 不推荐 / 慎用

- `vuex` —— 用 Pinia 替代
- `moment.js` —— 用 dayjs
- `axios-mock-adapter` 入生产 —— 仅 dev
- `jquery` —— 完全无必要

---

## 后端（Spring Boot 3）

### 平台

| 项                    | 推荐                           |
| --------------------- | ------------------------------ |
| JDK                   | 17 LTS（或 21 LTS）            |
| Spring Boot           | 3.2.x / 3.3.x                  |
| 构建                  | Maven 3.9+ 或 Gradle 8+        |

### 核心 starter

| 依赖                                          | 用途                     |
| --------------------------------------------- | ------------------------ |
| `spring-boot-starter-web`                     | Web/MVC                  |
| `spring-boot-starter-validation`              | Bean Validation          |
| `spring-boot-starter-security`                | 安全/鉴权                |
| `spring-boot-starter-data-redis`              | Redis                    |
| `spring-boot-starter-actuator`                | 健康检查/指标            |
| `spring-boot-starter-aop`                     | 切面                     |

### 数据访问

| 依赖                                                   | 用途                     |
| ------------------------------------------------------ | ------------------------ |
| `mybatis-plus-spring-boot3-starter` `^3.5.7`           | ORM                      |
| `mysql-connector-j`                                    | MySQL 驱动               |
| `flyway-core` + `flyway-mysql`                         | 数据库迁移               |
| `com.alibaba:druid-spring-boot-3-starter`              | 连接池（或用默认 Hikari） |

### 工具与转换

| 依赖                                       | 用途                |
| ------------------------------------------ | ------------------- |
| `org.projectlombok:lombok`                 | 减少样板代码        |
| `org.mapstruct:mapstruct` `^1.5`           | 对象转换            |
| `org.mapstruct:mapstruct-processor`        | MapStruct 注解处理  |
| `commons-lang3`                            | 工具类              |
| `guava`（按需）                            | 集合/缓存工具       |

### 鉴权 / JWT

| 依赖                                                           | 用途        |
| -------------------------------------------------------------- | ----------- |
| `io.jsonwebtoken:jjwt-api` `^0.12`                             | JWT API     |
| `io.jsonwebtoken:jjwt-impl`（runtime）                         | 实现        |
| `io.jsonwebtoken:jjwt-jackson`（runtime）                      | 序列化      |

### 文档与可观测

| 依赖                                                    | 用途                |
| ------------------------------------------------------- | ------------------- |
| `springdoc-openapi-starter-webmvc-ui` `^2.5`            | Swagger UI          |
| `net.logstash.logback:logstash-logback-encoder` `^7.4`  | JSON 日志           |
| `micrometer-registry-prometheus`                        | 指标导出（可选）    |

### AI（可选）

| 依赖                                        | 用途              |
| ------------------------------------------- | ----------------- |
| `spring-ai-openai-spring-boot-starter`      | OpenAI 兼容客户端 |

### 测试

| 依赖                                       | 用途                  |
| ------------------------------------------ | --------------------- |
| `spring-boot-starter-test`                 | JUnit 5 + Mockito     |
| `org.testcontainers:mysql`                 | 集成测试用真实 MySQL  |
| `org.testcontainers:junit-jupiter`         | JUnit 5 集成          |

---

## 版本约束

- 所有版本通过 `pom.xml` 的 `<dependencyManagement>` 或 `gradle/libs.versions.toml` **集中管理**
- **禁止**在子模块或单个 starter 内部覆盖版本号
- 升级跨主版本（e.g. Spring Boot 2 → 3、Vue 2 → 3）必须有专门的迁移 PR + 完整回归测试

---

## 安全更新

- 每月跑一次 `pnpm audit` / `mvn versions:display-dependency-updates`
- CVE 评级 High 以上的依赖**必须 7 天内升级或缓解**
- 使用 GitHub Dependabot / Renovate 自动 PR

---

## 不要做

| 反模式                                  | 正确做法                              |
| --------------------------------------- | ------------------------------------- |
| 在多个模块各自声明同一依赖的不同版本    | BOM / catalog 统一管理                |
| 升级时一次性升 N 个大库                 | 一次一个，分别提 PR                   |
| 依赖未声明范围（compile/runtime/test）  | 显式标注，特别是 runtime / test 依赖   |
| 使用 SNAPSHOT 入生产                    | 严禁                                  |
