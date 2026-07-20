---
change: standardize-backend-redis-cleanup
design-doc: docs/superpowers/specs/2026-07-20-standardize-backend-redis-cleanup-design.md
base-ref: 831398c3f961e22dea5de0ca3f26b88668ca1948
backend-base-ref: 8064ca321a666e6a680cfa98259ca4f52616b92d
archived-with: 2026-07-20-standardize-backend-redis-cleanup
---

# 统一后端 Redis 管理与遗留返回码清理 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** �?HydroCore 后端 Redis 底层统一为字符串序列化，并把业务对象缓存编解码收敛到 `RedisService` �?fastjson2 typed API，同时清理未使用�?`ResultCode` / `IErrorCode`�?
**Architecture:** `RedisConfig` 只负责提供字符串语义�?`RedisTemplate<String, String>` �?listener container；`RedisService` 是唯一业务 Redis 边界，内部用 fastjson2 普�?JSON 字符串写入，并在读取时强制调用方提供 `Class<T>` �?`com.alibaba.fastjson2.TypeReference<T>`。业务调用点只选择字符串、typed JSON、hash typed JSON、TTL、删除、批量删除和 SETNX 锁语义，不使�?Jackson Redis serializer，不使用 AutoType、`WriteClassName` �?`Object.class` 通用恢复业务对象�?
**Tech Stack:** Java 8、Spring Boot 2.6.13、Spring Data Redis、fastjson2 2.0.45、JJWT 0.11.5、JUnit 5、AssertJ、Mockito、Maven Surefire 2.22.2�?
## Global Constraints

- 工作目录�?`D:\project\HydroCore`，后端独�?Git repo 位于 `hydrocore-be`�?- 执行实施计划时只修改 `hydrocore-be` 内与�?change 直接相关的源码、测试和验证文档；不要还原他人改动�?- Redis key、hash key、TTL、JWT claim、HTTP response envelope 和数据库 schema 保持不变�?- Redis key、value、hash key、hash value 必须使用 `StringRedisSerializer`�?- Redis 缓存对象写入必须是普�?JSON 字符串；读取必须使用 `JSON.parseObject(json, clazz)` �?`JSON.parseObject(json, typeReference)`�?- 禁止 Redis 缓存路径使用 `JSONWriter.Feature.WriteClassName`、AutoType filter、`JSON.parseObject(json, Object.class)` �?`Object.class` 通用反序列化恢复业务对象�?- 不引�?Jackson Redis serializer、Redisson、Spring Cache abstraction、Kryo、Protostuff 或其他新缓存框架�?- `JwtUtil` 不再直接注入 `StringRedisTemplate`，但 token key 前缀、TTL 单位、refresh-window、stale-token、download-token 一次性消费和刷新锁语义保持不变�?- 删除 `com.siact.hydrocore.common.result.ResultCode` �?`IErrorCode` 前先用静态扫描确认没有生产调用；REST 响应码继续由 `ApiResponse<T>` �?`ApiResponseCode` 承担�?- Maven 测试命令必须显式�?`-DskipTests=false`，因�?`hydrocore-be/pom.xml` 当前 Surefire 配置默认跳过测试�?
---

## File Structure

- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/redis/RedisConfig.java`
  - 负责创建字符�?Redis template �?`RedisMessageListenerContainer`�?- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/redis/RedisService.java`
  - 负责所有业�?Redis 访问，包括字符串、typed JSON、hash typed JSON、TTL、删除、批量删除、keys 扫描�?SETNX 锁�?- Delete: `hydrocore-be/src/main/java/com/siact/hydrocore/common/redis/FastJson2JsonRedisSerializer.java`
  - �?FastJson Redis serializer，当前写�?`WriteClassName` 并启�?AutoType filter�?- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/utils/JwtUtil.java`
  - 从直�?`StringRedisTemplate` 迁移�?`RedisService`�?- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/service/impl/SysConfigServiceImpl.java`
  - 系统配置缓存迁移�?`RedisService#setJson/getJson`，移除缓存路径里�?`JacksonUtils` �?Jackson `TypeReference`�?- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/module/base/service/impl/FrontendCacheServiceImpl.java`
  - 前端配置 hash 访问迁移为字符串 hash API，保持现有返�?JSON 字符串包装行为�?- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/sec/sevice/impl/DataServiceImpl.java`
  - `nodeHistory` hash value 迁移�?typed JSON list�?- Delete: `hydrocore-be/src/main/java/com/siact/hydrocore/common/result/ResultCode.java`
- Delete: `hydrocore-be/src/main/java/com/siact/hydrocore/common/result/IErrorCode.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/common/redis/RedisConfigTest.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/common/redis/RedisServiceTest.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/common/utils/JwtUtilTest.java`
- Modify: `hydrocore-be/src/test/java/com/siact/hydrocore/architecture/RuntimeContractStaticScanTest.java`
- Create: `docs/superpowers/verification/2026-07-20-standardize-backend-redis-cleanup.md`
  - 记录�?Redis 缓存不兼容处理：等待 TTL 自然过期或部署时清理受影�?key�?
