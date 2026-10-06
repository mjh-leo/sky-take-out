---
tags:
  - 苍穹外卖
  - 缓存
  - 面试题
---
# Day08-缓存菜品、缓存套餐

## 一、缓存菜品

### 1. 问题是什么？

用户端打开小程序，要按分类查菜品列表。如果每次都查 数据库，并发一大就卡，用户点个商品等好几秒，体验很差。

**解决思路：** 把热点数据放到 Redis 里，查的时候先查 Redis，Redis 有就直接返回，没有再查数据库，然后把结果塞进 Redis。

### 2. 手动实现缓存

```java
// 构建redis key  
String key = "dish_" + categoryId;  
// 查询redis中是否存在菜品数据  
List<DishVO> list = (List<DishVO>) redisTemplate.opsForValue().get(key);  
// 如果存在，直接返回  
if (list != null && !list.isEmpty()) {  
    return Result.success(list);  
}  
// 如果不存在，从数据库中查询，并缓存到redis中  
Dish dish = new Dish();  
dish.setCategoryId(categoryId);  
dish.setStatus(StatusConstant.ENABLE);//查询起售中的菜品  
  
list = dishService.listWithFlavor(dish);  
// 缓存到redis中  
redisTemplate.opsForValue().set(key, list);  
return Result.success(list);
```

### 3.手动清理缓存
```java
Set keys = redisTemplate.keys(pattern);  
redisTemplate.delete(keys);
```
---
## 二、缓存套餐 - SpringCache

### 1. SpringCache 是什么？

SpringCache 是 Spring 提供的缓存抽象框架，**用注解就能加缓存，不用自己写 Redis 代码**。

底层可以接不同的缓存实现：Redis、Caffeine、EhCache。换缓存实现不用改业务代码。

### 2. 常用注解

| 注解 | 说明 | 什么时候用 |
|------|------|-----------|
| `@EnableCaching` | 开启缓存注解功能，加在启动类上 | 项目启动时必须加，不然注解不生效 |
| `@Cacheable` | 方法执行前先查缓存，有就直接返回；没有就执行方法并把返回值存缓存 | 查询方法 |
| `@CachePut` | 不管缓存有没有，都执行方法，然后把返回值更新到缓存 | 更新方法 |
| `@CacheEvict` | 方法执行后删除缓存 | 删除/修改方法 |

### 3. @Cacheable 原理：AOP + 反射

完整流程：

```
调用 controller.list()
       ↓
实际调用的是 Spring 生成的代理对象（AOP）
       ↓
代理对象内部：
  1. 反射拿到 list() 方法上的 @Cacheable 注解
     → 读出 cacheNames = "setmealCache"，key = "#categoryId"
  2. 反射解析 SpEL 表达式，算出真实的 key（比如 "setmealCache::100"）
  3. 拿这个 key 去 Redis 查
  4. 查到了 → 直接返回，原方法不执行
  5. 没查到 → 反射调用原方法 list()，查数据库
  6. 把方法返回值序列化后存进 Redis
  7. 返回结果
```

**所以：**
- **核心机制是 AOP 动态代理**——Spring 给你生成了一个代理对象，拦截方法调用
- **反射是实现手段**——代理内部用反射读注解、解析 SpEL key、反射调用原方法
- 不是"基于反射实现缓存"，是"AOP 拦截 + 反射读注解和参数"

**代码示例：**
```java
@GetMapping("/list")  
@Cacheable(cacheNames = "setmealCache", key = "#categoryId") // key: setmealCache::100  
public Result<List<Setmeal>> list(Long categoryId) {  
    Setmeal setmeal = new Setmeal();  
    setmeal.setCategoryId(categoryId);  
    setmeal.setStatus(StatusConstant.ENABLE);  
  
    List<Setmeal> list = setmealService.list(setmeal);  
    return Result.success(list);  
}
```

**修改/删除时清缓存：**
```java
@DeleteMapping()  
@CacheEvict(cacheNames = "setmealCache",allEntries = true)  
public Result delete(@RequestParam List<Long> ids) {  
    log.info("删除套餐:{}", ids);  
    setmealService.delete(ids);  
    return Result.success();  
}
```

### 3. 我踩的坑：导错包了

```java
// ❌ 错的（Swagger 的注解，完全没用）
import springfox.documentation.annotations.Cacheable;

// ✅ 对的（Spring Cache 的注解）
import org.springframework.cache.annotation.Cacheable;
```

