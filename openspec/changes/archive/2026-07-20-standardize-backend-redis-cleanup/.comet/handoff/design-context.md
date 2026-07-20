# Comet Design Handoff

- Change: standardize-backend-redis-cleanup
- Phase: design
- Mode: compact
- Context hash: c3c1881aa456c0a1576b15055bcceb419517b0fcd82d7c2f8e3078c39165bd4a

Generated-by: comet-handoff.sh

OpenSpec remains the canonical capability spec. This handoff is a deterministic, source-traceable context pack, not an agent-authored summary.

## openspec/changes/standardize-backend-redis-cleanup/proposal.md

- Source: openspec/changes/standardize-backend-redis-cleanup/proposal.md
- Lines: 1-31
- SHA256: b81e18f231119c83e7fe78fb3c265e1d0471c8bf93f49f1f69c3784c33e14375

```md
## Why

后端 Redis 能力目前没有完全收敛到一个清晰边界：`RedisConfig` 使用 FastJson 自定义序列化，`RedisService` 注入 raw `RedisTemplate`，`JwtUtil` 又直接操作 `StringRedisTemplate`，业务缓存需要各自处理序列化细节。与此同时，`common.result.ResultCode` 和 `IErrorCode` 已不再被业务引用，和当前 `ApiResponseCode` 运行期响应契约重复。

本变更用于统一 Redis 模板、序列化/反序列化和调用入口，同时删除已确认无用的遗留返回码类型，降低后端基础设施维护成本。

## What Changes

- 统一后端 Redis 配置，明确 key、value、hash key、hash value 的序列化策略，并避免业务层各自选择 JSON 库或模板类型。
- 收敛 Redis 访问入口，让普通对象缓存、字符串 token、hash/list/set、TTL、删除、分布式锁等能力通过统一服务或清晰分层的 Redis facade 暴露。
- 迁移 `JwtUtil`、系统配置缓存、前端配置缓存、历史节点数据缓存等调用点，避免绕过统一 Redis 管理边界。
- 为 Redis 序列化、TTL、hash 读写、token 字符串读写和锁释放行为补充 focused tests 或静态扫描验证。
- 删除无外部引用的 `common.result.ResultCode` 与 `IErrorCode`，确保后端只保留 `common.api.ApiResponseCode` 作为 REST 运行期响应码来源。
- 不改变现有 Redis key 命名、token 生命周期、接口响应 JSON envelope 或数据库 schema。

## Capabilities

### New Capabilities

- `backend-redis-management`: 规定后端 Redis 模板、序列化/反序列化、统一服务入口、调用边界和关键验证场景。

### Modified Capabilities

- `runtime-contracts`: 明确遗留 `ResultCode`/`IErrorCode` 不再属于后端运行期响应契约，删除后不得重新作为 API 错误码来源。

## Impact

- 后端代码：`hydrocore-be/src/main/java/com/siact/hydrocore/common/redis/`、`JwtUtil`、Redis 相关业务调用点、遗留返回码包。
- 后端测试：新增或调整 Redis 配置/服务单元测试、静态扫描测试，复用现有 `ApiResponse`/运行期契约测试。
- 依赖风险：若替换 FastJson Redis serializer，需要确认现有缓存兼容策略；本变更不承诺兼容已经写入 Redis 的旧类型元数据，必要时通过缓存过期或 key 清理处理。
- 运行影响：Redis key、TTL 和业务语义保持不变；只改变后端内部管理和序列化实现边界。

```

## openspec/changes/standardize-backend-redis-cleanup/design.md

- Source: openspec/changes/standardize-backend-redis-cleanup/design.md
- Lines: 1-109
- SHA256: ca9d992675baae78f866d2f7357daa92dbd30e876433bc5d5c4b7ee257648559

[TRUNCATED]

```md
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


```

Full source: openspec/changes/standardize-backend-redis-cleanup/design.md

## openspec/changes/standardize-backend-redis-cleanup/tasks.md

- Source: openspec/changes/standardize-backend-redis-cleanup/tasks.md
- Lines: 1-34
- SHA256: 31b52342cd66174c5c90523dacd1ff3e7649e3273d7c3ed94f3c034a99737a04

