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
