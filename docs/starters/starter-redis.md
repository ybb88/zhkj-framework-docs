# 分布式缓存与分布式锁：zhkj-framework-starter-redis

> **模块定位**: 基于 Spring Data Redis 的缓存、操作门面、发布订阅与分布式锁 Starter，提供 JSON 值序列化、按 `#` 后缀设置缓存 TTL 的缓存管理器、`ZhkjRedis` 操作门面、`@RPubSubListener` 订阅与 `@DistributedLock` 注解。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Redis 7.x `[待确认]`（仓库未声明 Redis 服务端版本）
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-redis`

---

## 📋 1. 架构定位与核心设计

1. **值序列化**: `RedisSerializer.json()` 作为默认值序列化器，供 `redisTemplate`、Spring Cache 与发布订阅共享。
2. **键前缀**: `RedisKeySerializer` 为键统一添加 `zhkj.redis.prefix` 前缀（默认 `zhkj:`），已带前缀的键不重复添加。
3. **单条缓存 TTL**: `RedisAutoCacheManager` 解析 `@Cacheable(cacheNames = "name#ttl")` 中的 `#` 后缀，为该缓存设置过期时间。
4. **分布式锁**: `@DistributedLock` 切面，锁状态存于 Redis，释放时校验持有者标识。切面优先级为 `HIGHEST_PRECEDENCE + 1`，位于事务切面外层。
5. **缓存作用域**: `CacheScope`、`CacheNamespaceResolver`、`ScopeCacheEvictor` 提供按全局、租户、维度、委托维度隔离缓存键的能力。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-redis</artifactId>
</dependency>
```

依赖（来自 `pom.xml`）: `zhkj-framework-core`、`spring-boot-autoconfigure`、`spring-boot-starter-data-redis`、`spring-boot-starter-cache`、`tools.jackson.core:jackson-databind`、`spring-boot-starter-aspectj`。

---

## 🧱 3. 包结构

```
io.github.ybb88.framework.redis
├── cache/
│   ├── ZhkjRedis.java                   # 操作门面
│   ├── CacheKey.java                    # 键 + 过期时间
│   ├── CacheNames.java                  # 缓存名常量接口 (sys / user / auth / datascope)
│   ├── CacheScope.java                  # GLOBAL / TENANT / DIMENSION / DELEGATED
│   ├── CacheNamespaceResolver.java      # 命名空间解析 SPI
│   ├── DefaultCacheNamespaceResolver.java
│   ├── ScopeCacheEvictor.java           # 按租户、维度、模式批量删除
│   └── ICacheKey.java
├── config/
│   ├── RedisTemplateConfiguration.java  # RedisTemplate、StringRedisTemplate、ZhkjRedis
│   ├── ZhkjRedisCacheAutoConfiguration.java  # CacheManager (@Primary)、作用域组件
│   ├── RedisAutoCacheManager.java       # 支持 # TTL 后缀
│   └── RedisPubSubConfiguration.java    # 监听容器、发布器、监听器探测
├── lock/
│   ├── annotation/DistributedLock.java
│   ├── aspect/DistributedLockAspect.java
│   ├── config/DistributedLockAutoConfiguration.java
│   ├── core/                            # LockExecutor、LockFailureStrategy、LockInstance
│   ├── exception/DistributedLockException.java
│   ├── executor/RedisTemplateLockExecutor.java
│   ├── resolver/LockKeyResolver.java
│   └── script/DistributedLockScripts.java
├── props/ZhkjRedisProperties.java       # zhkj.redis.*
├── pubsub/
│   ├── RPubSubListener.java             # 方法注解
│   ├── RPubSubListenerDetector.java     # BeanPostProcessor
│   ├── RPubSubListenerLazyFilter.java
│   ├── RPubSubEvent.java
│   ├── RPubSubPublisher.java
│   ├── RedisPubSubPublisher.java
│   └── ChannelUtil.java
└── serializer/RedisKeySerializer.java
```

---

## 🔁 4. 自动装配

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册:

- `RedisTemplateConfiguration`
- `ZhkjRedisCacheAutoConfiguration`
- `RedisPubSubConfiguration`
- `DistributedLockAutoConfiguration`

| 配置类 | Bean | 条件 |
| :--- | :--- | :--- |
| `RedisTemplateConfiguration` | `redisSerializer`（`RedisSerializer<Object>`，`RedisSerializer.json()`） | 缺失时注册 |
| | `redisTemplate`（`RedisTemplate<String, Object>`） | 名称缺失时注册；键使用 `RedisKeySerializer`，值使用 `redisSerializer` |
| | `stringRedisTemplate`（`StringRedisTemplate`） | 类型缺失时注册；键使用 `RedisKeySerializer`，值为字符串 |
| | `zhkjRedis`（`ZhkjRedis`） | 存在 `RedisTemplate` 且 `ZhkjRedis` 缺失时注册 |
| `ZhkjRedisCacheAutoConfiguration` | `redisCacheManager`（方法返回类型 `RedisCacheManager`，实例为 `RedisAutoCacheManager`，`@Primary`） | 始终注册 |
| | `cacheNamespaceResolver`、`scopeCacheEvictor` | 缺失时注册 |
| `RedisPubSubConfiguration` | `rPubSubListenerLazyFilter`、`redisMessageListenerContainer`（缺失时） | — |
| | `topicEventPublisher`（`RPubSubPublisher`）、`topicListenerDetector`（`BeanPostProcessor`） | 始终注册 |
| `DistributedLockAutoConfiguration` | `lockKeyResolver`、`lockExecutor`（`RedisTemplateLockExecutor`）、`distributedLockAspect` | `zhkj.redis.lock.enabled`（缺省视为 `true`） |

`RedisTemplateConfiguration` 标注 `@EnableCaching`；`ZhkjRedisCacheAutoConfiguration` 未标注。

---

## ⚙️ 5. 配置项

### 5.1 框架配置（`zhkj.redis`）

| 键 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `zhkj.redis.prefix` | `zhkj` | `RedisKeySerializer` 的键前缀，规范化为 `zhkj:`；键已包含前缀时不重复添加 |
| `zhkj.redis.lock.enabled` | `true` | 是否装配分布式锁 |

### 5.2 Spring 标准配置

连接参数使用 `spring.data.redis.*`。Spring Cache 参数使用 `spring.cache.*`（`CacheProperties`）:

| 键 | 说明 |
| :--- | :--- |
| `spring.cache.cache-names` | 启动时预创建的缓存名列表（使用默认配置） |
| `spring.cache.redis.time-to-live` | 默认 TTL |
| `spring.cache.redis.key-prefix` | 缓存键前缀（Spring Cache 写入 Redis 时） |
| `spring.cache.redis.use-key-prefix` | 是否使用前缀，默认按 Spring 规则 |
| `spring.cache.redis.cache-null-values` | 是否缓存 `null` |

```yaml
spring:
  data:
    redis:
      host: 127.0.0.1
      port: 6379
      timeout: 3000ms
  cache:
    redis:
      time-to-live: 1h
