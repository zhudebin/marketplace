# MyBatis-Plus 与 Mapper

MyBatis-Plus 配置、Mapper 写法、Wrapper、自定义 SQL、分页插件、批量、自动填充、逻辑删除与乐观锁。

## 配置

```java
// config/MybatisPlusConfig.java
@Configuration
@MapperScan("com.company.app.module.**.mapper")
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 分页
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        // 乐观锁
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        // 防全表更新/删除
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
        return interceptor;
    }
}
```

```yaml
# application.yml
mybatis-plus:
  mapper-locations: classpath*:mapper/**/*.xml
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.nologging.NoLoggingImpl   # SQL 日志走 logback
  global-config:
    db-config:
      id-type: auto
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
      table-prefix: t_
```

## Entity (DO) 定义

```java
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("t_order")
public class OrderDO extends BaseEntity {

    @TableId(type = IdType.AUTO)
    private Long id;

    private Long userId;

    private String reference;

    @TableField("status")
    private OrderStatus status;

    private BigDecimal total;

    @Version
    private Integer version;
}
```

```java
// common/mybatis/BaseEntity.java
@Data
public abstract class BaseEntity {

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createdAt;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updatedAt;

    @TableField(fill = FieldFill.INSERT)
    private Long createdBy;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Long updatedBy;

    @TableLogic
    private Integer deleted;
}
```

## 自动填充

```java
// common/mybatis/AutoFillMetaObjectHandler.java
@Component
public class AutoFillMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        Long userId = SecurityUserHelper.currentUserIdOrNull();
        LocalDateTime now = LocalDateTime.now();
        this.strictInsertFill(metaObject, "createdAt", LocalDateTime.class, now);
        this.strictInsertFill(metaObject, "updatedAt", LocalDateTime.class, now);
        this.strictInsertFill(metaObject, "createdBy", Long.class, userId);
        this.strictInsertFill(metaObject, "updatedBy", Long.class, userId);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updatedAt", LocalDateTime.class, LocalDateTime.now());
        this.strictUpdateFill(metaObject, "updatedBy", Long.class, SecurityUserHelper.currentUserIdOrNull());
    }
}
```

## Mapper 接口

```java
public interface OrderMapper extends BaseMapper<OrderDO> {

    // 简单的自定义查询用注解
    @Select("SELECT * FROM t_order WHERE user_id = #{userId} AND status = #{status} AND deleted = 0")
    List<OrderDO> selectByUserAndStatus(@Param("userId") Long userId, @Param("status") OrderStatus status);

    // 复杂查询用 XML（见下方）
    PageResult<OrderListVO> selectListVO(IPage<OrderListVO> page, @Param("query") OrderQueryDTO query);

    // 自定义更新
    @Update("UPDATE t_order SET status = #{status}, updated_at = NOW() WHERE id = #{id} AND deleted = 0")
    int updateStatus(@Param("id") Long id, @Param("status") OrderStatus status);
}
```

## XML SQL（复杂查询）

`resources/mapper/order/OrderMapper.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "https://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.company.app.module.order.mapper.OrderMapper">

    <select id="selectListVO" resultType="com.company.app.module.order.vo.OrderListVO">
        SELECT
            o.id,
            o.reference,
            o.status,
            o.total,
            o.created_at AS createdAt,
            u.username AS userName
        FROM t_order o
        LEFT JOIN t_user u ON u.id = o.user_id AND u.deleted = 0
        <where>
            o.deleted = 0
            <if test="query.status != null and query.status != ''">
                AND o.status = #{query.status}
            </if>
            <if test="query.keyword != null and query.keyword != ''">
                AND o.reference LIKE CONCAT('%', #{query.keyword}, '%')
            </if>
            <if test="query.userId != null">
                AND o.user_id = #{query.userId}
            </if>
        </where>
        ORDER BY o.id DESC
    </select>
</mapper>
```

> **关于 Wrapper vs XML**：
>
> - **简单单表 CRUD**：用 `LambdaQueryWrapper`
> - **多表 JOIN / 聚合 / 复杂条件**：用 XML（避免拼接代码丑陋）
> - **超过 3 个 if 条件分支**：建议 XML

## LambdaQueryWrapper

```java
// 推荐：lambda 写法，字段重命名编译期发现
List<OrderDO> orders = orderMapper.selectList(
    Wrappers.<OrderDO>lambdaQuery()
        .eq(OrderDO::getUserId, userId)
        .eq(OrderDO::getStatus, OrderStatus.PENDING)
        .ge(OrderDO::getCreatedAt, since)
        .orderByDesc(OrderDO::getCreatedAt)
        .last("LIMIT 100")
);

// 反模式：字符串字段名，重构易碎
new QueryWrapper<OrderDO>().eq("user_id", userId);
```

## 分页

```java
public PageResult<OrderListVO> page(OrderQueryDTO query) {
    Page<OrderListVO> page = new Page<>(query.getPageNum(), query.getPageSize());
    PageResult<OrderListVO> result = orderMapper.selectListVO(page, query);
    return PageResult.of(page, Function.identity());
}
```

