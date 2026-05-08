# Flyway 开发期"漂移"

| 项       | 值          |
| -------- | ----------- |
| 分类     | 数据库迁移  |
| 严重等级 | Warning     |

## 问题描述

团队多人同时开发期间，频繁遇到：

1. 启动报错：`Validate failed: Migration checksum mismatch for migration version 1.5`
2. 启动报错：`Detected resolved migration not applied to database: 1.7`
3. 同事 A 改了 `V1_5__add_user_table.sql`，本地能跑，CI 跑不过
4. 拉同事的分支，本地数据库已迁移到 `V1_8`，对方 PR 是 `V1_5__xxx`，导致 out-of-order
5. 测试库不知什么时候被人手动 drop 了表，Flyway 执行报错
6. 生产升级时跑出"未知"的迁移记录，不知是谁、何时加的

## 根因

- **已发布的迁移脚本被修改了**（checksum 改变）
- 多人并行开发时**版本号撞车**或**乱序**
- Flyway 默认严格校验：版本号必须连续递增、已应用的脚本不能改
- 测试/开发库**与脚本管理脱节**

## 解决方案

### 1. 铁律：已合并的迁移**永不修改**

```
V1_5__add_user_table.sql  ← 已合并到 main，禁止改
                            想改？写新脚本：V1_9__alter_user_add_phone.sql
```

校验规则：
- CI 中跑 `flyway:validate`，校验失败直接 fail PR
- Code Review 检查是否在改历史脚本

### 2. 版本号规则：用"业务里程碑 + 时间戳"

避免版本号撞车的两种方案：

#### 方案 A：时间戳版本（推荐多人协作）

```
V20260508_1430__add_user_phone.sql
V20260508_1530__create_order_table.sql
```

- 不会撞车
- 顺序天然按时间
- 配合 `flyway.outOfOrder=true` 允许"插队"

#### 方案 B：序号版本 + 强约定

```
V1_15__add_user_phone.sql
V1_16__create_order_table.sql
```

- 简洁，但需要人工分配号段
- 多人并行时容易撞，PR 顺序不能乱

> **小团队** B 够用；**4 人以上、并行特性多**用 A。

### 3. 开发库与脚本同源

- 本地数据库**不要手动改**结构，一律走 Flyway
- 启动时自动 migrate（`spring.flyway.enabled=true`）
- 出现 dirty 数据库时**重建**：`docker compose down -v && docker compose up`，绝不"手工修一修"

### 4. 配置 out-of-order

```yaml
spring:
  flyway:
    out-of-order: true   # 仅 dev / test 环境
    validate-on-migrate: true
```

> 生产环境**禁用** out-of-order，避免线上跳过迁移。

### 5. baseline：接管已有库

接入存量库时：

```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: 1
```

并把当前 schema 整体 dump 成 `V1__init_baseline.sql`。

### 6. 修复模式（紧急）

如果生产真的出现 checksum mismatch（极罕见，应避免）：

```sql
-- 先确认改动是否安全，再用 repair
DELETE FROM flyway_schema_history WHERE version='1.5' AND success=0;
-- 或 ./mvnw flyway:repair
```

> **`flyway:repair` 是最后手段**，不是日常工具。

### 7. 不在 Flyway 里做的事

| 不要做                           | 替代                            |
| -------------------------------- | ------------------------------- |
| 在迁移脚本里塞业务初始化数据     | 用 `R__seed_xxx.sql`（重复执行）或单独的 data migration |
| 在迁移里跑很慢的 DDL（大表加列） | 改用 in-place online DDL（gh-ost / pt-osc） |
| 在迁移里 `SELECT` 业务数据决定 DDL | 业务逻辑放代码层                |

## 关键启示

- **迁移脚本是 git 历史的一部分**，已合并的就是过去式
- 多人并行时**用时间戳版本**避免撞车
- 开发库不能手动改，要么 Flyway，要么重建
- `flyway:repair` 只能救急，平时不该用到

## 自检清单

- [ ] 我有没有在改已合并的 V*.sql？
- [ ] 版本号是否会和同事的 PR 冲突？
- [ ] 这个 DDL 在生产大表上是否会锁表？
- [ ] 业务数据有没有混进 DDL 脚本？
- [ ] dev / prod 的 `out-of-order` 配置是否正确？
