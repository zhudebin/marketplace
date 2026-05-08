# Controller 规范

REST 接口的写法、参数绑定、统一响应、错误传播与 OpenAPI 注解。

## 基本结构

```java
package com.company.app.module.order.controller;

import com.company.app.common.response.Result;
import com.company.app.module.order.convert.OrderConverter;
import com.company.app.module.order.dto.OrderCreateDTO;
import com.company.app.module.order.dto.OrderQueryDTO;
import com.company.app.module.order.service.OrderService;
import com.company.app.module.order.vo.OrderVO;
import com.company.app.common.response.PageResult;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@Tag(name = "订单管理")
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;
    private final OrderConverter orderConverter;

    @Operation(summary = "分页查询订单")
    @GetMapping
    public Result<PageResult<OrderVO>> list(@Valid OrderQueryDTO query) {
        PageResult<OrderVO> page = orderService.page(query);
        return Result.success(page);
    }

    @Operation(summary = "查询订单详情")
    @GetMapping("/{id}")
    public Result<OrderVO> detail(@PathVariable Long id) {
        OrderVO vo = orderService.findById(id);
        return Result.success(vo);
    }

    @Operation(summary = "创建订单")
    @PostMapping
    @PreAuthorize("hasAuthority('ORDER_CREATE')")
    public Result<OrderVO> create(@Valid @RequestBody OrderCreateDTO dto) {
        OrderVO vo = orderService.create(dto);
        return Result.success(vo);
    }

    @Operation(summary = "取消订单")
    @PostMapping("/{id}/cancel")
    public Result<Void> cancel(@PathVariable Long id) {
        orderService.cancel(id);
        return Result.success();
    }
}
```

## 强制规则

| 规则                                                          | 原因                                  |
| ------------------------------------------------------------- | ------------------------------------- |
| 出参一律 `Result<T>`                                          | 前端拦截器统一解包                    |
| `@RequestBody` 入参必须 `@Valid`                              | 校验失败由全局处理器返回字段错误      |
| Controller **不写业务逻辑**                                   | 仅做 HTTP 协议与 DTO/VO 转换          |
| Controller **不直接调 Mapper**                                | 必须经过 Service                      |
| 路径前缀统一 `/api/`                                          | 便于前端代理与网关路由                |
| 不返回 `OrderDO`                                              | DO 是数据库映射，不是接口契约         |
| 时间字段统一用 `LocalDateTime` + Jackson 全局格式             | 见 [configuration.md](./configuration.md) |
| `@PreAuthorize` 注解写在方法上                                | 详见 [authentication.md](./authentication.md) |

## 统一响应

```java
// common/response/Result.java
package com.company.app.common.response;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class Result<T> {
    private boolean success;
    private String code;
    private String message;
    private T data;

    public static <T> Result<T> success() {
        return new Result<>(true, "OK", "success", null);
    }

    public static <T> Result<T> success(T data) {
        return new Result<>(true, "OK", "success", data);
    }

    public static <T> Result<T> fail(String code, String message) {
        return new Result<>(false, code, message, null);
    }

    public static <T> Result<T> fail(ErrorCode error) {
        return new Result<>(false, error.getCode(), error.getMessage(), null);
    }
}
```

```java
// common/response/PageResult.java
package com.company.app.common.response;

import com.baomidou.mybatisplus.core.metadata.IPage;
import lombok.AllArgsConstructor;
import lombok.Data;

import java.util.List;
import java.util.function.Function;

@Data
@AllArgsConstructor
public class PageResult<T> {
    private List<T> records;
    private long total;
    private long pageNum;
    private long pageSize;

    public static <DO, VO> PageResult<VO> of(IPage<DO> page, Function<DO, VO> mapper) {
        List<VO> list = page.getRecords().stream().map(mapper).toList();
        return new PageResult<>(list, page.getTotal(), page.getCurrent(), page.getSize());
    }
}
```

## HTTP 动词约定

| 操作                      | 动词    | 路径示例                          |
| ------------------------- | ------- | --------------------------------- |
| 列表查询                  | GET     | `GET /api/orders`                 |
| 单个查询                  | GET     | `GET /api/orders/{id}`            |
| 创建                      | POST    | `POST /api/orders`                |
| 完整更新                  | PUT     | `PUT /api/orders/{id}`            |
| 部分更新                  | PATCH   | `PATCH /api/orders/{id}`          |
| 删除                      | DELETE  | `DELETE /api/orders/{id}`         |
| 业务动作（不是 CRUD）     | POST    | `POST /api/orders/{id}/cancel`    |
| 批量操作                  | POST    | `POST /api/orders/batch`          |
| 导出                      | GET     | `GET /api/orders/export`          |

