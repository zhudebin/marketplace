# 跨层思考指南

> 写一个特性时**永远从端到端思考**，不要只盯自己那一层。

## 核心心法

> **"这个数据是谁产生的？走过了哪几层？最终给谁看？路上每一站都做了什么？"**

每个特性都是一条**从用户操作到数据落库再回到用户屏幕**的链条。链条上任何一环错位，体验就崩。

---

## 一条完整链路（以"创建用户"为例）

```
用户点击"创建"
  └─ Vue 组件触发 onSubmit
       └─ Element Plus 表单校验（前端） ─ 失败 → 提示
       └─ 调 useUserApi.create(dto)
            └─ axios POST /api/users
                 └─ Spring Security 过滤器（JWT 校验）
                      └─ Controller 接收 CreateUserDTO
                           └─ @Valid 触发 Bean Validation ─ 失败 → 400 + VALIDATION_FAILED
                           └─ Service.create()
                                └─ 业务校验（邮箱重复？） ─ 失败 → BusinessException(USER_EMAIL_DUPLICATED)
                                └─ MapStruct: DTO → Entity
                                └─ Mapper.insert()
                                     └─ MyBatis-Plus → JDBC → MySQL
                                └─ 发布领域事件（可选）
                           └─ MapStruct: Entity → VO
                           └─ 返回 Result.success(VO)
                 └─ axios 拦截器解包 Result，返回 data
            └─ Pinia store 更新 list
       └─ ElMessage.success("创建成功")
       └─ 路由跳转 / 关闭弹窗
```

每一步都该问自己：**这一步如果失败，下游怎么知道？用户怎么感知？**

---

## 三类常见错位

### 1. 校验错位

| 错位                                  | 现象                          | 正解                          |
| ------------------------------------- | ----------------------------- | ----------------------------- |
| 只在前端校验                          | 用 Postman 直接打就过了        | **前后端都要校验**，前端只是体验，后端是兜底 |
| 只在后端校验                          | 用户提交后才知道错             | 前端 Element Plus 表单校验 + 即时提示 |
| 业务校验放在 Controller               | 多入口（定时任务/MQ）会绕过    | **业务校验必须在 Service**     |
| 格式校验放在 Service                  | DTO 形同虚设                   | 格式校验用 Bean Validation     |

### 2. 异常错位

| 错位                                  | 正解                                    |
| ------------------------------------- | --------------------------------------- |
| Service 抛 `RuntimeException`         | 抛 `BusinessException(ErrorCode)`        |
| Controller `try-catch` 包一层吞异常   | 一律由 `GlobalExceptionHandler` 处理     |
| 前端 `catch` 吞掉错误不提示           | `BusinessError` 通过 ElMessage 提示       |
| 后端把 SQL 异常 message 直接抛给前端  | 翻译成业务错误码 + 友好文案              |

### 3. 类型错位

| 错位                                  | 正解                                    |
| ------------------------------------- | --------------------------------------- |
| Long ID 用 number 序列化              | `@JsonSerialize(using=ToStringSerializer.class)` |
| 时间用时间戳                          | ISO 8601 字符串                         |
| 前端手写 DTO                          | `pnpm gen:api`                          |
| Entity 直接当成出参 VO                | DTO/VO 分离 + MapStruct                 |

---

## 思考矩阵

写新特性时，先填一遍这张矩阵：

|                  | 前端 | 后端 Controller | Service | Mapper | DB |
| ---------------- | ---- | --------------- | ------- | ------ | -- |
| **校验**         |      |                 |         |        |    |
| **异常**         |      |                 |         |        |    |
| **权限**         |      |                 |         |        |    |
| **日志**         |      |                 |         |        |    |
| **缓存**         |      |                 |         |        |    |
| **性能（分页）** |      |                 |         |        |    |
| **可观测**       |      |                 |         |        |    |

> 一格都不能空。空着说明你**没想清楚那一层在干嘛**。

---

## 前后端协作的三条铁律

### 铁律一：**类型由后端定义**

后端 `@Schema` → springdoc → 前端 `gen:api`。前端禁止猜字段。

### 铁律二：**错误由后端命名**

后端 `ErrorCode` 是唯一权威。前端拿到 `code` 决定怎么提示，**不要自己根据 message 字符串判断**。

### 铁律三：**状态由所属方持有**

- **服务端状态**（用户、订单、配置）→ 后端 + 前端缓存（Pinia / 直查）
- **UI 状态**（弹窗开关、表单临时值）→ 仅前端
- **会话状态**（登录态）→ 后端权威 + 前端缓存 token

混淆这三类是 90% 状态混乱的根源。

---

## 一个反例

**需求**：用户列表支持按角色筛选。

**糟糕实现**：
- 前端拉全量用户，前端 `filter()`
- 后端没做权限隔离，普通用户能看到管理员列表
- 角色字段直接返回数据库的 `tinyint`（1/2/3）
- 前端写 `if (role === 1) ...`

**问题**：
- 数据量大就崩（性能）
- 越权读取（安全）
- 字段含义随时可能变（耦合）
- 前后端都要维护 1/2/3 的对照（重复）

**正确实现**：
- 后端 `/api/users?role=ADMIN&page=1&size=20`，DB 层过滤
- Service 强制叠加当前用户的可见范围（`WHERE tenant_id = #{currentTenantId}`）
- 后端 enum 序列化为字符串：`ADMIN` / `USER` / `GUEST`
- 前端通过 `gen:api` 拿到 `'ADMIN' | 'USER' | 'GUEST'` 类型，编译期检查

---

## 总结

> 不存在"前端问题"或"后端问题"，只存在**链路问题**。
> 写代码前画一遍链路，写代码后**回放**一遍链路，问题会自动浮出来。
