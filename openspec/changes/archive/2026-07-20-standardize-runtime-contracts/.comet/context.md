# Comet Design Handoff

- Change: standardize-runtime-contracts
- Phase: design
- Mode: compact
- Context hash: d6cfb3a158e89a91a0968328c6c091b46d267302d77255e91da5a1cff7241b2d

Generated-by: comet-handoff.sh

OpenSpec remains the canonical capability spec. This handoff is a deterministic, source-traceable context pack, not an agent-authored summary.

## openspec/changes/standardize-runtime-contracts/proposal.md

- Source: openspec/changes/standardize-runtime-contracts/proposal.md
- Lines: 1-32
- SHA256: c02259b57b6d9667ae4381c8bed21f3c84c44facc12354299d325fd99ea1bf0a

```md
## Why

HydroCore 已经可以作为水处理系统二开基线，但运行期基础契约仍不够统一：后端存在多套返回值类型，前端 HTTP 适配对成功/失败的处理不一致，线程池配置分散且硬编码，日志内容也无法稳定表达问题来源、操作和上下文。

如果继续在这个基础上扩展水处理业务，后续设备接入、实时数据、告警、报表和第三方服务集成会放大这些不一致，导致排障困难、接口适配重复、异步任务容量不可控。

## What Changes

- 统一后端 API 响应契约，明确唯一对外响应 envelope、成功/失败判断规则、错误码语义和兼容迁移策略。
- 调整前端 HTTP 封装和错误事件管线，使前端只消费同一种响应结构，并在失败时稳定 reject/通知。
- 优化线程池基础配置，使 IO、CPU、后台任务和事件任务的线程池具备统一命名、配置来源、拒绝策略、关闭行为和可观测信息。
- 统一日志打印方式，要求日志准确表达操作、失败原因、关键上下文和异常堆栈，禁止 `System.out` / `printStackTrace` 作为运行期问题记录方式。
- 保留旧接口的必要兼容桥接，但新增和迁移后的代码不得继续引入新的响应包装类型或模糊日志格式。
- 不提交 Git commit；本 change 仅生成 OpenSpec/Comet 产物，后续实现等服务器和 Git 仓库策略就绪后再执行。

## Capabilities

### New Capabilities

- `runtime-contracts`: 定义 HydroCore 作为二开基线时必须满足的运行期契约，包括 API 响应、前端 HTTP 适配、线程池配置和日志/异常语义。

### Modified Capabilities

- 无。当前 `openspec/specs/` 尚无既有能力规格，本次以新增能力规格承载要求。

## Impact

- 后端影响范围包括 `hydrocore-be` 的通用返回类型、响应包装 advice、全局异常处理、安全异常返回、控制器返回类型、线程池配置、事件线程池配置、日志工具和关键服务日志。
- 前端影响范围包括 `hydrocore-fe/src/libs/http/axios.ts`、HTTP 错误事件类型、错误映射和调用方对响应数据的类型假设。
- 对外接口响应 JSON 结构将统一；如存在旧字段兼容需求，应在设计和任务中明确迁移方式。
- 验证范围包括后端编译/测试、前端构建、响应契约单元或集成测试、关键日志扫描和线程池配置绑定验证。
- 不包含真实水处理业务建模、设备控制算法、生产监控大屏、报表功能、数据库业务 schema 或外部部署流水线。

```

## openspec/changes/standardize-runtime-contracts/design.md

- Source: openspec/changes/standardize-runtime-contracts/design.md
- Lines: 1-73
- SHA256: 6a2bde0378a98bf0652f255ad55de668ea7207ea8ab4ca0c48a4d2bfca5f13d3

