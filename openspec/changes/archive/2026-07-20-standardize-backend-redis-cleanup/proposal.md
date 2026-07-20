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
