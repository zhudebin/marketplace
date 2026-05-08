# 后端开发规范索引

> **技术栈**：Spring Boot 3.x（Java 17+）+ MyBatis-Plus + MySQL + Flyway + Redis + Spring Security + JWT

## 关联规范

| 规范              | 位置         | 何时阅读             |
| ----------------- | ------------ | -------------------- |
| **共享代码标准**  | `../shared/` | 始终适用             |
| **思维指南**      | `../guides/` | 实现新特性前         |

---

## 文档列表

| 文件                                                       | 描述                                          | 何时阅读                       |
| ---------------------------------------------------------- | --------------------------------------------- | ------------------------------ |
| [directory-structure.md](./directory-structure.md)         | 模块划分、包结构、领域分层                    | 启动新特性                     |
| [controller-patterns.md](./controller-patterns.md)         | REST 接口、参数绑定、统一响应                 | 写/改接口时                    |
| [service-patterns.md](./service-patterns.md)               | 业务服务、事务、领域行为                      | 写业务逻辑时                   |
| [mapper-mybatis-plus.md](./mapper-mybatis-plus.md)         | BaseMapper、Wrapper、自定义 SQL、分页、批量   | 写数据访问时                   |
| [dto-and-validation.md](./dto-and-validation.md)           | DTO/VO/Entity 划分、Bean Validation、MapStruct | 类型/校验决策                  |
| [database.md](./database.md)                               | MySQL 规范、Flyway 迁移、字段约定             | 数据库变更时                   |
| [authentication.md](./authentication.md)                   | Spring Security + JWT、过滤器、权限注解       | 鉴权相关特性                   |
| [exception-handling.md](./exception-handling.md)           | 全局异常处理、错误码体系                      | 错误处理决策                   |
| [logging.md](./logging.md)                                 | Logback JSON、MDC、TraceId、敏感信息脱敏      | 调试与可观测性                 |
| [performance.md](./performance.md)                         | 缓存、并发、批处理、分页                      | 性能优化                       |
| [configuration.md](./configuration.md)                     | application.yml 多环境、Profile、外部化配置   | 配置变更                       |
| [ai-sdk-integration.md](./ai-sdk-integration.md)           | Spring AI、ChatClient、流式响应、Tool         | AI 特性                        |
| [quality.md](./quality.md)                                 | 提交前清单                                    | 提交前                         |

---

## 快速导航

### 模块结构

| 任务                  | 文件                                               |
| --------------------- | -------------------------------------------------- |
| 项目结构              | [directory-structure.md](./directory-structure.md) |
| 领域模块模式          | [directory-structure.md](./directory-structure.md) |
| Controller 写法       | [controller-patterns.md](./controller-patterns.md) |
| Service 写法          | [service-patterns.md](./service-patterns.md)       |
| Mapper 与 SQL         | [mapper-mybatis-plus.md](./mapper-mybatis-plus.md) |
| 命名约定              | [directory-structure.md](./directory-structure.md) |

### 类型与校验

| 任务                 | 文件                                       |
| -------------------- | ------------------------------------------ |
| DTO / VO / Entity 划分 | [dto-and-validation.md](./dto-and-validation.md) |
| Bean Validation       | [dto-and-validation.md](./dto-and-validation.md) |
| MapStruct 转换        | [dto-and-validation.md](./dto-and-validation.md) |
| 统一响应结构          | [controller-patterns.md](./controller-patterns.md) |

### 数据库

| 任务                       | 文件                                       |
| -------------------------- | ------------------------------------------ |
| MyBatis-Plus 基本用法      | [mapper-mybatis-plus.md](./mapper-mybatis-plus.md) |
| LambdaQueryWrapper         | [mapper-mybatis-plus.md](./mapper-mybatis-plus.md) |
| 分页插件                   | [mapper-mybatis-plus.md](./mapper-mybatis-plus.md) |
| 批量插入 / 更新            | [mapper-mybatis-plus.md](./mapper-mybatis-plus.md) |
| 逻辑删除、自动填充、乐观锁 | [mapper-mybatis-plus.md](./mapper-mybatis-plus.md) |
| Flyway 迁移                | [database.md](./database.md)               |
| 字段命名规范               | [database.md](./database.md)               |

### 错误与日志

