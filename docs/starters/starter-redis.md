# 分布式缓存与分布式锁：zhkj-framework-starter-redis

> **模块定位**: 高性能分布式缓存 Starter，基于 Spring Data Redis 深度定制，提供现代 `RedisSerializer.json()` 序列化、多级动态过期缓存管理器 `RedisAutoCacheManager`、生产级声明式分布式锁 `@DistributedLock` 与 Lua 持有者（Owner）校验防误删安全体系。  
> **适用基准**: OpenJDK 21 (LTS), Spring Boot 4.1.1, Redis 7.x  
> **模块坐标**: `io.github.ybb88:zhkj-framework-starter-redis`

---

## 📋 1. 架构定位与核心设计

分布式高并发应用中，缓存与分布式锁是解决数据库压力与并发防重提单的关键：

1. **Jackson 3 现代序列化工厂**：
   - 全面采用 Spring Data Redis 官方推荐的 `RedisSerializer.json()`，彻底淘汰存在类全限定名安全隐患与反序列化漏洞的旧式序列化器；
2. **多级动态过期缓存管理器 (`RedisAutoCacheManager`)**：
   - 支持在 `@Cacheable` 缓存名中使用 `#` 语法动态指定当前 Key 的 TTL 过期时间（如 `@Cacheable(cacheNames = "order#300")` 表示该 Key 缓存 300 秒），防止大批 Key 同时失效引发的缓存雪崩；
3. **生产级声明式分布式锁 (`@DistributedLock`)**：
   - 支持强大的 SpEL 动态参数解析构建锁 Key（如 `@DistributedLock(key = "'order:' + #dto.orderNo")`）；
   - 支持等待获取锁时间 `waitTime` 与租期 `leaseTime`；
4. **Lua 脚本持有者（Owner）校验防误删**：
   - 释放锁时通过原子 Lua 脚本严格比对当前持有者的唯一线程标识（UUID + ThreadId），只有持有者与锁中存储的值完全一致才执行删除，彻底杜绝因业务超时导致的“误删他人锁”灾难；
5. **强类型异常受控映射**：
   - 抢锁超时抛出的 `DistributedLockException` 严格继承自 `BaseFrameworkException`，由全局异常处理器统一拦截转为友好的 `400 Bad Request` 响应，绝不向前端暴露 500 未知系统崩溃。

---

## 📦 2. 依赖坐标

```xml
<dependency>
    <groupId>io.github.ybb88</groupId>
    <artifactId>zhkj-framework-starter-redis</artifactId>
</dependency>
```

---

## 🧱 3. 核心包结构与组件

```
io.github.ybb88.framework.redis
├── cache/
│   ├── RedisAutoCacheManager.java     # 支持动态 TTL 过期的缓存管理器
│   └── CacheProperties.java           # 缓存全局属性配置
├── lock/
│   ├── annotation/
│   │   └── DistributedLock.java       # 声明式分布式锁注解
│   ├── aspect/
│   │   └── DistributedLockAspect.java # 分布式锁 AOP 切面 (SpEL 解析、重试、释放)
│   ├── exception/
│   │   └── DistributedLockException.java # 强类型抢锁失败异常
│   └── script/
│       └── RedisLockScripts.java      # Lua 原子加锁与持有者防误删释放脚本
└── config/
    └── RedisAutoConfiguration.java    # RedisTemplate 与 RedisSerializer 自动装配
```

---

## ⚙️ 4. 核心配置清单 (`application.yml`)

```yaml
spring:
  data:
    redis:
      host: 127.0.0.1
      port: 6379
      password: "ENC(encrypted_redis_pwd)"
      database: 0
      timeout: 5000ms
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 2

zhkj:
  cache:
    enabled: true                      # 是否开启 Redis 增强缓存 (默认: true)
    default-expire: 86400              # 默认缓存过期时间 (秒)
```

---

## 💡 5. 典型使用范式

### 5.1 声明式防重提单分布式锁 (`@DistributedLock`)
```java
package io.github.ybb88.demo.controller;

import io.github.ybb88.framework.api.R;
import io.github.ybb88.framework.redis.lock.annotation.DistributedLock;
import io.github.ybb88.demo.dto.OrderCreateDto;
import io.github.ybb88.demo.vo.OrderVo;
import io.github.ybb88.demo.service.OrderService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    /**
     * 创建订单防重控制：
     * 1. 锁 Key 基于 SpEL 动态生成: order:create:ORD20261017001
     * 2. waitTime = 3000ms: 最多等待 3 秒抢锁，超时直接抛出 DistributedLockException
     * 3. leaseTime = 10000ms: 锁最大持有租期 10 秒，防止死锁
     * 4. message: 抢锁失败时返回给前端的友好提示
     */
    @PostMapping
    @DistributedLock(
        key = "'order:create:' + #dto.orderNo",
        waitTime = 3000,
        leaseTime = 10000,
        message = "订单正在处理中，请勿重复提交"
    )
    public R<OrderVo> createOrder(@RequestBody OrderCreateDto dto) {
        OrderVo vo = orderService.createOrder(dto);
        return R.data(vo);
    }
}
```

### 5.2 动态过期注解缓存 (`@Cacheable`)
```java
package io.github.ybb88.demo.service;

import io.github.ybb88.demo.vo.ProductVo;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class ProductService {

    /**
     * 使用 #300 语法声明该缓存的 TTL 为 300 秒 (5分钟)
     */
    @Cacheable(cacheNames = "product#300", key = "#id")
    public ProductVo getProductById(Long id) {
        // 若缓存未命中则查询数据库
        return loadFromDatabase(id);
    }
}
```

### 5.3 注入 `RedisTemplate` 原生操作
```java
@Autowired
private RedisTemplate<String, Object> redisTemplate;

public void saveSession(String key, Object data) {
    // 现代 API 推荐传入 java.time.Duration
    redisTemplate.opsForValue().set(key, data, Duration.ofMinutes(30));
}
```
