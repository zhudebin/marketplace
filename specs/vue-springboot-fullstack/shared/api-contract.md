# 统一 API 契约

> 所有 HTTP 接口共享同一套响应结构、错误约定与状态码语义。**前后端共同遵守**。

## 响应结构

```json
{
  "success": true,
  "code": "OK",
  "message": "success",
  "data": { ... }
}
```

| 字段      | 类型    | 含义                                                |
| --------- | ------- | --------------------------------------------------- |
| `success` | boolean | 业务是否成功（**不是** HTTP 是否成功）              |
| `code`    | string  | 业务错误码，成功固定 `OK`                           |
| `message` | string  | 给用户/开发者看的消息（可被前端直接展示）           |
| `data`    | T \| null | 业务载荷                                          |

> **HTTP 状态码** 与 **业务 success** 解耦，但需要遵循下面的对照表。

---

## HTTP 状态码语义

| HTTP | 含义                                         | `success` |
| ---- | -------------------------------------------- | --------- |
| 200  | 请求成功（无论业务成功失败均可使用）         | true/false |
| 201  | 资源创建成功                                 | true       |
| 204  | 成功且无返回体（删除等场景）                 | true       |
| 400  | 参数校验失败                                 | false      |
| 401  | 未登录或登录态失效                           | false      |
| 403  | 已登录但无权限                               | false      |
| 404  | 资源不存在                                   | false      |
| 409  | 冲突（重复提交、版本冲突）                   | false      |
| 422  | 业务规则不满足（语义层错误）                 | false      |
| 429  | 限流                                         | false      |
| 500  | 服务端内部错误                               | false      |
| 503  | 依赖不可用（DB、外部服务）                   | false      |

> **建议**：业务错误优先用 4xx + `success=false` + 业务 code，不要全部 200 + success=false。

---

## 错误响应

```json
{
  "success": false,
  "code": "USER_NOT_FOUND",
  "message": "用户不存在",
  "data": null
}
```

带字段错误的校验失败：

```json
{
  "success": false,
  "code": "VALIDATION_FAILED",
  "message": "参数校验失败",
  "data": {
    "errors": [
      { "field": "email", "message": "邮箱格式不正确" },
      { "field": "age", "message": "必须 ≥ 18" }
    ]
  }
}
```

---

## 错误码命名

| 规范            | 示例                                |
| --------------- | ----------------------------------- |
| 全大写 + 下划线 | `USER_NOT_FOUND`                    |
| 模块前缀（可选）| `ORDER_INVENTORY_INSUFFICIENT`      |
| 不带空格、无中文| 错误码是机器消费，message 给人看    |

> 错误码集中维护在后端 `ErrorCode` 枚举，参考 [backend/exception-handling.md](../backend/exception-handling.md)。

---

## 分页响应

```json
{
  "success": true,
  "code": "OK",
  "message": "success",
  "data": {
    "records": [ ... ],
    "total": 1234,
    "page": 1,
    "size": 20,
    "pages": 62
  }
}
```

> 字段名**严格遵循上表**，避免与 MyBatis-Plus 的 `IPage` 默认字段（`current` / `size`）混淆 —— 后端在 VO 层归一化为 `page`。

---

## 时间与日期

- 统一使用 **ISO 8601 字符串**：`2026-05-08T20:25:00+08:00`
- 后端 Jackson 配置 `JavaTimeModule` + `WRITE_DATES_AS_TIMESTAMPS=false`
- 前端不要自己拼时间字符串，使用 `dayjs` 解析

---

## 大数与精度

- `Long`（雪花 ID 等 > 2^53）**必须**序列化为字符串，避免 JS 精度丢失
- 金额字段使用 **BigDecimal + 字符串**，前端禁止用 `Number` 解析

```java
@JsonSerialize(using = ToStringSerializer.class)
private Long id;
```

---

## 鉴权

- 通过 `Authorization: Bearer <token>` 头传递 access token
- 401 后前端尝试 refresh token，失败则跳登录
- 详细流程见 [backend/authentication.md](../backend/authentication.md) 与 [frontend/authentication.md](../frontend/authentication.md)

---

## 幂等

| 场景       | 方案                                                |
| ---------- | --------------------------------------------------- |
| 创建资源   | 客户端生成 `Idempotency-Key` 头                     |
| 支付/扣费  | 业务层主动校验业务幂等键                            |
| 重复点击   | 前端 loading 防抖 + 后端唯一约束                    |

---

## 不要做

| 反模式                                  | 正确做法                                 |
| --------------------------------------- | ---------------------------------------- |
| 后端把异常 message 直接拼回 message      | 由 GlobalExceptionHandler 翻译为友好文案 |
| 前端用 `response.success` 自己判断状态码 | axios 拦截器统一处理                     |
| 不同接口分页字段名不一致                | 全部用 `records/total/page/size/pages`   |
| 把 ID 用 number 返回                    | Long 一律字符串                          |
