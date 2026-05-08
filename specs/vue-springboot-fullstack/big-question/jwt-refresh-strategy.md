# JWT 续签策略

| 项       | 值        |
| -------- | --------- |
| 分类     | 鉴权      |
| 严重等级 | Critical  |

## 问题描述

只有 access token，过期就强制登出，体验极差；但盲目延长有效期又带来安全风险。错误的 refresh 实现还会引出更糟的问题：

1. 多个并发请求同时 401，前端发起 N 次 refresh，互相覆盖 token
2. 用户在 A 设备登出，B 设备仍能用旧 token，直到自然过期
3. refresh token 也存在前端 localStorage，被 XSS 偷走后无法挽回
4. 后端没有黑名单/版本号机制，access token 注销后**仍可用**直到过期
5. 把 access token 放在 URL 里（GET 参数），被日志/Referer 泄漏

## 根因

- **access token 是无状态的 self-contained 令牌**：签发后无法吊销（除非建立服务端黑名单）
- 服务端不维护登录态时，"登出"只在前端有效
- 并发 refresh 没有做请求合并

## 解决方案

### 1. 双 token 模型

| Token         | 有效期    | 存放                      | 用途                   |
| ------------- | --------- | ------------------------- | ---------------------- |
| access token  | 15-30 min | 内存 / sessionStorage     | 调业务接口             |
| refresh token | 7-30 day  | httpOnly Cookie（推荐）   | 仅用于换 access        |

> refresh token 放 httpOnly Cookie 可挡 XSS。如果项目无法用 Cookie，至少别和 access 同处存放。

### 2. 后端：refresh 接口与黑名单

```java
@PostMapping("/api/auth/refresh")
public Result<TokenPair> refresh(@CookieValue("refresh_token") String rt) {
    // 1. 校验 refresh token 签名 + 类型 == "refresh"
    // 2. 查 Redis 看是否被吊销
    // 3. 滚动签发新 access + 新 refresh（rotation）
    // 4. 老 refresh 加入 Redis 黑名单（防重放）
}
```

关键点：
- **refresh token 单次使用**：每次 refresh 后旧的立即失效（rotation）
- **登出时把 access 的 jti 加 Redis 黑名单**，TTL = access 剩余有效期
- 把 `userId + tokenVersion` 一并签入 token；管理员重置密码时递增 `tokenVersion`，老 token 全部作废

### 3. 前端：请求队列合并

axios 拦截器（参考 [frontend/api-integration.md](../frontend/api-integration.md)）：

```typescript
let refreshPromise: Promise<string> | null = null;

http.interceptors.response.use(undefined, async (error) => {
  const status = error.response?.status;
  const config = error.config;

  if (status === 401 && !config._retry) {
    config._retry = true;
    refreshPromise = refreshPromise ?? refreshAccessToken();
    try {
      const newToken = await refreshPromise;
      config.headers.Authorization = `Bearer ${newToken}`;
      return http(config);
    } finally {
      refreshPromise = null;
    }
  }
  return Promise.reject(error);
});
```

要点：
- 同一时刻**只有一个** refresh 请求在飞
- 失败要让所有挂起的请求一起失败 → 跳登录

### 4. 登出全链路清理

```
用户点登出
  └─ 前端调 /api/auth/logout
       └─ 后端把 access jti 加 Redis 黑名单
       └─ 后端清空 refresh token cookie
       └─ 前端清空 Pinia / sessionStorage 中的 token & user
       └─ 跳登录页
```

> "前端只清 token，后端不动"是常见漏洞，旧 token 在过期前仍能调接口。

### 5. 安全增强

- access token TTL 越短越好（15 min 是常见折中）
- refresh token rotation + 复用检测（同一 refresh 用了两次 → 视为被盗，立即吊销整链）
- 每个 token 内嵌 `jti`，便于黑名单
- token 不要放 URL，**仅在 Authorization 头**
- 关键操作（改密、转账）即使有 access 也要二次校验（短信码 / 密码重输）

## 关键启示

- **JWT 不是"无服务端"的银弹**：登出、强制下线都需要服务端协助（黑名单 / tokenVersion）
- **refresh 必须是单次使用 + 滚动**，否则被偷一次后果与单 token 无异
- **并发 refresh 必须合并**，否则会出现"我刚换的 token 又被覆盖"
- 所有"登出"操作必须是**前后端联动**

## 自检清单

- [ ] access TTL ≤ 30 min
- [ ] refresh token 走 httpOnly Cookie 或独立存储
- [ ] refresh token 单次使用 + rotation
- [ ] 后端有 access 黑名单（Redis），登出时写入
- [ ] 前端有 refresh 请求合并机制
- [ ] 登出时前后端**都**清理
- [ ] token 内嵌 `jti`、`tokenVersion`、`type`
- [ ] 关键操作有二次校验