### Shared Interfaces

`RedisService` 对后续任务提供以下公共方法；删除或重命名方法时必须同步更新所有调用点和测试�?
```java
public void setString(String key, String value);
public void setString(String key, String value, long timeout, TimeUnit unit);
public String getString(String key);
public Boolean hasKey(String key);
public boolean expire(String key, long timeout);
public boolean expire(String key, long timeout, TimeUnit unit);
public long getExpire(String key);
public boolean delete(String key);
public boolean deleteAll(Collection<String> keys);
public Set<String> keys(String pattern);

public <T> void setJson(String key, T value);
public <T> void setJson(String key, T value, long timeout, TimeUnit unit);
public <T> T getJson(String key, Class<T> clazz);
public <T> T getJson(String key, com.alibaba.fastjson2.TypeReference<T> typeReference);

public void putHashString(String key, String hKey, String value);
public void putAllHashString(String key, Map<String, String> values);
public List<String> multiGetHashString(String key, Collection<String> hKeys);
public boolean deleteHash(String key, String hKey);

public <T> void putHashJson(String key, String hKey, T value);
public <T> void putHashJson(String key, String hKey, T value, long timeout, TimeUnit unit);
public <T> T getHashJson(String key, String hKey, Class<T> clazz);
public <T> T getHashJson(String key, String hKey, com.alibaba.fastjson2.TypeReference<T> typeReference);
public <T> List<T> multiGetHashJson(String key, Collection<String> hKeys, Class<T> clazz);

public boolean tryLock(String key, String value, long timeoutSeconds);
public void unlock(String key, String value);
```

## Task 1: 固定运行时契约静态扫�?
**Files:**
- Modify: `hydrocore-be/src/test/java/com/siact/hydrocore/architecture/RuntimeContractStaticScanTest.java`

**Interfaces:**
- Consumes: 当前 `scanJavaFiles()` �?`matchingLines(Path, List<String>)`�?- Produces: 三个静态扫描测试方法：
  - `runtimeCodeDoesNotInjectRedisTemplatesOutsideRedisBoundary()`
  - `redisCacheCodeDoesNotUseDynamicTypeMetadata()`
  - `runtimeCodeDoesNotUseLegacyResultCodeContract()`

- [x] **Step 1: 新增直接 template 注入扫描**

�?`RuntimeContractStaticScanTest` 增加允许边界判断，允�?`common/redis/RedisConfig.java` �?`common/redis/RedisService.java`，禁止其它生产代码出�?`RedisTemplate` / `StringRedisTemplate` 注入�?import�?
```java
@Test
void runtimeCodeDoesNotInjectRedisTemplatesOutsideRedisBoundary() throws IOException {
    List<String> forbidden = Arrays.asList(
            "org.springframework.data.redis.core.RedisTemplate",
            "org.springframework.data.redis.core.StringRedisTemplate",
            " RedisTemplate<",
            " StringRedisTemplate "
    );

    List<String> failures = scanJavaFiles()
            .filter(path -> !isRedisBoundary(path))
            .flatMap(path -> matchingLines(path, forbidden).stream())
            .collect(Collectors.toList());

    assertThat(failures).isEmpty();
}

private boolean isRedisBoundary(Path path) {
    return path.endsWith(Paths.get("common", "redis", "RedisConfig.java"))
            || path.endsWith(Paths.get("common", "redis", "RedisService.java"));
}
```

- [x] **Step 2: 新增 Redis 动态类型元数据扫描**

同一测试类增加禁止项，覆盖旧 serializer �?`WriteClassName`、AutoType filter 和业务缓�?`Object.class` 通用恢复�?
```java
@Test
void redisCacheCodeDoesNotUseDynamicTypeMetadata() throws IOException {
    List<String> forbidden = Arrays.asList(
            "JSONWriter.Feature.WriteClassName",
            "JSONReader.autoTypeFilter",
            "AUTO_TYPE_FILTER",
            "parseObject(json, Object.class",
            "parseObject(str, Object.class",
            "new FastJson2JsonRedisSerializer(Object.class)"
    );

    List<String> failures = scanJavaFiles()
            .flatMap(path -> matchingLines(path, forbidden).stream())
            .collect(Collectors.toList());

    assertThat(failures).isEmpty();
}
```

- [x] **Step 3: 新增遗留返回码扫�?*

同一测试类增加扫描，直接识别两个文件路径和生产代码引用�?
```java
@Test
void runtimeCodeDoesNotUseLegacyResultCodeContract() throws IOException {
    List<String> forbidden = Arrays.asList(
            "com.siact.hydrocore.common.result.ResultCode",
            "com.siact.hydrocore.common.result.IErrorCode",
            "ResultCode implements IErrorCode",
            "interface IErrorCode"
    );

    List<String> failures = scanJavaFiles()
            .filter(path -> !path.endsWith(Paths.get("common", "exception", "BaseErrorInfoInterface.java")))
            .flatMap(path -> matchingLines(path, forbidden).stream())
            .collect(Collectors.toList());

    assertThat(failures).isEmpty();
}
```

