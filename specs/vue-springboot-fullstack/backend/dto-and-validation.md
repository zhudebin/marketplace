# DTO/VO/Entity 与校验

明确 Entity（DO）/ DTO / VO 三层职责，配合 Bean Validation 与 MapStruct 完成边界转换。

## 三层职责

| 类型     | 后缀  | 用途                                  | 出现位置                              |
| -------- | ----- | ------------------------------------- | ------------------------------------- |
| Entity   | `DO`  | 数据库表映射                          | Mapper、Service 内部                  |
| DTO      | `DTO` | 接口入参（前端 → 后端）              | Controller `@RequestBody` / 查询参数 |
| VO       | `VO`  | 接口出参（后端 → 前端）              | Controller 返回                       |

> **铁律**：DO 永不出现在 Controller 签名中。

## DO 定义

详见 [mapper-mybatis-plus.md](./mapper-mybatis-plus.md)。要点：

- `@TableName` 显式指定表名
- 继承 `BaseEntity`（`createdAt/updatedAt/deleted/version`）
- 字段 camelCase（库表 snake_case 由全局配置自动映射）

```java
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("t_user")
public class UserDO extends BaseEntity {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String username;
    private String passwordHash;
    private String email;
    private UserStatus status;
}
```

## DTO 定义

接口入参，**必须**有 Bean Validation 注解，**禁止**包含数据库内部字段（如 `createdBy/updatedAt`）。

```java
@Data
@Schema(description = "创建用户请求")
public class UserCreateDTO {

    @NotBlank(message = "用户名不能为空")
    @Size(min = 3, max = 32, message = "用户名长度 3-32")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "用户名只能包含字母、数字、下划线")
    private String username;

    @NotBlank
    @Email(message = "邮箱格式不正确")
    private String email;

    @NotBlank
    @Size(min = 8, max = 64)
    private String password;

    @NotNull
    private UserRole role;
}
```

### 查询 DTO

```java
@Data
public class UserQueryDTO {
    private String keyword;
    private UserStatus status;

    @Min(1)
    private Integer pageNum = 1;

    @Min(1)
    @Max(200)
    private Integer pageSize = 20;
}
```

### 分组校验

不同接口对同一对象有不同必填要求时使用分组：

```java
public interface Create {}
public interface Update {}

@Data
public class UserDTO {
    @Null(groups = Create.class)
    @NotNull(groups = Update.class)
    private Long id;

    @NotBlank(groups = {Create.class, Update.class})
    private String username;
}
```

```java
@PostMapping
public Result<UserVO> create(@Validated(Create.class) @RequestBody UserDTO dto) { ... }

@PutMapping("/{id}")
public Result<UserVO> update(@PathVariable Long id, @Validated(Update.class) @RequestBody UserDTO dto) { ... }
```

## VO 定义

接口出参，按"消费场景"裁剪。**禁止**直接返回 DO。

```java
@Data
@Schema(description = "用户简要信息（列表用）")
public class UserVO {
    private Long id;
    private String username;
    private String email;
    private UserRole role;
    private UserStatus status;
    private LocalDateTime createdAt;
}

@Data
@Schema(description = "用户详情（含订单统计）")
public class UserDetailVO {
    private Long id;
    private String username;
    private String email;
    private UserRole role;
    private UserStatus status;
    private LocalDateTime createdAt;
    private Long totalOrderCount;
    private BigDecimal totalSpend;
}
```

### 不要在 VO 暴露敏感字段

```java
// 反模式：直接 BeanUtils.copyProperties(userDO, vo)，把 passwordHash 也拷过去
public class UserVO {
    private String passwordHash;  // ❌
}
```

VO 显式声明字段，从源头杜绝信息泄漏。

## Bean Validation 速查

| 注解                          | 用途                              |
| ----------------------------- | --------------------------------- |
| `@NotNull`                    | 不为 null（允许空字符串）         |
| `@NotBlank`                   | 字符串不为 null + 非空白          |
| `@NotEmpty`                   | 集合 / 数组 / 字符串非空          |
| `@Size(min, max)`             | 字符串/集合长度                   |
| `@Min(n)` / `@Max(n)`         | 数值范围                          |
| `@DecimalMin` / `@DecimalMax` | BigDecimal 范围                   |
| `@Email`                      | 邮箱                              |
| `@Pattern(regexp)`            | 正则                              |
| `@Past` / `@Future`           | 过去/未来时间                     |
| `@Valid`                      | 嵌套对象 / 集合元素递归校验       |

