## Why

HydroCore 的对外 API 规范已经统一，但后端源码里仍保留着面向前端返回的旧包装类型和相关兼容入口，容易让新代码继续依赖这些旧形态。现在把这些旧类型直接删掉，可以把“统一返回类型”从契约层落到源码层，减少误用和回流。

## What Changes

- 删除后端面向前端返回的旧包装类型及其剩余入口，包括 `common.R`、`common.result.R`、`ResponseEntity` 和 `ResponseBodyAdvice` 的对外包装职责。
- 将仍然引用这些旧类型的后端控制器、异常处理器和安全处理器全部收敛到统一的 `ApiResponse<T>`。
- 更新后端静态扫描和测试，保证未来不能再把旧包装类型引回对外返回路径。
- 保留上游 Feign 的 `com.siact.api...R` 返回类型不变，这类类型只用于对接外部/上游接口，不属于本次删除范围。

## Capabilities

### New Capabilities
- 无

### Modified Capabilities
- `runtime-contracts`: 将后端对前端的返回契约进一步收紧为“源码中不再保留旧包装类型”，同时明确上游 Feign 响应类型不在删除范围内。

## Impact

- 影响 `hydrocore-be` 中面向前端的控制器、全局异常处理器、安全处理器、相关测试和静态扫描。
- 影响 `openspec/specs/runtime-contracts/spec.md`，需要把“旧包装不再作为对外返回类型”的要求收紧为“旧包装类型直接删除”。
- 不影响上游 Feign 接口的 `com.siact.api...R` 类型，也不影响前端 HTTP adapter 的现有解包契约。