- [x] **Step 4: 运行静态扫描并确认当前失败**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
```

Expected: FAIL。失败列表至少包�?`common/utils/JwtUtil.java` �?`StringRedisTemplate`、`common/redis/FastJson2JsonRedisSerializer.java` �?`WriteClassName` / AutoType，以�?`common/result/ResultCode.java` / `IErrorCode.java`�?
## Task 2: 字符�?Redis 配置

**Files:**
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/redis/RedisConfig.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/common/redis/RedisConfigTest.java`

**Interfaces:**
- Consumes: Spring `RedisConnectionFactory`�?- Produces: Bean method `public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory connectionFactory)`，key、value、hash key、hash value serializer 都是 `StringRedisSerializer`�?
- [x] **Step 1: �?RedisConfig 失败测试**

创建 `RedisConfigTest`，用 Mockito mock connection factory，不连接真实 Redis�?
```java
package com.siact.hydrocore.common.redis;

import org.junit.jupiter.api.Test;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class RedisConfigTest {
    @Test
    void redisTemplateUsesStringSerializersForKeysValuesAndHashes() {
        RedisTemplate<String, String> template = new RedisConfig().redisTemplate(mock(RedisConnectionFactory.class));

        assertThat(template.getKeySerializer()).isInstanceOf(StringRedisSerializer.class);
        assertThat(template.getValueSerializer()).isInstanceOf(StringRedisSerializer.class);
        assertThat(template.getHashKeySerializer()).isInstanceOf(StringRedisSerializer.class);
        assertThat(template.getHashValueSerializer()).isInstanceOf(StringRedisSerializer.class);
    }
}
```

- [x] **Step 2: 运行 RedisConfig 测试并确认当前失�?*

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.redis.RedisConfigTest test
```

Expected: FAIL，原因是当前 value/hash value serializer �?`FastJson2JsonRedisSerializer`�?
- [x] **Step 3: 实现字符�?Redis template**

修改 `RedisConfig#redisTemplate`，删�?`FastJson2JsonRedisSerializer` 接线�?
```java
@Bean
public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory connectionFactory) {
    RedisTemplate<String, String> template = new RedisTemplate<>();
    template.setConnectionFactory(connectionFactory);

    StringRedisSerializer serializer = new StringRedisSerializer();
    template.setKeySerializer(serializer);
    template.setValueSerializer(serializer);
    template.setHashKeySerializer(serializer);
    template.setHashValueSerializer(serializer);

    template.afterPropertiesSet();
    return template;
}
```

- [x] **Step 4: 运行 RedisConfig 测试并确认通过**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.redis.RedisConfigTest test
```

Expected: PASS，四�?serializer 断言全部�?`StringRedisSerializer`�?
## Task 3: RedisService typed 边界

**Files:**
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/redis/RedisService.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/common/redis/RedisServiceTest.java`

**Interfaces:**
- Consumes: Task 2 �?`RedisTemplate<String, String>`�?- Produces: Shared Interfaces 中列出的字符串、typed JSON、hash string、hash typed JSON、TTL、删除、批量删除、keys 和锁方法�?
- [x] **Step 1: �?RedisService 字符串和 JSON 失败测试**

创建 `RedisServiceTest`，使�?mock `RedisTemplate<String, String>` �?Redis operations，验证写入的是字符串 JSON，不�?Java 类型元数据�?
```java
package com.siact.hydrocore.common.redis;

import com.alibaba.fastjson2.TypeReference;
import lombok.Data;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.springframework.data.redis.core.HashOperations;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;

import java.util.Arrays;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class RedisServiceTest {
    private RedisTemplate<String, String> redisTemplate;
    private ValueOperations<String, String> valueOperations;
    private HashOperations<String, String, String> hashOperations;
    private RedisService redisService;

    @BeforeEach
    void setUp() {
        redisTemplate = mock(RedisTemplate.class);
        valueOperations = mock(ValueOperations.class);
        hashOperations = mock(HashOperations.class);
        when(redisTemplate.opsForValue()).thenReturn(valueOperations);
        when(redisTemplate.opsForHash()).thenReturn(hashOperations);
        redisService = new RedisService(redisTemplate);
    }

    @Test
    void setJsonStoresPlainJsonStringWithoutClassMetadata() {
        CacheDto dto = new CacheDto();
        dto.setCode("A1");

        redisService.setJson("cache:1", dto, 5, TimeUnit.MINUTES);

        ArgumentCaptor<String> jsonCaptor = ArgumentCaptor.forClass(String.class);
        verify(valueOperations).set(eq("cache:1"), jsonCaptor.capture(), eq(5L), eq(TimeUnit.MINUTES));
        assertThat(jsonCaptor.getValue()).contains("\"code\":\"A1\"");
        assertThat(jsonCaptor.getValue()).doesNotContain("@type");
        assertThat(jsonCaptor.getValue()).doesNotContain("com.siact");
    }

    @Test
    void getJsonRequiresConcreteClass() {
        when(valueOperations.get("cache:1")).thenReturn("{\"code\":\"A1\"}");

        CacheDto dto = redisService.getJson("cache:1", CacheDto.class);

        assertThat(dto.getCode()).isEqualTo("A1");
    }

    @Test
    void getJsonRejectsObjectClass() {
        assertThatThrownBy(() -> redisService.getJson("cache:1", Object.class))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Object.class");
    }

    @Test
    void getJsonSupportsTypeReference() {
        when(valueOperations.get("cache:list")).thenReturn("[{\"code\":\"A1\"},{\"code\":\"B2\"}]");

        List<CacheDto> list = redisService.getJson("cache:list", new TypeReference<List<CacheDto>>() {});

        assertThat(list).extracting(CacheDto::getCode).containsExactly("A1", "B2");
    }

    @Data
    private static class CacheDto {
        private String code;
    }
}
```