### 嵌套校验

```java
@Data
public class OrderCreateDTO {
    @NotNull
    private Long customerId;

    @NotEmpty
    @Valid                                  // 必须加 @Valid 才会校验集合元素
    private List<OrderItemDTO> items;
}

@Data
public class OrderItemDTO {
    @NotNull
    private Long productId;

    @Min(1)
    private Integer quantity;
}
```

### 自定义校验

```java
@Documented
@Constraint(validatedBy = PhoneValidator.class)
@Target({FIELD, PARAMETER})
@Retention(RUNTIME)
public @interface Phone {
    String message() default "手机号格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneValidator implements ConstraintValidator<Phone, String> {
    private static final Pattern P = Pattern.compile("^1[3-9]\\d{9}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        return value == null || P.matcher(value).matches();
    }
}
```

## MapStruct 转换

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>

<!-- 在 maven-compiler-plugin 中加 annotationProcessorPaths -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>17</source>
        <target>17</target>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.5.5.Final</version>
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.30</version>
            </path>
            <!-- 注意：lombok 与 mapstruct 顺序敏感 -->
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok-mapstruct-binding</artifactId>
                <version>0.2.0</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

```java
@Mapper(componentModel = "spring")
public interface UserConverter {

    UserVO toVO(UserDO entity);

    UserDetailVO toDetailVO(UserDO entity);

    UserDO toDO(UserCreateDTO dto);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "passwordHash", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    void updateDOFromDTO(UserUpdateDTO dto, @MappingTarget UserDO entity);

    List<UserVO> toVOList(List<UserDO> entities);
}
```

注入即用：

```java
@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {
    private final UserMapper userMapper;
    private final UserConverter userConverter;

    public UserVO findById(Long id) {
        UserDO entity = userMapper.selectById(id);
        if (entity == null) throw new BusinessException(ErrorCode.USER_NOT_FOUND);
        return userConverter.toVO(entity);
    }
}
```

### 复杂映射

```java
@Mapper(componentModel = "spring")
public interface OrderConverter {

    @Mapping(source = "user.username", target = "userName")
    @Mapping(source = "createdAt", target = "createdAt", dateFormat = "yyyy-MM-dd HH:mm:ss")
    @Mapping(target = "totalText", expression = "java(String.format(\"￥%.2f\", entity.getTotal()))")
    OrderVO toVO(OrderDO entity);
}
```

> 复杂表达式（如多源拼装）建议写成 `default` 方法，比 `expression` 更可读。

## 校验失败处理

`@Valid` 失败抛 `MethodArgumentNotValidException`，由 `GlobalExceptionHandler` 统一捕获并返回字段错误，前端可回填表单。详见 [exception-handling.md](./exception-handling.md)。

## 时间字段约定

后端统一用 `LocalDateTime`，全局 Jackson 配置序列化为 ISO 8601 字符串：

```yaml
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
```

```java
@Data
public class OrderVO {
    private LocalDateTime createdAt;  // → "2026-05-08 20:00:00"
}
```

> 跨时区项目可改为 `Instant` + 毫秒时间戳，前端用 `dayjs` 转本地时间。

## 钱字段约定

- DO / DTO / VO 一律 `BigDecimal`
- 数据库列 `DECIMAL(15,2)`
- 永不使用 `Double` / `Float`

## 最佳实践

1. **DO/DTO/VO 三件套，DO 不出接口**
2. **DTO 用 `@Valid`**，校验前置
3. **VO 显式声明字段**，杜绝敏感泄漏
4. **MapStruct 做转换**，禁用 `BeanUtils.copyProperties`
5. **金额 `BigDecimal`，时间 `LocalDateTime`**

## 反模式

- 一个 `UserVO` 既给列表又给详情（应分 `UserVO` / `UserDetailVO`）
- DTO 里包含 `id/createdAt/updatedAt/deleted` 等内部字段
- 用 `Map<String, Object>` 替代 DTO/VO
- `BeanUtils.copyProperties` 散落各处（用 MapStruct）
- 字段类型用 `Date`（用 `LocalDateTime`）
- 校验逻辑写在 Service 里（应在 DTO + `@Valid`）
