# standardize-backend-redis-cleanup 验证记录

## Redis 缓存兼容性

本次变更将 Redis value 和 hash value 统一为字符串序列化，并由 `RedisService` 使用 fastjson2 typed API 显式编解码业务对象。旧 `FastJson2JsonRedisSerializer` 写入的 `WriteClassName` 类型元数据缓存值不作为兼容目标。

部署处理要求：

- 短 TTL key：等待 TTL 自然过期后观察缓存命中率和错误日志。
- 长 TTL 或无 TTL key：部署前清理受影响业务 key，包括 `sys:config:*`、`FrontendCache_*`、`nodeHistory`。
- token 相关 key 保持字符串 value 语义，key 前缀和 TTL 单位不变，包括 `token:*`、`token:refresh:*`、`token:stale:*`、`token:lock:*`、`download:*`。

## Review Fixes

- `RedisService#unlock` 使用 Lua compare-and-delete，避免 GET 后 DELETE 的非原子释放锁问题。
- `JwtUtil#refreshToken` 使用每次刷新生成的唯一锁值，并用同一个值释放锁。
- `RedisService` 拒绝 `TypeReference<Object>` 读取，避免通用 Object 反序列化入口回流。
- `SysConfigServiceImpl` 在缓存 JSON 损坏或旧格式无法解析时回退数据库查询。
- `TaosSqlBuilder` 对 `startTime` / `endTime` 使用严格 `yyyy-MM-dd HH:mm:ss` 解析和格式化，拒绝注入片段。

## 验证命令

```bash
mvn -q "-DskipTests=false" "-Dtest=com.siact.hydrocore.tdengine.util.TaosSqlBuilderTest,com.siact.hydrocore.common.redis.RedisServiceTest,com.siact.hydrocore.common.utils.JwtUtilTest,com.siact.hydrocore.module.system.service.impl.SysConfigServiceImplTest" test
mvn -q "-DskipTests=false" "-Dtest=com.siact.hydrocore.common.redis.RedisConfigTest,com.siact.hydrocore.common.redis.RedisServiceTest,com.siact.hydrocore.common.utils.JwtUtilTest,com.siact.hydrocore.architecture.RuntimeContractStaticScanTest,com.siact.hydrocore.common.api.ApiResponseTest,com.siact.hydrocore.common.exception.GlobalExceptionHandlerTest,com.siact.hydrocore.core.security.handler.SecurityHandlerResponseTest,com.siact.hydrocore.tdengine.util.TaosSqlBuilderTest,com.siact.hydrocore.module.system.service.impl.SysConfigServiceImplTest" test
mvn -q clean "-DskipTests=false" test
rg -n "StringRedisTemplate|FastJson2JsonRedisSerializer|WriteClassName|autoTypeFilter|AUTO_TYPE_FILTER|parseObject\([^\n]*Object\.class|common\.result\.ResultCode|common\.result\.IErrorCode" src/main/java
rg -n "setCacheObject|getCacheObject|setCacheMapValue|getCacheMapValue|getMultiCacheMapValue|setCacheMap" src/main/java
```

结果：前三个 Maven 命令均退出码 0；两条 `rg` 扫描均无匹配。
