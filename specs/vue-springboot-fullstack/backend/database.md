# 数据库与 Flyway

MySQL 字段命名约定、连接池、Flyway 迁移流程与回滚策略。

## MySQL 版本与字符集

- **版本**：MySQL 8.0+
- **字符集**：`utf8mb4` + `utf8mb4_0900_ai_ci`
- **存储引擎**：InnoDB

## 命名约定

| 对象       | 约定                            | 示例                |
| ---------- | ------------------------------- | ------------------- |
| 表         | `t_` + snake_case + 单数        | `t_order`、`t_user` |
| 字段       | snake_case                      | `created_at`        |
| 主键       | `id`，`BIGINT UNSIGNED`         |                     |
| 外键字段   | `{ref_table}_id`                | `user_id`           |
| 索引名     | `idx_` 普通 / `uk_` 唯一        | `idx_user_status`、`uk_user_email` |
| 布尔字段   | `is_xxx` 或 `xxx`，`TINYINT(1)` | `is_active`、`deleted` |
| 时间字段   | `xxx_at`，`DATETIME(3)`         | `created_at`        |

## 必备字段

每张业务表都包含：

```sql
id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
created_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
updated_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
created_by  BIGINT UNSIGNED NULL,
updated_by  BIGINT UNSIGNED NULL,
deleted     TINYINT(1) NOT NULL DEFAULT 0,
version     INT NOT NULL DEFAULT 0
```

> `version` 用于 MyBatis-Plus `@Version` 乐观锁，没有并发更新场景的表可省略。

## 索引

- 单列查询频次高 → 单列索引
- 复合查询遵循"最左前缀"
- 业务唯一性 → `UNIQUE KEY`，**且必须包含 `deleted`**（见下文逻辑删除）
- 大字段（TEXT/JSON）不建索引
- 时间范围查询字段建索引

```sql
CREATE TABLE t_order (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED NOT NULL,
    reference   VARCHAR(64) NOT NULL,
    status      VARCHAR(32) NOT NULL,
    total       DECIMAL(15,2) NOT NULL DEFAULT 0,
    created_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted     TINYINT(1) NOT NULL DEFAULT 0,
    version     INT NOT NULL DEFAULT 0,
    UNIQUE KEY uk_reference_deleted (reference, deleted),
    KEY idx_user_status (user_id, status),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

## 逻辑删除与唯一索引

`@TableLogic` 删除时实际是 `UPDATE deleted = 1`。如果唯一索引只在 `reference` 上，删除后的同名记录无法重建。

**解决方案 A（推荐）**：唯一索引带上 `deleted`：

```sql
UNIQUE KEY uk_reference_deleted (reference, deleted)
```

但 `deleted` 列只有 0/1 两个值，删两次后还是冲突。改用：

**解决方案 B**：删除时把 `deleted` 改为 `id` 值（保证唯一）：

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-not-delete-value: 0
      logic-delete-value: "id"   # 删除时填充为 id
```

或在数据库列改成 `BIGINT DEFAULT 0`，删除时由触发器/应用层写入 id。详见 [big-question/mybatis-plus-logic-delete.md](../big-question/mybatis-plus-logic-delete.md)。

## 数据源与连接池

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp?useSSL=false&characterEncoding=utf8mb4&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 600000
      max-lifetime: 1800000
      connection-timeout: 5000
      pool-name: MyAppHikariCP
      connection-test-query: SELECT 1
```

> **关键参数**：
>
> - `rewriteBatchedStatements=true`：开启批量 INSERT 重写为 multi-VALUES，性能提升数十倍
> - `useSSL=false`：内网部署常用；公网必须用 SSL
> - `serverTimezone=Asia/Shanghai`：避免 8 小时偏移
> - `maximum-pool-size`：通常 = (业务线程数 × 2) + 备用，根据压测调整

## Flyway 迁移

### 启用

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
    table: flyway_schema_history
    validate-on-migrate: true
    out-of-order: false
```

### 脚本命名

