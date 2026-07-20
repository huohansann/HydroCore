## Context

HydroCore 后端是 Spring Boot 2.6.13 基线工程，当前 Redis 能力集中在 `common.redis` 包，但边界不够完整：

- `RedisConfig` 暴露 `RedisTemplate<Object, Object>`，value/hash value 使用 `FastJson2JsonRedisSerializer<Object>`，key/hash key 使用 `StringRedisSerializer`。
- `RedisService` 注入 raw `RedisTemplate`，对外暴露对象、list、set、hash、TTL、删除、锁等方法，但调用者仍需要理解底层序列化行为。
- `JwtUtil` 直接注入 `StringRedisTemplate` 管理 token、refresh token、stale token、download token 和刷新锁，绕过了统一 Redis 服务边界。
- `SysConfigServiceImpl` 已经把复杂对象转成 JSON 字符串后再写入 Redis，说明“Redis 存字符串，服务层显式编解码”更贴近当前代码实际。
- `FrontendCacheServiceImpl` 和 `DataServiceImpl` 使用 `RedisService` 的 hash 能力，但读取后又通过 JSON 库转换，说明类型恢复应由明确 API 管理，而不是依赖全局动态类型反序列化。
- `common.result.ResultCode` 和 `IErrorCode` 没有被业务引用，响应契约已经收敛到 `common.api.ApiResponseCode`。

约束：本变更不改变 Redis key 命名、TTL、JWT 生命周期、HTTP API envelope、数据库 schema，也不要求兼容 Redis 中已经存在的旧 FastJson `WriteClassName` 类型元数据缓存；缓存可通过 TTL 或显式清理自然迁移。

## Goals / Non-Goals

**Goals:**

- Redis 配置由一个明确的配置类管理，key、value、hash key、hash value 在底层统一使用字符串序列化。
- Redis 对象缓存不依赖全局 Object serializer，而是由 `RedisService` 提供显式 JSON 编解码 API：写入时转 JSON 字符串，读取时必须传入 `Class<T>` 或 `TypeReference<T>`。
- JSON 实现复用项目已存在的 fastjson2 依赖，但禁止 AutoType、`WriteClassName` 和通用 `Object.class` 反序列化作为业务缓存恢复机制。
- Redis 调用入口收敛到一个公共服务边界：业务代码不直接注入 `RedisTemplate` 或 `StringRedisTemplate`，除非配置层或测试明确需要。
- `JwtUtil` 的字符串 token 存储、SETNX 锁、批量 key 删除等行为迁移到统一 Redis 服务的字符串/锁方法，保持原 key、TTL 和刷新语义。
- 删除 `ResultCode` 和 `IErrorCode`，并用静态扫描防止它们重新进入运行期响应契约。
- 增加 focused tests 覆盖 Redis 字符串配置、RedisService 显式 JSON API、JWT Redis 边界和遗留返回码清理。

**Non-Goals:**

- 不引入 Redisson、Redis Repository、Spring Cache abstraction、Kryo、Protostuff 或新的外部缓存框架。
- 不把 Redis 全局 value/hash value serializer 切换为 Jackson、JDK 原生序列化或任何动态类型 Object serializer。
- 不改变业务缓存结构、key 前缀、TTL 数值、JWT claim、接口响应结构或前端调用方式。
- 不迁移或保留线上 Redis 中已经存在的旧 FastJson 类型元数据缓存。
- 不重构 TDengine、MQTT、WebSocket、业务查询逻辑或异常体系命名。

## Decisions

### 1. Redis 底层统一字符串，类型恢复由 RedisService 显式管理

选择：`RedisConfig` 提供字符串语义的 Redis 模板，key、value、hash key、hash value 都使用 `StringRedisSerializer`。对象、集合和复杂 hash 值由 `RedisService` 在方法层显式调用 JSON 编解码：

- `setJson(String key, T value, ...)`
- `getJson(String key, Class<T> type)`
- `getJson(String key, TypeReference<T> typeReference)`
- `setHashJson(String key, String hashKey, T value, ...)`
- `getHashJson(String key, String hashKey, Class<T> type)`
- `getHashJson(String key, String hashKey, TypeReference<T> typeReference)`

原因：Spring Data Redis 官方支持 `StringRedisSerializer`，并提示不要在不可信环境依赖 Java 原生反序列化。相比全局 Jackson/Object serializer，字符串 Redis 更容易排查、更容易迁移，也不会把动态类型信息写进 Redis。当前 `SysConfigServiceImpl` 已经以 JSON 字符串方式缓存对象，迁移成本低。

替代方案：

- 使用 `GenericJackson2JsonRedisSerializer`。实现较少，但会把类型信息和全局 Object 反序列化引入 Redis；对 HydroCore 这种统一 `RedisService` 存多种对象的场景，排查和安全边界不如显式 typed API。
- 继续使用自定义 `FastJson2JsonRedisSerializer`。兼容旧缓存风险较低，但当前实现依赖 `WriteClassName` 与 AutoType filter，不适合作为长期通用对象反序列化边界。
- 使用 Kryo/Protostuff 等二进制序列化。性能可能更好，但引入依赖和可观测性成本，不符合本次“清理统一管理”的目标。

### 2. fastjson2 只作为显式 typed JSON 工具，不作为 AutoType serializer