| 任务                       | 文件                                           |
| -------------------------- | ---------------------------------------------- |
| 全局异常处理               | [exception-handling.md](./exception-handling.md) |
| 错误码体系                 | [exception-handling.md](./exception-handling.md) |
| 结构化日志（JSON）         | [logging.md](./logging.md)                     |
| TraceId 与 MDC             | [logging.md](./logging.md)                     |
| 敏感信息脱敏               | [logging.md](./logging.md)                     |

### 性能

| 任务                          | 文件                               |
| ----------------------------- | ---------------------------------- |
| Redis 缓存（@Cacheable）      | [performance.md](./performance.md) |
| 异步与线程池                  | [performance.md](./performance.md) |
| 批处理与分页                  | [performance.md](./performance.md) |
| 限流与幂等                    | [performance.md](./performance.md) |

### 鉴权

| 任务                       | 文件                                     |
| -------------------------- | ---------------------------------------- |
| JWT 签发与校验             | [authentication.md](./authentication.md) |
| 过滤器链与认证入口         | [authentication.md](./authentication.md) |
| `@PreAuthorize` 与角色权限 | [authentication.md](./authentication.md) |
| 登录态缓存（Redis）        | [authentication.md](./authentication.md) |

### AI 集成

| 任务                  | 文件                                             |
| --------------------- | ------------------------------------------------ |
| ChatClient 调用       | [ai-sdk-integration.md](./ai-sdk-integration.md) |
| 流式响应（SSE）       | [ai-sdk-integration.md](./ai-sdk-integration.md) |
| Tool / Function call  | [ai-sdk-integration.md](./ai-sdk-integration.md) |
| Prompt 管理           | [ai-sdk-integration.md](./ai-sdk-integration.md) |

---

## 核心规则汇总

| 规则                                                                | 参考                                       |
| ------------------------------------------------------------------- | ------------------------------------------ |
| **接口出参一律 `Result<T>`**，禁止裸返回 Entity                     | [controller-patterns.md](./controller-patterns.md) |
| **Controller 不写业务逻辑**，只做 DTO/VO 转换与调用 Service         | [controller-patterns.md](./controller-patterns.md) |
| **Service 必须依赖 Mapper 而非数据源**                              | [service-patterns.md](./service-patterns.md) |
| **Entity 不直接出/入接口**，使用 DTO/VO + MapStruct 转换            | [dto-and-validation.md](./dto-and-validation.md) |
| **DTO 必须 `@Valid`**，禁止在 Service 里再做参数格式校验            | [dto-and-validation.md](./dto-and-validation.md) |
| **禁止在 for 循环里调 Mapper**，使用 `inSql`/`saveBatch`            | [mapper-mybatis-plus.md](./mapper-mybatis-plus.md) |
| **所有 Schema 变更走 Flyway**，禁止手动改库                         | [database.md](./database.md)               |
| **业务异常抛 `BusinessException`**，由 `@RestControllerAdvice` 统一处理 | [exception-handling.md](./exception-handling.md) |
| **禁用 `System.out.println` 与 `e.printStackTrace()`**              | [logging.md](./logging.md)                 |
| **结构化日志使用占位符 `{}`**，禁止字符串拼接                       | [logging.md](./logging.md)                 |
| **JWT 鉴权过滤器只放行白名单接口**                                  | [authentication.md](./authentication.md)   |
| **配置项不允许硬编码**，外部化到 `application-{profile}.yml`        | [configuration.md](./configuration.md)     |
| **跨服务调用必须设超时与重试**                                      | [performance.md](./performance.md)         |
| **提交前跑完 [quality.md](./quality.md) 清单**                      | [quality.md](./quality.md)                 |

---

## 参考路径

| 资源              | 典型位置                                          |
| ----------------- | ------------------------------------------------- |
| 启动类            | `backend/src/main/java/com/company/app/Application.java` |
| MyBatis-Plus 配置 | `backend/src/main/java/com/company/app/config/MybatisPlusConfig.java` |
| Security 配置     | `backend/src/main/java/com/company/app/config/SecurityConfig.java` |
| JWT 工具          | `backend/src/main/java/com/company/app/common/security/JwtUtils.java` |
| 全局异常          | `backend/src/main/java/com/company/app/common/exception/GlobalExceptionHandler.java` |
| 统一响应          | `backend/src/main/java/com/company/app/common/response/Result.java` |
| 错误码            | `backend/src/main/java/com/company/app/common/response/ErrorCode.java` |
| Flyway 脚本       | `backend/src/main/resources/db/migration/`        |
| 配置文件          | `backend/src/main/resources/application*.yml`     |

---

**语言**：所有文档使用**简体中文**编写。
