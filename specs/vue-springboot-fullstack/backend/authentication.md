# 鉴权（Spring Security + JWT）

JWT 签发与校验、过滤器链、`@PreAuthorize` 权限注解、登录态缓存与登出黑名单。

## 总体架构

```
HTTP 请求
    ↓
[CorsFilter]                       全局 CORS
    ↓
[JwtAuthenticationFilter]          解析 Authorization → 校验 token → 写入 SecurityContext
    ↓
[FilterSecurityInterceptor]        基于 SecurityContext + @PreAuthorize 鉴权
    ↓
Controller
```

## 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JJWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

## SecurityConfig

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;
    private final RestAuthenticationEntryPoint authEntryPoint;
    private final RestAccessDeniedHandler accessDeniedHandler;

    private static final String[] PUBLIC_PATHS = {
        "/api/auth/login",
        "/api/auth/refresh",
        "/api/public/**",
        "/v3/api-docs/**",
        "/swagger-ui/**",
        "/swagger-ui.html",
        "/actuator/health",
    };

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(PUBLIC_PATHS).permitAll()
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                .anyRequest().authenticated()
            )
            .exceptionHandling(e -> e
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration cfg) throws Exception {
        return cfg.getAuthenticationManager();
    }
}
```

## JWT 工具

```java
@Component
@RequiredArgsConstructor
public class JwtUtils {

    @Value("${app.security.jwt.secret}")
    private String secret;

    @Value("${app.security.jwt.access-token-ttl-minutes:30}")
    private long accessTtlMinutes;

    @Value("${app.security.jwt.refresh-token-ttl-days:7}")
    private long refreshTtlDays;

    private SecretKey key;

    @PostConstruct
    void init() {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    public String generateAccessToken(Long userId, Collection<String> authorities) {
        Instant now = Instant.now();
        return Jwts.builder()
            .subject(userId.toString())
            .claim("type", "access")
            .claim("authorities", authorities)
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plus(accessTtlMinutes, ChronoUnit.MINUTES)))
            .signWith(key, Jwts.SIG.HS256)
            .compact();
    }

    public String generateRefreshToken(Long userId) {
        Instant now = Instant.now();
        return Jwts.builder()
            .subject(userId.toString())
            .claim("type", "refresh")
            .id(UUID.randomUUID().toString())
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plus(refreshTtlDays, ChronoUnit.DAYS)))
            .signWith(key, Jwts.SIG.HS256)
            .compact();
    }

    public Claims parse(String token) {
        return Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}
```

```yaml
app:
  security:
    jwt:
      secret: ${JWT_SECRET}                  # 生产环境必须 ≥ 32 字节并通过环境变量注入
      access-token-ttl-minutes: 30
      refresh-token-ttl-days: 7
```

## JWT 过滤器

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtils jwtUtils;
    private final TokenBlacklistService blacklist;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String token = resolveToken(req);

        if (token != null) {
            try {
                if (blacklist.isBlacklisted(token)) {
                    throw new JwtException("token revoked");
                }
                Claims claims = jwtUtils.parse(token);
                if (!"access".equals(claims.get("type"))) {
                    throw new JwtException("not access token");
                }
                Long userId = Long.parseLong(claims.getSubject());
                @SuppressWarnings("unchecked")
                List<String> auths = (List<String>) claims.get("authorities");
                List<SimpleGrantedAuthority> authorities = auths == null ? List.of()
                    : auths.stream().map(SimpleGrantedAuthority::new).toList();

                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(userId, null, authorities);
                auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException ex) {
                SecurityContextHolder.clearContext();
                log.debug("JWT 校验失败: {}", ex.getMessage());
                // 不直接 sendError，让后续 EntryPoint 统一处理
            }
        }
        chain.doFilter(req, res);
    }

    private String resolveToken(HttpServletRequest req) {
        String h = req.getHeader(HttpHeaders.AUTHORIZATION);
        if (h != null && h.startsWith("Bearer ")) {
            return h.substring(7);
        }
        return null;
    }
}
```

