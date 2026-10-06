---
tags:
  - 苍穹外卖
  - 订单
  - PageHelper
  - 面试题
---
# Day11 - 历史订单模块

## 一、历史订单查询

### 1. 踩坑：一对多 JOIN 分页错乱

**一开始的写法：** orders LEFT JOIN order_detail，一条 SQL 查订单和明细，用 PageHelper 分页。

**出现的问题：**

| 现象 | 原因 |
|------|------|
| 数据库 1 个订单有 5 条菜品，SQL 返回 5 行 | JOIN 后每行是一个订单+一个菜品 |
| records 里出现 5 个重复订单 VO | 5 行被当成 5 条记录 |
| orderDetailList 是 null，前端报 JS 错 | resultType 平铺映射，没法自动嵌套集合 |
| total 是 1，但返回了 5 条 | count 查的是订单数，分页 limit 按行算，对不上 |

### 2. 解决：两次查询，手动组装
```java
// 1. 分页只查订单主表
PageHelper.startPage(page, pageSize);
Page<Orders> pageOrders = orderMapper.orderList(dto);

// 2. 循环每个订单，单独查明细
List<OrderVO> voList = new ArrayList<>();
for (Orders order : pageOrders.getResult()) {
    OrderVO vo = new OrderVO();
    BeanUtils.copyProperties(order, vo);
    List<OrderDetail> details = orderDetailMapper.getByOrderId(order.getId());
    vo.setOrderDetailList(details);
    voList.add(vo);
}

return new PageResult(pageOrders.getTotal(), voList);
```

> **为什么不能 JOIN？** 1 个订单 N 条明细，JOIN 后变 N 行，分页按行算就错了。必须先分页查主表，再补明细。

### 3. PageHelper：为什么要用 Page\<T\> 接收？

| 接收方式 | 能拿到当前页数据 | 能拿到 total |
|---------|----------------|-------------|
| `List<T>` | ✅ | ❌ 拿不到 |
| `Page<T>` | ✅（page.getResult()） | ✅（page.getTotal()） |

**原理：**
- `startPage()` 把分页参数存 ThreadLocal
- MyBatis 拦截器自动执行 `count(*)` 查总数，并给 SQL 加 limit
- 返回结果被包装成 `Page<T>` 对象，里面既有数据列表，又有 total
> **结论：** 后端分页接口要返回 total 给前端，必须用 `Page<T>` 接收。
---
## 二、查询订单详情

比较简单，根据订单 id：
1. 查 orders 表 → 订单基本信息
2. 查 order_detail 表 → 菜品明细列表
3. BeanUtils 拷贝到 OrderVO，把明细 list set 进去返回
---
## 三、取消订单

还是做不了真实微信退款，模拟一下：
1. 更新订单状态为"已取消"
2. 更新支付状态为"退款"

>  **生产环境：** 取消订单要调微信退款接口，等微信退款成功回调后，再更新订单状态。不能直接本地改个状态就算退款了。
---
## 四、再来一单

### 1. 业务逻辑

点订单详情里的"再来一单"，把旧订单里的所有菜品**重新加到当前用户购物车**，用户去购物车结算。
### 2. stream().map() 转换对象
```java
// 把 List<OrderDetail> 转成 List<ShoppingCart>
List<ShoppingCart> cartList = orderDetailList.stream().map(detail -> {
    ShoppingCart cart = new ShoppingCart();
    BeanUtils.copyProperties(detail, cart, "id");  // 不拷 id，新记录自增
    cart.setUserId(userId);
    cart.setCreateTime(LocalDateTime.now());
    return cart;
}).collect(Collectors.toList());

shoppingCartMapper.insertBatch(cartList);
```
**stream().map() 是干什么的？**
- 遍历集合里每个元素
- 对每个元素做转换（OrderDetail → ShoppingCart）
- 收集成新的 List

相当于简化版 for 循环：
```java
List<ShoppingCart> list = new ArrayList<>();
for (OrderDetail detail : orderDetailList) {
    ShoppingCart cart = new ShoppingCart();
    // ... 转换
    list.add(cart);
}
```
### 3. @Param 注解的疑问

```java
void insertBatch(List<ShoppingCart> shoppingCartList);
```

