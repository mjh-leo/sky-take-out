---
tags:
  - 苍穹外卖
  - Redis
  - 缓存
  - 面试题
---
# Day06 - Redis 入门、店铺状态设置

## 一、Redis 是什么

Redis 是基于内存的 key-value 结构数据库，读写极快，常用作缓存。

**为什么用 Redis？**
- **快**：数据存在内存里，读写性能是数据库的 100 倍以上
- **存热点数据**：查询频繁、修改较少的数据放到 Redis，减少直接查询数据库的次数，减轻 MySQL 压力。
**和MySQL的区别：**

|      | MySQL     | Redis        |
| ---- | --------- | ------------ |
| 存储位置 | 磁盘        | 内存           |
| 速度   | 慢（毫秒级）    | 快（微秒级）       |
| 用途   | 存正式数据、持久化 | 缓存、临时数据、热点数据 |
| 数据结构 | 表（行/列）    | key-value    |

## 二、Redis 五种数据类型

| 类型                | 类比 Java       | 特点                    | 典型场景             |
| ----------------- | ------------- | --------------------- | ---------------- |
| string            | String        | 最简单，key-value         | 存字符串、数字、营业状态、验证码 |
| hash              | HashMap       | key 下面有多个 field-value | 存对象（用户信息、商品信息）   |
| list              | LinkedList    | 有序、可重复                | 消息队列、最新列表        |
| set               | HashSet       | 无序、不重复                | 标签、共同好友、去重       |
| sorted set / zset | TreeSet（不带分数） | 有序、不重复、带分数            | 排行榜、按分数排序        |

**本项目用的：** 店铺营业状态用 string 类型，key 是 `SHOP_STATUS`，value 是 `1`（营业）或 `0`（打烊）。

## 三、店铺营业状态设置

### 1. 需求

商家可以一键设置店铺"营业中"或"打烊"，用户端看到店铺状态。

**为什么用 Redis 不用数据库？**
- 营业状态是高频读取的数据（用户每次打开小程序都要看）
- 不需要持久化存表，重启了也无所谓
- 放 Redis 里读快，不压数据库
### 2. 代码

**Controller：**
```java
@GetMapping("/status/{status}")
public Result setStatus(@PathVariable Integer status) {
    redisTemplate.opsForValue().set("SHOP_STATUS", status);
    return Result.success();
}

@GetMapping("/status")
public Result getStatus() {
    String status = redisTemplate.opsForValue().get("SHOP_STATUS");
    return Result.success(status);
}
```
### 3. 讨论：写 Controller 还是 Service？

视频弹幕又开始了：
- **一派说**：写 Controller 就行，就一个 redis 读写，又不是三层架构，搞 Service 多此一举
- **另一派说**：应该写 Service，为了解耦、规范分层

我选择写在了 ServiceImpl 里，我感觉既然规定了三层，那controller就负责接收请求就行了，写到业务里如果有其他需求也好扩展。

---
## 四、为什么要设置 key 序列化

### 1. 不设置会怎么样？

Spring Boot 默认的 RedisTemplate，**key 用的是 JDK 序列化**。你存了一个 key 叫 `name`，打开 Redis 一看：

```
\xxac\xed\x00\x05t\x00\x04name
```

**全是乱码！** 根本不知道存的是什么，调试的时候想哭。

### 2. 为什么会乱码？

- Redis 存数据本质上是存**字节数组**
- 你传一个 Java 的 String 对象，RedisTemplate 要先把它**序列化成字节**才能存进去
- **默认用 JDK 序列化**：把 Java 对象的类型信息、类名这些全一起序列化了，所以存出来是一串乱码
- 打个比方：就像你寄快递，把整个快递盒、快递单、泡沫都一起塞到快递柜里了，别人打开一看全是包装材料，找不到东西

### 3. 设置之后怎么样？

```java
@Configuration
public class RedisConfiguration {
    @Bean
    public RedisTemplate redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate redisTemplate = new RedisTemplate();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        // 设置 key 的序列化器为 StringRedisSerializer
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        return redisTemplate;
    }
}
```

