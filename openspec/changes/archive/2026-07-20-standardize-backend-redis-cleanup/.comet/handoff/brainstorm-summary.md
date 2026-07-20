# Brainstorm Summary

- Change: standardize-backend-redis-cleanup
- Date: 2026-07-20

## 确认的技术方案

采用字符串 Redis 存储方案：Redis 底层 key、value、hash key、hash value 统一使用字符串序列化；`RedisService` 作为后端业务代码唯一 Redis 访问边界，负责字符串操作、显式 typed JSON 编解码、hash JSON、TTL、删除、批量删除和 SETNX 风格锁。JSON 实现复用项目已有 fastjson2，但禁止 AutoType、`WriteClassName` 和 `Object.class` 通用反序列化作为 Redis 缓存恢复机制。

`JwtUtil`、系统配置缓存、前端配置缓存和历史节点缓存迁移到统一服务边界。`ResultCode` 与 `IErrorCode` 在确认无引用后直接删除。

已放弃方案：

- 全局 Jackson serializer：官方支持且可行，但会引入另一套 Redis JSON 实现和动态 Object serializer 边界；用户明确希望采用不用 Jackson 的更好方式。
- 继续使用当前 `FastJson2JsonRedisSerializer`：兼容旧缓存风险较低，但依赖 `WriteClassName` 与 AutoType filter，不适合作为长期统一边界。
- Kryo/Protostuff 等二进制 serializer：性能可能更好，但排查困难且引入额外依赖，不符合本次清理目标。

## 关键取舍与风险

- 取舍：选择可读字符串 JSON 和显式目标类型，优先于自动恢复任意 Java 对象。
- 风险：Redis 中已有 FastJson 类型元数据缓存值可能无法被新字符串 JSON API 读取。本次不承诺兼容旧缓存，依赖 TTL 过期或部署时清理缓存。
- 风险：`JwtUtil` 的刷新锁和 stale-token 逻辑迁移后若测试不足，可能影响并发刷新。需要 focused tests 固定 SETNX、TTL、delete、stale token 行为。
- 风险：静态扫描禁止业务层直接注入 RedisTemplate 时，需要允许 Redis 配置类、Redis 服务类和测试例外。
- 风险：删除 `ResultCode` 可能影响仓库外未纳入编译的依赖；当前设计以本后端子仓库 `rg` 和 `mvn test` 为准。

## 测试策略

- Redis 配置测试：验证 key、value、hash key、hash value 都使用字符串 serializer。
- RedisService 测试：覆盖 string value、typed JSON value、hash JSON、TTL、delete/batch delete、SETNX lock、unlock。
- JwtUtil 测试：覆盖 token 写入、refresh window、stale token、download token 一次性消费、锁语义。
- 静态扫描：禁止生产业务代码直接注入 `RedisTemplate` / `StringRedisTemplate`，禁止 Redis 缓存路径使用 AutoType / `WriteClassName` / `Object.class` 通用反序列化，禁止 `common.result.ResultCode` / `IErrorCode` 残留。
- 回归验证：运行 Redis/JWT/ApiResponse/exception/security/runtime-contract focused tests 和完整 `mvn -q test`。

## Spec Patch

已回写 `backend-redis-management` delta spec：从“统一 JSON serializer”调整为“Redis 底层字符串存储 + RedisService 显式 typed JSON 编解码 + 禁用动态类型元数据”。
