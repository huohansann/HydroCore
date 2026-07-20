## 1. 基线验证

- [x] 1.1 在 `hydrocore-be` 中运行当前后端 focused tests，并记录 Redis 改造前后的验证结果。
- [x] 1.2 新增静态扫描，识别生产代码中直接注入 `RedisTemplate` / `StringRedisTemplate` 以及遗留 `common.result` 使用。

## 2. Redis 字符串配置与显式 JSON

- [x] 2.1 新增 Redis configuration/template 测试，断言 key、value、hash key、hash value 都使用字符串序列化。
- [x] 2.2 将 FastJson Redis serializer 接线替换为字符串 Redis template 配置，同时保持现有 Redis key 和 hash 结构不变。
- [x] 2.3 确认 Redis 配置和测试不再依赖 `FastJson2JsonRedisSerializer` 后删除该类。

## 3. Redis 服务边界

- [x] 3.1 重构 `RedisService`，注入类型明确的字符串 Redis template，并暴露字符串值、JSON 值、hash JSON、TTL、删除、批量删除和 SETNX 风格锁方法。
- [x] 3.2 为 `RedisService` 新增 focused tests，覆盖字符串读写、显式 typed JSON 读写、hash JSON 读写、TTL、删除、批量删除、锁获取和锁释放行为。
- [x] 3.3 禁止 RedisService 使用 AutoType、`WriteClassName` 或 `Object.class` 通用反序列化恢复业务缓存对象。
- [x] 3.4 迁移现有调用点后移除旧 Object 风格兼容入口，并由静态扫描覆盖。

## 4. 调用点迁移

- [x] 4.1 将 `JwtUtil` 从直接访问 `StringRedisTemplate` 迁移到共享 Redis 服务，保持 token key 前缀、TTL 单位、refresh-window、stale-token、download-token 消费和锁语义不变。
- [x] 4.2 迁移 `SysConfigServiceImpl` 的缓存序列化，使其一致使用统一 Redis 边界，业务代码不再临时选择 serializer。
- [x] 4.3 验证 `FrontendCacheServiceImpl` 和 `DataServiceImpl` 在字符串 hash JSON 下的缓存行为，按需调整转换逻辑以保持业务返回类型。

## 5. 遗留返回码清理

- [x] 5.1 确认生产代码和测试没有引用后，删除无用的 `common.result.ResultCode` 和 `IErrorCode`。
- [x] 5.2 扩展 runtime-contract 验证，防止 `ResultCode` 和 `IErrorCode` 重新作为 REST 响应码契约出现。

## 6. 最终验证

- [x] 6.1 运行 Redis、JWT、ApiResponse、exception-handler、security-handler 和 runtime-contract 相关 focused tests。
- [x] 6.2 在 `hydrocore-be` 中运行完整后端 `mvn -q clean -DskipTests=false test`。
- [x] 6.3 在变更验证报告中记录 Redis 缓存兼容性和清理缓存要求。