```md
## 1. 基线验证

- [ ] 1.1 在 `hydrocore-be` 中运行当前后端 focused tests，记录 Redis 改造前的基线结果。
- [ ] 1.2 新增或扩展静态扫描，识别生产代码中直接注入 `RedisTemplate` / `StringRedisTemplate` 以及遗留 `common.result` 使用的情况。

## 2. Redis 字符串配置与显式 JSON

- [ ] 2.1 新增 Redis configuration/template 测试，断言 key、value、hash key、hash value 都使用字符串序列化。
- [ ] 2.2 将 FastJson Redis serializer 接线替换为字符串 Redis template 配置，同时保持现有 Redis key 和 hash 结构不变。
- [ ] 2.3 确认 Redis 配置和测试不再依赖 `FastJson2JsonRedisSerializer` 后删除该类。

## 3. Redis 服务边界

- [ ] 3.1 重构 `RedisService`，注入类型明确的字符串 Redis templates，并暴露字符串值、JSON 值、hash JSON、list/set 操作、TTL、删除、批量删除和 SETNX 风格锁方法。
- [ ] 3.2 为 `RedisService` 新增 focused tests，覆盖字符串读写、显式 typed JSON 读写、hash JSON 读写、TTL、删除/批量删除、锁获取和锁释放行为。
- [ ] 3.3 禁止 RedisService 使用 AutoType、`WriteClassName` 或 `Object.class` 通用反序列化恢复业务缓存对象。
- [ ] 3.4 仅在现有调用点需要时保留兼容 overload，并用测试或静态扫描断言覆盖这些兼容入口。

## 4. 调用点迁移

- [ ] 4.1 将 `JwtUtil` 从直接访问 `StringRedisTemplate` 迁移到共享 Redis 服务，保持 token key 前缀、TTL 单位、refresh-window、stale-token、download-token 消费和锁语义不变。
- [ ] 4.2 迁移或简化 `SysConfigServiceImpl` 的缓存序列化，使其一致使用统一 Redis 边界，业务代码不再临时选择 serializer。
- [ ] 4.3 验证 `FrontendCacheServiceImpl` 和 `DataServiceImpl` 在字符串 hash JSON 下的缓存行为，仅在必须保持返回业务类型时调整转换逻辑。

## 5. 遗留返回码清理

- [ ] 5.1 确认生产代码和测试没有引用后，删除无用的 `common.result.ResultCode` 和 `IErrorCode`。
- [ ] 5.2 扩展 runtime-contract 验证，防止 `ResultCode` 和 `IErrorCode` 重新作为 REST 响应码契约出现。

## 6. 最终验证

- [ ] 6.1 运行 Redis、JWT、ApiResponse、exception-handler、security-handler 和 runtime-contract 静态扫描相关 focused tests。
- [ ] 6.2 在 `hydrocore-be` 中运行完整后端 `mvn -q test`。
- [ ] 6.3 在后端文档或变更验证报告中记录 Redis 缓存兼容性或清理缓存的要求。

```

## openspec/changes/standardize-backend-redis-cleanup/specs/backend-redis-management/spec.md

- Source: openspec/changes/standardize-backend-redis-cleanup/specs/backend-redis-management/spec.md
- Lines: 1-63
- SHA256: 55b617c4758e1c5b1538e2fce9d308a1980d5919cfd9cf7dbaa085943783e38c

