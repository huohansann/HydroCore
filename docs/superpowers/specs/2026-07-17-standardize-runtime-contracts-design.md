---
comet_change: standardize-runtime-contracts
role: technical-design
canonical_spec: openspec
---

# Standardize Runtime Contracts Design

## 背景

HydroCore 当前已经适合作为水处理系统二开基线，但运行期基础契约仍不统一。后端同时存在 `common.R`、`common.result.R`、`ResponseEntity` 和 `ResponseBodyAdvice` 隐式包装；前端 HTTP adapter 对业务失败只提示但不统一 reject；普通线程池硬编码，事件线程池另有配置模型；日志中还存在 `System.out.println`、`printStackTrace` 和无法准确定位问题的模糊文案。

本设计的目标是一次性建立清晰的运行期规约，让后续二开业务不再继承这些不确定性。

## 总体方案

采用“新增规范响应类型 + 直接替换旧返回类型”的方案。

后端新增唯一规范响应类型，建议命名为 `ApiResponse<T>`，字段固定为：

- `success`: 业务是否成功。
- `code`: 业务/系统错误码。
- `message`: 可展示或可排障的简短信息。
- `data`: 业务载荷。
- `traceId`: 请求追踪标识，用于关联日志和前端错误提示。

所有 controller、全局异常处理器、安全处理器都显式返回 `ApiResponse<T>`。旧返回类型不再作为 API 对外返回类型。`ResponseBodyAdvice` 不继续作为统一包装机制；完成入口迁移后删除该类，或仅保留与明确例外相关的非包装逻辑。

## 后端响应规约

新增 `ApiResponse<T>` 及工厂方法，例如：

- `success(data)`
- `success(data, message)`
- `fail(code, message)`
- `fail(code, message, data)`

错误码收敛时优先复用现有 `ResponseEnum` / `ResultCode` 的业务含义，但实现上应只保留一套对外错误码枚举或错误码接口。`common.R`、`common.result.R`、`ResponseEntity` 的对外使用必须替换；如果短期保留旧类，只能标记为废弃，并通过扫描确认没有 API 入口继续返回它们。

`GlobalExceptionHandler` 返回规范 envelope：

- 参数校验失败：`success=false`，`code=400`，`message` 使用明确校验信息。
- 未登录/Token 过期：`success=false`，`code=401`。
- 无权限：`success=false`，`code=403`。
- 业务异常：使用业务错误码和业务 message。
- 未知异常：`success=false`，`code=500`，message 不暴露敏感堆栈细节，详细异常进日志。

安全处理器不能手写旧 `ResponseEntity`，必须输出同一 `ApiResponse` JSON。

## ResponseBodyAdvice 处理

本 change 不保留 `ResponseBodyAdvice` 作为统一输出的核心能力。原因是它会隐藏真实返回类型，使 controller 看起来可以返回裸对象、字符串或旧包装类，长期会让二开代码继续扩散不一致。

实现阶段按以下顺序处理：

1. 扫描所有 controller、异常处理器、安全处理器返回类型。
2. 将它们显式改为 `ApiResponse<T>`。
3. 处理 `String`、文件流、下载接口、Swagger/框架文档等例外。
4. 确认没有入口依赖 advice 包装裸返回值。
5. 删除 `ResponseBodyAdvice`，或只保留明确例外需要的非包装逻辑。

## 前端 HTTP 规约

前端定义 `ApiEnvelope<T>`，与后端 envelope 对齐。`axios.ts` 是默认唯一解析层：

- `success=true`: resolve `data`。
- `success=false`: reject `AppHttpError`，并触发 `http:error`。
- 网络错误、超时、非 envelope 响应：统一映射为 `AppHttpError`。

`AppHttpError` 需要包含 `code`、`status`、`message`、`traceId`、`raw` 和可选 `action`。页面和服务代码不再重复判断 `code/message` envelope。

## 线程池规约

新增统一线程池配置属性和 executor builder，覆盖：

- IO 线程池。
- CPU 线程池。
- background 线程池。
- event 线程池。

每类线程池都必须有 core/max/queue/keepAlive/threadNamePrefix/rejection/shutdown 配置。`ThreadPoolConfig` 中的静态可变字段和重复硬编码应移除。事件线程池已有 `spring.event.thread-pool` 配置，迁移时保留配置键兼容，同时复用统一 builder。

默认拒绝策略可继续使用 `CallerRunsPolicy`，但必须写入文档，说明队列满时调用线程会承担执行压力。

## 日志规约

统一使用 SLF4J 参数化日志，优先使用 `@Slf4j`。运行期问题记录禁止使用 `System.out.println` 和 `printStackTrace`。

异常日志必须包含：

- 操作或接口上下文。
- 失败阶段。
- 关键业务标识。
- `traceId`。
- 异常对象。

预期场景不滥用 `error`：参数校验、鉴权失败、无数据、上游无返回等按实际影响使用 `warn`、`info` 或直接返回错误响应。

## 验证策略

后端验证：

- `mvn -q -DskipTests compile`
- `mvn -q test`
- 响应 envelope 测试。
- 异常和安全响应测试。
- 线程池配置绑定测试。
- 静态扫描旧返回类型和 `ResponseBodyAdvice` 依赖。
- 静态扫描 `System.out.println` 和 `printStackTrace`。

前端验证：

- HTTP adapter 成功解包。
- 业务失败 reject。
- 401/403 行为。
- 网络错误和超时映射。
- `pnpm.cmd run build`。

## 风险与处理

- 旧返回类型直接替换影响面较大：先做入口清单，按模块分组迁移，用编译和扫描收敛。
- 删除 `ResponseBodyAdvice` 会暴露裸返回入口：这是预期结果，必须显式修正返回类型。
- 前端失败改为 reject 可能暴露旧调用假设：统一修复调用方，避免页面继续消费 envelope。
- 线程池配置变更影响异步行为：保守默认值，保留事件配置键兼容。

## 实施边界

本设计只处理运行期基础契约，不新增水处理业务模型、设备控制算法、报表、生产数据库 schema、部署流水线或 Git 提交流程。
