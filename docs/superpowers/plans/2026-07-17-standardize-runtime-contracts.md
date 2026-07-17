---
change: standardize-runtime-contracts
design-doc: docs/superpowers/specs/2026-07-17-standardize-runtime-contracts-design.md
base-ref: unborn-head
---

# Standardize Runtime Contracts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立 HydroCore 运行期基础规约：后端显式返回 `ApiResponse<T>`，前端只消费一个 envelope，线程池统一配置，日志统一且可排障。

**Architecture:** 后端新增唯一响应类型 `ApiResponse<T>`，所有 controller、异常处理器和安全处理器直接返回或写出该规约，不再依赖 `ResponseBodyAdvice` 隐式包装。前端把 envelope 解析集中到 HTTP adapter，成功只返回业务 `data`，失败 reject `AppHttpError`。线程池通过统一 properties 和 builder 创建 IO、CPU、background、event executor，日志通过 SLF4J 参数化输出和静态扫描防止漂移。

**Tech Stack:** Java 8、Spring Boot 2.6.13、Spring MVC、Spring Security、Lombok、JUnit 5、Vue 3、TypeScript 5.9、Axios 1.13、pnpm。

## Global Constraints

- 语言：zh-CN。
- 只修改本 change 范围内文件；本计划本身只写入 `docs/superpowers/plans/2026-07-17-standardize-runtime-contracts.md`。
- 实施时不要提交 Git；当前仓库尚无提交历史，`git rev-parse HEAD` 当前失败，输出包含 `fatal: ambiguous argument 'HEAD': unknown revision or path not in the working tree.`，因此 `base-ref` 记录为 `unborn-head`。
- 后端 API 必须显式返回 `ApiResponse<T>`；`common.R`、`common.result.R`、`ResponseEntity` 不再作为 API 对外返回类型。
- 删除或停用 `ResponseBodyAdvice` 的统一包装职责；迁移完成后不得依赖 advice 包装裸对象、字符串或旧包装类。
- 前端业务调用方不再重复消费 envelope；默认只通过 HTTP adapter 解析 envelope。
- 运行期代码不得新增 `System.out.println` 或 `printStackTrace`。
- 验证命令必须至少覆盖：`mvn -q -DskipTests compile`、`mvn -q test`、`pnpm.cmd run build`、旧返回类型扫描、日志禁用项扫描。

---

## File Structure

### 后端响应规约

- Create: `hydrocore-be/src/main/java/com/siact/hydrocore/common/api/ApiResponse.java`
  - 唯一对外 API envelope，字段为 `success`、`code`、`message`、`data`、`traceId`。
- Create: `hydrocore-be/src/main/java/com/siact/hydrocore/common/api/ApiResponseCode.java`
  - 统一错误码枚举或错误码接口，收敛现有 `ResponseEnum` / `ResultCode` 的对外语义。
- Create: `hydrocore-be/src/main/java/com/siact/hydrocore/common/web/TraceIdResolver.java`
  - 从 MDC 或请求头解析 traceId；无请求上下文时返回空字符串或生成兼容值。
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/entity/ResponseEntity.java`
  - 标记 `@Deprecated`，仅保留编译兼容，不作为 API 返回类型使用。
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/R.java`
  - 标记 `@Deprecated`，不再用于 controller 返回。
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/result/R.java`
  - 标记 `@Deprecated`，不再用于 controller 返回。
- Delete: `hydrocore-be/src/main/java/com/siact/hydrocore/core/web/advice/ResponseBodyAdvice.java`
  - 删除统一包装职责。若删除导致编译失败，说明还有入口依赖隐式包装，必须显式迁移入口。
- Review: `hydrocore-be/src/main/java/com/siact/hydrocore/common/annotation/NoResponseAdvice.java`
  - 删除无引用注解，或仅在明确例外入口保留；本 change 目标是正常 API 不使用 response advice。
- Review: `hydrocore-be/src/main/java/com/siact/hydrocore/common/annotation/SuccessMessage.java`
  - 删除无引用注解，或迁移成 controller 显式 message。

### 后端入口迁移

- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/exception/GlobalExceptionHandler.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/core/security/handler/AuthenticationEntryPointImpl.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/core/security/handler/LogoutSuccessHandlerImpl.java`
- Modify controller files:
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/base/controller/ConfigFieldStoreController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/base/controller/DicController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/base/controller/FrontendCacheController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/base/controller/TplController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/device/controller/DeviceMappingController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/device/controller/DeviceRealtimeController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/controller/AuthController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/controller/SysConfigController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/controller/SysMenuController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/controller/SysOrganizationController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/controller/SysRoleController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/controller/SysUserController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/sec/controller/BaseDataController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/sec/controller/DataController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/sec/controller/DevController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/sec/controller/PropController.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/sec/controller/SecInsController.java`

### 前端 HTTP 规约

- Create: `hydrocore-fe/src/libs/http/envelope.ts`
  - 定义 `ApiEnvelope<T>`、`isApiEnvelope`、`unwrapApiEnvelope`、`toAppHttpError`。
- Modify: `hydrocore-fe/src/libs/http/axios.ts`
  - 成功 envelope resolve `data`，失败 envelope reject `AppHttpError` 并 emit `http:error`。
- Modify: `hydrocore-fe/src/services/event/http/types.ts`
  - 扩展 `AppHttpError`，包含 `code`、`status`、`message`、`traceId`、`raw`、`action`。
- Modify: `hydrocore-fe/src/services/event/http/error-mapper.ts`
- Modify: `hydrocore-fe/src/services/event/http/error-handler.ts`
- Modify: `hydrocore-fe/src/libs/eventbus/events.ts`
  - 将 `http:error` 事件类型从 `AxiosError` 调整为 `AppHttpError`。
- Modify request modules:
  - `hydrocore-fe/src/request/auth.ts`
  - `hydrocore-fe/src/request/cache.ts`
  - `hydrocore-fe/src/request/charts.ts`
  - `hydrocore-fe/src/request/device/mapping.ts`
  - `hydrocore-fe/src/request/device/realtime.ts`
  - `hydrocore-fe/src/request/system/config.ts`
  - `hydrocore-fe/src/request/system/menu.ts`
  - `hydrocore-fe/src/request/system/org.ts`
  - `hydrocore-fe/src/request/system/role.ts`
  - `hydrocore-fe/src/request/system/user.ts`
- Modify known page consumers that currently read `{ message, data }` from old response:
  - `hydrocore-fe/src/views/system/config/index.tsx`
  - `hydrocore-fe/src/views/system/config/components/config-form.tsx`
  - `hydrocore-fe/src/views/system/menu/index.tsx`
  - `hydrocore-fe/src/views/system/menu/components/menu-form.tsx`
  - `hydrocore-fe/src/views/system/org/index.tsx`
  - `hydrocore-fe/src/views/system/org/components/org-form.tsx`
  - `hydrocore-fe/src/views/system/points/index.tsx`
  - `hydrocore-fe/src/views/system/points/components/mapping-form.tsx`
  - `hydrocore-fe/src/views/system/role/index.tsx`
  - `hydrocore-fe/src/views/system/role/components/assign-menus.tsx`
  - `hydrocore-fe/src/views/system/role/components/role-form.tsx`
  - `hydrocore-fe/src/views/system/user/index.tsx`
  - `hydrocore-fe/src/views/system/user/components/assign-roles.tsx`
  - `hydrocore-fe/src/views/system/user/components/reset-password.tsx`
  - `hydrocore-fe/src/views/system/user/components/user-form.tsx`
  - `hydrocore-fe/src/views/trends/route.tsx`

### 线程池和日志

- Create: `hydrocore-be/src/main/java/com/siact/hydrocore/core/common/config/RuntimeThreadPoolProperties.java`
- Create: `hydrocore-be/src/main/java/com/siact/hydrocore/core/common/config/ThreadPoolTaskExecutorBuilder.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/core/common/config/ThreadPoolConfig.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/core/event/config/EventConfiguration.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/core/event/config/EventProperties.java`
- Modify: `hydrocore-be/src/main/resources/nacos/hydrocore.yml`
- Modify: `hydrocore-be/src/main/resources/nacos/README.md`
- Modify logging hot spots:
  - `hydrocore-be/src/main/java/com/siact/hydrocore/common/exception/GlobalExceptionHandler.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/common/utils/TimeUtil.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/common/utils/JepUtils.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/common/utils/ExcelUtils.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/common/redis/RedisService.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/sec/utils/ConvertUtils.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/tdengine/service/TaosDataServiceImpl.java`
  - `hydrocore-be/src/main/java/com/siact/hydrocore/sec/sevice/impl/DataServiceImpl.java`

### Tests and Verification

- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/common/api/ApiResponseTest.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/common/exception/GlobalExceptionHandlerTest.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/core/security/handler/SecurityHandlerResponseTest.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/core/common/config/RuntimeThreadPoolPropertiesTest.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/architecture/RuntimeContractStaticScanTest.java`
- Create: `hydrocore-fe/scripts/verify-http-contract.mjs`
- Modify: `hydrocore-fe/package.json`
  - 增加 `verify:http-contract` 脚本，不新增测试框架依赖。