zhkj:
  redis:
    prefix: zhkj
    lock:
      enabled: true
```

---

## 🕒 6. 缓存与单条 TTL

`@Cacheable` 的缓存名支持 `名称#时长` 语法。时长由 `DurationStyle.detectAndParse` 解析，未带单位的数字按秒处理（如 `30s`、`5m`、`2h`、`7d`）。

```java
@Service
public class ProductService {

    // 使用全局默认 TTL
    @Cacheable(cacheNames = "product", key = "#id")
    public ProductVo getProductById(Long id) {
        return loadFromDatabase(id);
    }

    // 该缓存有效期 5 分钟
    @Cacheable(cacheNames = "product#5m", key = "#id")
    public ProductVo getHotProduct(Long id) {
        return loadFromDatabase(id);
    }
}
```

`cacheNames` 中的 `#` 后缀由 `RedisAutoCacheManager.createRedisCache` 剥离，剩余部分作为缓存名。

---

## 🧰 7. 操作门面 `ZhkjRedis`

`ZhkjRedis` 按数据结构提供方法（以下仅列出分组与代表方法，完整列表见源码）:

| 分组 | 代表方法 |
| :--- | :--- |
| 字符串 | `set(String, Object)`、`setEx(String, Object, Duration)`、`setEx(String, Object, Long seconds)`、`get(String)`、`get(String, Supplier<T>)`、`getAndDel`、`mSet`、`mGet`、`incr`、`incrBy`、`decr`、`decrBy` |
| 键 | `del(...)`、`exists`、`expire(String, long)`、`expire(String, Duration)`、`expireAt`、`ttl`、`pttl`、`persist`、`type`、`keys(pattern)`、`scan(pattern[, count][, consumer])` |
| 哈希 | `hSet`、`hMset`、`hGet`、`hmGet`、`hDel`、`hExists`、`hGetAll`、`hKeys`、`hVals`、`hLen`、`hIncrBy` |
| 列表 | `lPush`、`rPush`、`lPop`、`rPop`、`lRange`、`lLen`、`lTrim`、`lRem` |
| 集合 | `sAdd`、`sMembers`、`sIsMember`、`sRem`、`sInter`、`sUnion`、`sDiff` |
| 有序集合 | `zAdd`、`zRange`、`zRevrange`、`zRangeByScore`、`zRank`、`zScore`、`zRem`、`zCard`、`zCount` |
| 发布 | `publish(String channel, T message, Function<T, byte[]> mapper)` |