```md
## Context

HydroCore 当前后端存在三类响应对象：`common.entity.ResponseEntity`、`common.result.R` 和 `common.R`。`ResponseBodyAdvice` 会把多种返回类型再次转换为 `ResponseEntity`，全局异常处理、安全异常处理和部分控制器也会直接构造不同响应对象。前端 `src/libs/http/axios.ts` 从响应中读取 `code` 和 `message`，但拦截器只提示错误而不统一 reject，调用方实际拿到的是 `response.data`，成功和失败路径边界不稳定。

线程池方面，`ThreadPoolConfig` 以静态字段和硬编码方式配置 IO、CPU、后台任务线程池，事件框架另有 `eventTaskExecutor`。线程名前缀、队列长度、拒绝策略、优雅关闭和配置入口没有统一约束。日志方面，代码中混合使用 `@Slf4j`、`LoggerFactory`、`System.out.println` 和 `printStackTrace`，部分日志只写 “error” 或 “参数校验不通过”，缺少操作、关键参数、异常类型和上下文。

本次 change 的目标是为后续水处理二开打稳运行期基础契约，先明确 OpenSpec 和 Comet 产物，不提交 Git，不进入实现阶段。

## Goals / Non-Goals

**Goals:**

- 后端统一对外响应 envelope，所有正常响应和异常响应遵循同一字段语义。
- 前端 HTTP 封装统一解析后端 envelope，成功只返回业务数据，失败走统一错误对象和事件管线。
- 线程池统一为可配置、可命名、可观测、可优雅关闭的基础设施，区分 IO、CPU、后台任务和事件任务用途。
- 日志统一使用 SLF4J/Lombok 风格，禁止运行期 `System.out` / `printStackTrace`，日志内容必须准确描述操作、失败原因和关键上下文。
- 直接迁移旧响应类型和入口代码，最终删除 `ResponseBodyAdvice` 的统一包装职责，使响应结构由显式返回类型保证。

**Non-Goals:**

- 不新增真实水处理业务模块、工艺模型、设备控制算法、报表或生产数据 schema。
- 不引入完整链路追踪平台、日志采集平台或 APM 产品。
- 不重写所有业务服务，只处理本次契约需要覆盖的公共入口和高风险调用点。
- 不提交 Git commit，不做远端仓库、分支或服务器部署配置。

## Decisions

### 1. 后端使用一个规范响应 envelope，并直接替换旧类型

采用一个规范响应模型作为对外 JSON 结构，字段为 `success`、`code`、`message`、`data`、`traceId`。其中 `success` 表示业务成功与否，`code` 表示业务/系统码，`message` 是可展示或可排障的简短信息，`data` 是业务载荷，`traceId` 用于关联日志。

备选方案是继续使用现有 `ResponseEntity` 的 `code/message/data` 三字段，并只在前端做适配。该方案改动较小，但无法明确业务成功语义，也不利于排障关联。最终选择显式 envelope，并将 `common.R`、`common.result.R`、`ResponseEntity` 的对外使用全部替换为新规约；旧类型如短期保留，只能作为内部废弃门面，不能再作为 API 返回类型。

`ResponseBodyAdvice` 不再作为统一返回的核心机制。迁移期间必须把 controller、异常处理器和安全处理器全部改为显式返回规范类型；完成扫描确认后删除 `ResponseBodyAdvice`，或只保留与文件流/第三方回调等明确例外相关的非包装逻辑。

### 2. 前端只在 HTTP 层理解 envelope

前端 `axios.ts` 负责检查 envelope：成功时返回 `data`，失败时构造统一 `AppHttpError` 并 reject，同时触发 `http:error` 事件。业务页面不再自行判断后端 `code/message` 结构。

备选方案是保留调用方自行判断 `code`。该方案会继续扩散重复判断和不一致提示，因此不采用。对少数需要原始响应的场景，可保留显式 opt-out 方法或局部配置，但默认路径必须统一。

### 3. 线程池使用属性绑定和统一创建逻辑

引入运行期线程池配置属性，集中定义 IO、CPU、background、event 四类池的核心线程数、最大线程数、队列容量、keepAlive、线程名前缀、等待关闭和拒绝策略。线程池创建逻辑应复用同一个 builder/helper，避免四个 bean 重复硬编码。

备选方案是只修改现有硬编码数值。该方案不能解决环境差异和后续二开容量调整问题，因此不采用。默认值应保守，允许通过配置文件覆盖。

### 4. 日志规则以“准确表达问题”为主

统一使用 SLF4J 参数化日志和 `@Slf4j`，异常日志必须包含操作名称、关键业务标识、失败阶段、异常对象；参数校验、鉴权失败、无数据等非系统异常不得滥用 `error`。工具类中的 `printStackTrace` 和 `System.out.println` 应替换为日志或移除。

备选方案是只清理明显的 `printStackTrace`。该方案无法解决“日志看不出真实问题”的核心诉求，因此本次要求同时修正高风险模块中的模糊日志。

## Risks / Trade-offs

- **一次性替换旧返回类型影响面较大** -> 先完成入口清单，再按 controller/handler 分组替换；通过编译、契约测试和静态扫描确认没有旧包装类型继续对外返回。
- **前端失败路径改为 reject 可能暴露隐藏调用问题** -> 分阶段修复依赖旧行为的调用方，关键接口增加手工验证或单元测试。
- **线程池默认值不适合所有部署环境** -> 使用可配置默认值和明确文档，后续生产环境按服务器规格调整。
- **日志规范容易停留在文档层面** -> 在任务中加入静态扫描，至少阻断新增 `System.out` / `printStackTrace` 和公共入口的模糊错误日志。

## Migration Plan

1. 定义规范响应模型和错误码语义，改造 `GlobalExceptionHandler`、安全处理器和 controller，使它们显式返回新规约。
2. 改造前端 HTTP adapter 和错误事件类型，使调用方默认拿到业务 `data`，失败统一进入 reject/事件。
3. 引入线程池配置属性和统一 builder，迁移 IO、CPU、background、event 四类线程池。
4. 删除 `ResponseBodyAdvice` 的统一包装职责，清理公共异常、线程池、事件、TDengine/SEC 关键路径日志，替换 `printStackTrace` / `System.out`。
5. 运行后端编译/测试、前端构建、旧返回类型扫描、日志扫描和关键响应契约测试。

## Open Questions

- 具体错误码表是否沿用现有 `ResponseEnum` / `ResultCode`，还是收敛为一个新的枚举，需要在 design 阶段结合调用点数量确认。
- `traceId` 来源使用现有请求上下文、MDC 过滤器新增值，还是先预留字段，需要在实现前检查已有过滤器和网关环境。
- 是否需要为少数下载、文件流或第三方回调接口保留 `@NoResponseAdvice` 例外，需要在实现前列出接口清单。

```