> **业务动作不要伪装成 PUT/PATCH**。`/cancel` `/approve` `/publish` 这类用 `POST + 子资源动词`。

## 参数绑定

### Path 变量

```java
@GetMapping("/{id}")
public Result<OrderVO> detail(@PathVariable Long id) { ... }
```

### Query 参数 → DTO

将所有 query 参数封装成一个 DTO 对象，**不要**写一堆 `@RequestParam`：

```java
// 反模式
@GetMapping
public Result<...> list(
    @RequestParam(required = false) String status,
    @RequestParam(required = false) String keyword,
    @RequestParam(defaultValue = "1") Integer pageNum,
    @RequestParam(defaultValue = "20") Integer pageSize) { ... }

// 推荐
@GetMapping
public Result<...> list(@Valid OrderQueryDTO query) { ... }
```

```java
@Data
public class OrderQueryDTO {
    private String status;
    private String keyword;

    @Min(1)
    private Integer pageNum = 1;

    @Min(1) @Max(200)
    private Integer pageSize = 20;
}
```

### Body

```java
@PostMapping
public Result<OrderVO> create(@Valid @RequestBody OrderCreateDTO dto) { ... }
```

### 文件上传

```java
@PostMapping(value = "/avatar", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public Result<String> uploadAvatar(@RequestPart("file") MultipartFile file) {
    String url = fileService.upload(file);
    return Result.success(url);
}
```

### 文件下载

下载场景**不**包 `Result<T>`：

```java
@GetMapping("/export")
public ResponseEntity<Resource> export(@Valid OrderQueryDTO query) {
    Resource res = orderService.export(query);
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=orders.xlsx")
        .body(res);
}
```

## 错误处理

Controller **不写 try/catch**，业务错误抛 `BusinessException`，由全局处理器统一转换成 `Result<Void>`。详见 [exception-handling.md](./exception-handling.md)。

```java
@PostMapping("/{id}/cancel")
public Result<Void> cancel(@PathVariable Long id) {
    orderService.cancel(id);  // 内部可能抛 BusinessException
    return Result.success();
}
```

## OpenAPI / Swagger 注解

```java
@Tag(name = "订单管理", description = "订单 CRUD 与状态流转")
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Operation(summary = "分页查询订单", description = "支持按状态、关键字筛选")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "成功"),
        @ApiResponse(responseCode = "401", description = "未登录"),
    })
    @GetMapping
    public Result<PageResult<OrderVO>> list(@Valid OrderQueryDTO query) { ... }
}
```

> **不要**给每个字段都打 `@Schema`，让 DTO 字段类型 + JavaDoc 说话即可。重要业务说明用 `@Schema(description = "...")`。

## 当前用户

```java
import com.company.app.common.security.SecurityUserHelper;

@PostMapping
public Result<OrderVO> create(@Valid @RequestBody OrderCreateDTO dto) {
    Long userId = SecurityUserHelper.currentUserId();  // 不通过参数传递
    return Result.success(orderService.create(userId, dto));
}
```

不要让前端把 `userId` 通过 DTO 传来——必须从安全上下文取，否则越权。

## 跨域 CORS

集中配置在 `WebConfig`（详见 [configuration.md](./configuration.md)），**不要**在 Controller 上加 `@CrossOrigin`。

## 最佳实践

1. **Controller 极薄**：参数绑定 + 调 Service + 转 VO + 包 Result
2. **入参用 DTO**，出参用 VO，禁止 DO 出入接口
3. **`@Valid` 必加**，校验交给框架
4. **当前用户从 SecurityContext 取**
5. **业务动作 POST + 子资源动词**

## 反模式

- Controller 里 `if/else` 写业务逻辑
- 直接返回 `OrderDO` 给前端
- `@RequestParam` 列十多个参数
- Controller 里 `try/catch` 业务异常并自行返回 fail
- 不加 `@Valid`，让 Service 再校验一遍
- 用 `Map<String, Object>` 承接入参或返回（破坏类型契约）
