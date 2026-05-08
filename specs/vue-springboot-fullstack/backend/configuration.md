# 配置管理

`application.yml` 多环境分层、外部化配置、敏感信息处理与 `@ConfigurationProperties`。

## 分层结构

```
src/main/resources/
├── application.yml                  # 通用配置（所有环境）
├── application-dev.yml              # 开发环境
├── application-test.yml             # 测试环境
├── application-prod.yml             # 生产环境
└── application-local.yml            # 本地个人覆盖（git ignore）
```

## 通用配置

```yaml
# application.yml
spring:
  application:
    name: myapp-backend

  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    default-property-inclusion: non_null
    deserialization:
      fail-on-unknown-properties: false
    serialization:
      write-dates-as-timestamps: false

  servlet:
    multipart:
      max-file-size: 20MB
      max-request-size: 50MB

  mvc:
    throw-exception-if-no-handler-found: true

  web:
    resources:
      add-mappings: false               # 关闭静态资源默认映射，避免 404 被吞

mybatis-plus:
  mapper-locations: classpath*:mapper/**/*.xml
  configuration:
    map-underscore-to-camel-case: true
  global-config:
    db-config:
      id-type: auto
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0

server:
  port: 8080
  compression:
    enabled: true
    mime-types: application/json, text/html, text/css, application/javascript
    min-response-size: 2048
  shutdown: graceful

management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  endpoint:
    health:
      show-details: when_authorized
  metrics:
    tags:
      application: ${spring.application.name}

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    enabled: ${SWAGGER_ENABLED:true}

app:
  security:
    jwt:
      secret: ${JWT_SECRET}
      access-token-ttl-minutes: 30
      refresh-token-ttl-days: 7
```

## 开发环境

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp_dev?useSSL=false&characterEncoding=utf8mb4&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    username: root
    password: root
    hikari:
      maximum-pool-size: 10

  data:
    redis:
      host: localhost
      port: 6379

  flyway:
    locations: classpath:db/migration/common,classpath:db/migration/dev

logging:
  level:
    com.company.app: debug
    com.company.app.module.**.mapper: debug

app:
  cors:
    allowed-origins: http://localhost:5173, http://localhost:3000
```

## 生产环境

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:3306/${DB_NAME}?useSSL=true&characterEncoding=utf8mb4&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 30
      minimum-idle: 5

  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}

  flyway:
    locations: classpath:db/migration/common,classpath:db/migration/prod

logging:
  level:
    root: info

springdoc:
  swagger-ui:
    enabled: false

app:
  cors:
    allowed-origins: https://app.example.com
```

## 敏感信息

| 类型                    | 位置                                |
| ----------------------- | ----------------------------------- |
| 数据库密码              | 环境变量 `DB_PASSWORD`              |
| Redis 密码              | 环境变量 `REDIS_PASSWORD`           |
| JWT secret              | 环境变量 `JWT_SECRET`               |
| 第三方 API key          | 环境变量 `XXX_API_KEY`              |
| OAuth client secret     | 环境变量                            |

> **永不**提交敏感信息到 git。`application-local.yml` 加入 `.gitignore`。

`application.yml` 中：

```yaml
spring:
  datasource:
    password: ${DB_PASSWORD}            # 必须存在
    # password: ${DB_PASSWORD:default}  # 仅开发可设默认值
```

> 生产部署的 `JWT_SECRET` 必须 ≥ 32 字节随机字符串：

```bash
openssl rand -base64 64
```

## 启动方式

```bash
# 本地开发
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# Docker
docker run -e SPRING_PROFILES_ACTIVE=prod -e DB_PASSWORD=*** myapp:latest

# Kubernetes（推荐用 ConfigMap + Secret）
```

## `@ConfigurationProperties` 类型化配置

```java
@Data
@ConfigurationProperties(prefix = "app.security.jwt")
public class JwtProperties {
    private String secret;
    private long accessTokenTtlMinutes = 30;
    private long refreshTokenTtlDays = 7;
}
```

```java
@SpringBootApplication
@EnableConfigurationProperties(JwtProperties.class)
public class Application { ... }
```

```java
@Component
@RequiredArgsConstructor
public class JwtUtils {
    private final JwtProperties props;
    // 用 props.getAccessTokenTtlMinutes() 等
}
```

> 优先用 `@ConfigurationProperties` 取代散落的 `@Value`，便于 IDE 提示与单元测试。

## 配置验证

```java
@Validated
@Data
@ConfigurationProperties(prefix = "app.security.jwt")
public class JwtProperties {
    @NotBlank
    private String secret;

    @Min(5) @Max(1440)
    private long accessTokenTtlMinutes = 30;

    @Min(1) @Max(90)
    private long refreshTokenTtlDays = 7;
}
```

启动时校验失败直接报错，避免运行到一半才发现配置错。

## OpenAPI / Swagger

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("MyApp API")
                .version("v1")
                .description("MyApp 后端 API 文档"))
            .components(new Components()
                .addSecuritySchemes("bearer", new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")))
            .addSecurityItem(new SecurityRequirement().addList("bearer"));
    }
}
```

访问：

- 文档 JSON：`http://localhost:8080/v3/api-docs`
- Swagger UI：`http://localhost:8080/swagger-ui.html`

> **生产环境必须关闭 Swagger UI**：`springdoc.swagger-ui.enabled=false`，但保留 `/v3/api-docs` 用于前端类型生成（受 IP 白名单或 internal 网络保护）。

## CORS 配置外部化

```yaml
app:
  cors:
    allowed-origins: ${ALLOWED_ORIGINS}
    allowed-methods: GET,POST,PUT,PATCH,DELETE,OPTIONS
    allowed-headers: "*"
    max-age: 3600
```

```java
@Data
@ConfigurationProperties(prefix = "app.cors")
public class CorsProperties {
    private List<String> allowedOrigins = new ArrayList<>();
    private List<String> allowedMethods = new ArrayList<>();
    private List<String> allowedHeaders = new ArrayList<>();
    private long maxAge = 3600;
}

@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {
    private final CorsProperties cors;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOriginPatterns(cors.getAllowedOrigins().toArray(String[]::new))
            .allowedMethods(cors.getAllowedMethods().toArray(String[]::new))
            .allowedHeaders(cors.getAllowedHeaders().toArray(String[]::new))
            .maxAge(cors.getMaxAge());
    }
}
```

## 健康检查

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        readiness:
          include: db, redis, diskSpace
        liveness:
          include: ping
```

K8s 配置 `/actuator/health/liveness` 和 `/actuator/health/readiness` 即可。

## 应用版本与构建信息

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

`/actuator/info` 返回版本号 + 构建时间 + git commit。

## 配置变更流程

| 类型           | 流程                                           |
| -------------- | ---------------------------------------------- |
| 业务配置（功能开关） | yml 中默认值 + 启动时读 → 重启生效              |
| 敏感信息       | 环境变量 / K8s Secret，不入库                  |
| 频繁变更的配置 | 考虑接入配置中心（Nacos/Apollo），本规范默认不用 |

## 反模式

- 把 `application-prod.yml` 中的密码写明文
- 用 `@Value` 串联十几个配置（用 `@ConfigurationProperties`）
- 不区分 profile，所有配置写一个文件
- 生产开 Swagger UI
- 默认值放在生产环境 yml 而不是公共 yml
- 配置文件里写 `127.0.0.1`（应外部化）
