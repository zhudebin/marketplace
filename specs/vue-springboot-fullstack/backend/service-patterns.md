# Service 规范

业务 Service 的职责边界、接口/实现分离、事务管理与领域行为。

## 接口与实现分离

强制使用接口 + 实现分离，便于 mock 测试和替换实现：

```java
// service/OrderService.java
public interface OrderService {
    OrderVO findById(Long id);
    PageResult<OrderVO> page(OrderQueryDTO query);
    OrderVO create(Long userId, OrderCreateDTO dto);
    void cancel(Long id);
}
```

```java
// service/impl/OrderServiceImpl.java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderServiceImpl implements OrderService {

    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;
    private final OrderConverter orderConverter;
    private final UserService userService;          // 跨模块走 Service
    private final ApplicationEventPublisher publisher;

    @Override
    public OrderVO findById(Long id) {
        OrderDO order = orderMapper.selectById(id);
        if (order == null) {
            throw new BusinessException(ErrorCode.ORDER_NOT_FOUND);
        }
        return orderConverter.toVO(order);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public OrderVO create(Long userId, OrderCreateDTO dto) {
        // 1. 业务校验
        UserVO user = userService.findById(userId);
        validateUserCanOrder(user);

        // 2. 落主表
        OrderDO order = orderConverter.toDO(dto);
        order.setUserId(userId);
        order.setStatus(OrderStatus.PENDING);
        order.setReference(generateReference());
        orderMapper.insert(order);

        // 3. 落明细
        List<OrderItemDO> items = dto.getItems().stream()
            .map(it -> orderConverter.toItemDO(it, order.getId()))
            .toList();
        orderItemMapper.insertBatch(items);

        // 4. 计算总额
        BigDecimal total = items.stream()
            .map(it -> it.getPrice().multiply(BigDecimal.valueOf(it.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        orderMapper.updateTotal(order.getId(), total);
        order.setTotal(total);

        // 5. 发事件（事务提交后再发，见下文）
        publisher.publishEvent(new OrderCreatedEvent(order.getId()));

        log.info("订单已创建 orderId={} userId={} total={}", order.getId(), userId, total);
        return orderConverter.toVO(order);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void cancel(Long id) {
        OrderDO order = orderMapper.selectById(id);
        if (order == null) {
            throw new BusinessException(ErrorCode.ORDER_NOT_FOUND);
        }
        if (order.getStatus() != OrderStatus.PENDING) {
            throw new BusinessException(ErrorCode.ORDER_CANNOT_CANCEL);
        }
        orderMapper.updateStatus(id, OrderStatus.CANCELLED);
    }

    private void validateUserCanOrder(UserVO user) {
        if (user.getStatus() != UserStatus.ACTIVE) {
            throw new BusinessException(ErrorCode.USER_NOT_ACTIVE);
        }
    }
}
```

## 强制规则

| 规则                                                              | 原因                                |
| ----------------------------------------------------------------- | ----------------------------------- |
| Service 接口在 `service/`，实现在 `service/impl/`                 | 解耦、易于 mock                     |
| 实现类用 `@Service` + 构造注入（`@RequiredArgsConstructor`）      | 不用字段注入                        |
| 业务异常抛 `BusinessException(ErrorCode.X)`                       | 由全局处理器统一返回                |
| 事务必须 `@Transactional(rollbackFor = Exception.class)`          | 默认只回滚 RuntimeException         |
| **跨模块调用走 Service 接口，不能直接 inject 别人的 Mapper**       | 维护模块边界                        |
| 单个 Service 类不超过 ~500 行                                     | 超出应拆子服务或助手类              |
| 入参用 DTO，出参用 VO                                             | 不返回 DO                           |

## 事务管理

### 事务方法的最小作用域

事务越短越好，把 IO 调用（HTTP、消息队列、文件）**踢出**事务：

```java
// 反模式：把外部调用包在事务内，可能撑爆连接池
@Transactional
public void create(...) {
    orderMapper.insert(order);
    notifySmsService.send(...);   // 慢 + 不可控
    paymentService.charge(...);   // 第三方
}

// 推荐：事务只包数据库操作
public void create(...) {
    Long orderId = doInsertInTx(dto);
    notifySmsService.send(...);     // 事务外
    paymentService.chargeAsync(orderId);
}

@Transactional(rollbackFor = Exception.class)
protected Long doInsertInTx(OrderCreateDTO dto) {
    // 仅 DB 操作
}
```

