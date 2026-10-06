---
tags:
  - 苍穹外卖
  - 套餐模块
  - 面试题
---
# Day05 - 套餐模块（独立完成 ）

## 一、这次是怎么写的？

视频没有写这一部分，我就自己写了这个模块（和菜品模块差不多，业务有一点点不一样吧）：
1. 先看接口文档，搞清楚套餐模块有哪些接口
2. 自己从前端发请求顺着捋下去，分析业务逻辑
3. AI辅助验证，自己写 Service 和 Mapper
4. 边写边调，写完才发现漏了一些业务逻辑

**相当于独立完成了整个套餐模块**，遇到几个问题：
- 写完才发现"套餐起售需要判断套餐内的菜品是不是起售状态"这个业务规则
- 新增套餐需要菜品列表，但菜品模块里没写 `/dish/list` 接口，又回头补了
- N+1 查询问题
## 二、业务逻辑的完整性

### 1. 套餐起售时的菜品状态校验

**问题：** 写完起售接口才发现，MessageConstant 里有个 `SETMEAL_ENABLE_FAILED`，意思是"套餐内包含未起售菜品，无法起售"。

**业务规则：** 套餐起售的前提是——套餐里包含的**所有菜品都必须是起售状态**。只要有一个菜品是停售的，整个套餐就不能起售。

**正确逻辑：**
```java
@Override
public void startOrStop(Long id, Integer status) {
    // 只有启售的时候才校验，停售不用校验
    if (StatusConstant.ENABLE.equals(status)) {
        // 1. 查套餐下所有菜品的 dishId
        List<Long> dishIds = setmealDishMapper.getDishBySetmealId(id)
                .stream().map(SetmealDish::getDishId).collect(Collectors.toList());

        // 2. 批量查这些菜品的状态
        List<Dish> dishList = dishMapper.getByIds(dishIds);

        // 3. 只要有一个菜品停售，套餐就不能起售
        for (Dish dish : dishList) {
            if (StatusConstant.DISABLE.equals(dish.getStatus())) {
                throw new SetmealEnableFailedException(MessageConstant.SETMEAL_ENABLE_FAILED);
            }
        }
    }
    // 校验通过，正常更新套餐状态
}
```
> **思考：** 我感觉看一看这些异常类、常量类啥的也是个方法，里面反映着业务规则。

### 2. 跨模块接口依赖

**问题：** 新增套餐页面需要选菜品，但菜品模块里只有分页查询接口，没有"查询菜品列表"的接口。前端调不到，我又回去在菜品模块补了一个 `/dish/list` 接口。
## 三、N+1 查询问题

### 1. 什么是 N+1 查询？

**就用套餐起售校验这段代码作为例子吧。** （实际来说好像套餐里也没有多少菜品）

业务：套餐起售前，要检查套餐内所有菜品是不是都起售了。

**错误写法（N+1）：**
```java
// 1. 查套餐下所有菜品的 dishId（1 次查询）
List<Long> dishIds = setmealDishMapper.getDishBySetmealId(id)
        .stream().map(SetmealDish::getDishId).collect(Collectors.toList());

// 2. 循环里每个菜品单独查一次状态（N 次查询）
for (Long dishId : dishIds) {
    Dish dish = dishMapper.getById(dishId);  // 每个菜品查一次！
    if (StatusConstant.DISABLE.equals(dish.getStatus())) {
        throw new SetmealEnableFailedException(...);
    }
}
```

**执行了 1 + N 次 SQL**：套餐里有 10 个菜品，就是 11 次查询；有 50 个菜品，就是 51 次。这就是 **N+1 问题**——1 次主查询 + N 次单条查询。
### 2. 正确写法（批量查询）

你写的就是正确做法：**先把所有 id 收集起来，一条 SQL 查完**。

```java
// 1. 查套餐下所有菜品的 dishId（1 次）
List<Long> dishIds = setmealDishMapper.getDishBySetmealId(id)
        .stream().map(SetmealDish::getDishId).collect(Collectors.toList());

// 2. 批量查询所有菜品的状态（1 次！不是 N 次）
List<Dish> dishList = dishMapper.getByIds(dishIds);
// 对应 SQL: SELECT * FROM dish WHERE id IN (1, 2, 3, 4...)

// 3. 内存里遍历判断（不查数据库了）
for (Dish dish : dishList) {
    if (StatusConstant.DISABLE.equals(dish.getStatus())) {
        throw new SetmealEnableFailedException(MessageConstant.SETMEAL_ENABLE_FAILED);
    }
}
```

**总共 2 次 SQL**
### 3. 为什么 N+1 是个问题？

- **数据库连接数有限**：查 50 个菜品就要 50 次查询，连接池很快被占满
- **网络开销**：每次 SQL 都有网络往返，50 次查询 = 50 次网络延迟
- **数据库压力大**：50 次简单查询 vs 1 次 IN 查询，数据库负载差几十倍
---
## ⚠️ 避坑 Tips

### 1. N+1 查询的坑
- **循环里查数据库是第一大性能杀手**。看到 for 循环里调 Mapper，就要警惕是不是 N+1。
- **MyBatis 的嵌套查询也会导致 N+1**。`<collection>` 标签用嵌套查询而不是联表查询，同样会 N+1。
- **分页大列表更要注意**。一页 10 条还好，一页 100 条就是 101 次查询。
### 2. 业务完整性的坑
- **状态变更前一定要校验关联数据**。比如套餐起售前，要检查套餐里的所有菜品是不是都起售了；菜品停售前，要检查有没有套餐在用这个菜品。这些都是业务规则，不是技术问题。
### 3. 跨模块依赖的坑
- **一个模块用到另一个模块的数据，先确认接口有没有**。不要写完才发现依赖的接口没写。
- **接口设计要考虑通用性**。菜品列表接口不只是套餐用，以后可能别的地方也用，设计成通用的。
---
## 🎯 面试题

### Q1：什么是 N+1 查询问题？怎么解决？

> **回答框架：**
> 1. **是什么**：主查询查 N 条记录，然后 for 循环里每条记录再查一次关联数据，总共 1 + N 次 SQL
> 2. **为什么是问题**：数据库连接有限、网络开销大、数据库压力大
> 3. **解决方案**：
>    - **多表联查**：一条 SQL 把关联数据查出来（推荐）
>    - **批量查询**：先查主表，收集 id，一条 SQL 查出所有关联数据，内存里组装
> 4. **项目里的例子**：套餐起售时要校验套餐内所有菜品是不是都起售了。一开始如果循环里每个菜品单独查一次状态，就是 N+1。后来改成把所有 dishId 收集起来，用 `getByIds()` 一条 IN 查询批量查，2 次 SQL 搞定。

> **一句话**：N+1 就是循环里查数据库，解决思路是"一次性批量查出来，别循环查"。
---
## 💡 今日insight

今天最大的收获不是写了多少代码，而是理解了"独立做"和"跟着教程敲"的区别。
跟着教程敲，代码是对的，因为视频已经把坑都踩过了。但自己做，才发现：
- 有的时候脑子突然短路，不知道下一步是什么
- 业务规则写之前就要想清楚，而不是写完代码才想起来
- 跨模块依赖不是写完才补的，是设计阶段就要考虑

