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