---

## Task 1: Runtime Contract Inventory

**Files:**
- Read: all backend controller files listed in File Structure
- Read: `hydrocore-be/src/main/java/com/siact/hydrocore/common/exception/GlobalExceptionHandler.java`
- Read: `hydrocore-be/src/main/java/com/siact/hydrocore/core/security/handler/AuthenticationEntryPointImpl.java`
- Read: `hydrocore-be/src/main/java/com/siact/hydrocore/core/security/handler/LogoutSuccessHandlerImpl.java`
- Read: `hydrocore-be/src/main/java/com/siact/hydrocore/core/web/advice/ResponseBodyAdvice.java`
- Read: `hydrocore-fe/src/libs/http/axios.ts`
- Read: `hydrocore-fe/src/request/**/*.ts`
- Read: `hydrocore-fe/src/views/**/*.tsx`

**Interfaces:**
- Consumes: existing return types and frontend call-site assumptions.
- Produces: concrete migration list used by Tasks 2, 3, and 4.

- [ ] **Step 1: Scan backend API return wrappers**

Run:

```powershell
rg -n "com\.siact\.hydrocore\.common\.R|com\.siact\.hydrocore\.common\.result\.R|ResponseEntity<|ResponseEntity\.|ResponseBodyAdvice|@RestController|@RestControllerAdvice" hydrocore-be\src\main\java
```

Expected: output lists each old wrapper usage and every REST entry point. Record entries under these categories in the task notes of the implementation session: controller return, exception handler return, security handler writer, advice wrapper, external Feign DTO.

- [ ] **Step 2: Scan backend raw and String response risks**

Run:

```powershell
rg -n "public .* (String|Object|Map|List|Boolean|boolean|void)\s+\w+\(" hydrocore-be\src\main\java\com\siact\hydrocore\module hydrocore-be\src\main\java\com\siact\hydrocore\sec\controller
```

Expected: every REST method returning a raw business type is identified for explicit `ApiResponse<T>` migration. File download or stream endpoints are marked as explicit exceptions only when they write `HttpServletResponse`, return `ResponseEntity<Resource>`, or use a documented framework response.

- [ ] **Step 3: Scan frontend envelope consumers**

Run:

```powershell
rg -n "ResponseModel|response\.data|const \{ message, data \}|result\.data|result\.message|AxiosError<ResponseModel|code !== 200|http:error" hydrocore-fe\src
```

Expected: all request modules still typed as `ResponseModel<T>` and all page components reading old `{ message, data }` are listed for Task 4.

- [ ] **Step 4: Scan logging violations**

Run:

```powershell
rg -n "System\.out\.println|printStackTrace\(|log\.error\(\"\\{\\} error|发生.*异常" hydrocore-be\src\main\java
```

Expected: known violations include `TimeUtil`, `JepUtils`, `ExcelUtils`, `RedisService`, `ConvertUtils`, `GlobalExceptionHandler`, plus any TDengine or SEC service vague logs found by the scan.

- [ ] **Step 5: Do not commit**

Run:

```powershell
git status --short
```

Expected: working tree may contain existing uncommitted changes. Do not run `git add` or `git commit`; record only the files this implementation will own.

## Task 2: Backend Canonical ApiResponse

**Files:**
- Create: `hydrocore-be/src/main/java/com/siact/hydrocore/common/api/ApiResponse.java`
- Create: `hydrocore-be/src/main/java/com/siact/hydrocore/common/api/ApiResponseCode.java`
- Create: `hydrocore-be/src/main/java/com/siact/hydrocore/common/web/TraceIdResolver.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/entity/ResponseEntity.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/R.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/result/R.java`
- Test: `hydrocore-be/src/test/java/com/siact/hydrocore/common/api/ApiResponseTest.java`

