---
comet_change: standardize-backend-redis-cleanup
role: technical-design
canonical_spec: openspec
archived-with: 2026-07-20-standardize-backend-redis-cleanup
status: final
---

# 统一后端 Redis 管理与遗留返回码清理技术设计

## 背景

HydroCore 后端当前 Redis 相关代码集中在 `common.redis`，但实际使用边界分散：

- `RedisConfig` 使用 `FastJson2JsonRedisSerializer<Object>` 作为 value/hash value serializer，并通过 `WriteClassName` 写入类型信息。
- `RedisService` 注入 raw `RedisTemplate`，业务调用点难以从方法签名判断序列化语义。
- `JwtUtil` 直接使用 `StringRedisTemplate`，绕过公共 Redis 服务。
- `SysConfigServiceImpl` 已经手动把对象写成 JSON 字符串再缓存，说明当前项目里“字符串 Redis + 显式 JSON 编解码”已有实践基础。
- `ResultCode` 与 `IErrorCode` 仅彼此引用，和现有 `ApiResponseCode` 重复。

网上资料核对后的结论是：Spring Data Redis 官方支持 `StringRedisSerializer`，也提示不要在不可信环境依赖 Java 原生反序列化；fastjson2 官方文档说明 AutoType 默认关闭，开启需要谨慎。因此本设计不采用 Jackson 全局 serializer，也不保留当前 AutoType 风格的 FastJson Redis serializer。

参考：

- Spring Data Redis serializers: https://docs.spring.io/spring-data/data-redis/docs/current-SNAPSHOT/reference/html/
- Fastjson2 AutoType: https://alibaba.github.io/fastjson2/autotype_cn.html

## 目标

- Redis 底层存储统一为字符串：key、value、hash key、hash value 都使用 `StringRedisSerializer`。
- `RedisService` 成为业务代码唯一 Redis 访问边界，统一管理字符串操作、显式 JSON 编解码、hash JSON、TTL、删除、批量删除和 SETNX 风格锁。
- JSON 编解码复用项目已有 fastjson2 依赖，但必须通过明确目标类型读取缓存。
- 禁止 Redis 缓存路径使用 AutoType、`WriteClassName` 或 `Object.class` 通用反序列化恢复业务对象。
- `JwtUtil` 不再直接注入 `StringRedisTemplate`，token key、TTL、刷新窗口、stale-token、download-token 和锁语义保持不变。
- 删除无引用的 `ResultCode` 与 `IErrorCode`，保留 `ApiResponseCode` 作为 REST 响应码来源。

## 非目标

- 不引入 Jackson Redis serializer、Redisson、Spring Cache abstraction、Kryo、Protostuff 或其他新缓存框架。
- 不改变 Redis key 命名、TTL、JWT claim、HTTP response envelope 或数据库 schema。
- 不兼容 Redis 中已经存在的旧 FastJson `WriteClassName` 缓存值。部署时通过 TTL 自然过期或显式清理处理。
- 不重构 TDengine、MQTT、WebSocket、业务查询逻辑或异常体系命名。

## 方案

### Redis 配置

`RedisConfig` 提供字符串语义模板。推荐保留一个 `StringRedisTemplate` 或等价的 `RedisTemplate<String, String>`，并确保 key、value、hash key、hash value 都是 `StringRedisSerializer`。`RedisMessageListenerContainer` 保持现有连接工厂配置。

删除 `FastJson2JsonRedisSerializer` 的 Redis 配置接线。若 fastjson2 仍被 DTO/上游接口解析使用，保留 Maven 依赖；本次只删除 Redis serializer 类和使用点。

### RedisService 边界

`RedisService` 对业务层暴露语义明确的方法：

- 字符串值：`setString`、`getString`、`delete`、`deleteAll`、`hasKey`、`expire`、`getExpire`
- JSON 值：`setJson`、`getJson(String, Class<T>)`、`getJson(String, TypeReference<T>)`
- hash JSON：`putHashJson`、`getHashJson`、`multiGetHashJson`、`deleteHash`
- 集合/list/set：保留现有调用点需要的方法，底层以 JSON 字符串或字符串集合表达，避免通用 Object 反序列化
- 锁：`setIfAbsent` 或 `tryLock`、`unlock`