### 自调用导致 `@Transactional` 失效

```java
// 反模式：A 调 B，事务不生效（绕过代理）
public void a() {
    this.b();  // 直接 this 调用
}

@Transactional
public void b() { ... }
```

解决：拆到另一个 Service / 用 `AopContext.currentProxy()` / 用 `TransactionTemplate`。

```java
// 推荐
@Transactional(rollbackFor = Exception.class)
public void b() { ... }

// 在另一个 Service 调
public class AService {
    private final BService bService;
    public void a() { bService.b(); }
}
```

### 事件发布的事务一致性

事件应在**事务提交后**再触发副作用，避免事务回滚但邮件已发：

```java
// 发布
publisher.publishEvent(new OrderCreatedEvent(orderId));

// 监听器
@Component
public class OrderEventListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onCreated(OrderCreatedEvent event) {
        // 此时事务已提交，可安全调用外部
        notifyService.notify(event.getOrderId());
    }
}
```

## 业务校验放哪

| 校验类型              | 位置                 |
| --------------------- | -------------------- |
| 字段格式（必填、长度、邮箱） | DTO + `@Valid`（Controller） |
| 业务前置条件（状态机、唯一性、库存） | Service              |
| 并发一致性（乐观锁、版本号） | Service + Mapper（详见 mybatis-plus） |
| 数据库强约束（唯一索引、外键） | Mapper 抛错 → Service 转 BusinessException |

## 跨模块协作

### 通过 Service 接口

```java
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {
    private final UserService userService;
    private final ProductService productService;
}
```

### 避免循环依赖

如果 A.Service 与 B.Service 互相依赖，通常意味着模块边界划错。解决办法：

1. 抽公共 Service 到第三方模块
2. 一方改用领域事件解耦
3. 重新审视边界

## 异步与并发

### `@Async`

```java
@Async("ioTaskExecutor")
public void sendNotification(Long orderId) { ... }
```

注意：

- 必须在 `@SpringBootApplication` 上加 `@EnableAsync`
- 不能在同类内部调用（自调用问题）
- 异步方法的事务是独立的（不会继承调用方事务）

线程池配置见 [performance.md](./performance.md)。

### 并行批处理

```java
List<CompletableFuture<ResultVO>> futures = ids.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> processOne(id), ioExecutor))
    .toList();

List<ResultVO> results = futures.stream()
    .map(CompletableFuture::join)
    .toList();
```

## Service 内部分层

当 Service 类增大时按子职责拆分（**不是**拆为多个 Service 暴露给外部）：

```
service/order/
├── OrderService.java                       # 对外接口
├── impl/
│   ├── OrderServiceImpl.java               # 主服务
│   ├── helper/
│   │   ├── OrderPriceCalculator.java       # 价格计算（包内可见）
│   │   ├── OrderStatusMachine.java         # 状态机
│   │   └── OrderReferenceGenerator.java    # 编号生成
│   └── strategy/
│       └── DiscountStrategy.java
```

`@Component`（包级可见）即可，不必再开放为 Service 接口。

## ServiceImpl 与 MyBatis-Plus 的 IService

可继承 `ServiceImpl<OrderMapper, OrderDO>` 拿到 `getById/save/list` 等便捷方法：

```java
@Service
public class OrderServiceImpl
    extends ServiceImpl<OrderMapper, OrderDO>
    implements OrderService {
    // 直接用 this.getById、this.save、this.lambdaQuery() 等
}
```

但**注意**：

- 这些方法返回 `DO`，对外接口仍要转 `VO`
- 别让 `IService` 的方法（如 `removeById`）被 Controller 直接调（破坏封装）

## 最佳实践

1. **接口 + 实现分离**
2. **事务最小作用域**，外部调用踢出去
3. **业务异常抛 `BusinessException`**
4. **跨模块走 Service 接口**
5. **事件用 `AFTER_COMMIT`** 保证事务一致性

## 反模式

- 在 Service 中 `try/catch` 后吞掉异常返回 null
- `@Transactional` 注解写在 `private` 方法上（不生效）
- 在事务内调用第三方 HTTP 接口
- 一个 ServiceImpl 兼任 5+ 领域的职责
- 多个 Service 通过 Mapper 直接通信跨越边界