**Interfaces:**
- Consumes: existing success code `200`, validation code `400`, unauthorized `401`, forbidden `403`, server error `500`.
- Produces:
  - `ApiResponse<T>` with getters `getSuccess()`, `getCode()`, `getMessage()`, `getData()`, `getTraceId()`.
  - Static factories `success(T data)`, `success(T data, String message)`, `fail(int code, String message)`, `fail(int code, String message, T data)`.
  - `ApiResponseCode` values `SUCCESS(200, "操作成功")`, `BAD_REQUEST(400, "请求参数错误")`, `UNAUTHORIZED(401, "未登录或token已过期")`, `FORBIDDEN(403, "没有相关权限")`, `INTERNAL_ERROR(500, "服务器内部错误")`.

- [ ] **Step 1: Write ApiResponse tests first**

Create `ApiResponseTest.java` with these cases:

```java
package com.siact.hydrocore.common.api;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ApiResponseTest {
    @Test
    void successResponseContainsCanonicalFields() {
        ApiResponse<String> response = ApiResponse.success("payload", "操作成功", "trace-1");

        assertThat(response.getSuccess()).isTrue();
        assertThat(response.getCode()).isEqualTo(200);
        assertThat(response.getMessage()).isEqualTo("操作成功");
        assertThat(response.getData()).isEqualTo("payload");
        assertThat(response.getTraceId()).isEqualTo("trace-1");
    }

    @Test
    void failureResponseContainsCanonicalFields() {
        ApiResponse<Void> response = ApiResponse.fail(400, "名称不能为空", "trace-2");

        assertThat(response.getSuccess()).isFalse();
        assertThat(response.getCode()).isEqualTo(400);
        assertThat(response.getMessage()).isEqualTo("名称不能为空");
        assertThat(response.getData()).isNull();
        assertThat(response.getTraceId()).isEqualTo("trace-2");
    }
}
```

Run:

```powershell
mvn -q -Dtest=com.siact.hydrocore.common.api.ApiResponseTest test
```

Expected: FAIL because `ApiResponse` does not exist yet.

- [ ] **Step 2: Add `ApiResponseCode`**

Implement `ApiResponseCode` as an enum with `getCode()` and `getMessage()`:

```java
package com.siact.hydrocore.common.api;

public enum ApiResponseCode {
    SUCCESS(200, "操作成功"),
    BAD_REQUEST(400, "请求参数错误"),
    UNAUTHORIZED(401, "未登录或token已过期"),
    FORBIDDEN(403, "没有相关权限"),
    INTERNAL_ERROR(500, "服务器内部错误");

    private final int code;
    private final String message;

    ApiResponseCode(int code, String message) {
        this.code = code;
        this.message = message;
    }

    public int getCode() {
        return code;
    }

    public String getMessage() {
        return message;
    }
}
```

Run:

```powershell
mvn -q -DskipTests compile
```

Expected: FAIL only if `ApiResponse` references are still missing; no enum syntax errors.

- [ ] **Step 3: Add `ApiResponse<T>`**

Implement `ApiResponse<T>` as immutable enough for API use and Jackson serialization:

```java
package com.siact.hydrocore.common.api;

import lombok.AccessLevel;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

@Data
@Builder
@NoArgsConstructor(access = AccessLevel.PRIVATE)
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class ApiResponse<T> implements Serializable {
    private static final long serialVersionUID = 1L;

    private Boolean success;
    private Integer code;
    private String message;
    private T data;
    private String traceId;

    public static <T> ApiResponse<T> success(T data) {
        return success(data, ApiResponseCode.SUCCESS.getMessage(), "");
    }

    public static <T> ApiResponse<T> success(T data, String message) {
        return success(data, message, "");
    }

    public static <T> ApiResponse<T> success(T data, String message, String traceId) {
        return ApiResponse.<T>builder()
                .success(true)
                .code(ApiResponseCode.SUCCESS.getCode())
                .message(message)
                .data(data)
                .traceId(traceId)
                .build();
    }

    public static <T> ApiResponse<T> fail(int code, String message) {
        return fail(code, message, null, "");
    }

    public static <T> ApiResponse<T> fail(int code, String message, String traceId) {
        return fail(code, message, null, traceId);
    }

    public static <T> ApiResponse<T> fail(int code, String message, T data, String traceId) {
        return ApiResponse.<T>builder()
                .success(false)
                .code(code)
                .message(message)
                .data(data)
                .traceId(traceId)
                .build();
    }
}
```

Run:

```powershell
mvn -q -Dtest=com.siact.hydrocore.common.api.ApiResponseTest test
```

Expected: PASS for `ApiResponseTest`.

- [ ] **Step 4: Add traceId resolver**

Create `TraceIdResolver`:

```java
package com.siact.hydrocore.common.web;

import org.slf4j.MDC;
import org.springframework.web.context.request.RequestAttributes;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;

public final class TraceIdResolver {
    private static final String TRACE_ID = "traceId";
    private static final String TRACE_HEADER = "X-Trace-Id";

    private TraceIdResolver() {
    }

    public static String currentTraceId() {
        String fromMdc = MDC.get(TRACE_ID);
        if (fromMdc != null && !fromMdc.trim().isEmpty()) {
            return fromMdc;
        }
        RequestAttributes attributes = RequestContextHolder.getRequestAttributes();
        if (attributes instanceof ServletRequestAttributes) {
            HttpServletRequest request = ((ServletRequestAttributes) attributes).getRequest();
            String fromHeader = request.getHeader(TRACE_HEADER);
            if (fromHeader != null && !fromHeader.trim().isEmpty()) {
                return fromHeader;
            }
        }
        return "";
    }
}
```

Run:

```powershell
mvn -q -DskipTests compile
```

Expected: PASS or only unrelated existing compile errors. `TraceIdResolver` compiles under Java 8.

- [ ] **Step 5: Deprecate legacy wrappers**

Add `@Deprecated` and class-level Javadoc to `ResponseEntity`, `common.R`, and `common.result.R`:

```java
/**
 * @deprecated API responses must use {@link com.siact.hydrocore.common.api.ApiResponse}.
 */
@Deprecated
```