```
src/main/resources/db/migration/
├── V1__init.sql
├── V2__add_user_status.sql
├── V3__create_order_table.sql
└── V20260508_001__add_index_on_order.sql
```

格式：`V{版本号}__{描述}.sql`，下划线两个。版本号建议用日期：`V20260508_001`，避免多人合作冲突。

### 脚本规范

```sql
-- V20260508_001__create_order_table.sql
-- 描述：创建订单表，初始化基础字段与索引
-- 作者：xxx
-- 关联需求：JIRA-1234

CREATE TABLE t_order (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED NOT NULL,
    reference   VARCHAR(64) NOT NULL,
    status      VARCHAR(32) NOT NULL DEFAULT 'PENDING',
    total       DECIMAL(15,2) NOT NULL DEFAULT 0,
    created_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    updated_at  DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    deleted     TINYINT(1) NOT NULL DEFAULT 0,
    version     INT NOT NULL DEFAULT 0,
    UNIQUE KEY uk_reference_deleted (reference, deleted),
    KEY idx_user_status (user_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单主表';
```

### 强制规则

| 规则                                                | 原因                                |
| --------------------------------------------------- | ----------------------------------- |
| **已发布脚本绝对不能修改**                          | Flyway 校验 checksum 失败启动报错   |
| 修正错误必须新增 `Vx_fix.sql` 而不是改老脚本        | 同上                                |
| 大表 DDL 必须用 `pt-online-schema-change` 或 gh-ost | 避免长时间锁表                      |
| 索引添加单独脚本                                    | 便于回滚和审计                      |
| **不允许 `DROP TABLE` / `DROP COLUMN`** 直接进生产 | 软删除：先停用使用 → 下个版本再 DROP |
| 字段重命名不允许直接 RENAME                         | 应"加新列 → 双写 → 切读 → 删旧列"   |
| 数据脚本（DML）单独 `Vx_data_xxx.sql`              | 与 DDL 区分，便于 review            |

### 开发期 schema 漂移

详见 [big-question/flyway-dev-drift.md](../big-question/flyway-dev-drift.md)。摘要：

- **禁止**用 `mybatis-plus` 自动建表 / `spring.jpa.hibernate.ddl-auto`
- 本地开发也走 Flyway
- 出现 checksum 不一致：`flyway:repair` 仅在确认无误时使用

### 多环境

```
db/migration/
├── common/                    # 所有环境共享
│   └── V1__init.sql
├── dev/                       # 仅 dev 环境（如测试数据）
│   └── V900__seed_dev.sql
└── prod/                      # 仅 prod
    └── V900__seed_prod.sql
```

```yaml
spring:
  flyway:
    locations: classpath:db/migration/common,classpath:db/migration/${spring.profiles.active}
```

## 回滚策略

Flyway 社区版不支持自动回滚。约定：

1. 任何破坏性变更**先做兼容版本**（保留旧字段 + 加新字段）
2. 上线后观察一个版本，确认无问题再删
3. 紧急回滚：手写 `V{n+1}__rollback_xxx.sql` 反向变更脚本

## 慢查询监控

开启慢查询日志：

```sql
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;            -- 1 秒
```

应用侧用 P6Spy 或 Druid 监控（开发环境）。

## 备份

- 生产环境每日全量 + binlog 增量
- 上线前 schema 变更必须能在影子库验证一次

## 最佳实践

1. **必备字段五件套** + 主键 BIGINT UNSIGNED
2. **业务唯一索引含 `deleted`**
3. **Flyway 脚本不可变**
4. **大表 DDL 用 online schema change 工具**
5. **HikariCP 参数调优**

## 反模式

- 修改已发布的 Flyway 脚本
- `DROP COLUMN` 直接进生产
- 用 `varchar(255)` 默认值（按业务实际长度声明）
- 业务表无 `created_at` / `updated_at`
- `TIMESTAMP` 字段（用 `DATETIME(3)` 避免 2038 问题）
- 库表 charset 不一致（混用 utf8 与 utf8mb4 → emoji 写入失败）
