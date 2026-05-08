# MyBatis-Plus 逻辑删除踩坑

| 项       | 值          |
| -------- | ----------- |
| 分类     | ORM / 数据库 |
| 严重等级 | Critical    |

## 问题描述

启用 MyBatis-Plus 逻辑删除（`@TableLogic`）后，出现一系列"诡异"现象：

1. 唯一索引"无效"：删除一个 email 后再创建同 email，竟然报"邮箱已存在"
2. 自定义 SQL（XML 或 `@Select`）查询时**没有自动加** `is_deleted = 0`，把已删数据带回来了
3. `count()` / `selectList()` 数量对得上，但 join 别的表后突然多出一堆已删行
4. 关联子查询里的 `is_deleted` 没生效，返回了"幽灵"数据

## 根因

1. **逻辑删除 ≠ 真删**：行还在表里，唯一索引仍然冲突
2. MyBatis-Plus 的逻辑删除**只对 `BaseMapper` 提供的方法自动注入** `is_deleted = 0`：
   - `selectById` / `selectList` / `update` / `delete` ✅
   - **自定义 XML / `@Select` ❌（必须自己写）**
3. join 时只对**主表**自动注入 `is_deleted`，**关联表不会自动加**
4. 子查询、`exists` 子句也不自动加

## 解决方案

### 1. 唯一索引改为联合索引

```sql
-- 错误：单列唯一
ALTER TABLE user ADD UNIQUE uk_user_email (email);

-- 正确：把 is_deleted 加进去
ALTER TABLE user ADD UNIQUE uk_user_email (email, is_deleted);
```

但这样仍有问题：删了 A，再删一次同 email，又冲突。**更好的做法**：用 `is_deleted` 存"删除时间戳"或"自增删除序号"，而不是 0/1 布尔。

#### 推荐方案：删除标记用时间戳

```java
@TableLogic(value = "0", delval = "UNIX_TIMESTAMP()")
private Long isDeleted;
```

```sql
ALTER TABLE user ADD UNIQUE uk_user_email (email, is_deleted);
```

- 未删：`is_deleted = 0`
- 删除时填当前时间戳，每次都不同 → 不会与历史已删行冲突

### 2. 自定义 SQL 必须显式加条件

```xml
<!-- 错误 -->
<select id="findByEmail" resultType="UserDO">
  SELECT * FROM user WHERE email = #{email}
</select>

<!-- 正确 -->
<select id="findByEmail" resultType="UserDO">
  SELECT * FROM user WHERE email = #{email} AND is_deleted = 0
</select>
```

> Code Review 必须把 `is_deleted = 0` 当作 SQL 自查项。

### 3. join 必须对每张表都加

```sql
SELECT u.*, o.id AS order_id
FROM user u
LEFT JOIN `order` o ON o.user_id = u.id AND o.is_deleted = 0
WHERE u.is_deleted = 0;
```

### 4. 用 SQL 注入器统一处理（高级）

可以自定义 `ISqlInjector` 给所有 join 自动追加 `is_deleted` 条件，但**复杂度高**，团队规模小不建议。

## 关键启示

- **逻辑删除是写库语义的延伸，不是免费的午餐**
- 唯一索引必须把 `is_deleted` 纳入考量，且 `is_deleted` 最好是非布尔
- 自定义 SQL **永远要显式过滤** `is_deleted`
- 任何含 join 的查询，对每张表问一句"这张要不要 is_deleted = 0"
- 涉及到统计、报表、对账时，明确"看不看已删数据"是产品决策，要文档化

## 自检清单

- [ ] 是否所有需要"软删"的表都有 `is_deleted` 字段？
- [ ] 唯一索引是否包含 `is_deleted`？
- [ ] 所有自定义 SQL 都显式过滤了 `is_deleted`？
- [ ] join 时每张表都加了 `is_deleted = 0`？
- [ ] `is_deleted` 是时间戳（推荐）还是 0/1（注意冲突）？
