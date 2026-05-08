# 后端目录结构

定义 Spring Boot 工程的包结构与领域分层。

## 顶层结构

```
backend/
├── pom.xml                                    # 或 build.gradle
├── src/
│   ├── main/
│   │   ├── java/com/company/app/
│   │   │   ├── Application.java               # 启动类
│   │   │   ├── common/                        # 跨模块通用
│   │   │   │   ├── response/                  # Result, ErrorCode
│   │   │   │   ├── exception/                 # BusinessException, GlobalExceptionHandler
│   │   │   │   ├── security/                  # JwtUtils, SecurityUserHelper
│   │   │   │   ├── mybatis/                   # MetaObjectHandler, BaseEntity
│   │   │   │   ├── annotation/                # 自定义注解
│   │   │   │   └── util/                      # 工具类（无业务）
│   │   │   ├── config/                        # @Configuration 集中地
│   │   │   │   ├── MybatisPlusConfig.java
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   ├── RedisConfig.java
│   │   │   │   ├── OpenApiConfig.java
│   │   │   │   ├── WebConfig.java             # CORS、拦截器、消息转换
│   │   │   │   └── AsyncConfig.java
│   │   │   ├── infrastructure/                # 三方客户端、消息队列
│   │   │   │   ├── ai/
│   │   │   │   └── storage/
│   │   │   └── module/                        # 业务领域
│   │   │       ├── auth/                      # 登录、JWT 签发、登出
│   │   │       ├── user/
│   │   │       └── order/
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── logback-spring.xml
│   │       ├── mapper/                        # MyBatis XML（如使用）
│   │       │   └── order/OrderMapper.xml
│   │       └── db/migration/                  # Flyway 脚本
│   │           ├── V1__init.sql
│   │           └── V2__add_index.sql
│   └── test/
│       └── java/com/company/app/
└── target/ 或 build/
```

## 业务模块结构

每个领域模块（`module/[domain]/`）保持自洽：

```
module/order/
├── controller/
│   └── OrderController.java
├── service/
│   ├── OrderService.java                # 接口
│   └── impl/
│       └── OrderServiceImpl.java         # 实现（@Service）
├── mapper/
│   └── OrderMapper.java                  # extends BaseMapper<OrderDO>
├── entity/
│   └── OrderDO.java                      # 数据库实体（@TableName）
├── dto/                                  # 入参（接口契约）
│   ├── OrderCreateDTO.java
│   ├── OrderUpdateDTO.java
│   └── OrderQueryDTO.java
├── vo/                                   # 出参（接口契约）
│   ├── OrderVO.java
│   └── OrderDetailVO.java
├── convert/
│   └── OrderConverter.java               # MapStruct
├── enums/
│   └── OrderStatus.java
└── event/                                # 领域事件（可选）
    └── OrderCreatedEvent.java
```

> **关键**：`controller/service/mapper/entity/dto/vo/convert` 是 **七件套**，每个领域模块尽量都要齐备。

## 文件职责

### `controller/XxxController.java`

- 使用 `@RestController` + `@RequestMapping("/api/orders")`
- 仅做 HTTP 协议层与 DTO/VO 转换
- 通过 `@Valid` 做参数校验
- 调 Service 拿 DO/Entity 后用 Converter 转 VO 返回
- 统一返回 `Result<T>`
- 详见 [controller-patterns.md](./controller-patterns.md)

### `service/XxxService.java` + `impl/XxxServiceImpl.java`

- 接口与实现分离
- 业务逻辑、事务边界（`@Transactional`）
- 不直接 inject 其他模块的 Mapper（应通过其他模块的 Service）
- 详见 [service-patterns.md](./service-patterns.md)

### `mapper/XxxMapper.java`

- `extends BaseMapper<XxxDO>`
- MyBatis-Plus 注解 + 复杂 SQL（XML 或 `@Select` 注解）
- 详见 [mapper-mybatis-plus.md](./mapper-mybatis-plus.md)

### `entity/XxxDO.java`

