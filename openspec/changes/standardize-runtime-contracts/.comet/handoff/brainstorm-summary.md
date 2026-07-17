# Brainstorm Summary

- Change: standardize-runtime-contracts
- Date: 2026-07-17
- Language: zh-CN

## 确认的技术方案

采用“新增规范响应类型 + 直接替换旧返回类型”的方案。

后端新增唯一规范响应类型，建议命名为 `ApiResponse<T>`，对外 JSON 字段固定为 `success`、`code`、`message`、`data`、`traceId`。所有 controller、全局异常处理器、安全处理器都显式返回该类型，不再依赖 `ResponseBodyAdvice` 做隐式包装。

`common.R`、`common.result.R`、`ResponseEntity` 不再作为 API 对外返回类型。实现阶段需要按调用点直接替换为新规约；如果短期保留旧类，只能作为废弃兼容代码存在，并必须通过静态扫描确认没有 API 入口继续返回旧类型。

`ResponseBodyAdvice` 的统一包装职责不保留。完成入口迁移后删除该类；若确实存在文件流、第三方回调、Swagger/框架文档等例外，只能用显式注解和清单记录，不得继续承担“兜底包装所有返回值”的职责。

前端 `axios.ts` 只消费新 envelope：成功时 resolve `data`，失败时 reject 统一 `AppHttpError`，错误事件管线从该对象读取 `code`、`message`、`traceId`、HTTP status 和 raw context。

线程池仍采用统一配置属性和统一 builder：IO、CPU、background、event 四类池都进入统一创建逻辑。事件线程池已有 `spring.event.thread-pool` 配置，应在不破坏现有配置键的前提下复用 builder。

日志统一使用 SLF4J 参数化风格，公共异常、安全、响应、线程池、事件、TDengine/SEC 高风险路径必须准确表达操作、失败原因和关键上下文。运行期代码不得使用 `System.out.println` 或 `printStackTrace` 记录问题。

## 关键取舍与风险

- 直接替换旧返回类型会比兼容桥接改动更大，但最终基线更干净，适合二开项目长期维护。
- 删除 `ResponseBodyAdvice` 能避免隐式包装造成的返回结构不可见问题，但要求 controller 和 handler 返回类型全部显式迁移。
- 旧类型必须通过静态扫描控制，防止后续代码继续引入 `common.R`、`common.result.R`、`ResponseEntity`。
- 前端失败路径改为 reject 后，可能暴露依赖旧 resolve 行为的调用点，需要在实现阶段集中修复。
- 事件线程池已有配置模型，不应粗暴重写；应统一创建逻辑，同时兼容现有配置。

## 测试策略

- 后端：响应 envelope 测试、异常处理测试、安全错误测试、旧返回类型静态扫描、线程池属性绑定测试、上下文加载测试、日志扫描。
- 前端：HTTP adapter 成功解包、业务失败 reject、401/403、网络错误、超时、生产构建。
- 集成命令：`mvn -q -DskipTests compile`、`mvn -q test`、`pnpm.cmd run build`。

## Spec Patch

已回写 delta spec：删除“旧返回类型由 ResponseBodyAdvice 兼容转换”的要求，改为“API 入口直接返回新规约，完成迁移后删除或停用 ResponseBodyAdvice 包装职责”。