Expected: code still compiles, but future use is visible during reviews and IDE inspection. Do not add adapter methods that encourage continued external use.

## Task 3: Backend Entry Points Return ApiResponse Directly

**Files:**
- Modify: `GlobalExceptionHandler.java`
- Modify: `AuthenticationEntryPointImpl.java`
- Modify: `LogoutSuccessHandlerImpl.java`
- Modify: all controller files listed in File Structure
- Delete: `ResponseBodyAdvice.java`
- Test: `GlobalExceptionHandlerTest.java`
- Test: `SecurityHandlerResponseTest.java`
- Test: `RuntimeContractStaticScanTest.java`

**Interfaces:**
- Consumes: `ApiResponse<T>` and `TraceIdResolver.currentTraceId()`.
- Produces: backend API entry points that return `ApiResponse<T>` directly and no normal API wrapping by advice.

- [ ] **Step 1: Write static scan test for forbidden API return types**

Create `RuntimeContractStaticScanTest.java` with a source scan that excludes tests and external generated files:

```java
package com.siact.hydrocore.architecture;

import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;

class RuntimeContractStaticScanTest {
    private static final Path MAIN_JAVA = Paths.get("src/main/java");

    @Test
    void apiEntrypointsDoNotReturnLegacyWrappersOrUseResponseAdvice() throws IOException {
        try (Stream<Path> files = Files.walk(MAIN_JAVA)) {
            List<String> violations = files
                    .filter(path -> path.toString().endsWith(".java"))
                    .filter(path -> !path.toString().endsWith("common/R.java"))
                    .filter(path -> !path.toString().endsWith("common/result/R.java"))
                    .filter(path -> !path.toString().endsWith("common/entity/ResponseEntity.java"))
                    .filter(path -> !path.toString().contains("sec/utils/SiactSecApiFeignUtil.java"))
                    .flatMap(path -> scan(path,
                            "com.siact.hydrocore.common.R",
                            "com.siact.hydrocore.common.result.R",
                            "com.siact.hydrocore.common.entity.ResponseEntity",
                            "ResponseEntity<",
                            "ResponseEntity.",
                            "implements org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice"))
                    .collect(Collectors.toList());

            assertThat(violations).isEmpty();
        }
    }

    @Test
    void runtimeCodeDoesNotUseConsoleOutputOrPrintStackTrace() throws IOException {
        try (Stream<Path> files = Files.walk(MAIN_JAVA)) {
            List<String> violations = files
                    .filter(path -> path.toString().endsWith(".java"))
                    .flatMap(path -> scan(path, "System.out.println", "printStackTrace("))
                    .collect(Collectors.toList());

            assertThat(violations).isEmpty();
        }
    }

    private Stream<String> scan(Path path, String... tokens) {
        try {
            String content = new String(Files.readAllBytes(path), StandardCharsets.UTF_8);
            return Stream.of(tokens)
                    .filter(content::contains)
                    .map(token -> path + " contains " + token);
        } catch (IOException e) {
            throw new IllegalStateException("Failed to scan " + path, e);
        }
    }
}
```

Run:

```powershell
mvn -q -Dtest=com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
```

Expected: FAIL before migration because `ResponseBodyAdvice`, `ResponseEntity`, `System.out.println`, and `printStackTrace` still exist in runtime paths.

- [ ] **Step 2: Migrate `GlobalExceptionHandler`**

Change method return types from `ResponseEntity<String>` to `ApiResponse<String>` and log with accurate levels. Use this shape:

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse<String> handleMethodArgumentNotValidException(MethodArgumentNotValidException e, HandlerMethod method) {
        String traceId = TraceIdResolver.currentTraceId();
        String message = e.getBindingResult().getAllErrors().isEmpty()
                ? ApiResponseCode.BAD_REQUEST.getMessage()
                : e.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        log.warn("request validation failed, handler={}, traceId={}, message={}", handlerName(method), traceId, message);
        return ApiResponse.fail(ApiResponseCode.BAD_REQUEST.getCode(), message, traceId);
    }

    @ExceptionHandler(Exception.class)
    public ApiResponse<String> handleException(Exception e, HandlerMethod method) {
        String traceId = TraceIdResolver.currentTraceId();
        log.error("unhandled request exception, handler={}, traceId={}, exceptionType={}, message={}",
                handlerName(method), traceId, e.getClass().getName(), e.getMessage(), e);
        return ApiResponse.fail(ApiResponseCode.INTERNAL_ERROR.getCode(), ApiResponseCode.INTERNAL_ERROR.getMessage(), traceId);
    }

    private String handlerName(HandlerMethod method) {
        return method == null ? "unknown" : method.getBeanType().getSimpleName() + "#" + method.getMethod().getName();
    }
}
```

Expected: validation/auth/business failures do not log as unknown crashes; unknown exceptions hide stack details from client but keep stack trace in logs.

- [ ] **Step 3: Migrate security handlers**

In `AuthenticationEntryPointImpl` and `LogoutSuccessHandlerImpl`, serialize `ApiResponse`:

```java
ApiResponse<Void> result = ApiResponse.fail(
        ApiResponseCode.UNAUTHORIZED.getCode(),
        ApiResponseCode.UNAUTHORIZED.getMessage(),
        TraceIdResolver.currentTraceId());
```

For logout success:

```java
ApiResponse<Void> result = ApiResponse.success(null, ApiResponseCode.SUCCESS.getMessage(), TraceIdResolver.currentTraceId());
```

Run:

```powershell
mvn -q -DskipTests compile
```

Expected: PASS or compile failures only in controllers that still import old wrappers.

- [ ] **Step 4: Migrate controllers by module**

For each controller file, replace old imports and method signatures. Use this mapping:

```java
// Old
public R<List<DicVO>> list() {
    return R.success(service.list());
}

