# CORS 与 Cookie SameSite

| 项       | 值              |
| -------- | --------------- |
| 分类     | 跨域 / Cookie   |
| 严重等级 | Warning         |

## 问题描述

前后端分离开发，部署在不同域名下，常见错误：

1. 浏览器报 `CORS policy: No 'Access-Control-Allow-Origin' header`
2. CORS 预检请求 `OPTIONS` 返回 401（被 Spring Security 拦了）
3. 配置了 `allowedOrigins: "*"` + `allowCredentials: true`，浏览器报 `The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '*'`
4. refresh token 走 Cookie，跨域时 Cookie **不发送**
5. 本地 `localhost:5173` 调远程接口，Cookie 莫名其妙没生效
6. iOS Safari 一切正常，Chrome 报错（或反之）
7. 上线后接口正常，但 Cookie SameSite=None 在 Safari 上死活不存

## 根因

- **CORS 预检 (preflight)** 是 `OPTIONS` 请求，**必须放行**且不需要鉴权
- Spring Security 在 CORS 之前生效时会先返回 401
- `allowedOrigins: "*"` 与 `allowCredentials: true` 互斥（W3C 规定）
- **Cookie 跨域必须满足三件套**：`SameSite=None; Secure; Partitioned`（最新 Chrome）
- `Secure` 要求 HTTPS，本地 `http://localhost` 是例外
- Safari 对第三方 Cookie 限制最严，必须严格满足规范才存

## 解决方案

### 1. Spring Boot 端 CORS 配置

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        // ⚠ allowCredentials=true 时，Origin 不能是 *，必须明确列出
        config.setAllowedOriginPatterns(List.of(
            "http://localhost:5173",
            "https://app.example.com",
            "https://*.example.com"   // 通配子域可用 OriginPatterns
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setExposedHeaders(List.of("Authorization", "X-Trace-Id"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

> **关键点**：`setAllowedOrigins` 不支持通配，必须用 `setAllowedOriginPatterns`。

### 2. Spring Security 必须把 CORS 放在最前

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .cors(cors -> {})              // 启用上面的 CorsConfigurationSource
        .csrf(AbstractHttpConfigurer::disable)
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()  // 预检放行
            .requestMatchers("/api/auth/**").permitAll()
            .anyRequest().authenticated()
        )
        .build();
}
```

> 不显式放行 `OPTIONS`，预检请求**会被** JWT 过滤器拦截返回 401，CORS 直接挂。

### 3. 前端 axios 携带 Cookie

```typescript
const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  withCredentials: true,   // ← 必须
});
```

### 4. Cookie 设置三件套

后端写 cookie：

```java
ResponseCookie cookie = ResponseCookie.from("refresh_token", token)
    .httpOnly(true)
    .secure(true)             // 必须 HTTPS（本地 localhost 例外）
    .sameSite("None")         // 跨站必须
    .path("/")
    .maxAge(Duration.ofDays(30))
    .build();
response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
```

### 5. 同源 vs 跨站 vs 跨域

| 场景                         | 是否同源 | SameSite 要求       |
| ---------------------------- | -------- | ------------------- |
| 前后端同域名同端口           | 同源     | `Lax` 即可          |
| 前后端不同子域（同 SLD）     | 跨站     | `Lax` 通常可        |
| 前后端不同主域               | 跨站     | `None; Secure`      |
| 浏览器扩展 / 不同站跳转      | 跨站     | `None; Secure`      |

> **能合并到同域就合并**（前端走 nginx 反代到后端 `/api/*`），最省心。

### 6. 推荐部署方案

```
              ┌─────────────────────┐
              │   nginx / 网关       │
   ┌──────────►  /         → 前端    │
   │          │  /api/*    → 后端    │
   │          └─────────────────────┘
浏览器
   │
   └─ 与浏览器的所有交互都同域 → 没有 CORS、没有 SameSite 烦恼
```

> 这是最稳的部署方式，**生产强烈推荐**。CORS 配置主要为开发期的 vite dev server。

### 7. 开发期：vite proxy 取代 CORS

`vite.config.ts`：

```typescript
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
    }
  }
}
```

前端调 `/api/users` 直接由 vite 代理到后端，**浏览器视角下是同源**，无需任何 CORS 配置。**强烈推荐这种方式**。

### 8. 排错顺序

CORS 出问题时按顺序检查：

1. 后端返回了 `Access-Control-Allow-Origin` 吗？
2. `allowCredentials=true` 时，`Allow-Origin` 是不是具体 origin（不是 `*`）？
3. 预检 `OPTIONS` 是不是 200？是不是被 Security 拦了 401？
4. 前端 `withCredentials` 设了吗？
5. Cookie 是 `SameSite=None; Secure` 吗？
6. 是不是 HTTPS（或 localhost）？
7. 浏览器是不是隐身模式 / 第三方 Cookie 被禁？

## 关键启示

- **CORS 配置 + Security 配置 + Cookie 属性**，三者要协同，缺一就崩
- `allowedOrigins: "*"` 与 Cookie 不共存
- 生产**优先用反代消除 CORS**，比配置 CORS 更稳
- 开发用 vite proxy，避开 CORS 而不是与之搏斗
- Safari 对 Cookie 最严格，**以 Safari 为兼容基线**

## 自检清单

- [ ] CORS 用 `OriginPatterns` 不用 `*`
- [ ] `allowCredentials=true`
- [ ] Security 显式放行 `OPTIONS`
- [ ] 前端 axios `withCredentials: true`
- [ ] Cookie `SameSite=None; Secure; HttpOnly`
- [ ] 生产是否可改为同域反代
- [ ] 开发是否使用 vite proxy
- [ ] Safari 上验证过登录流程