- 不加 @Param 也没报错？MyBatis 默认参数名可能能识别
- **保险做法**：加 `@Param("shoppingCartList")`，XML 里就用 `collection="shoppingCartList"`
- 如果是 List 或数组，不加 @Param 时默认叫 `list` 或 `array`

---
## 五、int vs Integer 的坑

测试时发现订单详情打包费显示 0，查了半天：

- 前端默认打包费是 2，下单页面有值
- 但支付完查看订单详情，打包费显示 0
- 数据库里 `pack_amount` 字段是 `int DEFAULT NULL`
- Orders 实体类里用了 `int`（基本类型）
- 把 `int` 改成 `Integer` 后，打包费正确显示 2

**对比：**

| | int（基本类型） | Integer（包装类） |
|---|---|---|
| 默认值 | 0 | null |
| 能不能存 null | 不能，null 被强制转成 0 | 能，null 就是 null |
| 语义 | 0 既表示"传了0"也表示"没传" | null 明确表示"没有值" |
| 对应数据库 | `int DEFAULT NULL` 容易出问题 | 能正确映射 NULL |

> **教训：** 实体类里对应数据库可能为 NULL 的字段，**一律用包装类型 Integer/Long/Double，不要用基本类型 int/long/double**。基本类型有默认值 0，会把"没有值"和"值就是 0"混淆，排查问题时特别坑。

---
## ⚠️ 避坑 Tips

### 1. 一对多分页查询的坑
- **主表和子表 JOIN 后分页会错乱**。1 个订单有 5 条菜品明细，JOIN 后返回 5 行，PageHelper 按行数分页就错了
- **正确做法**：先分页查主表（orders），再循环查每个订单的明细（order_detail），手动组装
- **这其实就是 N+1 查询**。1 次查订单 + N 次查明细，但订单分页一般一页就几条，N 很小，可以接受
### 2. PageHelper 的坑
- **要用 Page\<T\> 接收才能拿到 total**。用 List\<T\> 接收只能拿到当前页数据，拿不到总条数
- **startPage() 只对下一条 SQL 生效**，中间插别的查询就失效
- **Page 对象既是 List 又是分页信息**，自动转成 Page 就能拿 total
### 3. int vs Integer 的坑
- **数据库字段允许 NULL，Java 实体类必须用包装类型**。int 会把 null 变成 0，丢失信息
- **MyBatis 参数日志要看仔细**。日志显示 `1(Integer)` 说明是包装类型
- **改完代码一定要重启**。IDEA 有时用旧的 class 文件，改了不生效
---
## 🎯 面试高频题

### Q1：int 和 Integer 有什么区别？实体类为什么用包装类型？

> **回答框架：**
> 1. **int 是基本类型**，默认值 0，不能存 null
> 2. **Integer 是包装类**，默认值 null，可以表示"没有值"
> 3. **为什么实体类用包装类**：
>    - 数据库字段允许 NULL，基本类型 int 会把 null 映射成 0
>    - 分不清"没传值"和"值就是 0"，导致业务数据不对
>    - 比如订单打包费，前端传了 2，结果详情页显示 0，改成 Integer 后才正确
> 4. **自动装箱/拆箱**：int 和 Integer 可以自动转换，但 NPE 风险要注意

### Q2：PageHelper 的原理和注意事项？

> **回答框架：**
> 1. **原理**：startPage() 把分页参数存 ThreadLocal，MyBatis 拦截器拦截下一条 SQL，自动加 limit 和 count
> 2. **注意事项**：
>    - 只对下一条 SQL 生效
>    - 要用 Page\<T\> 接收才能拿 total
>    - 一对多 JOIN 查询会分页错乱，要手动组装
> 3. **本项目踩的坑**：历史订单 JOIN 订单明细，分页错乱（1个订单有多行），改成两次查询解决、


第十一天，有无敲这部分的！咋恁难啊！苍穹外卖day11，今天写了用户端历史订单模块。全部都要自己分析业务，确实要炸了，分析的老是不全，写得方法也不一样，一下子写不那么完美。还因为一个类型问题改了半天bug，我甚至都以为是数据库还是连接池出问题了。。

