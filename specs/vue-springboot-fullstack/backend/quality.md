# 后端提交前清单

> 推送/合并前必须依次完成。任何一项未通过都视为 **不可提交**。

## 编译与静态检查

- [ ] `./mvnw clean compile` / `./gradlew compileJava` 全绿
- [ ] 无 IDE 警告（未使用 import、未使用变量、@Deprecated 调用）
- [ ] Checkstyle / Spotless 通过（如启用）
- [ ] 无 `TODO` / `FIXME` 留在主流程未跟进

## 类型与契约

- [ ] 接口出参一律 `Result<T>`，未裸返回 Entity
- [ ] DTO 全部加 `@Valid` 与字段级校验注解
- [ ] Entity ↔ DTO ↔ VO 转换走 MapStruct，未手写 setter 拷贝
- [ ] 新增字段同步更新 OpenAPI 描述（`@Schema`）
- [ ] 前端类型已通过 `pnpm gen:api` 重新生成（参考 [shared/types-sync.md](../shared/types-sync.md)）

## 数据库

- [ ] 所有 Schema 变更通过 Flyway 迁移脚本（`Vxxx__*.sql`），未手动改库
- [ ] 已发布的迁移脚本 **未被修改**
- [ ] 新表/新字段命名符合 `database.md` 规范（snake_case、`_at` 时间后缀）
- [ ] 涉及大表的 DDL 已评估在线 DDL 风险

## 测试

- [ ] 单元测试：Service 层关键分支已覆盖
- [ ] 集成测试：Controller + 关键 Mapper 走通（`@SpringBootTest` + Testcontainers）
- [ ] `./mvnw test` 全绿
- [ ] 关键业务路径手动跑过一遍

## 安全

- [ ] 新接口默认登录可见，公开接口已显式加入白名单
- [ ] 角色权限通过 `@PreAuthorize` 标注
- [ ] 用户输入没有直接拼 SQL（一律走 MyBatis-Plus / 占位符）
- [ ] 没有把 token / 密码 / API Key 写进日志或返回体
- [ ] CORS / CSRF / 文件上传白名单符合预期

## 配置

- [ ] 新增配置项已在所有 profile（`dev` / `test` / `prod`）补齐或显式 fallback
- [ ] 敏感配置走环境变量，未硬编码
- [ ] `application.yml` 注释清楚每项配置含义

## 错误与日志

- [ ] 业务异常抛 `BusinessException`，未直接抛 `RuntimeException`
- [ ] 全局异常处理器已覆盖新错误码
- [ ] 关键日志用占位符 `{}`，未字符串拼接
- [ ] 没有 `e.printStackTrace()` / `System.out.println`
- [ ] 异常日志带 traceId（MDC）

## 性能

- [ ] for 循环里没有调 Mapper（用 `inSql` / `saveBatch`）
- [ ] 分页查询已设上限（默认 ≤ 100 / 页）
- [ ] 缓存键有过期时间，且考虑了空值/异常缓存
- [ ] 跨服务/外部调用设置了超时和重试

## 文档

- [ ] Swagger UI（`/swagger-ui.html`）能正确加载并显示新接口
- [ ] 接口描述、参数、响应示例完整
- [ ] README / 模块文档已更新（如有重大变更）

---

## 自动化（建议落地到 CI）

```bash
./mvnw -B verify          # 编译 + 测试
./mvnw spotless:check     # 格式
./mvnw spotbugs:check     # 静态分析
./mvnw flyway:validate    # 迁移脚本一致性
```

> CI 失败的 PR **禁止合并**。
