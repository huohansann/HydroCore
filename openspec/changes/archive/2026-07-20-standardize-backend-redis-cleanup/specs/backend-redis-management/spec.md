## ADDED Requirements

### Requirement: Unified Redis string storage configuration
HydroCore backend MUST define Redis key, value, hash key, and hash value serialization in a single Redis configuration boundary.
HydroCore 后端必须在一个 Redis 配置边界内定义 key、value、hash key 和 hash value 的序列化方式。key、value、hash key 和 hash value 必须使用字符串序列化。业务缓存对象必须由共享 Redis 服务显式编码为 JSON 字符串，并在读取时通过调用方提供的目标类型显式反序列化。

#### Scenario: Redis template uses string serialization
- **WHEN** 后端 Redis template bean 被创建
- **THEN** key、value、hash key 和 hash value serializer 都使用字符串序列化

#### Scenario: JSON decoding requires explicit target type
- **WHEN** 后端业务代码从 Redis 读取对象缓存
- **THEN** 调用方必须通过共享 Redis 服务提供 `Class<T>` 或 `TypeReference<T>` 这类明确目标类型

### Requirement: No dynamic type metadata for Redis cache values
Redis cache values MUST NOT rely on AutoType, WriteClassName, or Object.class generic deserialization to restore business objects.
HydroCore 后端 Redis 缓存不得依赖 AutoType、`WriteClassName` 或 `Object.class` 通用反序列化恢复业务对象。JSON 库可以使用项目已有 fastjson2，但只能通过显式目标类型进行缓存值反序列化。

#### Scenario: Redis cache does not write Java type metadata
- **WHEN** 后端写入对象缓存到 Redis
- **THEN** Redis value 或 hash value 中不包含用于通用对象恢复的 Java 类型元数据

#### Scenario: AutoType is not used for Redis cache reads
- **WHEN** 后端从 Redis 读取 JSON 缓存值
- **THEN** Redis 缓存读取路径不启用 AutoType，不使用 `Object.class` 通用反序列化恢复业务对象

### Requirement: Unified Redis service boundary
Backend business code MUST access Redis through the shared Redis service or facade instead of direct template injection.
HydroCore 后端业务代码必须通过共享 Redis service 或 facade 访问 Redis。该服务必须覆盖现有代码使用的字符串值、JSON 值、hash JSON、list/set、TTL、删除、批量删除和 SETNX 风格锁获取。

#### Scenario: Existing cache callers use the shared service
- **WHEN** 系统配置缓存、前端配置缓存、节点历史缓存或 JWT token 存储访问 Redis
- **THEN** 调用点使用共享 Redis 服务边界，并保持现有 key 名称、TTL 值和业务语义

#### Scenario: Direct RedisTemplate injection is restricted
- **WHEN** 静态验证扫描后端生产代码
- **THEN** 除 Redis 配置、共享 Redis service/facade 和明确范围内的测试外，不存在直接注入 `RedisTemplate` 或 `StringRedisTemplate` 的代码

### Requirement: Redis token operations preserve existing behavior
JWT and download-token Redis operations MUST preserve existing key prefixes, TTL units, refresh-window behavior, stale-token behavior, one-time download-token consumption, and refresh lock behavior.
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
Redis migration notes MUST state whether existing values written by the old serializer remain compatible or must be cleaned up or allowed to expire.
Redis 清理必须说明旧 serializer 写入的既有 Redis value 是否兼容。若不兼容，实现必须记录清理缓存、等待 TTL 过期或部署处理要求。

#### Scenario: Serializer compatibility is assessed
- **WHEN** Redis 缓存写入格式发生变化
- **THEN** 变更文档说明旧缓存值是否仍可读取，或必须被清理/等待过期

### Requirement: Redis behavior is verified
Implementation MUST include focused verification for Redis configuration and shared Redis service behavior.
实现必须包含 Redis 配置和服务行为的 focused verification，覆盖字符串 serializer 接线、字符串 value 操作、显式 typed JSON 操作、现有调用点使用的 hash JSON 操作、TTL、删除/批量删除和锁获取/释放。

#### Scenario: Redis verification runs
- **WHEN** 后端验证运行
- **THEN** Redis 配置和共享服务行为由自动化测试或静态扫描覆盖，足以发现 serializer 漂移、动态类型反序列化漂移和直接 template 使用漂移
