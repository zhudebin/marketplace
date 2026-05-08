# 常见问题与解决方案

> 在 Vue + Spring Boot 全栈项目中遇到过的真实坑及定型解决方案。本目录只收录"不知道就会踩、知道了就能绕过"的问题。

## 严重等级

| 级别     | 描述                                       |
| -------- | ------------------------------------------ |
| Critical | 构建失败、数据丢失或安全风险               |
| Warning  | 体验受损但有 workaround                    |
| Info     | 视觉/可用性轻微问题，识别后即可修复        |

---

## 索引

| 问题                                                                       | 分类             | 严重等级 |
| -------------------------------------------------------------------------- | ---------------- | -------- |
| [mybatis-plus-logic-delete.md](./mybatis-plus-logic-delete.md)             | ORM / 数据库     | Critical |
| [jwt-refresh-strategy.md](./jwt-refresh-strategy.md)                       | 鉴权             | Critical |
| [flyway-dev-drift.md](./flyway-dev-drift.md)                               | 数据库迁移       | Warning  |
| [element-plus-on-demand.md](./element-plus-on-demand.md)                   | 前端构建         | Warning  |
| [springboot3-jakarta-migration.md](./springboot3-jakarta-migration.md)     | 升级 / 兼容      | Warning  |
| [cors-samesite.md](./cors-samesite.md)                                     | 跨域 / Cookie    | Warning  |

---

## 如何贡献

发现新坑？请按以下流程沉淀：

1. 用描述性 kebab-case 命名新建 `.md` 文件
2. 遵循统一格式：**问题描述 / 根因 / 解决方案 / 关键启示**
3. 把分类和严重等级补到上面的索引表
4. 尽量提供可复现的最小代码片段