## openspec/changes/standardize-runtime-contracts/tasks.md

- Source: openspec/changes/standardize-runtime-contracts/tasks.md
- Lines: 1-46
- SHA256: 3d88b0c5617d45eb83d1837788b739cf967d0d4f91d59b0bf5adae1d86905c3b

```md
## 1. Response Contract Inventory

- [ ] 1.1 扫描后端控制器、异常处理、安全处理器和 advice，列出所有返回 `ResponseEntity`、`common.R`、`common.result.R`、裸对象和字符串的入口。
- [ ] 1.2 扫描前端 HTTP 调用方，确认哪些调用依赖 `response.data`、`code/message` 或失败时仍 resolve 的旧行为。
- [ ] 1.3 梳理现有 `ResponseEnum`、`ResultCode`、HTTP 状态码和安全错误码，确定需要保留、合并或迁移的错误码集合。

## 2. Backend Unified Response

- [ ] 2.1 定义规范后端响应 envelope 类型和工厂方法，包含 `success`、`code`、`message`、`data`、`traceId` 字段。
- [ ] 2.2 将 controller、全局异常处理器和安全处理器的返回类型直接替换为规范 envelope，不再依赖 `ResponseBodyAdvice` 做隐式包装。
- [ ] 2.3 删除或停用 `ResponseBodyAdvice` 的统一包装职责；如存在文件流、第三方回调或框架文档接口例外，必须用显式注解和清单记录。
- [ ] 2.4 将 `common.R`、`common.result.R`、`ResponseEntity` 标记为废弃或删除可删除部分，确保它们不再作为 API 对外返回类型。
- [ ] 2.5 增加静态扫描，确认后端 API 入口没有继续返回 `common.R`、`common.result.R`、`ResponseEntity` 或依赖 `ResponseBodyAdvice` 包装裸对象。
- [ ] 2.6 补充后端响应契约测试，覆盖成功响应、参数校验失败、业务异常、鉴权/授权失败和未知异常。

## 3. Frontend HTTP Contract

- [ ] 3.1 定义前端 `ApiEnvelope<T>` 和扩展后的 `AppHttpError` 类型，包含后端错误码、消息、traceId、HTTP 状态和 raw context。
- [ ] 3.2 改造 `src/libs/http/axios.ts`，成功 envelope 只 resolve `data`，失败 envelope reject 统一错误对象并触发 `http:error`。
- [ ] 3.3 改造 `error-handler`、`error-mapper` 和相关调用方，使错误提示、登出、重定向和通知逻辑从统一错误对象读取信息。
- [ ] 3.4 修复依赖旧失败 resolve 行为的页面或服务调用，确保业务代码不再重复判断后端 envelope。
- [ ] 3.5 补充前端 HTTP adapter 测试或最小验证用例，覆盖成功解包、业务失败、401/403、网络错误和超时。

## 4. Thread Pool Runtime Configuration

- [ ] 4.1 定义线程池配置属性类，覆盖 IO、CPU、background、event 四类线程池的 core/max/queue/keepAlive/prefix/rejection/shutdown 设置。
- [ ] 4.2 抽取统一 `ThreadPoolTaskExecutor` 创建逻辑，消除 `ThreadPoolConfig` 中重复硬编码和静态可变配置。
- [ ] 4.3 将 `eventTaskExecutor` 迁移到同一配置模型，保持事件发布器 bean 名称和注入兼容。
- [ ] 4.4 在默认配置文件和文档中记录线程池默认值、调整方式、拒绝策略和优雅关闭行为。
- [ ] 4.5 补充线程池配置绑定或上下文加载测试，确认默认值和覆盖值生效。

## 5. Logging Standardization

- [ ] 5.1 制定日志规则并在代码中落地：统一 SLF4J 参数化日志，错误日志包含操作、阶段、关键标识、traceId 和异常对象。
- [ ] 5.2 清理公共异常处理、响应 advice、线程池、事件框架、TDengine 和 SEC 关键路径中的模糊或误导性日志。
- [ ] 5.3 替换运行期代码中的 `System.out.println` 和 `printStackTrace`，保留必要调试信息时改为合适级别日志。
- [ ] 5.4 调整校验失败、鉴权失败、无数据、上游无返回等预期场景的日志级别，避免滥用 `error`。
- [ ] 5.5 增加静态扫描命令或测试，防止新增 `System.out`、`printStackTrace` 和明显无上下文的公共错误日志。

## 6. Verification

- [ ] 6.1 运行 `mvn -q -DskipTests compile` 验证后端编译。
- [ ] 6.2 运行 `mvn -q test` 验证后端测试。
- [ ] 6.3 运行 `pnpm.cmd run build` 验证前端构建。
- [ ] 6.4 运行响应契约、旧返回类型、线程池配置和日志静态扫描验证，并记录结果。
- [ ] 6.5 更新本 change 的验证记录和剩余风险，不提交 Git commit。

```