// New
public ApiResponse<List<DicVO>> list() {
    return ApiResponse.success(service.list());
}
```

For operations where page code previously consumed `{ message, data }`, return boolean payload and explicit message:

```java
public ApiResponse<Boolean> remove(@PathVariable Long id) {
    boolean removed = service.removeById(id);
    return ApiResponse.success(removed, removed ? "删除成功" : "删除失败");
}
```

Run after each module group:

```powershell
mvn -q -DskipTests compile
```

Expected: compile errors shrink after each group. Finish when no controller imports `com.siact.hydrocore.common.R`, `com.siact.hydrocore.common.result.R`, or `com.siact.hydrocore.common.entity.ResponseEntity`.

- [ ] **Step 5: Delete `ResponseBodyAdvice` wrapper**

Delete `hydrocore-be/src/main/java/com/siact/hydrocore/core/web/advice/ResponseBodyAdvice.java`. Then run:

```powershell
rg -n "ResponseBodyAdvice|NoResponseAdvice|SuccessMessage" hydrocore-be\src\main\java
```

Expected: no `ResponseBodyAdvice` implementation remains. `NoResponseAdvice` and `SuccessMessage` either have no references and can be deleted in the same task, or have explicit documented non-wrapping use.

- [ ] **Step 6: Add exception and security response tests**

Create tests that serialize handlers and assert canonical fields:

```java
assertThat(json).contains("\"success\":false");
assertThat(json).contains("\"code\":401");
assertThat(json).contains("\"message\":\"未登录或token已过期\"");
assertThat(json).contains("\"traceId\"");
```

Run:

```powershell
mvn -q -Dtest=com.siact.hydrocore.common.exception.GlobalExceptionHandlerTest,com.siact.hydrocore.core.security.handler.SecurityHandlerResponseTest test
```

Expected: PASS. Failure means an error path still emits an old envelope or omits `traceId`.

- [ ] **Step 7: Run backend response static scan**

Run:

```powershell
mvn -q -Dtest=com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
```

Expected: response wrapper section PASS after API migration; logging section may still fail until Task 6.

## Task 4: Frontend HTTP Adapter Consumes One Envelope

**Files:**
- Create: `hydrocore-fe/src/libs/http/envelope.ts`
- Modify: `hydrocore-fe/src/libs/http/axios.ts`
- Modify: `hydrocore-fe/src/services/event/http/types.ts`
- Modify: `hydrocore-fe/src/services/event/http/error-mapper.ts`
- Modify: `hydrocore-fe/src/services/event/http/error-handler.ts`
- Modify: `hydrocore-fe/src/libs/eventbus/events.ts`
- Modify: request modules and known page consumers listed in File Structure
- Create: `hydrocore-fe/scripts/verify-http-contract.mjs`
- Modify: `hydrocore-fe/package.json`

**Interfaces:**
- Consumes: backend `ApiResponse<T>` JSON envelope.
- Produces:
  - `ApiEnvelope<T>` with `success: boolean; code: number; message: string; data: T; traceId?: string`.
  - `AppHttpError` with `code?: number; status?: number; message: string; traceId?: string; raw?: unknown; action?: ...`.
  - `http.request<T>()` resolves to business `T`, not `ApiEnvelope<T>`.

- [ ] **Step 1: Add envelope helper**

Create `envelope.ts` with this API:

```ts
import type { AxiosError, AxiosResponse } from 'axios';
import type { AppHttpError } from '@/services/event/http/types';
import { HttpStatus } from '@/constants/HttpStatus';

export interface ApiEnvelope<T> {
  success: boolean;
  code: number;
  message: string;
  data: T;
  traceId?: string;
}

export function isApiEnvelope<T = unknown>(value: unknown): value is ApiEnvelope<T> {
  const body = value as Partial<ApiEnvelope<T>>;
  return !!body && typeof body === 'object' && typeof body.success === 'boolean' && typeof body.code === 'number' && typeof body.message === 'string';
}

export function toAppHttpError(error: AxiosError | ApiEnvelope<unknown>, status?: number, raw?: unknown): AppHttpError {
  if (isApiEnvelope(error)) {
    return {
      code: error.code,
      status,
      message: error.message,
      traceId: error.traceId,
      raw: raw ?? error,
      action: error.code === HttpStatus.UNAUTHORIZED ? { type: 'logout' } : { type: 'notify' },
    };
  }
  return {
    status: error.response?.status,
    message: error.message || '网络异常或请求失败',
    raw: error,
    action: error.response?.status === HttpStatus.UNAUTHORIZED ? { type: 'logout' } : { type: 'notify' },
  };
}

export function unwrapApiEnvelope<T>(response: AxiosResponse<ApiEnvelope<T>>): T {
  const envelope = response.data;
  if (!isApiEnvelope<T>(envelope)) {
    throw toAppHttpError({
      success: false,
      code: response.status,
      message: '后端响应结构不符合 ApiEnvelope 规约',
      data: undefined,
    }, response.status, response.data);
  }
  if (!envelope.success) {
    throw toAppHttpError(envelope, response.status, response.data);
  }
  return envelope.data;
}
```

Run:

```powershell
pnpm.cmd run build
```

Expected: FAIL until imports and adapter are migrated.

- [ ] **Step 2: Change `axios.ts` request semantics**

Make `request<T>` return business data:

```ts
public request = async <T>(config: AxiosRequestConfig): Promise<T> => {
  const response = await this.instance.request<ApiEnvelope<T>>(config);
  return unwrapApiEnvelope<T>(response);
};
```

In response interceptor, refresh token only and return response; error interceptor emits typed error:

```ts
async (error: AxiosError) => {
  this.refreshTokenIfNeeded(error.response);
  const appError = toAppHttpError(error);
  eventbus.emit('http:error', appError);
  return Promise.reject(appError);
}
```

Expected: `window.$message.error(message)` is removed from `axios.ts`; user-facing behavior goes through `http:error`.

- [ ] **Step 3: Update frontend error event types**

Change `events.ts`:

```ts
import type { AppHttpError } from '@/services/event/http/types';

export type AppEventTypes = {
  'auth:logout': { reason?: string };
  'http:error': AppHttpError;
  'websocket:error': IFrame;
  notify: { type: 'info' | 'success' | 'error' | 'warning'; message: string; title?: string; meta?: any };
};
```

Change `types.ts`:

```ts
export interface AppHttpError {
  code?: number;
  status?: number;
  message: string;
  traceId?: string;
  raw?: unknown;
  action?: { type: 'logout' | 'redirect' | 'notify'; redirectTo?: string };
}
```

Run:

```powershell
pnpm.cmd run build
```

Expected: remaining failures identify request modules and page consumers that still expect `ResponseModel<T>`.

- [ ] **Step 4: Update request modules to return business data**

Replace `http.request<ResponseModel<T>>` with `http.request<T>`. Example:

```ts
// Old
export const getCurrentUser = () => http.request<ResponseModel<UserDTO>>({ url: 'auth/current', method: 'GET' });