- [x] **Step 2: �?RedisService hash、删除和锁失败测�?*

在同一测试类增加下面测试，覆盖现有调用点需要的 hash string、hash typed JSON、批量删除和 SETNX 锁�?
```java
@Test
void hashJsonStoresAndReadsTypedList() {
    CacheDto dto = new CacheDto();
    dto.setCode("A1");
    redisService.putHashJson("nodeHistory", "field-1", Arrays.asList(dto), 60, TimeUnit.SECONDS);

    ArgumentCaptor<String> jsonCaptor = ArgumentCaptor.forClass(String.class);
    verify(hashOperations).put(eq("nodeHistory"), eq("field-1"), jsonCaptor.capture());
    assertThat(jsonCaptor.getValue()).contains("\"code\":\"A1\"");
    verify(redisTemplate).expire("nodeHistory", 60, TimeUnit.SECONDS);

    when(hashOperations.get("nodeHistory", "field-1")).thenReturn("[{\"code\":\"A1\"}]");
    List<CacheDto> cached = redisService.getHashJson("nodeHistory", "field-1", new TypeReference<List<CacheDto>>() {});
    assertThat(cached).extracting(CacheDto::getCode).containsExactly("A1");
}

@Test
void multiGetHashStringKeepsFrontendCacheValuesAsStrings() {
    when(hashOperations.multiGet("FrontendCache_7", Arrays.asList("theme", "layout")))
            .thenReturn(Arrays.asList("dark", "{\"compact\":true}"));

    List<String> values = redisService.multiGetHashString("FrontendCache_7", Arrays.asList("theme", "layout"));

    assertThat(values).containsExactly("dark", "{\"compact\":true}");
}

@Test
void deleteAllAndTryLockUseStringTemplateOperations() {
    Collection<String> keys = Arrays.asList("token:refresh:7:a", "token:refresh:7:b");
    when(redisTemplate.delete(keys)).thenReturn(2L);
    when(valueOperations.setIfAbsent("token:lock:old", "1", 10, TimeUnit.SECONDS)).thenReturn(true);

    assertThat(redisService.deleteAll(keys)).isTrue();
    assertThat(redisService.tryLock("token:lock:old", "1", 10)).isTrue();
}
```

- [x] **Step 3: 运行 RedisService 测试并确认当前失�?*

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.redis.RedisServiceTest test
```

Expected: FAIL，原因是当前 `RedisService` 没有 constructor injection、typed JSON、hash string �?typed hash JSON 方法�?
- [x] **Step 4: 实现 RedisService 最小可用边�?*

�?raw `RedisTemplate` 字段替换�?final `RedisTemplate<String, String>`，使�?constructor injection。JSON 读写只使�?fastjson2 typed API，`Class<T>` 入口拒绝 `Object.class`�?
```java
private final RedisTemplate<String, String> redisTemplate;

public RedisService(RedisTemplate<String, String> redisTemplate) {
    this.redisTemplate = redisTemplate;
}

public <T> void setJson(String key, T value, long timeout, TimeUnit unit) {
    setString(key, JSON.toJSONString(value), timeout, unit);
}

public <T> T getJson(String key, Class<T> clazz) {
    rejectObjectClass(clazz);
    String json = getString(key);
    return json == null ? null : JSON.parseObject(json, clazz);
}

public <T> T getJson(String key, TypeReference<T> typeReference) {
    String json = getString(key);
    return json == null ? null : JSON.parseObject(json, typeReference);
}

private void rejectObjectClass(Class<?> clazz) {
    if (Object.class.equals(clazz)) {
        throw new IllegalArgumentException("Redis JSON reads require a concrete target type, not Object.class");
    }
}
```

同时保留调用点需要的 `keys(String pattern)`，但不要恢复旧的 `setCacheObject/getCacheObject/setCacheMapValue/getCacheMapValue` 通用 Object API�?
- [x] **Step 5: 运行 RedisService 测试并确认通过**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.redis.RedisServiceTest test
```

