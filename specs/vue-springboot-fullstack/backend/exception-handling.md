# 全局异常与统一响应

业务异常与统一响应、错误码体系、`@RestControllerAdvice` 全局处理器。

## 设计目标

- Controller 全部"快乐路径"，不写 `try/catch`
- 业务错误抛 `BusinessException(ErrorCode)`
- 系统错误（数据库连不上、空指针）由全局兜底
- 所有返回符合 `Result<T>` 结构（参见 [shared/api-contract.md](../shared/api-contract.md)）

## 错误码定义

```java
package com.company.app.common.response;

import lombok.Getter;

@Getter
public enum ErrorCode {

    // ===== 通用 =====
    OK("OK", "success"),
    INTERNAL_ERROR("INTERNAL_ERROR", "服务器开小差了，请稍后再试"),
    BAD_REQUEST("BAD_REQUEST", "请求参数错误"),
    VALIDATION_FAILED("VALIDATION_FAILED", "参数校验失败"),
    UNAUTHORIZED("UNAUTHORIZED", "未登录或登录已过期"),
    FORBIDDEN("FORBIDDEN", "无权限"),
    NOT_FOUND("NOT_FOUND", "资源不存在"),
    CONFLICT("CONFLICT", "资源冲突"),
    RATE_LIMITED("RATE_LIMITED", "请求过于频繁，请稍后再试"),

    // ===== 鉴权 =====
    LOGIN_FAILED("LOGIN_FAILED", "用户名或密码错误"),
    INVALID_REFRESH_TOKEN("INVALID_REFRESH_TOKEN", "登录态已失效，请重新登录"),

    // ===== 用户 =====
    USER_NOT_FOUND("USER_NOT_FOUND", "用户不存在"),
    USER_NOT_ACTIVE("USER_NOT_ACTIVE", "账号未激活"),
    USERNAME_EXISTS("USERNAME_EXISTS", "用户名已被占用"),

    // ===== 订单 =====
    ORDER_NOT_FOUND("ORDER_NOT_FOUND", "订单不存在"),
    ORDER_CANNOT_CANCEL("ORDER_CANNOT_CANCEL", "当前状态不允许取消"),
    INVENTORY_NOT_ENOUGH("INVENTORY_NOT_ENOUGH", "库存不足"),

    // ===== 并发 =====
    CONCURRENT_MODIFICATION("CONCURRENT_MODIFICATION", "数据已被他人修改，请刷新后重试");

    private final String code;
    private final String message;

    ErrorCode(String code, String message) {
        this.code = code;
        this.message = message;
    }
}
```

### 错误码命名规范

- 全大写 + 下划线
- `{LAYER/DOMAIN}_{REASON}`：`USER_NOT_FOUND`、`ORDER_CANNOT_CANCEL`
- 不要带数字（`E0001`）：可读性差
- 同一含义全局唯一

### 新增错误码 SOP

1. 业务模块新增前先在 `ErrorCode` 中检索同类错误码
2. 不复用通用错误码描述具体业务（`BAD_REQUEST` ≠ `USERNAME_EXISTS`）
3. message 用对用户友好的话，不暴露内部细节
4. PR 提交时同步更新前端类型（如有错误码常量）

## 业务异常

```java
package com.company.app.common.exception;

import com.company.app.common.response.ErrorCode;
import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {

    private final String code;
    private final transient Object[] args;

    public BusinessException(ErrorCode error) {
        super(error.getMessage());
        this.code = error.getCode();
        this.args = null;
    }

    public BusinessException(ErrorCode error, String customMessage) {
        super(customMessage);
        this.code = error.getCode();
        this.args = null;
    }

    public BusinessException(ErrorCode error, Object... args) {
        super(formatMessage(error.getMessage(), args));
        this.code = error.getCode();
        this.args = args;
    }

    private static String formatMessage(String template, Object... args) {
        return args == null || args.length == 0 ? template : String.format(template, args);
    }
}
```

使用：