或在仅单表查询时直接用 BaseMapper：

```java
Page<OrderDO> page = new Page<>(pageNum, pageSize);
orderMapper.selectPage(page, Wrappers.<OrderDO>lambdaQuery()
    .eq(OrderDO::getStatus, OrderStatus.PENDING));
return PageResult.of(page, orderConverter::toVO);
```

## 批量插入

### 使用 IService

```java
@Service
public class OrderItemServiceImpl extends ServiceImpl<OrderItemMapper, OrderItemDO> {

    public void saveItems(List<OrderItemDO> items) {
        this.saveBatch(items, 500);  // 每 500 条 flush 一次
    }
}
```

### 自定义批量

如果用 `INSERT ... VALUES (...), (...)` 单条 SQL 性能更好：

```xml
<insert id="insertBatch">
    INSERT INTO t_order_item (order_id, product_id, quantity, price)
    VALUES
    <foreach collection="items" item="item" separator=",">
        (#{item.orderId}, #{item.productId}, #{item.quantity}, #{item.price})
    </foreach>
</insert>
```

## 关键反模式：循环里调 Mapper

```java
// 反模式：N 次往返
for (Long id : ids) {
    OrderDO o = orderMapper.selectById(id);
    // ...
}

// 正确：一次 inSql
List<OrderDO> orders = orderMapper.selectList(
    Wrappers.<OrderDO>lambdaQuery().in(OrderDO::getId, ids)
);
Map<Long, OrderDO> byId = orders.stream()
    .collect(Collectors.toMap(OrderDO::getId, Function.identity()));
```

## 逻辑删除

加 `@TableLogic` 后，`selectById` / `delete` 会自动追加 `WHERE deleted = 0`：

```java
@TableLogic
private Integer deleted;

// delete 实际上执行 UPDATE ... SET deleted = 1
orderMapper.deleteById(id);
```

> **重要陷阱**：业务唯一索引必须包含 `deleted` 列，否则同一业务键删除后无法重新创建。详见 [big-question/mybatis-plus-logic-delete.md](../big-question/mybatis-plus-logic-delete.md)。

## 乐观锁

```java
@Version
private Integer version;
```

更新时 MP 自动追加 `WHERE version = #{oldVersion}`，并 `version = oldVersion + 1`。如果返回 0 行，调用方应判定并重试或抛错：

```java
int rows = orderMapper.updateById(order);
if (rows == 0) {
    throw new BusinessException(ErrorCode.CONCURRENT_MODIFICATION);
}
```

## 枚举处理

```yaml
mybatis-plus:
  configuration:
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
```

```java
@Getter
public enum OrderStatus {
    PENDING("PENDING"),
    PAID("PAID"),
    CANCELLED("CANCELLED");

    @EnumValue          // 写入数据库的值
    private final String code;

    @JsonValue          // 返回前端的值
    OrderStatus(String code) { this.code = code; }
}
```

数据库列类型：`VARCHAR(32)`，与 `code` 一致。

## JSON 字段

```java
@TableField(typeHandler = JacksonTypeHandler.class)
private List<OrderTagDTO> tags;
```

```java
@TableName(value = "t_order", autoResultMap = true)
public class OrderDO { ... }
```

> 必须 `autoResultMap = true`，否则 `selectById` 不会反序列化 JSON。

## 防全表 update/delete

`BlockAttackInnerInterceptor` 已在配置中启用。它会拦截不带 `WHERE` 的 `UPDATE` / `DELETE`。

## SQL 日志

开发期开启慢 SQL 日志，定位 N+1：

```yaml
logging:
  level:
    com.company.app.module.**.mapper: debug
```

或集成 `p6spy` 输出真实参数化 SQL。

## 测试

```java
@SpringBootTest
@AutoConfigureMybatisPlus
@Transactional
class OrderMapperTest {

    @Autowired
    private OrderMapper orderMapper;

    @Test
    void shouldInsertAndSelect() {
        OrderDO o = new OrderDO();
        // ...
        orderMapper.insert(o);
        OrderDO loaded = orderMapper.selectById(o.getId());
        assertThat(loaded).isNotNull();
    }
}
```

## 最佳实践

1. **简单单表用 LambdaQueryWrapper，多表 JOIN 用 XML**
2. **禁止 for 循环里调 Mapper**，用 `inSql` 或批量
3. **业务唯一索引必须含 `deleted` 列**
4. **`@Version` 乐观锁判断更新行数**
5. **JSON 字段 `autoResultMap = true`**
6. **批量插入 `saveBatch` 或 `<foreach>`**

## 反模式

- 在 Service 里手写 SQL 字符串拼接
- 字符串字段名 `eq("user_id", ...)`
- 用 `BaseMapper.delete(null)` 或不带条件的 update
- 直接返回 `OrderDO` 给 Controller
- XML 中重复定义 `<resultMap>`（用 `resultType` + 自动 camelCase 映射）
- 不加 `@Version` 的并发更新