## 登录接口

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public Result<TokenVO> login(@Valid @RequestBody LoginDTO dto) {
        return Result.success(authService.login(dto));
    }

    @PostMapping("/refresh")
    public Result<TokenVO> refresh(@Valid @RequestBody RefreshDTO dto) {
        return Result.success(authService.refresh(dto.getRefreshToken()));
    }

    @PostMapping("/logout")
    public Result<Void> logout(@RequestHeader("Authorization") String authHeader) {
        authService.logout(authHeader.substring(7));
        return Result.success();
    }

    @GetMapping("/me")
    public Result<UserVO> me() {
        return Result.success(authService.currentUser());
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class AuthServiceImpl implements AuthService {

    private final UserMapper userMapper;
    private final PasswordEncoder passwordEncoder;
    private final JwtUtils jwtUtils;
    private final TokenBlacklistService blacklist;

    public TokenVO login(LoginDTO dto) {
        UserDO user = userMapper.selectOne(
            Wrappers.<UserDO>lambdaQuery().eq(UserDO::getUsername, dto.getUsername()));

        if (user == null || !passwordEncoder.matches(dto.getPassword(), user.getPasswordHash())) {
            throw new BusinessException(ErrorCode.LOGIN_FAILED);
        }
        if (user.getStatus() != UserStatus.ACTIVE) {
            throw new BusinessException(ErrorCode.USER_NOT_ACTIVE);
        }

        List<String> auths = loadAuthorities(user.getId());
        String access = jwtUtils.generateAccessToken(user.getId(), auths);
        String refresh = jwtUtils.generateRefreshToken(user.getId());

        return new TokenVO(access, refresh);
    }

    public TokenVO refresh(String refreshToken) {
        Claims claims;
        try {
            claims = jwtUtils.parse(refreshToken);
        } catch (JwtException e) {
            throw new BusinessException(ErrorCode.INVALID_REFRESH_TOKEN);
        }
        if (!"refresh".equals(claims.get("type"))) {
            throw new BusinessException(ErrorCode.INVALID_REFRESH_TOKEN);
        }
        Long userId = Long.parseLong(claims.getSubject());

        List<String> auths = loadAuthorities(userId);
        String access = jwtUtils.generateAccessToken(userId, auths);
        // 滚动刷新（可选）
        String newRefresh = jwtUtils.generateRefreshToken(userId);
        return new TokenVO(access, newRefresh);
    }

    public void logout(String accessToken) {
        try {
            Claims claims = jwtUtils.parse(accessToken);
            long ttl = claims.getExpiration().getTime() - System.currentTimeMillis();
            if (ttl > 0) {
                blacklist.blacklist(accessToken, Duration.ofMillis(ttl));
            }
        } catch (JwtException ignored) {}
    }
}
```

## Token 黑名单（Redis）

```java
@Service
@RequiredArgsConstructor
public class TokenBlacklistService {

    private final StringRedisTemplate redis;
    private static final String PREFIX = "auth:blacklist:";

    public void blacklist(String token, Duration ttl) {
        redis.opsForValue().set(PREFIX + sha256(token), "1", ttl);
    }

    public boolean isBlacklisted(String token) {
        return redis.hasKey(PREFIX + sha256(token));
    }

    private String sha256(String s) { /* ... */ }
}
```

> 黑名单 key 用 token 的 SHA256，避免存原始 token。

## 权限注解

```java
@PreAuthorize("hasAuthority('ORDER_CREATE')")
@PostMapping
public Result<OrderVO> create(@Valid @RequestBody OrderCreateDTO dto) { ... }

@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public Result<Void> delete(@PathVariable Long id) { ... }

@PreAuthorize("hasAuthority('ORDER_VIEW') and #userId == authentication.principal")
@GetMapping("/users/{userId}/orders")
public Result<List<OrderVO>> userOrders(@PathVariable Long userId) { ... }
```

> Authority vs Role：
>
> - `Role`：粗粒度（`ADMIN`、`USER`），存储为 `ROLE_ADMIN`
> - `Authority`：细粒度功能权限（`ORDER_CREATE`、`USER_DELETE`）
> 推荐**只用 Authority**，避免 Role 与 Authority 混用造成困惑。

## 当前用户工具

```java
public final class SecurityUserHelper {
    private SecurityUserHelper() {}

    public static Long currentUserId() {
        Long id = currentUserIdOrNull();
        if (id == null) throw new BusinessException(ErrorCode.UNAUTHORIZED);
        return id;
    }

    public static Long currentUserIdOrNull() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated() || auth.getPrincipal() == null) return null;
        Object p = auth.getPrincipal();
        if (p instanceof Long l) return l;
        return null;
    }

    public static boolean hasAuthority(String authority) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return auth != null && auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals(authority));
    }
}
```

## 鉴权失败的统一返回

```java
@Component
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res, AuthenticationException ex)
            throws IOException {
        ResponseUtils.write(res, HttpStatus.UNAUTHORIZED,
            Result.fail(ErrorCode.UNAUTHORIZED));
    }
}

@Component
public class RestAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest req, HttpServletResponse res, AccessDeniedException ex)
            throws IOException {
        ResponseUtils.write(res, HttpStatus.FORBIDDEN,
            Result.fail(ErrorCode.FORBIDDEN));
    }
}
```

## 密码

- 注册/重置密码用 `BCryptPasswordEncoder`（默认强度 10）
- 数据库列 `password_hash VARCHAR(64)`
- **永不**返回密码哈希（VO 不含 `passwordHash` 字段）

## CORS

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOriginPatterns("https://*.example.com", "http://localhost:5173")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .exposedHeaders("X-Trace-Id")
            .allowCredentials(false)
            .maxAge(3600);
    }
}
```

> 不要 `allowedOrigins("*")` + `allowCredentials(true)`：浏览器会拒绝。

## 多租户 / 数据权限

如有需求，应该在 SecurityContext 中放 `tenantId`，并在 MyBatis 拦截器追加 SQL 条件。本规范不展开。

## 最佳实践

1. **JWT 短 + Refresh 长**
2. **Logout 写黑名单**
3. **`@PreAuthorize` 在方法上**，不要把鉴权逻辑写进 Service
4. **当前用户从 SecurityContext 取**，不通过参数
5. **密码 BCrypt**，VO 永不含 hash
6. **CORS 集中配置**，不在 Controller 上 `@CrossOrigin`

## 反模式

- 把 token 放 URL query / 表单字段
- 用 `MD5(password)` 存密码
- 自己实现"是否登录"的拦截器（用 Spring Security）
- 把权限校验写在 Service 入口
- `permitAll()` 误开放接口（漏配 `authenticated()`）
- JWT secret 写死在代码里
