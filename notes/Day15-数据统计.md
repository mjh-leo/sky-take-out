---
tags:
  - 苍穹外卖
  - 数据统计
  - ApacheEcharts
  - 面试题
---
# Day15 - 数据统计

## 一、Apache ECharts

**ECharts 是基于 JavaScript 的数据可视化图表库**，前端用它画图表（折线图、柱状图、饼图）。
**和后端的关系：** 后端只负责把统计数据查出来、按前端规定的格式返回；**画图是前端的事**（ECharts 拿到数据画成折线图/柱状图）。
**分工：**
```
后端：查订单表/用户表 → 统计 → 返回 {日期集合, 数值集合}
前端：拿到数据 → ECharts 画折线图/柱状图 → 展示
```
## 二、营业额统计

### 业务规则

- 营业额 = **订单状态为已完成**的订单金额合计
- 折线图展示，X 轴日期，Y 轴营业额
- 按时间区间，展示**每天**的营业额

### 实现思路

```
前端传 ?begin=开始日期&end=结束日期
    ↓
后端：算出 begin~end 的每一天（日期集合）
    ↓
遍历每一天，查当天营业额（状态=已完成 的 sum(amount)）
    ↓
两个集合：日期集合 + 营业额集合，返回给前端
```
> **思考：** 为什么不直接 `GROUP BY` 一把查出来？因为数据库里没有订单的日期可能没数据，图表上会缺一天。简单的做法是**先造出完整日期集合**，每天单独查，没有营业额就补 0，保证折线图连续，但这样对数据库查询次数大。可以用 `GROUP BY` 一把查出来，然后手动写补0代码。

## 三、用户统计

### 业务规则

- 折线图，X 轴日期，Y 轴用户数
- 展示每天的**用户总量**和**新增用户量**（两条线）
### 实现思路

三个集合：`dateList`（日期）、`totalUserList`（总用户数）、`newUserList`（新增用户数）。
先 put end，再 put begin，先查总用户数，再查新增用户数，这样就不用写两个map了。
## 四、订单统计

### 业务规则

- **有效订单 = 状态为已完成**的订单
- 折线图，展示每天的**订单总数**和**有效订单数**（两条线）
### 实现思路
和用户数量统计差不多。
### stream 求和：
```java
Integer totalOrderCount = totalOrderList.stream().reduce(Integer::sum).get();
```
**reduce(Integer::sum) 是什么？** 把集合里的数字逐个累加。
## 五、销量排名统计

### 业务规则

- 展示销量**前10**的商品（菜品 + 套餐）
- 柱状图**降序**展示
- 销量 = **销售份数**（不是金额）

### SQL 实现

```sql
SELECT od.name, sum(od.number) number
FROM order_detail od
INNER JOIN orders o 
ON od.order_id = o.id
WHERE o.status = 5
AND o.order_time > '2026-09-20' AND o.order_time < '2026-10-03'
GROUP BY od.name
ORDER BY number DESC
LIMIT 0,10
```

**逐步拆解：**

| 部分                        | 作用         |
| ------------------------- | ---------- |
| `od.name, sum(od.number)` | 查菜名 + 累加份数 |
| `INNER JOIN orders o`     | 订单明细关联订单表  |
| `o.status = 5`            | 只要已完成订单的明细 |
| `order_time` 区间           | 按时间筛选      |
| `GROUP BY od.name`        | 按菜名分组      |
| `ORDER BY number DESC`    | 按销量降序      |
| `LIMIT 0,10`              | 取前10       |

老师的写法是逗号连接（`FROM order_detail od, orders o WHERE od.order_id = o.id`），是**隐式内连接**，效果和 INNER JOIN 一样。显式 INNER JOIN 更规范，推荐。

### 内连接还是外连接？怎么判断？

**三种连接的区别：**

| 连接 | 结果 | 场景 |
|------|------|------|
| INNER JOIN 内连接 | 两边**都有**匹配才出现 | 要两边都成立的数据 |
| LEFT JOIN 左连接 | 左表**全部**保留，右表没匹配就 NULL | 以左表为主，右表可缺 |
| RIGHT JOIN 右连接 | 右表全部保留，左表没匹配就 NULL | 以右表为主 |

**怎么判断用哪个？问自己一句话：**

> **"我要的数据，是不是必须两边的记录都存在？"**

- 这里统计"已完成订单里的菜品销量"：只有订单表有记录（status=5 的订单），明细才有意义；只有明细表有记录，才能统计销量 → **两边都必须有 → 内连接**
- 如果需求是"所有订单都显示，哪怕没有明细，明细显示 0" → 订单为主表 → **LEFT JOIN**
### 避坑：别名必须对应 DTO 字段名

数据库查出来有两个字段：`name`、`number`。

**关键坑：** `sum(od.number) number` 必须起别名 `number`，因为封装的 `GoodsSalesDTO` 里字段名是 `number`。

**为什么对不上就不行？** MyBatis 把查询结果映射到对象时，是**按列名找属性名**的。列名叫 `number` 才找得到 DTO 的 `number` 属性；如果列名叫 `sum(od.number)`，映射失败，`number` 就是 null，前端就不显示销量。

---
## ⚠️ 避坑 Tips

### 1. SQL 统计的坑
- **聚合函数必须起别名**。`sum(od.number)` 不写 `number`，MyBatis 映射不到 DTO 属性，结果就是 null。**别名要和 DTO 字段名一模一样**
- **GROUP BY 的字段要写全**。按 `od.name` 分组，SELECT 里就只能出现 name 和聚合函数，其他字段会报错或取随机值
### 2. 连接查询的坑
- **JOIN 条件别漏**。`ON od.order_id = o.id` 不写就变成笛卡尔积，数据翻倍，统计全错
- **多表统计确认是"交集"还是"全量"**。交集用 INNER JOIN，全量用 LEFT/RIGHT JOIN，先想清楚需求再写
- **性能**：大表 JOIN 要建索引（order_id 外键索引），否则慢查询
---
## 🎯 面试高频题

### Q1：营业额/用户/订单统计功能怎么实现的？

> **回答框架：**
> 1. **前端**：选时间区间，`?begin=&end=` 传给后端
> 2. **后端**：
>    - 算出区间内的**日期集合**
>    - 遍历每天，用 Mapper 查当天数据（营业额=已完成订单 sum(amount)，用户=count，订单=count）
>    - 无数据补 0，保证折线图连续
> 3. **返回**：dateList 转成字符串+ 数值 List 转成字符串（根据前端要求），前端 ECharts 画图
> 4. **关键**：统计口径要对——营业额只算已完成订单、有效订单=已完成

### Q2：MyBatis 查询结果映射不上，属性是 null，怎么排查？

> **回答框架：**
> 1. **别名**：聚合函数必须起别名，别名要和 DTO 属性名一致（sum(number) → number）
> 2. **驼峰映射**：检查是否开启 map-underscore-to-camel-case（create_time → createTime）
> 3. **字段类型**：数据库字段和 Java 属性类型要匹配（INT → Integer）
> 4. **列名 vs 属性名**：不一致就要配 resultMap 或别名
> 5. **项目例子**：销量统计 sum(od.number) 没起别名，number 就是 null，前端不显示销量

第十五天，真快啊，外卖马上要结束了。苍穹外卖day15，今天写了数据统计模块，写几个统计接口，业务其实大差不差，了解了这个ECharts图表，重温下sql，明天就写完了，这一部分马上完结撒花喽~