Expected: PASS；测试输出中没有 `WriteClassName`、AutoType �?`Object.class` 通用恢复路径�?
## Task 4: 迁移 JWT Redis 调用�?
**Files:**
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/common/utils/JwtUtil.java`
- Create: `hydrocore-be/src/test/java/com/siact/hydrocore/common/utils/JwtUtilTest.java`

**Interfaces:**
- Consumes: `RedisService#setString/getString/hasKey/delete/deleteAll/keys/tryLock/unlock`�?- Produces: `JwtUtil` 保持现有 public methods：`generateToken`、`parseToken`、`parseTokenAllowExpired`、`getSessionId`、`isTokenValid`、`refreshToken`、`deleteToken`、`deleteRefreshTokens`、`generateDownloadToken`、`consumeDownloadToken`�?
- [x] **Step 1: �?JWT 生成和下�?token 测试**

创建 `JwtUtilTest`，通过 `ReflectionTestUtils` 设置配置字段，mock `RedisService`�?
```java
package com.siact.hydrocore.common.utils;

import com.siact.hydrocore.common.redis.RedisService;
import com.siact.hydrocore.module.system.dto.LoginUser;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.springframework.test.util.ReflectionTestUtils;

import java.util.Collections;
import java.util.Date;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class JwtUtilTest {
    private static final String SECRET = "01234567890123456789012345678901";

    private RedisService redisService;
    private JwtUtil jwtUtil;

    @BeforeEach
    void setUp() {
        redisService = mock(RedisService.class);
        jwtUtil = new JwtUtil();
        ReflectionTestUtils.setField(jwtUtil, "secretKey", SECRET);
        ReflectionTestUtils.setField(jwtUtil, "expiration", 60000L);
        ReflectionTestUtils.setField(jwtUtil, "refreshWindow", 120000L);
        ReflectionTestUtils.setField(jwtUtil, "staleTtl", 5000L);
        ReflectionTestUtils.setField(jwtUtil, "redisService", redisService);
    }

    @Test
    void generateTokenStoresTokenAndRefreshWindowWithMillisecondTtl() {
        LoginUser user = loginUser();

        String token = jwtUtil.generateToken(user);

        assertThat(token).isNotBlank();
        verify(redisService).setString(eq("token:" + token), eq("7"), eq(60000L), eq(TimeUnit.MILLISECONDS));
        verify(redisService).setString(org.mockito.ArgumentMatchers.startsWith("token:refresh:7:"), eq(token), eq(120000L), eq(TimeUnit.MILLISECONDS));
    }

    @Test
    void consumeDownloadTokenReadsDeletesAndReturnsLoginUser() {
        when(redisService.getString("download:abc")).thenReturn("7:acct:Alice");

        LoginUser user = jwtUtil.consumeDownloadToken("abc");

        assertThat(user.getId()).isEqualTo(7L);
        assertThat(user.getAccount()).isEqualTo("acct");
        assertThat(user.getUsername()).isEqualTo("Alice");
        verify(redisService).delete("download:abc");
    }

    private LoginUser loginUser() {
        LoginUser user = new LoginUser();
        user.setId(7L);
        user.setAccount("acct");
        user.setUsername("Alice");
        return user;
    }
}
```

- [x] **Step 2: �?refresh-window、stale-token 和锁测试**

�?`JwtUtilTest` 增加测试，固定过�?token 刷新行为�?
```java
@Test
void refreshTokenReturnsStaleTokenWhenConcurrentRefreshAlreadyCompleted() {
    when(redisService.getString("token:stale:old")).thenReturn("new-token");

    assertThat(jwtUtil.refreshToken("old")).isEqualTo("new-token");
}

@Test
void refreshTokenUsesSetNxLockAndDeletesConsumedRefreshWindow() {
    String oldToken = expiredToken("session-a");
    when(redisService.getString("token:stale:" + oldToken)).thenReturn(null);
    when(redisService.hasKey("token:refresh:7:session-a")).thenReturn(true);
    when(redisService.tryLock("token:lock:" + oldToken, "1", 10)).thenReturn(true);

    String newToken = jwtUtil.refreshToken(oldToken);

    assertThat(newToken).isNotBlank();
    verify(redisService).setString(eq("token:stale:" + oldToken), eq(newToken), eq(5000L), eq(TimeUnit.MILLISECONDS));
    verify(redisService).delete("token:refresh:7:session-a");
    verify(redisService).unlock("token:lock:" + oldToken, "1");
}

@Test
void deleteRefreshTokensDeletesAllMatchingKeys() {
    when(redisService.keys("token:refresh:7:*")).thenReturn(Collections.singleton("token:refresh:7:a"));

    jwtUtil.deleteRefreshTokens(7L);

    verify(redisService).deleteAll(Collections.singleton("token:refresh:7:a"));
}

private String expiredToken(String sessionId) {
    return Jwts.builder()
            .setSubject("acct")
            .claim("userId", 7L)
            .claim("sessionId", sessionId)
            .setIssuedAt(new Date(System.currentTimeMillis() - 120000L))
            .setExpiration(new Date(System.currentTimeMillis() - 60000L))
            .signWith(SignatureAlgorithm.HS256, SECRET)
            .compact();
}
```