// New
export const getCurrentUser = () => http.request<UserDTO>({ url: 'auth/current', method: 'GET' });
```

For mutation APIs, return `boolean` or concrete DTO directly:

```ts
export const remove = (id: number) => http.request<boolean>({ url: `sysuser/${id}`, method: 'DELETE' });
```

Run:

```powershell
rg -n "http\.request<ResponseModel|AxiosError<ResponseModel|ResponseModel<" hydrocore-fe\src
```

Expected: no request module still calls `http.request<ResponseModel<...>>`. If global `ResponseModel` remains for unrelated compatibility, it must not be used by default HTTP calls.

- [ ] **Step 5: Update page consumers**

Replace old result envelope reads:

```ts
// Old
const { message, data } = await userApi.remove(id);
window.$message[data ? 'success' : 'error'](message);

// New
const removed = await userApi.remove(id);
window.$message[removed ? 'success' : 'error'](removed ? '删除成功' : '删除失败');
```

For create/update forms:

```ts
const saved = isEdit.value ? await api.update(formModel) : await api.create(formModel);
window.$message[saved ? 'success' : 'error'](saved ? '保存成功' : '保存失败');
```

Run:

```powershell
rg -n "const \{ message, data \}|result\.data|result\.message" hydrocore-fe\src\views
```

Expected: no page depends on old envelope `message/data` for normal request handling.

- [ ] **Step 6: Add frontend contract verification script**

Create `hydrocore-fe/scripts/verify-http-contract.mjs` as a static guard that checks adapter ownership:

```js
import fs from 'node:fs';
import path from 'node:path';

const root = process.cwd();
const files = [
  'src/libs/http/axios.ts',
  'src/libs/http/envelope.ts',
  'src/services/event/http/types.ts',
  'src/libs/eventbus/events.ts',
];

const read = file => fs.readFileSync(path.join(root, file), 'utf8');
const failures = [];

const axios = read(files[0]);
if (!axios.includes('unwrapApiEnvelope')) failures.push('axios.ts must use unwrapApiEnvelope');
if (axios.includes('window.$message.error')) failures.push('axios.ts must not display envelope errors directly');
if (axios.includes('resolve(response.data)')) failures.push('axios.ts must not resolve raw response.data');

const envelope = read(files[1]);
for (const token of ['ApiEnvelope', 'isApiEnvelope', 'toAppHttpError', 'unwrapApiEnvelope']) {
  if (!envelope.includes(token)) failures.push(`envelope.ts missing ${token}`);
}

const events = read(files[3]);
if (!events.includes("'http:error': AppHttpError")) failures.push('http:error must emit AppHttpError');

if (failures.length) {
  console.error(failures.join('\n'));
  process.exit(1);
}
console.log('frontend http contract verification passed');
```

Add script to `package.json`:

```json
"verify:http-contract": "node scripts/verify-http-contract.mjs"
```

Run:

```powershell
pnpm.cmd run verify:http-contract
pnpm.cmd run build
```

Expected: both PASS. If build fails, remaining TypeScript errors point to old envelope assumptions.

## Task 5: Unified Thread Pool Configuration and Builder

**Files:**
- Create: `RuntimeThreadPoolProperties.java`
- Create: `ThreadPoolTaskExecutorBuilder.java`
- Modify: `ThreadPoolConfig.java`
- Modify: `EventConfiguration.java`
- Modify: `EventProperties.java`
- Modify: `hydrocore-be/src/main/resources/nacos/hydrocore.yml`
- Modify: `hydrocore-be/src/main/resources/nacos/README.md`
- Test: `RuntimeThreadPoolPropertiesTest.java`

**Interfaces:**
- Consumes: existing bean names `threadIoPoolTaskExecutor`, `threadCpuPoolTaskExecutor`, `backgroundTaskExecutor`, `eventTaskExecutor`.
- Produces: property-driven executors with `coreSize`, `maxSize`, `queueCapacity`, `keepAliveSeconds`, `threadNamePrefix`, `allowCoreThreadTimeout`, `waitForTasksToCompleteOnShutdown`, `awaitTerminationSeconds`, `rejectionPolicy`.

- [ ] **Step 1: Write thread pool binding test**

Create `RuntimeThreadPoolPropertiesTest.java`:

```java
package com.siact.hydrocore.core.common.config;