```java
if (order == null) throw new BusinessException(ErrorCode.ORDER_NOT_FOUND);

throw new BusinessException(ErrorCode.INVENTORY_NOT_ENOUGH,
    "商品 %s 库存不足，剩余 %d 件", product.getName(), product.getStock());
```

## 全局异常处理器

```java
@RestControllerAdvice
@Slf4j
@RequiredArgsConstructor
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<Result<Void>> handleBusiness(BusinessException ex, HttpServletRequest req) {
        log.warn("业务异常 path={} code={} msg={}", req.getRequestURI(), ex.getCode(), ex.getMessage());
        return ResponseEntity
            .status(httpStatusOf(ex.getCode()))
            .body(Result.fail(ex.getCode(), ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Result<List<FieldError>>> handleValidation(
            MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .toList();
        return ResponseEntity.badRequest()
            .body(new Result<>(false, ErrorCode.VALIDATION_FAILED.getCode(),
                ErrorCode.VALIDATION_FAILED.getMessage(), errors));
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<Result<List<FieldError>>> handleConstraint(
            ConstraintViolationException ex) {
        List<FieldError> errors = ex.getConstraintViolations().stream()
            .map(cv -> new FieldError(cv.getPropertyPath().toString(), cv.getMessage()))
            .toList();
        return ResponseEntity.badRequest()
            .body(new Result<>(false, ErrorCode.VALIDATION_FAILED.getCode(),
                ErrorCode.VALIDATION_FAILED.getMessage(), errors));
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<Result<Void>> handleBodyParse(HttpMessageNotReadableException ex) {
        log.warn("请求体解析失败 msg={}", ex.getMostSpecificCause().getMessage());
        return ResponseEntity.badRequest().body(Result.fail(ErrorCode.BAD_REQUEST));
    }

    @ExceptionHandler(MissingServletRequestParameterException.class)
    public ResponseEntity<Result<Void>> handleMissingParam(MissingServletRequestParameterException ex) {
        return ResponseEntity.badRequest()
            .body(Result.fail(ErrorCode.BAD_REQUEST.getCode(),
                "缺少参数: " + ex.getParameterName()));
    }

    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public ResponseEntity<Result<Void>> handleTypeMismatch(MethodArgumentTypeMismatchException ex) {
        return ResponseEntity.badRequest()
            .body(Result.fail(ErrorCode.BAD_REQUEST.getCode(),
                "参数类型错误: " + ex.getName()));
    }

    @ExceptionHandler(NoHandlerFoundException.class)
    public ResponseEntity<Result<Void>> handle404(NoHandlerFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Result.fail(ErrorCode.NOT_FOUND));
    }

    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResponseEntity<Result<Void>> handleMethod(HttpRequestMethodNotSupportedException ex) {
        return ResponseEntity.status(HttpStatus.METHOD_NOT_ALLOWED)
            .body(Result.fail(ErrorCode.BAD_REQUEST.getCode(), "方法不被允许: " + ex.getMethod()));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<Result<Void>> handleDataIntegrity(DataIntegrityViolationException ex) {
        log.warn("数据完整性异常 msg={}", ex.getMostSpecificCause().getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(Result.fail(ErrorCode.CONFLICT));
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<Result<Void>> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(Result.fail(ErrorCode.FORBIDDEN));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Result<Void>> handleAny(Exception ex, HttpServletRequest req) {
        // 关键：未识别的异常一定要打 ERROR 级别，含堆栈
        log.error("未捕获异常 path={}", req.getRequestURI(), ex);
        return ResponseEntity.internalServerError()
            .body(Result.fail(ErrorCode.INTERNAL_ERROR));
    }

    private HttpStatus httpStatusOf(String code) {
        return switch (code) {
            case "UNAUTHORIZED" -> HttpStatus.UNAUTHORIZED;
            case "FORBIDDEN" -> HttpStatus.FORBIDDEN;
            case "NOT_FOUND" -> HttpStatus.NOT_FOUND;
            case "CONFLICT", "CONCURRENT_MODIFICATION" -> HttpStatus.CONFLICT;
            case "RATE_LIMITED" -> HttpStatus.TOO_MANY_REQUESTS;
            default -> HttpStatus.BAD_REQUEST;
        };
    }

    public record FieldError(String field, String message) {}
}
```