- [x] **Step 3: 运行 JWT 测试并确认当前失�?*

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.utils.JwtUtilTest test
```

Expected: FAIL，原因是当前 `JwtUtil` 没有 `redisService` 字段，且直接使用 `StringRedisTemplate`�?
- [x] **Step 4: 迁移 JwtUtil �?RedisService**

将字段替换为�?
```java
@Autowired
private RedisService redisService;
```

按下面映射替�?Redis 操作�?
```java
redisTemplate.opsForValue().set(key, value, ttl, unit) -> redisService.setString(key, value, ttl, unit)
redisTemplate.opsForValue().get(key) -> redisService.getString(key)
redisTemplate.hasKey(key) -> redisService.hasKey(key)
redisTemplate.delete(key) -> redisService.delete(key)
redisTemplate.delete(keys) -> redisService.deleteAll(keys)
redisTemplate.keys(pattern) -> redisService.keys(pattern)
redisTemplate.opsForValue().setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS) -> redisService.tryLock(lockKey, "1", 10)
redisTemplate.delete(lockKey) in finally -> redisService.unlock(lockKey, "1")
```

保持常量值不变：`TOKEN_PREFIX = "token:"`、`REFRESH_PREFIX = "token:refresh:"`、`STALE_PREFIX = "token:stale:"`、`LOCK_PREFIX = "token:lock:"`、`DOWNLOAD_PREFIX = "download:"`�?
- [x] **Step 5: 运行 JWT 测试并确认通过**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.utils.JwtUtilTest test
```

Expected: PASS；`generateToken` �?token TTL 仍是 `TimeUnit.MILLISECONDS`，refresh lock TTL 仍是 10 秒�?
## Task 5: 迁移业务缓存调用�?
**Files:**
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/module/system/service/impl/SysConfigServiceImpl.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/module/base/service/impl/FrontendCacheServiceImpl.java`
- Modify: `hydrocore-be/src/main/java/com/siact/hydrocore/sec/sevice/impl/DataServiceImpl.java`
- Modify: `hydrocore-be/src/test/java/com/siact/hydrocore/architecture/RuntimeContractStaticScanTest.java`

**Interfaces:**
- Consumes: Task 3 �?`RedisService#setJson/getJson/putAllHashString/multiGetHashString/delete/deleteHash/putHashJson/getHashJson`�?- Produces: 所有现有业务缓存调用点不再使用通用 Object Redis 读写；系统配置和节点历史缓存读取显式传目标类型�?
- [x] **Step 1: 增加业务缓存 typed API 静态扫�?*

�?`RuntimeContractStaticScanTest` 增加测试，防止业务代码继续调用旧 RedisService 通用 Object API�?
```java
@Test
void businessRedisCallersUseExplicitStringOrTypedJsonMethods() throws IOException {
    List<String> forbidden = Arrays.asList(
            ".setCacheObject(",
            ".getCacheObject(",
            ".setCacheMapValue(",
            ".getCacheMapValue(",
            ".getMultiCacheMapValue(",
            ".setCacheMap("
    );

    List<String> failures = scanJavaFiles()
            .filter(path -> !isRedisBoundary(path))
            .flatMap(path -> matchingLines(path, forbidden).stream())
            .collect(Collectors.toList());

    assertThat(failures).isEmpty();
}
```

