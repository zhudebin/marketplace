# 代码质量与命名

> 跨前后端共同遵守的命名、注释与禁用项。

## 命名约定

### 通用

| 类别        | 风格                | 示例                                  |
| ----------- | ------------------- | ------------------------------------- |
| 文件夹      | kebab-case          | `user-management/`                    |
| 包/模块     | 全小写              | `com.company.app.user`                |
| 常量        | UPPER_SNAKE_CASE    | `MAX_RETRY_COUNT`                     |
| 布尔变量    | `is/has/can` 前缀   | `isLoading`、`hasPermission`          |
| 枚举        | UPPER_SNAKE_CASE    | `UserStatus.ACTIVE`                   |

### 前端

| 类别         | 风格         | 示例                       |
| ------------ | ------------ | -------------------------- |
| Vue 组件文件 | PascalCase   | `UserList.vue`             |
| Composable   | `useXxx`     | `useAuth.ts`               |
| Store        | `useXxxStore` | `useUserStore.ts`           |
| Type         | PascalCase   | `interface UserVO {}`      |

### 后端

| 类别         | 风格              | 示例                         |
| ------------ | ----------------- | ---------------------------- |
| 类           | PascalCase        | `UserService`                |
| 方法         | camelCase 动词开头 | `findById`、`createUser`      |
| Controller   | `XxxController`    | `UserController`             |
| Service 接口 | `XxxService`       | `UserService`                |
| 实现类       | `XxxServiceImpl`   | `UserServiceImpl`            |
| Mapper       | `XxxMapper`        | `UserMapper`                 |
| Entity       | 与表名同义但 PascalCase | `UserDO`                |
| DTO          | `XxxDTO`           | `CreateUserDTO`              |
| VO           | `XxxVO`            | `UserVO`                     |

### 数据库

| 对象      | 风格                    | 示例                  |
| --------- | ----------------------- | --------------------- |
| 表        | snake_case 单数         | `user`、`order_item`  |
| 字段      | snake_case              | `created_at`          |
| 时间字段  | `_at` 后缀              | `updated_at`          |
| 布尔字段  | `is_` 前缀              | `is_deleted`          |
| 索引      | `idx_<table>_<col>`     | `idx_user_email`      |
| 唯一索引  | `uk_<table>_<col>`      | `uk_user_email`       |
| 外键      | `fk_<from>_<to>`        | `fk_order_user`       |

---

## 注释规范

### 何时写注释

- ✅ 复杂业务规则的来源（例如"产品定义：库存按 SKU 维度"）
- ✅ 看起来像 bug 但故意为之的代码（"WORKAROUND: ..."）
- ✅ 安全/性能相关的关键点
- ❌ 描述代码"做了什么"（让代码自解释）
- ❌ 自动生成的 Getter/Setter

### Java JavaDoc

```java
/**
 * 创建用户。同名邮箱将抛 {@link BusinessException}（USER_EMAIL_DUPLICATED）。
 *
 * @param dto 创建参数
 * @return 新用户视图
 */
public UserVO create(CreateUserDTO dto) { ... }
```

### TypeScript JSDoc

```typescript
/**
 * 调用后端登录接口，成功后写入 token 与 user 到 Pinia
 * @throws BusinessError 当账号或密码错误
 */
export async function login(req: LoginReq): Promise<UserVO> { ... }
```

---

## 禁用项

### 全局禁用

- `console.log` 出现在生产代码（仅允许 `console.warn` / `console.error`，且必须有上下文）
- `debugger` 不允许提交
- `@SuppressWarnings("all")` —— 必须细化到具体规则
- 中文标点出现在代码标识符（注释和字符串字面量除外）

### Java

- `e.printStackTrace()`
- `System.out.println` / `System.err.println`
- `new Date()`（用 `LocalDateTime`）
- `SimpleDateFormat`（非线程安全，用 `DateTimeFormatter`）
- `Optional` 作为字段或方法参数（仅用于返回值）
- 直接抛 `RuntimeException`（一律 `BusinessException`）

### TypeScript

- `any`（必须用 `unknown` 或具体类型；如确需 any，必须 `// eslint-disable-next-line` 并写理由）
- `as any` 强转
- `// @ts-ignore`（用 `// @ts-expect-error` 并写理由）
- 默认导出（除框架要求外）

---

## 函数复杂度

| 指标            | 阈值      |
| --------------- | --------- |
| 单函数行数      | ≤ 50 行   |
| 圈复杂度        | ≤ 10      |
| 单文件行数      | ≤ 400 行  |
| 函数参数        | ≤ 5 个，多了用对象/DTO |

> 超阈值的代码必须拆分或在 Review 中被显式说明。

---

## 提交信息

使用 [Conventional Commits](https://www.conventionalcommits.org/zh-hans/)：

```
feat(user): 支持邮箱登录
fix(order): 修复并发下单重复扣库存
refactor(auth): 抽取 JwtUtils
docs(spec): 更新 mybatis-plus 章节
```

| 类型       | 含义               |
| ---------- | ------------------ |
| `feat`     | 新功能             |
| `fix`      | 修 bug             |
| `refactor` | 重构（不改行为）   |
| `perf`     | 性能优化           |
| `docs`     | 文档               |
| `test`     | 测试               |
| `chore`    | 构建/工具          |
| `style`    | 格式化（不改行为） |

---

## 不要做

| 反模式                              | 正确做法                          |
| ----------------------------------- | --------------------------------- |
| 用魔法字符串/魔法数字               | 抽常量或枚举                      |
| 巨型 if-else 链                     | 策略模式 / 表驱动                 |
| Service 跨调时拿 Controller 的 DTO  | 转成内部模型再传                  |
| Vue 模板写超长表达式                | 抽 computed                       |
| 一个函数同时做查询、计算、写库       | 拆成纯函数 + 副作用函数            |