选择：继续使用项目已有 fastjson2 依赖作为 Redis JSON 编解码实现，但只允许通过调用方传入明确目标类型来反序列化。实现中不得使用 `JSONWriter.Feature.WriteClassName`、AutoType 或 `Object.class` 通用反序列化恢复业务对象。

原因：项目已经大量使用 fastjson2，避免新增 Jackson Redis 方案可以减少技术栈分叉。显式类型 API 的边界比 AutoType 更清楚，Redis 内容也是普通 JSON 字符串，便于运维排查。

替代方案：

- 改用 Jackson typed API。安全边界也可控，但项目当前已有 fastjson2，用户已倾向不用 Jackson；本次没有足够收益引入第二套 Redis JSON 实现。
- 保留 AutoType 白名单。实现省事，但仍然把 Redis 内容与 Java 类型名耦合，后续重构类名或包名时风险较高。

### 3. RedisService 作为唯一业务入口，但允许配置层暴露模板 Bean

选择：配置层保留必要的 Redis template Bean；业务层通过 `RedisService` 访问字符串、对象 JSON、hash JSON、list/set、TTL、删除和锁。静态扫描禁止非配置/测试代码直接注入 `RedisTemplate` 或 `StringRedisTemplate`。

原因：JWT 需要字符串语义，系统配置和业务缓存需要 JSON 语义，统一服务可以用清晰方法区分两类操作，同时避免业务层直接操作模板导致序列化策略分叉。

替代方案：

- 将 token 单独抽成 `TokenRedisStore`。边界更细，但当前直接 Redis 调用点较少，先把底层能力收敛到 `RedisService` 更务实；后续若 token 逻辑继续增长，再拆专用 store。
- 完全隐藏所有 RedisTemplate Bean。Spring 生态和测试会变得不方便，配置层也需要模板 Bean，因此不采用。

### 4. 删除 ResultCode 与 IErrorCode，不迁移到 ApiResponseCode

选择：直接删除 `common.result.ResultCode` 和 `IErrorCode`，不新增兼容 adapter。保留 `ApiResponseCode` 作为 REST 响应码来源；异常包里的 `BaseErrorInfoInterface#getResultCode()` 属于另一套异常接口命名，本次不改名。

原因：扫描显示 `ResultCode` 和 `IErrorCode` 只彼此引用，没有业务引用。保留兼容层会继续制造“有两套错误码”的错觉。

替代方案：

- 标记 deprecated 后保留一个周期。适合外部模块仍编译依赖该类型的场景，但当前子仓库内没有引用；如果实施阶段发现外部发布 API 依赖，再回到 design 阶段重新评估。

## Risks / Trade-offs

- [Risk] Redis 中已有 FastJson 带类型信息的缓存值无法被新字符串 JSON API 读取。→ Mitigation：本变更只承诺 key/TTL/业务语义不变，不承诺旧缓存兼容；实施说明中记录清理或等待 TTL 过期策略。
- [Risk] 显式 typed API 要求调用点传入目标类型，改造工作比全局 Object serializer 多。→ Mitigation：只迁移现有调用点需要的方法，并保留清晰命名的兼容入口，避免一次性重写所有 RedisService 方法。
- [Risk] `JwtUtil` 迁移可能影响 token 刷新并发锁。→ Mitigation：为 token key 写入、refresh window、stale token、SETNX 锁和锁释放补单元测试或 mock RedisTemplate 验证。
- [Risk] 直接删除 `ResultCode` 后外部未纳入仓库的编译依赖失败。→ Mitigation：先在后端子仓库全量 `rg` 和 `mvn test` 验证；若存在外部二方包依赖，改为迁移公告或 deprecated 保留需要重新打开范围。

## Migration Plan

1. 新增 Redis configuration/service 测试，先固定 key/value/hash key/hash value 都使用字符串序列化，以及 string value、JSON value、hash JSON、TTL、delete、SETNX/lock 行为。
2. 调整 `RedisConfig`，提供命名清晰的字符串 Redis template；移除 `FastJson2JsonRedisSerializer` 使用。
3. 重构 `RedisService` 注入类型和方法边界，新增字符串 value、JSON value、hash JSON、set-if-absent、批量删除等能力，使 `JwtUtil` 不再直接操作 `StringRedisTemplate`。
4. 迁移 `JwtUtil`、`SysConfigServiceImpl`、`FrontendCacheServiceImpl`、`DataServiceImpl` 等调用点，保持 key、TTL 和业务语义不变。
5. 删除 `FastJson2JsonRedisSerializer`、`ResultCode`、`IErrorCode` 中确认无用的类型；fastjson2 依赖仍可保留给业务 DTO/上游解析和 RedisService typed JSON 使用。
6. 增加静态扫描，禁止业务层直接注入 RedisTemplate/StringRedisTemplate，禁止 AutoType/WriteClassName 作为 Redis 缓存机制，禁止 `common.result.ResultCode`/`IErrorCode` 残留。
7. 运行后端 focused tests 和 `mvn -q test`。

Rollback：如果 Redis 访问迁移引发运行异常，可回滚 RedisConfig/RedisService/JwtUtil 相关提交，并清理新写入测试缓存；文档和 ResultCode 删除应随实现回滚保持一致。

## Open Questions

- 是否要求兼容线上 Redis 中已存在的 FastJson `WriteClassName` 缓存值？当前设计假设不要求，依赖 TTL 或部署时清理缓存。
- 是否接受保留 Redis template Bean 供 Spring/配置层使用，但业务代码禁止直接注入？当前设计采用此边界。
