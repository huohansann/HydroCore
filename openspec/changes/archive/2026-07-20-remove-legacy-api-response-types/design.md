## Context

HydroCore 已经有统一的 `ApiResponse<T>` 对外契约，但后端源码里仍残留面向前端返回的旧包装类型和 `ResponseBodyAdvice` 包装入口。它们继续存在会让新代码误以为还能回到旧模式，也会让静态扫描和代码审查很难一眼判断哪些类型是真正允许的。

本次变更只处理 frontend-facing 的返回类型边界，不改上游 Feign 集成契约。`com.siact.api...R` 这类类型属于外部/上游对接，不在本次删除范围内。

## Goals / Non-Goals

**Goals:**

- 直接删除后端对前端返回的旧包装类型和 `ResponseBodyAdvice` 的正常包装职责。
- 保持控制器、异常处理器和安全处理器只输出 `ApiResponse<T>`。
- 让静态扫描能够明确区分 frontend-facing 旧包装与上游 Feign 集成类型。

**Non-Goals:**

- 不修改前端 HTTP adapter 的解包协议。
- 不修改线程池、日志或其他 runtime-contracts 范围外的能力。
- 不重构上游 Feign 的 `com.siact.api...R` 类型或其调用方式。

## Decisions

### 1. 直接删除 frontend-facing 旧包装，而不是保留废弃门面

`common.R`、`common.result.R`、`ResponseEntity` 这类类型对前端 API 来说已经没有保留价值。继续保留只会让后续代码继续依赖它们，削弱统一返回类型的约束力。

Alternative considered: keep them as deprecated compatibility shells. Rejected because它会延长双轨期，且当前 runtime-contracts 已经提供了新的标准返回类型，继续保留只会增加噪音。

### 2. 明确把上游 Feign 响应类型排除在删除范围外

`com.siact.api...R` 只用于上游集成，不属于前端返回契约。把它们一起删掉会把本次改动从“清理前端 API 边界”扩大成“重写外部集成契约”，不符合当前目标。

Alternative considered: 把 Feign 返回也统一改成 `ApiResponse<T>`。Rejected because that requires external contract coordination and is a different change boundary.

### 3. 用静态扫描和测试锁定删除结果

删除旧类型后，最容易复发的问题是新代码又把它们引回控制器或异常处理器。用静态扫描和契约测试做门禁，比只靠代码审查更稳。

Alternative considered: rely on manual review only. Rejected because this change is about tightening a low-level contract, and regressions are cheap to prevent automatically.

## Risks / Trade-offs

- 旧类型删除后，残留 import 或引用会先导致编译失败。→ 先收敛调用点，再删定义，最后跑编译和静态扫描。
- Feign 与 frontend-facing 类型名都含 `R`，容易混淆。→ 约束扫描范围只看 backend frontend-facing 代码路径，明确排除上游集成包。
- 如果未来有人想重新引入旧包装，会缺少兼容层。→ 这是刻意选择；需要新 change 才能恢复。

## Migration Plan

1. 删除 frontend-facing 旧包装类型和 `ResponseBodyAdvice` 的正常包装职责。
2. 将仍然返回旧包装的控制器、异常处理器和安全处理器迁移到 `ApiResponse<T>`.
3. 更新静态扫描与测试，确认 frontend-facing 代码路径不再出现这些旧类型。
4. 保留上游 Feign `com.siact.api...R` 不变，并在扫描中明确排除它们。

## Open Questions

- 无。边界已经明确：frontend-facing 旧包装删除，Feign 集成类型保留。