方法命名应让调用方知道读写的是字符串还是 JSON。实现中 JSON 读取必须要求目标类型，不提供“给我一个 Object”的通用恢复方法。

### JSON 策略

使用 fastjson2 typed API：

- 写入：对象转普通 JSON 字符串
- 读取：`JSON.parseObject(json, clazz)` 或 `JSON.parseObject(json, typeReference)`
- 禁止：`JSONWriter.Feature.WriteClassName`
- 禁止：AutoType filter
- 禁止：`JSON.parseObject(json, Object.class)` 用于业务缓存恢复

这样 Redis 中存储的是可读 JSON，排查和迁移简单，也不会把 Java 包名/类名写进缓存。

### 调用点迁移

`JwtUtil` 改为依赖 `RedisService`：

- token 存储仍写 `token:<tokenValue>` -> `userId`
- refresh token 仍写 `token:refresh:<userId>:<sessionId>` -> `tokenValue`
- stale token 仍写 `token:stale:<oldToken>` -> `newToken`
- refresh lock 仍使用 `token:lock:<oldToken>` 和 10 秒 TTL
- download token 仍写 `download:<token>`，消费后删除

`SysConfigServiceImpl` 可以直接使用 `setJson/getJson` 读取 `SysConfigDTO` 或 `List<SysConfigDTO>`，不再在业务类里自己选择 JSON 工具。

`FrontendCacheServiceImpl` 和 `DataServiceImpl` 保持 hash key 结构，但 hash value 改为字符串 JSON，由 `RedisService` typed API 负责恢复目标类型。

### 遗留返回码

删除 `common.result.ResultCode` 与 `IErrorCode`。扩展 runtime-contract 静态扫描：

- 禁止生产代码残留 `com.siact.hydrocore.common.result.ResultCode`
- 禁止生产代码残留 `com.siact.hydrocore.common.result.IErrorCode`
- 确认 API 响应仍通过 `ApiResponse<T>` 和 `ApiResponseCode`

异常包中的 `BaseErrorInfoInterface#getResultCode()` 是另一套异常接口命名，本次不改名，避免扩大范围。

## 风险与处理

- 旧缓存不兼容：旧 FastJson 类型元数据缓存可能无法读取。处理方式是在验证报告或后端文档中记录 TTL 等待或部署清理要求。
- typed API 改造量增加：相比全局 Object serializer，需要调用点传目标类型。处理方式是只覆盖现有调用点需要的 API，不做大范围抽象。
- token 刷新并发风险：`JwtUtil` 迁移必须用测试固定 stale-token 和 SETNX 锁行为。
- 外部依赖风险：`ResultCode` 删除前后用 `rg` 和 `mvn test` 验证；若发现仓库外发布 API 依赖，需回到设计阶段重新确认兼容策略。

## 测试策略

- Redis 配置测试：断言 key、value、hash key、hash value 都使用字符串 serializer。
- RedisService 测试：覆盖 string value、typed JSON value、hash JSON、TTL、delete/batch delete、tryLock/unlock。
- JwtUtil 测试：覆盖生成 token、刷新窗口、stale token、并发锁、download token 一次性消费。
- 静态扫描测试：禁止业务生产代码直接注入 `RedisTemplate` / `StringRedisTemplate`，禁止 Redis 缓存路径使用 AutoType / `WriteClassName` / `Object.class` 通用反序列化，禁止 `ResultCode` / `IErrorCode` 残留。
- 回归：运行 Redis/JWT/ApiResponse/exception/security/runtime-contract focused tests 和完整 `mvn -q test`。

## 实施顺序

1. 先写 Redis 配置、RedisService、JwtUtil 和静态扫描测试，记录当前失败点。
2. 改 Redis 配置为字符串模板，移除 FastJson Redis serializer 接线。
3. 重构 `RedisService`，补齐字符串、typed JSON、hash JSON 和锁 API。
4. 迁移 `JwtUtil`、`SysConfigServiceImpl`、`FrontendCacheServiceImpl`、`DataServiceImpl`。
5. 删除 `FastJson2JsonRedisSerializer`、`ResultCode`、`IErrorCode`。
6. 补文档或验证报告中的 Redis 缓存清理说明。
7. 运行 focused tests 和完整后端测试。
