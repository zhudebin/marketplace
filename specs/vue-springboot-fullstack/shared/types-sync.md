# 类型同步：OpenAPI → TypeScript

> 后端是**类型源头**。前端禁止手写后端返回的 DTO 类型。

## 整体链路

```
Java DTO/VO + @Schema  ──springdoc──►  /v3/api-docs (JSON)
                                            │
                                openapi-typescript / orval
                                            ▼
                            frontend/src/types/api.ts (生成物，禁手改)
```

---

## 后端：暴露 OpenAPI

`pom.xml`：

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
</dependency>
```

`application.yml`：

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

> 配置类示例见 [backend/configuration.md](../backend/configuration.md)。

### 注解规范

```java
@Schema(description = "用户视图")
public record UserVO(
    @Schema(description = "用户 ID", example = "1001") String id,
    @Schema(description = "邮箱") String email,
    @Schema(description = "状态：ACTIVE/INACTIVE") UserStatus status
) {}
```

要求：

- 每个 DTO/VO 类加 `@Schema(description=...)`
- 关键字段加 `description` 与 `example`
- 枚举显式列出取值

---

## 前端：生成类型

推荐工具：[`openapi-typescript`](https://github.com/openapi-ts/openapi-typescript)（轻量，零运行时）。

`package.json`：

```json
{
  "scripts": {
    "gen:api": "openapi-typescript http://localhost:8080/v3/api-docs -o src/types/api.ts"
  }
}
```

生成物示例：

```typescript
// src/types/api.ts （生成，禁手改）
export interface paths { ... }
export interface components {
  schemas: {
    UserVO: {
      id: string;
      email: string;
      status: 'ACTIVE' | 'INACTIVE';
    };
    Result_UserVO_: {
      success: boolean;
      code: string;
      message: string;
      data?: components['schemas']['UserVO'];
    };
  };
}
```

业务侧使用：

```typescript
import type { components } from '@/types/api';

type UserVO = components['schemas']['UserVO'];
```

> 用 `type` 别名简化访问，避免业务代码到处写 `components['schemas']['XXX']`。

---

## CI 校验

在 CI 增加一步：

```bash
pnpm gen:api
git diff --exit-code src/types/api.ts
```

> 若 diff 非空，说明前端类型未与后端同步，**直接 fail PR**。

---

## 版本演进

| 变更类型              | 处理方式                              |
| --------------------- | ------------------------------------- |
| 后端新增可选字段      | 前端无需改动，重新 `gen:api` 即可     |
| 后端新增必填字段      | 同步前端调用方，分批灰度              |
| 后端删除字段          | 视为 **破坏性变更**，需走废弃流程     |
| 字段含义变更          | 必须改名，禁止"原地变义"              |

### 字段废弃流程

1. 标 `@Deprecated` + `@Schema(deprecated = true)`
2. 通知前端，给定下线时间
3. 一个迭代后再删除

---

## 不要做

| 反模式                                  | 正确做法                       |
| --------------------------------------- | ------------------------------ |
| 前端手写 DTO interface                  | 一律走 `gen:api`               |
| 后端 DTO 用 `Map<String,Object>` 返回   | 显式定义 VO 类                 |
| `@Schema` 描述空着                      | 必填，前端 IDE 提示要靠它      |
| 后端 enum 没序列化为字符串              | 配置 Jackson 输出枚举名称      |
