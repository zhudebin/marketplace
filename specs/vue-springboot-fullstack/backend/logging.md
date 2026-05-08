# 日志（Logback + JSON）

结构化日志输出、TraceId、MDC、敏感信息脱敏与 ELK 接入。

## 总体原则

1. **结构化优先**：JSON 格式输出到 stdout，由日志采集器（Filebeat / Loki / Fluent Bit）入 ELK / Loki
2. **全链路 TraceId**：每个请求一个 traceId，贯穿从 Controller 到 Mapper 的所有日志
3. **占位符而非字符串拼接**：`log.info("user={} action={}", userId, action)`
4. **禁用 `System.out` / `e.printStackTrace()`**

## 依赖

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

`spring-boot-starter-web` 已经包含 logback-classic，不需要再加。

## logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds">

    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>
    <springProperty scope="context" name="ACTIVE_PROFILE" source="spring.profiles.active" defaultValue="dev"/>

    <!-- ============ 控制台（开发期人类可读） ============ -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{HH:mm:ss.SSS} %highlight(%-5level) [%thread] [%X{traceId:-}] %cyan(%logger{36}) - %msg%n
            </pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- ============ JSON（生产期采集用） ============ -->
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>Asia/Shanghai</timeZone>
                    <pattern>yyyy-MM-dd HH:mm:ss.SSS</pattern>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                          "app": "${APP_NAME}",
                          "env": "${ACTIVE_PROFILE}",
                          "level": "%level",
                          "logger": "%logger{40}",
                          "thread": "%thread",
                          "message": "%message",
                          "traceId": "%X{traceId}",
                          "userId": "%X{userId}"
                        }
                    </pattern>
                </pattern>
                <stackTrace>
                    <fieldName>stack</fieldName>
                    <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                        <maxDepthPerThrowable>40</maxDepthPerThrowable>
                        <rootCauseFirst>true</rootCauseFirst>
                    </throwableConverter>
                </stackTrace>
            </providers>
        </encoder>
    </appender>

    <!-- ============ Profile 选择 ============ -->
    <springProfile name="dev">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
        <logger name="com.company.app" level="DEBUG"/>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>

</configuration>
```

> 容器化部署只输出到 stdout，由 K8s / Docker 日志驱动 + 采集器统一处理，**不要**写本地文件。

## TraceId 与 MDC

每个请求进入时生成 / 透传 traceId 并写入 MDC，使所有日志自动带上：

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class TraceIdFilter extends OncePerRequestFilter {

    public static final String TRACE_ID = "traceId";
    public static final String HEADER = "X-Trace-Id";

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String traceId = req.getHeader(HEADER);
        if (!StringUtils.hasText(traceId)) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }
        MDC.put(TRACE_ID, traceId);
        res.setHeader(HEADER, traceId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove(TRACE_ID);
        }
    }
}
```

登录后写入 userId（在 JwtAuthenticationFilter 之后）：

```java
MDC.put("userId", String.valueOf(userId));
```

> 子线程不会自动继承 MDC，使用线程池时需要手动透传（见 [performance.md](./performance.md) 异步小节）。

## 日志规范

### 占位符

```java
// 推荐
log.info("订单创建 orderId={} userId={} total={}", orderId, userId, total);

// 反模式
log.info("订单创建 orderId=" + orderId);                  // 字符串拼接，无法被 JSON 结构化
log.info(String.format("orderId=%s", orderId));           // 同上
```

### 异常打印

```java
// 推荐：最后一个参数是 throwable
log.error("订单处理失败 orderId={}", orderId, ex);

// 反模式：只打 message 丢堆栈
log.error("订单处理失败 orderId={} msg={}", orderId, ex.getMessage());

// 反模式：堆栈打印到 stdout
ex.printStackTrace();
```

### 日志级别

| 级别  | 用途                                              |
| ----- | ------------------------------------------------- |
| ERROR | 系统异常、未捕获异常、需要人工介入                |
| WARN  | 业务异常、可恢复的失败、降级触发                  |
| INFO  | 关键业务节点（订单创建、支付完成、登录）          |
| DEBUG | 详细诊断信息，仅本地/灰度开启                     |
| TRACE | 极详细调试，几乎不用                              |