设置之后，key 就是**纯字符串**，打开 Redis 一看：

```
name
SHOP_STATUS
```

 **思考：** 那 value 要不要设置？

- **value 也要设置**，不然存对象也是乱码
- 一般 value 用 `Jackson2JsonRedisSerializer`，存数字类型就是数字，存对象就是转成json字符串
---
## 五、Java 操作 Redis 五种类型

注入 `RedisTemplate` 就能用：

```java
@Autowired
private RedisTemplate redisTemplate;
```

| 类型 | API | 常用操作 |
|------|-----|---------|
| string | `opsForValue()` | set / get / setIfAbsent（setnx，做分布式锁） |
| hash | `opsForHash()` | put / get / keys / values / delete |
| list | `opsForList()` | leftPush / range / leftPop / size |
| set | `opsForSet()` | add / members / size / union / remove |
| zset | `opsForZSet()` | add / range / incrementScore / remove |

**通用命令：**
```java
redisTemplate.keys("*");       // 查所有 key
redisTemplate.hasKey("name");  // 判断 key 是否存在
redisTemplate.type(key);       // 看是什么类型
redisTemplate.delete("xxx");   // 删除
```

**常用技巧：**
- `set(key, value, 60, TimeUnit.SECONDS)`：设置过期时间
- `setIfAbsent(key, value)`：key 不存在才 set，就是 setnx，分布式锁原理

---
## ⚠️ 避坑 Tips

### 1. Redis 不是万能的
- **Redis 是缓存，不是数据库**。重要数据一定要落库，Redis 挂了/重启了数据会丢（除非开了持久化，但也不是 100% 可靠）
- **不要把所有数据都塞 Redis**。只放热点数据（频繁查、不常变的），冷数据放数据库
- **内存有限**。Redis 占内存很贵，不要存大量没用的数据
### 2. RedisTemplate 的坑
- 默认序列化问题。Spring Boot 默认的 RedisTemplate 使用 JDK 序列化，存入 Redis 的 key 会出现二进制乱码，不方便在 Redis 客户端查看。生产环境一般手动配置 `StringRedisSerializer` 序列化器。
- **StringRedisSerializer 和 RedisTemplate 的区别**： `RedisTemplate` 是操作 Redis 的模板工具；`StringRedisSerializer` 是**序列化器**，作用是把对象转成字符串，用来设置 key/value 的序列化规则。
---
## 🎯 面试题

### Q1：Redis 为什么快？
> **回答框架：**
> 1. **纯内存操作**：数据存在内存里，不用读磁盘，速度快几个数量级
> 2. **单线程模型**：Redis 6.0 之前是单线程，避免了线程切换和锁竞争。每次处理一个命令，没有上下文切换开销
> 3. **I/O 多路复用**：用 epoll 机制，一个线程处理大量连接，不用每个连接开一个线程
> 4. **高效数据结构**：Redis 底层数据结构是精心设计的（跳表、压缩列表、哈希表），操作效率高
> **一句话**：Redis 快的核心是"内存 + 单线程 + I/O 多路复用"。

### Q2：为什么要用 Redis？直接用数据库不行吗？

> **回答框架：**
> 1. **性能问题**：数据库查一次要几毫秒，Redis 查一次只要几微秒。热点数据放 Redis，速度提升 100 倍
> 2. **数据库压力问题**：高并发场景下，所有请求都查数据库，数据库扛不住。加 Redis 做缓存，大部分请求直接从 Redis 拿，不用查数据库
> 3. **功能问题**：Redis 有很多数据库没有的功能，比如排行榜（zset）、分布式锁、过期时间
> 4. **本项目例子**：店铺营业状态，用户每次打开小程序都要看，如果查数据库，一次请求多一次数据库连接。放 Redis 里，读快，还不压数据库

> **一句话**：Redis 是缓存，用来扛高并发、存热点数据，减轻数据库压力。