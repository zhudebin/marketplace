# Spring Boot 3 / Jakarta 迁移

| 项       | 值          |
| -------- | ----------- |
| 分类     | 升级 / 兼容 |
| 严重等级 | Warning     |

## 问题描述

从 Spring Boot 2.x 升 3.x（或新项目随手抄旧代码）时遇到：

1. 一上来一片红：`package javax.servlet does not exist`
2. `import javax.validation.*` 找不到
3. `@Validated` 还在但 `javax.validation.Valid` 编译不过
4. springfox `Swagger 2/3` 注解满天报错（与 SB3 不兼容）
5. JPA 的 `javax.persistence.*` 全部失效
6. `WebSecurityConfigurerAdapter` 已经被删除，老 SecurityConfig 写法编译不过
7. 第三方库（旧版 mybatis-plus / jjwt / spring-cloud）启动报 `NoClassDefFoundError: javax/servlet/http/HttpServletRequest`

## 根因

- **Spring Boot 3 把命名空间从 `javax.*` 切到 `jakarta.*`**（Jakarta EE 9+）
- **Java 17 是最低基线**
- Spring Security 6 移除了 `WebSecurityConfigurerAdapter`，改成函数式配置
- 旧库（仍在 `javax`）**会运行时找不到类**，必须升级到适配 Jakarta 的版本
- springfox 已停止维护，Swagger 必须迁移到 springdoc-openapi

## 解决方案

### 1. JDK & 构建

```xml
<properties>
  <java.version>17</java.version>
  <maven.compiler.release>17</maven.compiler.release>
</properties>
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.2.0</version>
</parent>
```

### 2. 命名空间替换

| 旧（javax）                          | 新（jakarta）                          |
| ------------------------------------ | -------------------------------------- |
| `javax.servlet.*`                    | `jakarta.servlet.*`                    |
| `javax.servlet.http.HttpServletRequest` | `jakarta.servlet.http.HttpServletRequest` |
| `javax.validation.Valid`             | `jakarta.validation.Valid`             |
| `javax.validation.constraints.*`     | `jakarta.validation.constraints.*`     |
| `javax.persistence.*`                | `jakarta.persistence.*`                |
| `javax.annotation.PostConstruct`     | `jakarta.annotation.PostConstruct`     |

> 一键替换（IDEA 全局 Replace in Path）：
> `javax.servlet` → `jakarta.servlet`
> `javax.validation` → `jakarta.validation`
> `javax.persistence` → `jakarta.persistence`
> `javax.annotation` → `jakarta.annotation`

### 3. 第三方库版本

| 库                  | 适配 SB3 的版本                |
| ------------------- | ------------------------------ |
| MyBatis-Plus        | `mybatis-plus-spring-boot3-starter` ≥ 3.5.5 |
| JJWT                | ≥ 0.12.x                       |
| SpringDoc           | `springdoc-openapi-starter-webmvc-ui` ≥ 2.x |
| Druid               | `druid-spring-boot-3-starter`  |
| Lombok              | ≥ 1.18.30                      |
| MapStruct           | ≥ 1.5.5                        |
| Logstash Encoder    | ≥ 7.4                          |

> **不要用 SB2 时代的 starter 包**（如 `mybatis-plus-boot-starter`），否则启动报 `javax/servlet`。

### 4. Spring Security 6 改写

旧（SB2）：

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/public/**").permitAll()
            .anyRequest().authenticated();
    }
}
```

新（SB3 / Security 6）：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

> 注意 `antMatchers` → `requestMatchers`，`authorizeRequests` → `authorizeHttpRequests`。

### 5. Swagger 迁移

把 `springfox-swagger2 / springfox-boot-starter` 全部移除，换：

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.5.0</version>
</dependency>
```

注解：

| 旧（springfox）       | 新（springdoc / OpenAPI 3） |
| --------------------- | --------------------------- |
| `@Api`                | `@Tag`                      |
| `@ApiOperation`       | `@Operation`                |
| `@ApiParam`           | `@Parameter`                |
| `@ApiModel`           | `@Schema`                   |
| `@ApiModelProperty`   | `@Schema`                   |

UI 路径：`/swagger-ui/index.html` 或自定义 `springdoc.swagger-ui.path`。

### 6. 配置项变化（节选）

| 旧                                    | 新                              |
| ------------------------------------- | ------------------------------- |
| `server.servlet.session.cookie.same-site` 不受限 | 同左，但配合 SameSite=None 必须 secure=true |
| `spring.profiles.active` 多 profile 用逗号 | 同上，但 `spring.profiles.include` 行为变了 |
| `spring.flyway.enabled=true` 默认 true | 同上，但 driver 路径变 |

### 7. 运行期排错

启动报 `NoClassDefFoundError: javax/servlet/http/HttpServletRequest`：

```bash
# 找出哪个依赖还引了 javax.servlet
./mvnw dependency:tree | grep javax.servlet
```

90% 是某个库版本太老，找它的 SB3/Jakarta 适配版替换掉。

## 关键启示

- **包名替换是机械的，但漏一个就启动失败**——用 IDE 全局替换 + 编译期检查兜底
- **依赖版本必须全部升级**，混用必炸
- **Security 6 写法已经函数式化**，老的 Adapter 不能再用
- **springfox 已死**，遇到必迁 springdoc

## 自检清单

- [ ] JDK ≥ 17
- [ ] 所有 `javax.*` 已替换为 `jakarta.*`
- [ ] `mybatis-plus` 等核心库都是 SB3/Jakarta 适配版
- [ ] SecurityConfig 已重写为函数式
- [ ] 移除 springfox，改用 springdoc-openapi
- [ ] `dependency:tree` 中没有遗留的 `javax.servlet`
- [ ] CI 跑通完整集成测试
