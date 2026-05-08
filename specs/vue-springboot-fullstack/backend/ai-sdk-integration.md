# AI SDK 集成（后端）

> 适用范围：基于 **Spring AI** 的对话、流式输出、Tool Calling 与 Prompt 管理。

## 推荐依赖

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

> 通过 BOM 管理版本，避免与 Spring Boot 的传递依赖冲突。

---

## 配置

`application.yml`：

```yaml
spring:
  ai:
    openai:
      base-url: ${AI_BASE_URL:https://api.openai.com}
      api-key: ${AI_API_KEY}
      chat:
        options:
          model: ${AI_MODEL:gpt-4o-mini}
          temperature: 0.3
```

> **API Key 必须走环境变量**，禁止入库。参考 [configuration.md](./configuration.md)。

---

## ChatClient 注入

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("你是数据质量平台的智能助手，回答必须简洁、基于事实。")
            .build();
    }
}
```

业务侧：

```java
@Service
@RequiredArgsConstructor
public class AiAssistantService {

    private final ChatClient chatClient;

    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

---

## 流式响应（SSE）

Controller：

```java
@GetMapping(value = "/api/ai/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> stream(@RequestParam String q) {
    return aiAssistantService.streamAnswer(q)
        .map(chunk -> ServerSentEvent.<String>builder()
            .event("delta")
            .data(chunk)
            .build())
        .concatWith(Mono.just(ServerSentEvent.<String>builder()
            .event("done")
            .data("[DONE]")
            .build()));
}
```

Service：

```java
public Flux<String> streamAnswer(String question) {
    return chatClient.prompt()
        .user(question)
        .stream()
        .content();
}
```

> 前端消费方式见 [frontend/ai-sdk-integration.md](../frontend/ai-sdk-integration.md)。

### SSE 关键约定

| 事件名  | 用途                       |
| ------- | -------------------------- |
| `delta` | 增量文本片段               |
| `tool`  | 工具调用元数据（可选）     |
| `error` | 错误（含 code / message）  |
| `done`  | 流结束标识                 |

> **保持事件名前后端一致**，便于联调。

---

## Tool Calling（函数调用）

定义工具：

```java
@Component
public class DataQualityTools {

    @Tool(description = "查询某表近 7 天的质量评分")
    public Double getQualityScore(@ToolParam(description = "表名") String tableName) {
        // 调本地 service 取数
        return qualityService.recentScore(tableName);
    }
}
```

注册到 ChatClient：

```java
chatClient.prompt()
    .tools(dataQualityTools)
    .user("查 user_event 表最近 7 天的质量分")
    .call()
    .content();
```

### 安全要求

- Tool 方法**必须做权限检查**（与普通 Service 一视同仁）
- Tool 入参**必须做 Bean Validation**
- Tool **不允许直接执行任意 SQL**，仅暴露白名单业务方法

---

## Prompt 模板管理

将 Prompt 放到 `resources/prompts/`：

```
resources/
└── prompts/
    ├── analyze-quality.st
    └── summarize-issues.st
```

加载并渲染：

```java
@Value("classpath:/prompts/analyze-quality.st")
private Resource analyzeQualityPrompt;

public String analyze(String tableName, String metrics) {
    PromptTemplate template = new PromptTemplate(analyzeQualityPrompt);
    Prompt prompt = template.create(Map.of(
        "tableName", tableName,
        "metrics", metrics
    ));
    return chatClient.prompt(prompt).call().content();
}
```

> **禁止在 Java 代码里硬编码长 Prompt**，所有 Prompt 必须外置文件，便于运营/产品迭代。

---

## 上下文与会话

- 短对话用入参 `messages` 自带历史
- 长会话用 `ChatMemory`（Redis 实现）
- 每会话**强制截断**：超过 N 条消息或超过 token 阈值时，移除最旧的非系统消息

```java
@Bean
public ChatMemory chatMemory(StringRedisTemplate redis) {
    return new RedisChatMemory(redis, Duration.ofHours(2));
}
```

---

## 重试与超时

```yaml
spring:
  ai:
    openai:
      chat:
        options:
          temperature: 0.3
        connect-timeout: 5s
        read-timeout: 60s
```

业务侧 Spring Retry：

```java
@Retryable(
    retryFor = { TransientAiException.class },
    maxAttempts = 3,
    backoff = @Backoff(delay = 500, multiplier = 2)
)
public String robustAsk(String q) { ... }
```

> 流式接口**不重试**，避免重复发送给前端。

---

## 成本与可观测

- 每次调用记录：`model / promptTokens / completionTokens / cost`
- 通过 MDC 注入 `traceId`，便于在日志中关联用户请求 → AI 调用，参考 [logging.md](./logging.md)
- 提供管理端"用量看板"：按租户、按用户、按功能维度统计

---

## 安全红线

- **禁止**把用户原始 PII 直接拼到 Prompt（先脱敏，参考 [logging.md](./logging.md) 的 `MaskUtils`）
- **禁止**让 AI 直接生成 SQL 后无审查执行
- **禁止**把鉴权 token / API key / 内部地址放到 Prompt 上下文
- 所有 AI 接口**必须做限流**，参考 [performance.md](./performance.md)

---

## 不要做

| 反模式                                | 正确做法                              |
| ------------------------------------- | ------------------------------------- |
| Prompt 写死在代码字符串里             | 放 `resources/prompts/*.st`           |
| 流式接口边写边 flush 但没 `done` 事件 | 必须以 `done` 结束，前端才能关闭流    |
| Tool 方法不校验权限                   | 与普通 Service 一致，走 `@PreAuthorize` |
| 把 `OpenAiApi` 直接注入 Controller    | 经 `ChatClient` 与业务 Service 包装    |