## openspec/changes/standardize-runtime-contracts/specs/runtime-contracts/spec.md

- Source: openspec/changes/standardize-runtime-contracts/specs/runtime-contracts/spec.md
- Lines: 1-76
- SHA256: 7f9fecbb823848e2001909e2891197f1ae85f39133c32e90691096f5be24a2ca

```md
## ADDED Requirements

### Requirement: Unified API response envelope
HydroCore backend APIs SHALL expose one canonical JSON response envelope for normal and error responses. The envelope MUST include `success`, `code`, `message`, `data`, and `traceId` fields with stable semantics.

#### Scenario: Successful controller response
- **WHEN** a backend controller returns a successful business payload
- **THEN** the HTTP response body uses the canonical envelope with `success=true`, a success `code`, a non-empty `message`, the payload in `data`, and the current request `traceId`

#### Scenario: Business or validation failure
- **WHEN** a backend request fails because of business validation, argument validation, authentication, authorization, or an unhandled server exception
- **THEN** the HTTP response body uses the same canonical envelope with `success=false`, an accurate `code`, an accurate `message`, `data` empty unless explicitly needed, and the current request `traceId`

#### Scenario: No legacy response wrapper output
- **WHEN** backend controllers, exception handlers, and security handlers return API results
- **THEN** they return the canonical response type directly and do not return `common.R`, `common.result.R`, or `ResponseEntity`

#### Scenario: Response advice removal
- **WHEN** all backend API entry points have been migrated to the canonical response type
- **THEN** `ResponseBodyAdvice` is removed or disabled for normal API wrapping, so response structure is explicit in controller and handler code

### Requirement: Frontend HTTP adapter consumes one contract
The frontend HTTP adapter SHALL be the only default layer that interprets the backend response envelope. Successful requests MUST resolve to business `data`, and failed envelope responses MUST reject with a typed HTTP error.

#### Scenario: Successful API call
- **WHEN** the backend returns the canonical envelope with `success=true`
- **THEN** the frontend HTTP adapter resolves the request promise with the envelope `data` value

#### Scenario: Failed API call
- **WHEN** the backend returns the canonical envelope with `success=false`
- **THEN** the frontend HTTP adapter rejects the request promise with an `AppHttpError` containing the backend `code`, `message`, `traceId`, HTTP status when available, and raw response context

#### Scenario: Transport error
- **WHEN** Axios receives a network error, timeout, or non-envelope transport failure
- **THEN** the frontend HTTP adapter emits the existing HTTP error event with a typed `AppHttpError` and does not pretend the request returned successful business data

### Requirement: Configurable runtime thread pools
HydroCore backend SHALL define runtime thread pools through a unified configurable mechanism. IO, CPU, background, and event thread pools MUST have explicit names, bounded queues, configurable sizing, rejection policy, and shutdown behavior.

#### Scenario: Default thread pool creation
- **WHEN** the backend application starts without deployment-specific overrides
- **THEN** it creates IO, CPU, background, and event executors using documented safe defaults and distinct thread name prefixes

#### Scenario: Environment-specific sizing
- **WHEN** deployment configuration overrides a thread pool size, queue capacity, keep-alive value, or shutdown wait setting
- **THEN** the corresponding executor uses the configured value without source code changes

#### Scenario: Rejected task behavior
- **WHEN** a thread pool reaches its configured capacity
- **THEN** the configured rejection policy is applied consistently and the behavior is documented for operators and developers

### Requirement: Accurate structured logging
HydroCore runtime logging SHALL use one SLF4J-compatible style and MUST accurately describe the operation, failure stage, relevant identifiers, and exception details for actionable troubleshooting.

#### Scenario: Exception handling log
- **WHEN** global exception handling records an exception
- **THEN** the log entry includes the operation or handler context, exception type/message, request trace identifier when available, and the exception object for stack trace capture

#### Scenario: Validation or expected denial
- **WHEN** the system rejects a request because of validation, authentication, authorization, or missing optional upstream data
- **THEN** the log level and message reflect the expected condition accurately and do not describe it as an unknown system crash

#### Scenario: Forbidden runtime console output
- **WHEN** backend runtime code records operational problems
- **THEN** it MUST NOT use `System.out.println` or `printStackTrace`; it uses the unified logging style or removes non-useful debug output

### Requirement: Contract verification gates
The change implementation SHALL include verification that protects the runtime contracts from drifting during future secondary development.

#### Scenario: Backend verification
- **WHEN** backend verification runs
- **THEN** it covers response envelope behavior, exception response behavior, thread pool configuration binding, and a static scan for forbidden runtime console output

#### Scenario: Frontend verification
- **WHEN** frontend verification runs
- **THEN** it covers successful envelope unwrapping, failed envelope rejection, transport error mapping, and the production build

```