## 字段错误的前端约定

校验失败响应：

```json
{
  "success": false,
  "code": "VALIDATION_FAILED",
  "message": "参数校验失败",
  "data": [
    { "field": "username", "message": "用户名长度 3-32" },
    { "field": "email", "message": "邮箱格式不正确" }
  ]
}
```

前端根据 `code === "VALIDATION_FAILED"` 把 `data` 数组回填到表单（详见 [frontend/form-validation.md](../frontend/form-validation.md)）。

## 日志策略

| 异常类型              | 日志级别 | 是否打堆栈 |
| --------------------- | -------- | ---------- |
| `BusinessException`   | WARN     | 否         |
| 校验异常              | INFO     | 否         |
| `DataIntegrityViolationException` | WARN | 否（看根因 message） |
| 其他未捕获异常        | ERROR    | 是         |
| 鉴权异常（已被 EntryPoint 处理）| 不在此处记录 |   |

> 不要给业务异常打 ERROR 级别，会污染告警。

## 异常 → HTTP 状态码

| 错误类型              | HTTP 状态 |
| --------------------- | --------- |
| 校验失败              | 400       |
| 业务异常（默认）      | 400       |
| 未登录                | 401       |
| 无权限                | 403       |
| 资源不存在            | 404       |
| 方法不允许            | 405       |
| 资源冲突 / 并发更新   | 409       |
| 限流                  | 429       |
| 系统异常              | 500       |

> 前端不强依赖 HTTP 状态，统一看 `body.code`。但合理的状态码方便日志监控（如 5xx 触发告警）。

## 不要在 Controller 写 try/catch

```java
// 反模式
@PostMapping
public Result<OrderVO> create(@Valid @RequestBody OrderCreateDTO dto) {
    try {
        return Result.success(orderService.create(dto));
    } catch (Exception e) {
        log.error("创建失败", e);
        return Result.fail("ORDER_CREATE_FAILED", "创建失败");
    }
}

// 正确
@PostMapping
public Result<OrderVO> create(@Valid @RequestBody OrderCreateDTO dto) {
    return Result.success(orderService.create(dto));
}
```

## Service 内部 try/catch

仅在以下场景使用：

1. 调用第三方服务失败需降级（捕获后返回兜底数据）
2. 想把检查异常转为业务异常
3. 资源清理（用 try-with-resources 优先）

```java
try {
    smsService.send(phone, msg);
} catch (SmsException e) {
    log.warn("短信发送失败 phone={}", phone, e);
    // 降级：写入待发队列
    smsRetryQueue.push(phone, msg);
}
```

不要：

```java
// 反模式：吞错返回 null
try {
    return userMapper.selectById(id);
} catch (Exception e) {
    log.error("err", e);
    return null;
}
```

## 自定义异常体系（可选）

如果业务复杂，可在 `BusinessException` 之上派生：

```java
public class NotFoundException extends BusinessException { ... }
public class ConflictException extends BusinessException { ... }
public class ForbiddenException extends BusinessException { ... }
```

便于按异常类型统一打不同级别日志，但**多数项目用 ErrorCode 即可**，不必过度设计。

## 最佳实践

1. **错误码集中维护**于 `ErrorCode`
2. **业务异常用 `BusinessException`**
3. **Controller 不写 try/catch**
4. **未识别异常打 ERROR + 堆栈**
5. **校验失败返回字段级错误**

## 反模式

- 在 Controller 里写 `try { ... } catch (Exception e) { return fail(...) }`
- 把业务异常打 ERROR 级别
- 自定义大量异常类（`OrderNotFoundException`、`UserNotFoundException`、…），用 ErrorCode 即可
- `Result.fail("订单不存在")` 不带 code（前端无法编程判断）
- 全部错误返回 `200 OK + success:false`（合理用 HTTP 状态便于监控）