`CacheKey` 封装键与过期时间，`set(CacheKey, Object)` 按其 `expire` 设置 TTL。

```java
@Autowired
private ZhkjRedis zhkjRedis;

public void demo() {
    zhkjRedis.setEx("session:token:abc", tokenInfo, 600L);     // 600 秒
    Long counter = zhkjRedis.incrBy("page:view:1001", 1L);
    TokenInfo cached = zhkjRedis.get("session:token:abc");
}
```

---

## 📣 8. 发布订阅

### 8.1 发布

```java
@Autowired
private RPubSubPublisher publisher;

public void notifyRefresh() {
    publisher.publish("CONFIG_REFRESH_CHANNEL", new RefreshCommand("dict"));
}
```

`RedisPubSubPublisher` 通过 `ZhkjRedis.publish` 发送，消息体由容器中注入的 `RedisSerializer<Object>`（默认为 `RedisSerializer.json()`）序列化。

### 8.2 订阅

```java
@Component
public class CacheRefreshListener {

    @RPubSubListener("CONFIG_REFRESH_CHANNEL")
    public void onMessage(RPubSubEvent<RefreshCommand> event) {
        log.info("收到广播 channel={}, msg={}", event.getChannel(), event.getMsg());
    }
}
```

`@RPubSubListener` 规则:

- 标注在方法上，`value` 为频道名；含 `*`、`?` 或成对的 `[...]` 时按模式订阅（`PatternTopic`），否则为精确订阅（`ChannelTopic`）。判定由 `ChannelUtil.isPattern` 实现，其循环 `for (int i = length - 1; i > 0; i--)` 不检查下标 0，因此仅首字符为通配符的频道（如 `*abc`）会被判为精确订阅。
- 方法参数个数为 0 或 1（超过 1 个时启动期抛出 `IllegalArgumentException`）；有参时接收 `RPubSubEvent<Object>`（属性 `pattern`、`channel`、`msg`）。
- 消息体由容器中的 `RedisSerializer` 反序列化。

> **代码现状注意**: 源码核对确认两端频道名不一致。`ZhkjRedis.publish` 通过 `keySerialize(channel)` 发送，频道名经 `RedisKeySerializer` 追加 `zhkj.redis.prefix` 前缀（默认 `zhkj:`）；`RPubSubListenerDetector` 以注解原始 `value` 订阅（`ChannelUtil.getTopic`）。即默认配置下发布至 `zhkj:<channel>`，订阅 `<channel>`（见 `ZhkjRedis.java:1148-1151`、`RPubSubListenerDetector.java:44`）。该结论仅来自源码阅读，运行时行为未验证。

---

## 🔒 9. 分布式锁 `@DistributedLock`