- [x] **Step 2: 运行静态扫描并确认当前失败**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
```

Expected: FAIL，失败点包括 `SysConfigServiceImpl`、`FrontendCacheServiceImpl` �?`DataServiceImpl` 的旧 RedisService 调用�?
- [x] **Step 3: 迁移 SysConfigServiceImpl**

替换缓存读取和写入，移除 `com.fasterxml.jackson.core.type.TypeReference` 与缓存路径里�?`JacksonUtils` import，新�?`com.alibaba.fastjson2.TypeReference`�?
```java
SysConfigDTO cached = redisService.getJson(cacheKey, SysConfigDTO.class);
if (cached != null) {
    return cached;
}
```

模块列表缓存读取使用�?
```java
List<SysConfigDTO> cached = redisService.getJson(cacheKey, new TypeReference<List<SysConfigDTO>>() {});
if (cached != null) {
    return cached;
}
```

缓存写入改为�?
```java
private void writeCache(String key, Object value) {
    try {
        redisService.setJson(key, value, CACHE_EXPIRE_HOURS, TimeUnit.HOURS);
    } catch (Exception e) {
        log.error("写入配置缓存失败: {}", e.getMessage());
    }
}
```

缓存删除改为�?
```java
redisService.delete(CACHE_KEY_PREFIX + scCode);
redisService.delete(CACHE_KEY_MODULE + module.name());
```

- [x] **Step 4: 迁移 FrontendCacheServiceImpl**

保持 key `FrontendCache_` 和现有返�?JSON 字符串包装行为；�?hash value 作为字符串写入和读取�?
```java
HashMap<String, String> paramMap = new HashMap<>();
paramMap.put(key, value);
redisService.putAllHashString(getFrontendCache(userId), paramMap);
```

读取�?field�?
```java
List<String> cacheValueList = redisService.multiGetHashString(getFrontendCache(userId), Arrays.asList(keyArr));
```

结果构造保持：

```java
String cacheVal = cacheValueList.get(i);
resultMap.put(keyArr[i], ObjectUtils.isEmpty(cacheVal) ? null : JSON.toJSONString(cacheVal));
```

删除改为�?
```java
redisService.delete(getFrontendCache(userId));
redisService.deleteHash(getFrontendCache(userId), key);
```

- [x] **Step 5: 迁移 DataServiceImpl nodeHistory hash 缓存**

新增 fastjson2 `TypeReference` import，并�?`queryNoteIntervalVal` �?hash object 读取改为 typed list�?
```java
List<IntervalDataDto> cached = redisService.getHashJson(
        REDISKEY_NODEHISTORY,
        mdsObjStr,
        new com.alibaba.fastjson2.TypeReference<List<IntervalDataDto>>() {}
);
if (cached == null) {
    log.info("nodeHistory 缓存中没有需要通过接口获取!!!");
    R<List<PropValFMResultVo>> propData = propService.nodeHistory(vo);
    List<PropValFMResultVo> propDataDataList = analysisSiactSecData(propData);
    List<IntervalDataDto> dataDtoListCopy = new ArrayList<>();
    propDataDataList.forEach(itemData -> {
        String modelCode = itemData.getModelCode();
        analysisRequestData(dataDtoListCopy, itemData, modelCode);
    });
    dataDtoList = dataDtoListCopy;
    if (CollectionUtils.isNotEmpty(dataDtoList)) {
        redisService.putHashJson(REDISKEY_NODEHISTORY, mdsObjStr, dataDtoList, nodeHistoryTimeOut, TimeUnit.SECONDS);
    }
} else {
    dataDtoList = cached;
}
```

- [x] **Step 6: 运行业务调用点静态扫描并确认通过**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
```

Expected: PASS；生产代码中�?`common.redis` 边界外没有直�?`RedisTemplate` / `StringRedisTemplate`，业务调用点没有�?RedisService 通用 Object API�?
## Task 6: 删除 FastJson Redis serializer 和遗留返回码

**Files:**
- Delete: `hydrocore-be/src/main/java/com/siact/hydrocore/common/redis/FastJson2JsonRedisSerializer.java`
- Delete: `hydrocore-be/src/main/java/com/siact/hydrocore/common/result/ResultCode.java`
- Delete: `hydrocore-be/src/main/java/com/siact/hydrocore/common/result/IErrorCode.java`
- Modify: `hydrocore-be/src/test/java/com/siact/hydrocore/architecture/RuntimeContractStaticScanTest.java`

**Interfaces:**
- Consumes: Task 1 �?Task 5 的静态扫描�?- Produces: 生产代码中不存在�?Redis serializer、`ResultCode` �?`IErrorCode` 文件；异常包里的 `BaseErrorInfoInterface#getResultCode()` 保持不变�?
- [x] **Step 1: 删除�?Redis serializer**

删除 `FastJson2JsonRedisSerializer.java`，并确认 `RedisConfig` 没有 import 或实例化它�?
Run from `hydrocore-be`:

```bash
rg -n "FastJson2JsonRedisSerializer|WriteClassName|autoTypeFilter|AUTO_TYPE_FILTER" src/main/java
```

Expected: no matches�?
- [x] **Step 2: 删除�?ResultCode �?IErrorCode**

删除 `ResultCode.java` �?`IErrorCode.java`。不要改名异常体系里�?`BaseErrorInfoInterface#getResultCode()`、`CommonEnum#getResultCode()`、`CustomException` �?`ActiveException`，它们不属于�?change �?REST 响应码清理范围�?
Run from `hydrocore-be`:

```bash
rg -n "common\\.result\\.ResultCode|common\\.result\\.IErrorCode|ResultCode implements IErrorCode|interface IErrorCode" src/main/java
```