```md
## ADDED Requirements

### Requirement: Unified Redis string storage configuration
HydroCore 后端必须在一个 Redis 配置边界内定义 key、value、hash key 和 hash value 的序列化方式。key、value、hash key 和 hash value 必须使用字符串序列化。业务缓存对象必须由共享 Redis 服务显式编码为 JSON 字符串，并在读取时通过调用方提供的目标类型显式反序列化。

#### Scenario: Redis template uses string serialization
- **WHEN** 后端 Redis template bean 被创建
- **THEN** key、value、hash key 和 hash value serializer 都使用字符串序列化

#### Scenario: JSON decoding requires explicit target type
- **WHEN** 后端业务代码从 Redis 读取对象缓存
- **THEN** 调用方必须通过共享 Redis 服务提供 `Class<T>` 或 `TypeReference<T>` 这类明确目标类型

### Requirement: No dynamic type metadata for Redis cache values
HydroCore 后端 Redis 缓存不得依赖 AutoType、`WriteClassName` 或 `Object.class` 通用反序列化恢复业务对象。JSON 库可以使用项目已有 fastjson2，但只能通过显式目标类型进行缓存值反序列化。

#### Scenario: Redis cache does not write Java type metadata
- **WHEN** 后端写入对象缓存到 Redis
- **THEN** Redis value 或 hash value 中不包含用于通用对象恢复的 Java 类型元数据

#### Scenario: AutoType is not used for Redis cache reads
- **WHEN** 后端从 Redis 读取 JSON 缓存值
- **THEN** Redis 缓存读取路径不启用 AutoType，不使用 `Object.class` 通用反序列化恢复业务对象

### Requirement: Unified Redis service boundary
HydroCore 后端业务代码必须通过共享 Redis service 或 facade 访问 Redis。该服务必须覆盖现有代码使用的字符串值、JSON 值、hash JSON、list/set、TTL、删除、批量删除和 SETNX 风格锁获取。

#### Scenario: Existing cache callers use the shared service
- **WHEN** 系统配置缓存、前端配置缓存、节点历史缓存或 JWT token 存储访问 Redis
- **THEN** 调用点使用共享 Redis 服务边界，并保持现有 key 名称、TTL 值和业务语义

#### Scenario: Direct RedisTemplate injection is restricted
- **WHEN** 静态验证扫描后端生产代码
- **THEN** 除 Redis 配置、共享 Redis service/facade 和明确范围内的测试外，不存在直接注入 `RedisTemplate` 或 `StringRedisTemplate` 的代码

### Requirement: Redis token operations preserve existing behavior
JWT 和下载 token 的 Redis 操作必须在使用统一 Redis 服务边界后保持当前 token key 前缀、refresh-window 语义、stale-token 行为、下载 token 消费行为和刷新锁行为。

#### Scenario: Token is generated and stored
- **WHEN** 登录 token 被生成
- **THEN** Redis 存储 `token:<tokenValue>` 和 `token:refresh:<userId>:<sessionId>`，且 value 和 TTL 单位与当前行为一致

#### Scenario: Expired token refresh uses stale cache and lock
- **WHEN** 过期 token 在刷新窗口内刷新
- **THEN** 系统使用相同 stale-token key 和 SETNX 风格锁语义，避免并发请求重复刷新

#### Scenario: Download token is consumed once
- **WHEN** 下载 token 被成功消费
- **THEN** 系统读取相同 `download:<token>` value 格式，删除该 Redis key，并返回与当前行为一致的登录用户字段

### Requirement: Redis cache migration is operationally explicit
Redis 清理必须说明旧 serializer 写入的既有 Redis value 是否兼容。若不兼容，实现必须记录清理缓存、等待 TTL 过期或部署处理要求。

#### Scenario: Serializer compatibility is assessed
- **WHEN** Redis 缓存写入格式发生变化
- **THEN** 变更文档说明旧缓存值是否仍可读取，或必须被清理/等待过期

### Requirement: Redis behavior is verified
实现必须包含 Redis 配置和服务行为的 focused verification，覆盖字符串 serializer 接线、字符串 value 操作、显式 typed JSON 操作、现有调用点使用的 hash JSON 操作、TTL、删除/批量删除和锁获取/释放。

#### Scenario: Redis verification runs
- **WHEN** 后端验证运行
- **THEN** Redis 配置和共享服务行为由自动化测试或静态扫描覆盖，足以发现 serializer 漂移、动态类型反序列化漂移和直接 template 使用漂移

```

## openspec/changes/standardize-backend-redis-cleanup/specs/runtime-contracts/spec.md

- Source: openspec/changes/standardize-backend-redis-cleanup/specs/runtime-contracts/spec.md
- Lines: 1-12
- SHA256: 68f89e1b01b378715eeeb72a760dde2dcabffb671478f89f72b6cd98ae9bb9ee

```md
## ADDED Requirements

### Requirement: No legacy backend ResultCode contract
HydroCore backend SHALL use `com.siact.hydrocore.common.api.ApiResponseCode` as the canonical REST response code source. The legacy `com.siact.hydrocore.common.result.ResultCode` and `IErrorCode` types MUST NOT remain in production code after they are confirmed unused, and they MUST NOT be reintroduced as frontend-facing API error code contracts.

#### Scenario: Legacy result code types are absent
- **WHEN** backend production source is scanned after the cleanup
- **THEN** `com.siact.hydrocore.common.result.ResultCode` and `com.siact.hydrocore.common.result.IErrorCode` are absent

#### Scenario: API responses continue to use ApiResponseCode
- **WHEN** controllers, exception handlers, or security handlers construct REST response envelopes
- **THEN** they continue to use `ApiResponse<T>` and `ApiResponseCode` semantics rather than legacy result-code types

```
