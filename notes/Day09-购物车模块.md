---
tags:
  - 苍穹外卖
  - 数据库设计
  - 面试题
---
# Day09 - 购物车模块

## 一、数据库设计

###  shopping_cart 表为什么要冗余 name、image、amount？

购物车里存了三个冗余字段：商品名称、图片、单价。

| 原因       | 解释                                                                            |
| -------- | ----------------------------------------------------------------------------- |
| **性能**   | 查购物车列表时，不用 JOIN dish 表和 setmeal 表，一条 SQL 就够了。不然每条购物车记录都要关联菜品表查名称和图片，多表 JOIN 慢 |
| **历史快照** | 加购物车时的价格要固定下来。之后菜品涨价了，购物车里还是当时加的价格，不能跟着变。amount 是加购物车那一刻的单价快照                 |
| **解耦**   | 菜品改名了、图片换了，购物车里显示的还是加购时的样子。这符合用户预期——我加购的时候是什么样，结算时就应该是什么样                     |

**什么时候可以冗余？**
- 数据不常变（菜品名称、价格相对稳定）
- 查询频繁（购物车每次打开都要看）
- 冗余带来的性能收益 > 数据一致性维护成本

>  **反例：** 菜品的 `category_id` 就不冗余在购物车里——因为几乎不会按分类查购物车，冗余了也没用，还多一个字段要维护。

---

## 二、添加购物车

### 1. BaseContext 为什么取到的是 userId？

**思考：** BaseContext.getCurrentId() 取到的是 userId，为什么不是 empId？

注册了两个拦截器，拦截不同的路径：

| 拦截器 | 拦截路径 | 存的 id |
|--------|---------|---------|
| JwtTokenAdminInterceptor | `/admin/**` | 员工 id |
| JwtTokenUserInterceptor | `/user/**` | 用户 id |

购物车接口在 `/user/shoppingCart` 下，走的是 UserInterceptor，解析 token 后存的是 userId。BaseContext 本身不区分员工还是用户，它只是个 ThreadLocal 容器，**谁往里面存的，就是谁**。

### 2. 更新数量时为什么能拿到 id？

**理清三个场景：**

| 场景 | 怎么拿 id |
|------|----------|
| **insert 后要拿自增 id** | 需要 `useGeneratedKeys="true" keyProperty="id"`，MyBatis 把自增 id 回填到对象里 |
| **select 查询** | 查出来的结果里本来就有 id，自动封装到对象属性上 |
| **update/delete 按 id 改** | 先 select 查出来（带 id），再 update |

添加购物车的逻辑是：
1. 先查购物车里有没有这个商品（同一个用户 + 同一个 dish_id/setmeal_id + 同一个口味）
2. 有 → 数量 +1（用查出来的 id 做 update）
3. 没有 → 新增一条（insert 后回填 id）

> **我混淆的点：** 脑子迷糊了把 select 查询自动封装 id 和 insert 后回填 id 搞混了。select 本来就有 id，不用额外配置；insert 才需要 useGeneratedKeys。

---

## 三、查看购物车

**我犯的错：** 重复写了一个只传 userId 的查询 SQL（已经写过一个查询的动态sql）：
```java
@Select("select * from shopping_cart where user_id = #{userId}")
List<ShoppingCart> showShoppingCart(Long userId);
```
**其实不用。** 可以构造一个 ShoppingCart 对象作为查询条件：
```java
public List<ShoppingCart> showShoppingCart() {
    Long userId = BaseContext.getCurrentId();
    ShoppingCart shoppingCart = ShoppingCart.builder()
            .userId(userId)
            .build();
    return shoppingCartMapper.list(shoppingCart);
}
```
---
## 四、删除购物车

### 1. 清空购物车
简单，按 userId 删全部：
```sql
delete from shopping_cart where user_id = #{userId}
```
### 2. 删除一个商品（数量 -1）

**业务逻辑：**
1. 前端传 dishId 或 setmealId
2. 先查购物车里这条记录的数量
3. 如果数量 == 1 → 直接删除这条记录
4. 如果数量 > 1 → 数量 -1，更新 number
**踩坑：** 写完接口没通，一看请求方法写错了——前端发的是 POST，我写成了 `@DeleteMapping`。
**教训：** 接口联调时先看请求方法对不对（GET/POST/PUT/DELETE），再看参数对不对。

---
## ⚠️ 避坑 Tips

### 1. 冗余字段的坑
- **冗余不是越多越好**。每加一个冗余字段，数据变更时就要多维护一个地方。只冗余"查询频繁、变更不频繁"的字段
- **冗余字段要明确语义**。amount 是"加购物车时的单价快照"，不是菜品当前单价。改菜品价格不影响购物车
- **一致性兜底**：菜品改了名称/图片，购物车不更新是对的——这是用户加购时的快照
### 2. MyBatis 的坑
- **insert 要拿自增 id**：必须加 `useGeneratedKeys="true" keyProperty="id"`，不然对象里 id 是 null
- **select 不用额外配置**：查出来什么字段就封装什么字段
---
## 🎯 面试高频题

### Q1：数据库为什么要冗余字段？有什么优缺点？

> **回答框架：**
> 1. **是什么**：在一张表里存其他表的字段，比如购物车存菜品名称、图片、价格
> 2. **优点**：
>    - 减少 JOIN，查询快（购物车列表不用关联菜品表）
>    - 历史快照（加购时的价格固定，菜品涨价不影响购物车）
> 3. **缺点**：
>    - 数据冗余，占存储空间
>    - 数据变更时要同步更新冗余字段，维护成本高
> 4. **什么时候冗余**：查询频繁、数据相对稳定、对性能有要求的场景
> 5. **项目例子**：购物车冗余 name、image、amount，查购物车一条 SQL 搞定，价格是加购时的快照
### Q2：购物车功能怎么设计的？

> **回答框架：**
> 1. **一张 shopping_cart 表**：user_id + dish_id/setmeal_id + dish_flavor + number + 冗余字段
> 2. **添加购物车**：先查有没有同商品（同用户+同菜品+同口味），有就数量+1，没有就新增
> 3. **查看购物车**：按 userId 查全部
> 4. **删除一个商品**：数量-1，到0就删除记录
> 5. **清空购物车**：按 userId 删全部
> 6. **关键点**：amount 是加购时的价格快照；dish_id 和 setmeal_id 二选一（菜品或套餐）

第九天，购物车模块走起。苍穹外卖day09，今天写了购物车模块，设计数据库冗余字段确实对后面轻松太多，更新数量时脑子短路了忘了 select 本来就有 id，不用额外配置。然后自己写了删除单个商品，减和加差不多吧反过来，最后测试我看没减掉一看接口没通然后就看到是post请求。