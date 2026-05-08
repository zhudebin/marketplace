# 性能：缓存、并发、批处理

Redis 缓存（含 `@Cacheable`）、线程池、异步、批处理与限流。

## Redis 配置

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      database: 0
      timeout: 3000
      lettuce:
        pool:
          max-active: 32
          max-idle: 8
          min-idle: 2
```

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory cf, ObjectMapper om) {
        RedisTemplate<String, Object> tpl = new RedisTemplate<>();
        tpl.setConnectionFactory(cf);

        StringRedisSerializer keySer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer valSer = new GenericJackson2JsonRedisSerializer(om);

        tpl.setKeySerializer(keySer);
        tpl.setHashKeySerializer(keySer);
        tpl.setValueSerializer(valSer);
        tpl.setHashValueSerializer(valSer);
        tpl.afterPropertiesSet();
        return tpl;
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory cf, ObjectMapper om) {
        RedisCacheConfiguration base = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer(om)));

        Map<String, RedisCacheConfiguration> custom = Map.of(
            "user", base.entryTtl(Duration.ofMinutes(30)),
            "userAuthorities", base.entryTtl(Duration.ofMinutes(5)),
            "config", base.entryTtl(Duration.ofHours(2))
        );

        return RedisCacheManager.builder(cf)
            .cacheDefaults(base)
            .withInitialCacheConfigurations(custom)
            .build();
    }
}
```

## 缓存使用

### 注解式（适合简单 K-V）

```java
@Cacheable(cacheNames = "user", key = "#id", unless = "#result == null")
public UserVO findById(Long id) {
    UserDO entity = userMapper.selectById(id);
    return userConverter.toVO(entity);
}

@CacheEvict(cacheNames = "user", key = "#dto.id")
public void update(UserUpdateDTO dto) { ... }

@CacheEvict(cacheNames = "user", allEntries = true)
public void clearAll() { ... }
```

### 手动缓存（适合复杂场景：批量、TTL 抖动、空值穿透）

```java
@Service
@RequiredArgsConstructor
public class ProductCache {
    private final StringRedisTemplate redis;
    private final ProductMapper productMapper;
    private final ObjectMapper om;

    private static final String KEY = "product:%d";
    private static final Duration TTL = Duration.ofMinutes(10);

    public ProductVO get(Long id) throws JsonProcessingException {
        String key = KEY.formatted(id);
        String json = redis.opsForValue().get(key);
        if (json != null) {
            if ("__NULL__".equals(json)) return null;
            return om.readValue(json, ProductVO.class);
        }
        ProductDO entity = productMapper.selectById(id);
        if (entity == null) {
            // 空值缓存防穿透，TTL 短一些
            redis.opsForValue().set(key, "__NULL__", Duration.ofSeconds(30));
            return null;
        }
        ProductVO vo = ProductConverter.INSTANCE.toVO(entity);
        // TTL 抖动防雪崩
        long ttl = TTL.toSeconds() + ThreadLocalRandom.current().nextLong(60);
        redis.opsForValue().set(key, om.writeValueAsString(vo), Duration.ofSeconds(ttl));
        return vo;
    }
}
```

## 缓存防三害

| 问题   | 现象                                | 应对                                |
| ------ | ----------------------------------- | ----------------------------------- |
| 穿透   | 查不存在的 key，每次打 DB           | 空值缓存（短 TTL）+ 布隆过滤器      |
| 击穿   | 热点 key 过期瞬间海量并发           | 分布式锁 + 双检；或永不过期 + 后台刷新 |
| 雪崩   | 大量 key 同时过期                   | TTL 抖动；多级缓存                  |

### 双检锁（缓存击穿）

```java
public ProductVO get(Long id) {
    String key = "product:" + id;
    String lockKey = "lock:product:" + id;

    String json = redis.opsForValue().get(key);
    if (json != null) return parse(json);

    Boolean got = redis.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(10));
    if (Boolean.TRUE.equals(got)) {
        try {
            // 二次检查
            json = redis.opsForValue().get(key);
            if (json != null) return parse(json);
            // 加载并写入
            ProductVO vo = loadFromDb(id);
            redis.opsForValue().set(key, toJson(vo), TTL);
            return vo;
        } finally {
            redis.delete(lockKey);
        }
    } else {
        sleep(50);
        return get(id);  // 重试一次
    }
}
```

## 异步与线程池

### 启用异步

```java
@SpringBootApplication
@EnableAsync
public class Application { ... }
```

### 线程池配置

```java
@Configuration
public class AsyncConfig {

    @Bean("ioTaskExecutor")
    public ThreadPoolTaskExecutor ioTaskExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(8);
        exec.setMaxPoolSize(32);
        exec.setQueueCapacity(200);
        exec.setKeepAliveSeconds(60);
        exec.setThreadNamePrefix("io-");
        exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        exec.setTaskDecorator(new MdcTaskDecorator());   // 透传 MDC
        exec.setWaitForTasksToCompleteOnShutdown(true);
        exec.setAwaitTerminationSeconds(30);
        exec.initialize();
        return exec;
    }

    @Bean("cpuTaskExecutor")
    public ThreadPoolTaskExecutor cpuTaskExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        int cores = Runtime.getRuntime().availableProcessors();
        exec.setCorePoolSize(cores);
        exec.setMaxPoolSize(cores);
        exec.setQueueCapacity(500);
        exec.setThreadNamePrefix("cpu-");
        exec.initialize();
        return exec;
    }
}
```