Expected: no matches�?
- [x] **Step 3: 运行静态扫描并确认通过**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
```

Expected: PASS；`ApiResponseCode` 仍由 `ApiResponseTest`、`GlobalExceptionHandlerTest` �?`SecurityHandlerResponseTest` 覆盖�?
## Task 7: 记录 Redis 缓存迁移处理要求

**Files:**
- Create: `docs/superpowers/verification/2026-07-20-standardize-backend-redis-cleanup.md`

**Interfaces:**
- Consumes: 设计文档的非目标和风险处理要求�?- Produces: 运维可执行的缓存兼容性说明，明确�?`WriteClassName` 缓存不兼容�?
- [x] **Step 1: 创建验证报告**

创建验证报告，内容使用下面文本：

````markdown
# standardize-backend-redis-cleanup Verification Notes

## Redis 缓存兼容�?
�?change �?Redis value �?hash value 统一为字符串序列化，并由 `RedisService` 使用 fastjson2 typed API 显式编解码业务对象。旧 `FastJson2JsonRedisSerializer` 写入�?`WriteClassName` 类型元数据缓存值不作为兼容目标�?
部署处理要求�?
- 对短 TTL key，等�?TTL 自然过期后再观察缓存命中率和错误日志�?- 对长 TTL 或无 TTL key，部署前清理受影响业�?key：`sys:config:*`、`FrontendCache_*`、`nodeHistory`�?- token 相关 key 保持字符�?value 语义，key 前缀�?TTL 单位不变：`token:*`、`token:refresh:*`、`token:stale:*`、`token:lock:*`、`download:*`�?
## 验证命令

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.redis.RedisConfigTest,com.siact.hydrocore.common.redis.RedisServiceTest,com.siact.hydrocore.common.utils.JwtUtilTest,com.siact.hydrocore.architecture.RuntimeContractStaticScanTest test
mvn -q -DskipTests=false test
```
````

- [x] **Step 2: 检查报告不声明�?serializer 兼容**

Run from `D:\project\HydroCore`:

```bash
rg -n "兼容目标|WriteClassName|sys:config|FrontendCache_|nodeHistory" docs/superpowers/verification/2026-07-20-standardize-backend-redis-cleanup.md
```

Expected: matches include `�?FastJson2JsonRedisSerializer 写入�?WriteClassName 类型元数据缓存值不作为兼容目标` and the three business key groups `sys:config:*`、`FrontendCache_*`、`nodeHistory`�?
## Task 8: 最�?focused 验证和完整后端测�?
**Files:**
- Test: `hydrocore-be/src/test/java/com/siact/hydrocore/common/redis/RedisConfigTest.java`
- Test: `hydrocore-be/src/test/java/com/siact/hydrocore/common/redis/RedisServiceTest.java`
- Test: `hydrocore-be/src/test/java/com/siact/hydrocore/common/utils/JwtUtilTest.java`
- Test: `hydrocore-be/src/test/java/com/siact/hydrocore/architecture/RuntimeContractStaticScanTest.java`
- Test: `hydrocore-be/src/test/java/com/siact/hydrocore/common/api/ApiResponseTest.java`
- Test: `hydrocore-be/src/test/java/com/siact/hydrocore/common/exception/GlobalExceptionHandlerTest.java`
- Test: `hydrocore-be/src/test/java/com/siact/hydrocore/core/security/handler/SecurityHandlerResponseTest.java`

**Interfaces:**
- Consumes: Tasks 1-7 的实现�?- Produces: 可提交的后端验证结果�?
- [x] **Step 1: 运行 focused tests**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false -Dtest=com.siact.hydrocore.common.redis.RedisConfigTest,com.siact.hydrocore.common.redis.RedisServiceTest,com.siact.hydrocore.common.utils.JwtUtilTest,com.siact.hydrocore.architecture.RuntimeContractStaticScanTest,com.siact.hydrocore.common.api.ApiResponseTest,com.siact.hydrocore.common.exception.GlobalExceptionHandlerTest,com.siact.hydrocore.core.security.handler.SecurityHandlerResponseTest test
```

Expected: PASS；Redis serializer、RedisService typed API、JWT token Redis 行为、runtime contract、API envelope、exception handler �?security handler 均通过�?
- [x] **Step 2: 运行完整后端测试**

Run from `hydrocore-be`:

```bash
mvn -q -DskipTests=false test
```

Expected: PASS；若失败来自需要外部服务的既有集成测试，记录失败测试类、失败原因和与本 change 的关系，再运�?Task 8 Step 1 �?focused tests 作为�?change 的最低合格证据�?
- [x] **Step 3: 最终源码扫�?*

Run from `hydrocore-be`:

```bash
rg -n "StringRedisTemplate|FastJson2JsonRedisSerializer|WriteClassName|autoTypeFilter|AUTO_TYPE_FILTER|parseObject\\([^\\n]*Object\\.class|common\\.result\\.ResultCode|common\\.result\\.IErrorCode" src/main/java
```

Expected: no matches。若出现 `BaseErrorInfoInterface#getResultCode()`、`CommonEnum#getResultCode()`、`ActiveException` �?`CustomException` 的异常体系命名，不作为失败处理，除非�?import �?`common.result` 包�?
- [x] **Step 4: 检查待提交文件范围**

Run from `hydrocore-be`:

```bash
git status --short
```

Expected: 只包含本计划列出的后端源码、测试和验证报告文件；没有无关格式化或未解释的删除�?