import org.junit.jupiter.api.Test;
import org.springframework.boot.context.properties.bind.Bindable;
import org.springframework.boot.context.properties.bind.Binder;
import org.springframework.boot.env.OriginTrackedMapPropertySource;
import org.springframework.core.env.StandardEnvironment;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RuntimeThreadPoolPropertiesTest {
    @Test
    void bindsRuntimeThreadPoolOverrides() {
        Map<String, Object> source = new HashMap<>();
        source.put("hydrocore.thread-pools.io.core-size", "8");
        source.put("hydrocore.thread-pools.io.max-size", "16");
        source.put("hydrocore.thread-pools.io.queue-capacity", "256");
        source.put("hydrocore.thread-pools.io.thread-name-prefix", "hydro-io-");

        StandardEnvironment environment = new StandardEnvironment();
        environment.getPropertySources().addFirst(new OriginTrackedMapPropertySource("test", source));

        RuntimeThreadPoolProperties properties = Binder.get(environment)
                .bind("hydrocore.thread-pools", Bindable.of(RuntimeThreadPoolProperties.class))
                .orElseThrow(IllegalStateException::new);

        assertThat(properties.getIo().getCoreSize()).isEqualTo(8);
        assertThat(properties.getIo().getMaxSize()).isEqualTo(16);
        assertThat(properties.getIo().getQueueCapacity()).isEqualTo(256);
        assertThat(properties.getIo().getThreadNamePrefix()).isEqualTo("hydro-io-");
    }
}
```

Run:

```powershell
mvn -q -Dtest=com.siact.hydrocore.core.common.config.RuntimeThreadPoolPropertiesTest test
```

Expected: FAIL until properties class exists.

- [ ] **Step 2: Add `RuntimeThreadPoolProperties`**

Implement nested pool settings with conservative defaults:

```java
@Data
@ConfigurationProperties(prefix = "hydrocore.thread-pools")
public class RuntimeThreadPoolProperties {
    private Pool io = new Pool(16, 32, 1024, 60, "hydro-io-");
    private Pool cpu = new Pool(Runtime.getRuntime().availableProcessors(), Runtime.getRuntime().availableProcessors() * 2, 1024, 60, "hydro-cpu-");
    private Pool background = new Pool(2, 4, 10, 60, "hydro-background-");
    private Pool event = new Pool(10, 50, 1000, 60, "event-handler-");

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Pool {
        private int coreSize;
        private int maxSize;
        private int queueCapacity;
        private int keepAliveSeconds;
        private String threadNamePrefix;
        private boolean allowCoreThreadTimeout;
        private boolean waitForTasksToCompleteOnShutdown = true;
        private int awaitTerminationSeconds = 60;
        private String rejectionPolicy = "caller-runs";
    }
}
```

Expected: defaults cover IO、CPU、background、event. Keep event prefix compatible with existing `EventProperties`.

- [ ] **Step 3: Add executor builder**

Create builder:

```java
@Component
public class ThreadPoolTaskExecutorBuilder {
    public ThreadPoolTaskExecutor build(RuntimeThreadPoolProperties.Pool config) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(config.getCoreSize());
        executor.setMaxPoolSize(config.getMaxSize());
        executor.setQueueCapacity(config.getQueueCapacity());
        executor.setKeepAliveSeconds(config.getKeepAliveSeconds());
        executor.setThreadNamePrefix(config.getThreadNamePrefix());
        executor.setAllowCoreThreadTimeOut(config.isAllowCoreThreadTimeout());
        executor.setWaitForTasksToCompleteOnShutdown(config.isWaitForTasksToCompleteOnShutdown());
        executor.setAwaitTerminationSeconds(config.getAwaitTerminationSeconds());
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

Run:

```powershell
mvn -q -DskipTests compile
```

Expected: PASS or only references pending migration in config classes.

- [ ] **Step 4: Refactor `ThreadPoolConfig`**

Enable properties and inject builder:

```java
@Configuration
@EnableAsync
@EnableConfigurationProperties(RuntimeThreadPoolProperties.class)
@RequiredArgsConstructor
public class ThreadPoolConfig {
    private final RuntimeThreadPoolProperties properties;
    private final ThreadPoolTaskExecutorBuilder builder;

    @Bean(name = "threadIoPoolTaskExecutor")
    public Executor threadIoPoolTaskExecutor() {
        return builder.build(properties.getIo());
    }

    @Bean(name = "threadCpuPoolTaskExecutor")
    public Executor threadCpuPoolTaskExecutor() {
        return builder.build(properties.getCpu());
    }

    @Bean(name = "backgroundTaskExecutor")
    public Executor backgroundTaskExecutor() {
        return builder.build(properties.getBackground());
    }
}
```

Expected: remove static mutable fields and repeated hardcoded executor creation.

- [ ] **Step 5: Reuse builder in event configuration**

Keep bean name `eventTaskExecutor` and preserve `spring.event.thread-pool` compatibility. In `EventConfiguration`, convert `EventProperties.ThreadPool` to `RuntimeThreadPoolProperties.Pool`, then call the same builder.

Run:

```powershell
mvn -q -DskipTests compile
```

Expected: `AsyncEventPublisher` still injects `@Qualifier("eventTaskExecutor") Executor` successfully.

- [ ] **Step 6: Document defaults and CallerRunsPolicy**

Add `hydrocore.thread-pools` defaults to `hydrocore-be/src/main/resources/nacos/hydrocore.yml` and describe in `nacos/README.md`:

```yaml
hydrocore:
  thread-pools:
    io:
      core-size: 16
      max-size: 32
      queue-capacity: 1024
      keep-alive-seconds: 60
      thread-name-prefix: hydro-io-
      rejection-policy: caller-runs
```

Expected: operators know queue full behavior: `caller-runs` makes the submitting thread execute the task, applying backpressure instead of silently dropping work.

- [ ] **Step 7: Run thread pool tests**

Run:

```powershell
mvn -q -Dtest=com.siact.hydrocore.core.common.config.RuntimeThreadPoolPropertiesTest test
```

Expected: PASS.

## Task 6: Accurate Structured Logging

**Files:**
- Modify: `GlobalExceptionHandler.java`
- Modify: `TimeUtil.java`
- Modify: `JepUtils.java`
- Modify: `ExcelUtils.java`
- Modify: `RedisService.java`
- Modify: `ConvertUtils.java`
- Modify: `TaosDataServiceImpl.java`
- Modify: `DataServiceImpl.java`
- Test: `RuntimeContractStaticScanTest.java`

**Interfaces:**
- Consumes: `TraceIdResolver.currentTraceId()`.
- Produces: parameterized SLF4J logs with operation, stage, identifiers, traceId, and exception object where applicable.

- [ ] **Step 1: Replace console output and stack trace calls**

For every `printStackTrace()`:

```java
log.warn("time parse failed, input={}, pattern={}, traceId={}", input, pattern, TraceIdResolver.currentTraceId(), e);
```

For every `System.out.println` used as operational output:

```java
log.debug("calculation result, value={}, traceId={}", calc, TraceIdResolver.currentTraceId());
```

Expected: no runtime code uses console output for operational problems.

- [ ] **Step 2: Normalize `GlobalExceptionHandler` log levels**

Use `warn` for validation/auth/authorization/business denials and `error` only for unexpected exceptions. Example:

```java
log.warn("business exception handled, handler={}, traceId={}, message={}", handlerName(method), traceId, e.getMessage());
log.error("unhandled request exception, handler={}, traceId={}, exceptionType={}, message={}", handlerName(method), traceId, e.getClass().getName(), e.getMessage(), e);
```

Expected: logs describe the actual condition and do not label expected denials as system crashes.

- [ ] **Step 3: Normalize TDengine and SEC service logs**

Replace vague logs such as `"{} error"` or `"请求数字孪生...发生异常"` with operation-specific messages:

```java
log.warn("sec api returned empty data, operation=queryDevice, deviceCode={}, traceId={}", deviceCode, TraceIdResolver.currentTraceId());
log.error("tdengine query failed, operation=queryRealtimeData, sql={}, traceId={}", sql, TraceIdResolver.currentTraceId(), e);
```

Expected: each log contains operation and key identifier. Expected empty upstream data uses `warn` or `info`, not `error`.

- [ ] **Step 4: Run logging scan**

Run:

```powershell
rg -n "System\.out\.println|printStackTrace\(|log\.error\(\"\\{\\} error" hydrocore-be\src\main\java
```

Expected: no output. If output remains in non-runtime samples, move samples out of runtime source or justify with explicit exclusion in `RuntimeContractStaticScanTest`.

- [ ] **Step 5: Run static scan test**

Run:

```powershell
mvn -q -Dtest=com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
```

Expected: PASS for both old response wrapper and forbidden logging scans.

## Task 7: Verification Gates and Build Commands

**Files:**
- Test: backend tests created in Tasks 2, 3, 5, and 6
- Test: `hydrocore-fe/scripts/verify-http-contract.mjs`
- Modify only if needed: `hydrocore-fe/package.json`

**Interfaces:**
- Consumes: all implementation outputs.
- Produces: repeatable commands that protect the runtime contracts from drift.

- [ ] **Step 1: Backend compile**

Run from `hydrocore-be`:

```powershell
mvn -q -DskipTests compile
```

Expected: PASS. No Java compile errors, no missing imports after deleting `ResponseBodyAdvice`.

- [ ] **Step 2: Backend tests**

Run from `hydrocore-be`:

```powershell
mvn -q test
```

Expected: PASS. If project-level Surefire config still skips tests, run targeted tests explicitly:

```powershell
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.api.ApiResponseTest,com.siact.hydrocore.common.exception.GlobalExceptionHandlerTest,com.siact.hydrocore.core.security.handler.SecurityHandlerResponseTest,com.siact.hydrocore.core.common.config.RuntimeThreadPoolPropertiesTest,com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
```

Expected: targeted tests PASS and show contract regressions if old wrappers or forbidden logs return.

- [ ] **Step 3: Backend static command scan**

Run from repo root:

```powershell
rg -n "ResponseBodyAdvice|implements org\.springframework\.web\.servlet\.mvc\.method\.annotation\.ResponseBodyAdvice|ResponseEntity<|ResponseEntity\.|com\.siact\.hydrocore\.common\.R|com\.siact\.hydrocore\.common\.result\.R|System\.out\.println|printStackTrace\(" hydrocore-be\src\main\java
```

Expected: no output except deprecated wrapper class declarations if they remain for compatibility. No controller, exception handler, security handler, or advice implementation should appear.

- [ ] **Step 4: Frontend contract verification**

Run from `hydrocore-fe`:

```powershell
pnpm.cmd run verify:http-contract
```

Expected: output contains `frontend http contract verification passed`.

- [ ] **Step 5: Frontend production build**

Run from `hydrocore-fe`:

```powershell
pnpm.cmd run build
```

Expected: PASS. Vue type checking catches old `ResponseModel<T>` assumptions and pages still trying to read envelope `message/data`.

- [ ] **Step 6: OpenSpec task status**

After implementation and verification, update `openspec/changes/standardize-runtime-contracts/tasks.md` checkboxes that correspond to completed work. Do not commit.

Expected: tasks 1.1 through 6.5 reflect actual completion. Any failed verification remains unchecked with a note in the implementation handoff.

## Task 8: Final Contract Review Without Git Commit

**Files:**
- Read: `openspec/changes/standardize-runtime-contracts/specs/runtime-contracts/spec.md`
- Read: `openspec/changes/standardize-runtime-contracts/tasks.md`
- Read: implementation diffs

**Interfaces:**
- Consumes: verification outputs from Task 7.
- Produces: final implementation handoff for Comet verify phase.

- [ ] **Step 1: Check spec coverage**

Run:

```powershell
openspec.cmd status --change standardize-runtime-contracts --json
```

Expected: change still exists and implementation can be compared against the delta spec. The command does not require a Git commit.

- [ ] **Step 2: Check only intended files changed**

Run:

```powershell
git status --short
```

Expected: changed files match this plan plus OpenSpec task status updates. Because `git rev-parse HEAD` fails in this repository, do not try to create a commit or compare against `HEAD`.

- [ ] **Step 3: Record verification evidence**

Record exact command outcomes for:

```powershell
mvn -q -DskipTests compile
mvn -q test
pnpm.cmd run verify:http-contract
pnpm.cmd run build
rg -n "ResponseBodyAdvice|ResponseEntity<|ResponseEntity\.|System\.out\.println|printStackTrace\(" hydrocore-be\src\main\java
```

Expected: compile/test/build commands PASS; static scan has no forbidden runtime usage.

- [ ] **Step 4: Stop before commit**

Do not run:

```powershell
git add
git commit
git push
```

Expected: implementation remains uncommitted until a proper Git repository and baseline strategy exist.

---

## Self-Review

### Spec coverage

- Unified API response envelope: covered by Tasks 2, 3, and 7. `ApiResponse<T>` includes `success`、`code`、`message`、`data`、`traceId`; controller、exception handler、安全 handler 显式返回或写出该规约。
- No legacy response wrapper output: covered by Task 3 static scan and deletion of `ResponseBodyAdvice`; `common.R`、`common.result.R`、`ResponseEntity` are deprecated compatibility classes only.
- Frontend HTTP adapter consumes one contract: covered by Task 4 and Task 7; adapter resolves business `data` and rejects typed `AppHttpError`.
- Configurable runtime thread pools: covered by Task 5; IO、CPU、background、event share properties and builder with documented `caller-runs` behavior.
- Accurate structured logging: covered by Task 6 and static scan in Task 7; console output and stack trace calls are removed from runtime code.
- Contract verification gates: covered by Task 7; backend compile/tests/static scan and frontend contract/build commands are explicit.

### 占位符扫描

- This plan contains no implementation placeholders and no deferred marker words for future filling.
- Every task lists concrete files, commands, expected results, and sample code for the key interfaces.
- The plan intentionally omits Git commit steps because the user explicitly requested no commits before a proper Git repository exists.

### 类型一致性

- Backend canonical type is consistently `com.siact.hydrocore.common.api.ApiResponse<T>`.
- Backend canonical fields are consistently `success`、`code`、`message`、`data`、`traceId`.
- Frontend envelope type is consistently `ApiEnvelope<T>` with the same five fields.
- Frontend request API is consistently `http.request<T>() -> Promise<T>` where `T` is business data, not `ResponseModel<T>` or `ApiEnvelope<T>`.
- Frontend error type is consistently `AppHttpError` and `http:error` emits `AppHttpError`.
- Thread pool properties consistently use `RuntimeThreadPoolProperties.Pool` and existing bean names remain stable.