> **CallerRunsPolicy**：拒绝策略选择回退到调用方执行，比 `AbortPolicy` 更安全（不会丢任务），但要注意调用方阻塞。

### `@Async` 用法

```java
@Async("ioTaskExecutor")
public CompletableFuture<Void> sendNotification(Long orderId) {
    notifyClient.send(orderId);
    return CompletableFuture.completedFuture(null);
}
```

> 注意：
>
> - 不能在同类内部 `this.sendNotification()` 调用（绕过代理）
> - 异步方法的事务独立于调用方
> - 必须显式指定线程池名（避免使用默认 ForkJoinPool）

### 并行调用

```java
public OrderDetailVO detail(Long id) {
    OrderDO order = orderMapper.selectById(id);
    if (order == null) throw new BusinessException(ErrorCode.ORDER_NOT_FOUND);

    CompletableFuture<UserVO> userF = CompletableFuture.supplyAsync(
        () -> userService.findById(order.getUserId()), ioTaskExecutor);
    CompletableFuture<List<OrderItemVO>> itemsF = CompletableFuture.supplyAsync(
        () -> orderItemService.listByOrderId(order.getId()), ioTaskExecutor);

    return new OrderDetailVO(orderConverter.toVO(order), userF.join(), itemsF.join());
}
```

## 批处理

### 批量数据库写

```java
// 推荐：单条 SQL 多 VALUES（结合 rewriteBatchedStatements=true）
public void batchInsert(List<OrderItemDO> items) {
    Lists.partition(items, 500).forEach(orderItemMapper::insertBatch);
}
```

### 批量任务

```java
public void processOrders(List<Long> ids) {
    int chunkSize = 100;
    for (List<Long> chunk : Lists.partition(ids, chunkSize)) {
        List<CompletableFuture<Void>> futures = chunk.stream()
            .map(id -> CompletableFuture.runAsync(() -> processOne(id), ioTaskExecutor))
            .toList();
        CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new)).join();
    }
}
```

### Semaphore 限并发

```java
private final Semaphore sem = new Semaphore(10);

public void callExternal() throws InterruptedException {
    sem.acquire();
    try {
        externalApi.call();
    } finally {
        sem.release();
    }
}
```

## 限流

### 单机限流（Bucket4j）

```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.10.1</version>
</dependency>
```

```java
@Component
public class RateLimiter {
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    public boolean tryConsume(String key, int capacity, Duration period) {
        Bucket bucket = buckets.computeIfAbsent(key, k -> Bucket.builder()
            .addLimit(b -> b.capacity(capacity).refillIntervally(capacity, period))
            .build());
        return bucket.tryConsume(1);
    }
}
```

### 分布式限流（Redis + Lua）

通过 `Redisson` 的 `RRateLimiter` 或自写 Lua 脚本。生产环境推荐网关层（Nginx / Spring Cloud Gateway）做。

## 重试

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

```java
@EnableRetry
@SpringBootApplication
public class Application { ... }
```

```java
@Retryable(
    retryFor = {ExternalApiException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 200, multiplier = 2.0, maxDelay = 2000)
)
public PaymentResult charge(...) { ... }

@Recover
public PaymentResult chargeFallback(ExternalApiException e, Long orderId) {
    log.error("支付重试耗尽 orderId={}", orderId, e);
    throw new BusinessException(ErrorCode.PAYMENT_FAILED);
}
```

> **注意**：仅幂等接口可重试；写入类必须幂等设计（外部 token / orderId 去重）。

## 分页深分页问题

`LIMIT 1000000, 20` 性能急剧下降。深分页用游标：

```java
// 反模式：跳页深分页
SELECT * FROM t_order LIMIT 1000000, 20;

// 推荐：游标分页（前端带上一页最后 id）
SELECT * FROM t_order WHERE id > #{lastId} ORDER BY id LIMIT 20;
```

> 业务需要"跳到第 N 页"功能时（管理后台）才用 OFFSET，并加上 hard limit（最多 1000 页）。

## HTTP 客户端

调用第三方 API：

- 必须设连接超时与读超时（推荐 3s 连接、10s 读）
- 必须设连接池上限
- 必须有重试策略（针对幂等接口）
- 必须有降级（熔断）

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public RestClient restClient() {
        return RestClient.builder()
            .requestFactory(new SimpleClientHttpRequestFactory() {{
                setConnectTimeout((int) Duration.ofSeconds(3).toMillis());
                setReadTimeout((int) Duration.ofSeconds(10).toMillis());
            }})
            .build();
    }
}
```

## 监控指标

启用 Spring Boot Actuator + Micrometer：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  endpoint:
    health:
      show-details: when_authorized
```

关键指标：

- HTTP 接口 RT / QPS / 错误率（自动采集）
- 数据库连接池（HikariCP 自动）
- 线程池利用率（自定义）
- Redis 命中率（自定义）

## 最佳实践

1. **缓存有 TTL + 抖动**
2. **空值缓存防穿透**
3. **线程池显式命名 + MDC 透传**
4. **批处理分块 + 并行**
5. **重试只对幂等接口**
6. **深分页用游标**

## 反模式

- 用默认 ForkJoinPool 跑业务（不可控）
- `@Cacheable` 不设 unless 导致缓存 null
- 缓存与 DB 双写不一致（应"先更 DB 再删缓存"或读穿写穿）
- 同步链调用 N 个外部接口（应并行）
- 不分块的批量任务一次跑十万条（OOM 风险）
- 远程调用不设超时（拖垮整个服务）