### 不要输出无意义日志

```java
// 反模式
log.info("进入 createOrder 方法");
log.info("查询用户：{}", userId);
log.info("查询用户结果：{}", user);
log.info("退出 createOrder 方法");
```

INFO 应该是"业务事件"，不是"代码流程"。代码流程用 DEBUG。

## 敏感信息脱敏

### 必须脱敏的字段

- 密码 / token / refreshToken / API Key
- 手机号 / 身份证 / 银行卡
- email（部分场景）

### 方案 A：Jackson 注解

```java
public class UserVO {
    @JsonSerialize(using = MaskedSerializer.class)
    private String phone;
}
```

### 方案 B：Logback PatternLayout 全局过滤

```xml
<pattern>...{token=)([\w.\-]+),"$1***"}...</pattern>
```

### 方案 C：参数构造前主动脱敏

```java
log.info("登录成功 user={} phoneSuffix={}", user.getUsername(), maskPhone(user.getPhone()));
```

> 不论方案，**绝不打印密码原文 / token 全文**。

## 接口请求日志

可选：用 AOP / 过滤器记录所有 API 入参出参（含耗时）：

```java
@Aspect
@Component
@Slf4j
public class ApiLoggingAspect {

    @Around("execution(* com.company.app.module..controller..*(..))")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        String method = pjp.getSignature().toShortString();
        try {
            Object result = pjp.proceed();
            long cost = System.currentTimeMillis() - start;
            log.info("API ok method={} cost={}ms", method, cost);
            return result;
        } catch (Throwable ex) {
            long cost = System.currentTimeMillis() - start;
            log.warn("API err method={} cost={}ms err={}", method, cost, ex.getMessage());
            throw ex;
        }
    }
}
```

> 慎重输出 body：含敏感数据时建议只打字段名 + 长度。

## SQL 日志

开发期开启 MyBatis-Plus + p6spy 输出真实 SQL 与耗时；生产关闭：

```yaml
logging:
  level:
    com.company.app.module.**.mapper: ${SQL_LOG_LEVEL:info}
```

```bash
SQL_LOG_LEVEL=debug pnpm start:dev   # 仅开发
```

## 异步任务的 MDC 透传

线程池切换会丢 MDC，需要包装：

```java
@Bean("ioTaskExecutor")
public ThreadPoolTaskExecutor ioTaskExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(8);
    exec.setMaxPoolSize(32);
    exec.setQueueCapacity(200);
    exec.setThreadNamePrefix("io-");
    exec.setTaskDecorator(new MdcTaskDecorator());
    exec.initialize();
    return exec;
}

public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> ctx = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (ctx != null) MDC.setContextMap(ctx);
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

## 慢日志

```java
long start = System.currentTimeMillis();
// 业务...
long cost = System.currentTimeMillis() - start;
if (cost > 1000) {
    log.warn("慢调用 method=createOrder cost={}ms userId={}", cost, userId);
}
```

考虑封装为 `@Slow` 注解，统一阈值与监控。

## 与前端联动

响应头返回 `X-Trace-Id`，前端在错误提示中带上：

```
请求失败：订单不存在（traceId=abc123）
```

便于客服 / 运维定位。前端 axios 实例已示例（详见 [frontend/api-integration.md](../frontend/api-integration.md)）。

## 最佳实践

1. **JSON 输出 + stdout**
2. **TraceId 全链路**
3. **占位符 + 末尾 throwable**
4. **业务事件 INFO**，代码流程 DEBUG
5. **敏感信息脱敏**
6. **线程池透传 MDC**

## 反模式

- `System.out.println` / `e.printStackTrace()`
- 字符串拼接 / `String.format` 拼接日志
- 业务异常打 ERROR
- 把整个 DTO/请求体打印到日志
- 写本地文件而非 stdout（容器场景）
- 没有 traceId，事故时无法定位单次请求