注解加了不生效，排查半天才发现是导错包了。

> **教训：**  IDE 自动导包的时候要看一眼，别光看类名一样就回车。
---
## 三、缓存更新策略

### 1. 什么时候更新缓存？

| 操作    | 缓存怎么做         |
| ----- | ------------- |
| 新增菜品  | 不用管（下次查询自动加载） |
| 修改菜品  | 删对应分类的缓存      |
| 删除菜品  | 删对应分类的缓存      |
| 起售/停售 | 删对应分类的缓存      |

**原则：** 只要数据变了，就把缓存删掉。下次查询时自动从数据库加载最新数据。
### 2. 缓存的 key 怎么设计？

- 菜品缓存：`dish_{categoryId}`，按分类隔离
- 套餐缓存：`setmeal_{categoryId}`
- 不要所有菜品用一个 key，不然改一个菜品要清全部缓存，缓存命中率太低
### 3. 过期时间

- 热点数据设过期时间（比如 30 分钟）
- 防止数据库改了但缓存没删，导致一直是旧数据
- 即使删缓存的逻辑漏了，过期时间到了也会自动更新
---
## ⚠️ 避坑 Tips

### 1. 缓存和数据库一致性
- **先更数据库，再删缓存**。顺序别反了——先删缓存再更数据库，并发下可能有别的请求把旧数据又塞回缓存
- **删缓存比更新缓存安全**。不用算新缓存值，下次查询自动加载
- **设过期时间兜底**。即使删缓存逻辑漏了，过期时间到了自动更新
### 2. SpringCache 的坑
- **导错包**：两个 Cacheable，一定要导 `org.springframework.cache.annotation.Cacheable`
- **key 的写法**：`key = "#categoryId"` 用 SpEL 表达式，别写错变量名
- **@Cacheable 只对 public 方法生效**，同类内自调用不生效（和事务一样的代理问题）
### 3. 缓存设计的坑
- **不要什么都缓存**。频繁改的数据、查询量小的数据，别加缓存，反而增加复杂度
- **key 要有规律**，方便批量删除。比如都用 `dish_` 前缀
- **缓存不是越多越好**。缓存占内存，命中率低的缓存是浪费
---
## 🎯 面试高频题

### Q1：缓存穿透、击穿、雪崩是什么？怎么解决？

> **背下来：**
> - **穿透**：查不存在的数据，每次都穿到数据库。解决：缓存空值、布隆过滤器
> - **击穿**：一个热点 key 过期，瞬间大量请求打到数据库。解决：热点数据永不过期、加互斥锁
> - **雪崩**：大量 key 同时过期，数据库崩了。解决：过期时间加随机值、多级缓存

### Q2：缓存和数据库怎么保证一致性？

> **回答框架：**
> 1. **先更新数据库，再删除缓存**
> 2. **为什么是删缓存而不是更新缓存**：并发下更新缓存容易不一致，删缓存最简单，下次查询自动加载
> 3. **为什么先更数据库再删缓存**：反过来的话，删完缓存还没更数据库，别的请求又把旧数据读出来塞回缓存了
> 4. **兜底**：设过期时间，即使删缓存漏了，到期也会更新
> 5. **强一致场景怎么办**： Redis 事务、读写加锁，或者直接查数据库

### Q3：@Cacheable 原理是什么？

> **回答框架：**
>@Cacheable 底层基于 AOP 动态代理实现，Spring 会为添加该注解的类生成代理对象。 代理内部大量用到反射：容器启动阶段通过反射读取方法上的`@Cacheable`注解，获取`cacheNames`、key 等信息，并解析 SpEL 表达式；方法执行时，代理先拦截请求，使用解析好的 key 查询 Redis 缓存。
- 如果缓存命中，直接返回缓存数据，不执行目标方法；
- 如果缓存未命中，通过反射调用原方法查询数据库，拿到返回结果存入 Redis 后返回。

第八天，今天缓一缓。苍穹外卖day08，今天写了缓存菜品和套餐，先是手动的加到缓存、删除缓存，然后通过加注解进行缓存。@Cacheable 底层基于 AOP 动态代理实现，容器启动阶段，通过反射读取注解信息并解析 SpEL 表达式；方法执行时被代理拦截，先查 Redis。缓存命中直接返回，不执行目标方法；缓存未命中，反射调用目标方法查询数据库，将结果存入 Redis 后返回。