- 数据库映射对象，`@TableName("t_order")`
- **不允许直接出现在 Controller 入参/返回值中**
- 字段命名 camelCase（与库表 snake_case 由 `mapUnderscoreToCamelCase` 自动映射）

### `dto/XxxDTO.java` 与 `vo/XxxVO.java`

- DTO：入参（前端 → 后端）
- VO：出参（后端 → 前端）
- 详见 [dto-and-validation.md](./dto-and-validation.md)

### `convert/XxxConverter.java`

- `@Mapper(componentModel = "spring")` 的 MapStruct 接口
- 在 Service 边界完成 DO ↔ DTO/VO 转换

## 命名约定

### Java 类

| 类型           | 后缀         | 示例                       |
| -------------- | ------------ | -------------------------- |
| 数据库实体     | `DO`         | `OrderDO`                  |
| 接口入参       | `DTO`        | `OrderCreateDTO`           |
| 接口出参       | `VO`         | `OrderVO`、`OrderDetailVO` |
| 控制器         | `Controller` | `OrderController`          |
| 服务接口       | `Service`    | `OrderService`             |
| 服务实现       | `ServiceImpl`| `OrderServiceImpl`         |
| Mapper         | `Mapper`     | `OrderMapper`              |
| 转换器         | `Converter`  | `OrderConverter`           |
| 异常           | `Exception`  | `OrderNotFoundException`   |
| 配置类         | `Config`     | `MybatisPlusConfig`        |
| 枚举           | 无后缀       | `OrderStatus`              |
| 事件           | `Event`      | `OrderCreatedEvent`        |

### 包名

全小写、单数：`module.order` 而不是 `module.orders`。

### 数据库表

- 表名：`t_` 前缀 + snake_case 单数：`t_order`、`t_user`
- 字段：snake_case：`created_at`、`user_id`
- 主键：`id`，类型 `BIGINT UNSIGNED AUTO_INCREMENT` 或 雪花 ID（视项目而定）

## 跨模块调用

A 模块需要 B 模块数据时：

```java
// 推荐：通过 Service
@RequiredArgsConstructor
@Service
public class OrderServiceImpl implements OrderService {
    private final UserService userService;  // 注入服务接口

    public OrderVO create(OrderCreateDTO dto) {
        UserVO user = userService.findById(dto.getUserId());
        // ...
    }
}

// 反模式：直接 inject 别人的 Mapper
public class OrderServiceImpl {
    private final UserMapper userMapper;  // ❌ 跨越服务边界
}
```

## 何时新建模块

新建 `module/xxx/` 当：

1. 是一个独立的领域实体（`order` / `user` / `product`）
2. 有独立的生命周期与多个相关操作
3. 可能被其他模块依赖

不新建模块的场景：

1. 单个工具方法 → `common/util/`
2. 单条 SQL 查询 → 放到对应业务模块
3. 跨多个模块的协调 → 在 `module/{coordinator}/` 中

## 参考路径

| 资源              | 位置                                                                          |
| ----------------- | ----------------------------------------------------------------------------- |
| 启动类            | `com.company.app.Application`                                                 |
| 统一响应          | `com.company.app.common.response.Result`                                      |
| 错误码            | `com.company.app.common.response.ErrorCode`                                   |
| 业务异常          | `com.company.app.common.exception.BusinessException`                          |
| 全局异常处理      | `com.company.app.common.exception.GlobalExceptionHandler`                     |
| MyBatis-Plus 配置 | `com.company.app.config.MybatisPlusConfig`                                    |
| 安全配置          | `com.company.app.config.SecurityConfig`                                       |
| 元对象处理器      | `com.company.app.common.mybatis.AutoFillMetaObjectHandler`                    |
| 安全上下文工具    | `com.company.app.common.security.SecurityUserHelper`                          |

## 反模式

- 按"层"建顶层包（`controller/` / `service/` / `mapper/` 顶层平铺）→ 模块边界消失
- DO 直接出现在接口签名中
- Service 直接 inject 数据库 `DataSource` / `JdbcTemplate`
- Util 类承载业务逻辑
- 一个 Controller 调五个模块的 Service（应当封装到协调层）