```java
@Service
public class OrderPaymentService {

    @DistributedLock(
        key = "'pay:' + #order.orderNo",
        waitTime = 3,
        leaseTime = 30,
        failureStrategy = LockFailureStrategy.THROW_EXCEPTION,
        message = "订单正在支付处理中，请勿重复操作"
    )
    @Transactional(rollbackFor = Exception.class)
    public PayResult processPayment(OrderDto order) {
        return executePay(order);
    }

    @DistributedLock(key = "'sync:batch:task'", waitTime = 0, failureStrategy = LockFailureStrategy.RETURN_NULL)
    public void runDailySyncTask() {
        doSyncWork();
    }

    @DistributedLock(
        key = "'query:hot:' + #productId",
        failureStrategy = LockFailureStrategy.FALLBACK_METHOD,
        fallbackMethod = "queryProductFallback"
    )
    public ProductVo queryProduct(Long productId) {
        return queryFromDbAndCache(productId);
    }

    public ProductVo queryProductFallback(Long productId) {
        return getDegradedProductCache(productId);
    }
}
```

### 9.1 注解属性

| 属性 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `key` | String | 必填 | 锁标识，支持 SpEL（`#参数名`、`#对象.属性`、字符串拼接；仅当 key 含 `#`、`'` 或 `+` 时求值） |
| `prefix` | String | `""` | 为空时使用 `zhkj:lock:`；非空且不以 `:` 结尾时自动补 `:` |
| `waitTime` | long | `0` | 等待抢锁时长（单位见 `timeUnit`），`0` 表示非阻塞 |
| `leaseTime` | long | `30` | 锁租期，到期自动释放 |
| `timeUnit` | TimeUnit | `SECONDS` | `waitTime` 与 `leaseTime` 的单位 |
| `failureStrategy` | LockFailureStrategy | `THROW_EXCEPTION` | 抢锁失败处理策略 |
| `message` | String | `当前业务正在处理中，请稍后重试` | `THROW_EXCEPTION` 时的异常消息 |
| `fallbackMethod` | String | `""` | `FALLBACK_METHOD` 时调用的同类方法名，参数类型须与原方法一致 |

### 9.2 失败策略（`LockFailureStrategy`）

| 值 | 行为 |
| :--- | :--- |
| `THROW_EXCEPTION` | 抛出 `DistributedLockException`（继承 `BaseFrameworkException`，错误码 `FAILURE`） |
| `RETURN_NULL` | 返回 `null` |
| `FALLBACK_METHOD` | 调用同类的 `fallbackMethod`；方法不存在时抛出 `DistributedLockException` |

### 9.3 执行机制

- 锁存于 Redis，`RedisTemplateLockExecutor` 通过 Lua 脚本（`DistributedLockScripts`）原子加锁与释放。
- 持有者标识为 `JVM 进程 UUID（无横线）+ ":" + 线程 ID`，释放时比对持有者。
- 同一线程重复加锁为可重入（执行器注释描述为 Redis Hash 计数器）。
- 等待模式下以 10–50 毫秒间隔重试，直至超过 `waitTime`。
- 切面 `@Order(Ordered.HIGHEST_PRECEDENCE + 1)`，锁在 `@Transactional` 外层获取，在事务结束后释放。

---

## 🧩 10. 缓存作用域

`CacheScope` 取值: `GLOBAL`、`TENANT`、`DIMENSION`、`DELEGATED`。`DefaultCacheNamespaceResolver.resolvePrefix(缓存名, 作用域)` 生成的前缀（末尾均附加缓存名）为:

| 作用域 | 前缀 |
| :--- | :--- |
| `GLOBAL` | `g:<缓存名>:` |
| `TENANT` | `t:<租户ID>:<缓存名>:` |
| `DIMENSION` | `t:<租户ID>:d:<维度ID>:<缓存名>:`（无维度 ID 时省略 `d:` 段） |
| `DELEGATED` | 一律拒绝解析；当前无可核验的逐请求业务委托授权，不生成跨租户缓存命名空间 |

`ScopeCacheEvictor` 提供 `evictByTenant(tenantId)`、`evictByDimension(tenantId, dimensionId)`、`evictByPattern(pattern)`，均返回删除数量，使用 `StringRedisTemplate` 扫描删除。

`CacheNames` 为常量接口，定义 `SYS_CACHE`（`sys`）、`USER_CACHE`（`user`）、`AUTH_CACHE`（`auth`）、`DATA_SCOPE_CACHE`（`datascope`）。

---

## 🔗 11. 相关文档

- 核心工具与上下文: [core](./core.md)
- Web 层: [starter-web](./starter-web.md)
