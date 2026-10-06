---
tags:
  - 苍穹外卖
  - 事务
  - 动态SQL
  - 多表联查
  - MinIO
  - 面试题
---
# Day04 - 菜品模块（事务、多表、动态SQL）

## 一、文件上传：AliOSS vs MinIO

新增菜品要上传图片，视频用的是阿里云 OSS，我学过minio，所以用的是 MinIO。搜了搜对比：

| | 阿里云 OSS | MinIO |
|---|---|---|
| 类型 | 云服务（付费） | 开源私有化部署（免费） |
| 部署 | 开箱即用，不用管运维 | 自己装在服务器上，要管运维 |
| 费用 | 按存储+流量收费 | 自己服务器成本，0 额外费用 |
| 适合场景 | 生产环境、快速上线 | 学习练手、私有化部署 |
| 兼容性 | 阿里云 SDK | 兼容 S3 协议，SDK 通用 |

**MinIO 配置步骤（自己跑一遍）：**
1. 下载安装 MinIO，启动 `minio server ./data`
2. 创建 bucket（就相当于是文件夹），设置读写权限
3. 项目里加 MinIO 依赖，配置 endpoint、accessKey、secretKey、bucket
4. 写工具类：上传文件 → 返回访问 URL
## 二、Spring 事务 @Transactional

### 1. 为什么要加事务？

新增菜品不是只插一张表：
1. 往 `dish` 表插一条菜品数据
2. 往 `dish_flavor` 表插 N 条口味数据

**问题：** 如果插完 dish 表，插 flavor 表时报错了怎么办？dish 表已经插进去了，但口味没插进去——**数据不一致**。

**解决：** 加 `@Transactional`，要么全成功，要么全失败然后回滚。

### 2. 事务四大特性（ACID）

| 特性 | 英文 | 含义 |
|------|------|------|
| 原子性 | Atomicity | 事务里的操作要么全做，要么全不做 |
| 一致性 | Consistency | 事务前后，数据库从一个一致状态到另一个一致状态 |
| 隔离性 | Isolation | 多个事务之间互不干扰 |
| 持久性 | Durability | 事务提交后，数据永久保存 |

> **简单理解：** 转账——你转给我 100 块，原子性=要么扣了你的加了我的，要么都没变；一致性=总钱数不变；隔离性=咱俩同时转钱互不干扰；持久性=转完了不能赖账。

### 3. 代码示例

```java
@Override
@Transactional  // 加上这个注解，事务就生效了
public void saveWithFlavor(DishDTO dishDTO) {
    Dish dish = new Dish();
    BeanUtils.copyProperties(dishDTO, dish);
    
    // 1. 插入菜品表
    dishMapper.insert(dish);
    
    // 2. 获取自增生成的菜品 id
    Long dishId = dish.getId();
    
    // 3. 遍历口味，设置 dishId，批量插入口味表
    List<DishFlavor> flavors = dishDTO.getFlavors();
    if (flavors != null && flavors.size() > 0) {
        flavors.forEach(df -> df.setDishId(dishId));
        dishFlavorMapper.insertBatch(flavors);
    }
    // 中间任何一步报错，前面的操作全部回滚
}
```

### 4. 修改菜品为什么"先删再增"？

修改口味时，视频的做法是：
1. 更新 dish 表
2. 删除原来的所有口味（`deleteByDishId`）
3. 再插入新的口味列表

**为什么不用"逐个对比更新"？**
- 用户可能删了几个口味、加了几个口味、改了几个口味
- 逐个 diff 太麻烦，代码复杂
- **先删再增最简单可靠**：不管原来是什么，全删了重新插，结果一定是对的
- 只要加了事务，删了再插也不会出问题（失败就回滚）

**思考**：这就是工程思维——不要过度设计。能跑对、好维护比"优雅"更重要。先删再增虽然多了几次 SQL，但逻辑简单，不容易出 bug。但是任何需要对比使用的吧，还是根据需求、成本等综合考虑后选择合适的使用。

## 三、多表联查 + 动态 SQL

### 1. 菜品分页查询（LEFT JOIN）

```sql
SELECT d.*, c.name AS categoryName
FROM dish d
LEFT JOIN category c ON d.category_id = c.id
WHERE d.name LIKE CONCAT('%', #{name}, '%')
  AND d.category_id = #{categoryId}
  AND d.status = #{status}
ORDER BY d.create_time DESC
```

**要点：**
- **LEFT JOIN（左外连接）**：以 dish 表为主表，即使 category 表没匹配上，dish 记录也保留。如果用 INNER JOIN，category 匹配不上的菜品就查不出来了。
- **`c.name AS categoryName`**：起别名，和 VO 里的属性名对应，MyBatis 才能自动封装。不然结果集里的列名是 `name`，和 dish 自己的 `name` 冲突。

### 2. 动态 SQL 

MyBatis 的动态 SQL 就是：**根据传入的参数，动态拼接 SQL 语句**。

| 标签 | 作用 | 类似 |
|------|------|------|
| `<if>` | 条件判断，满足才拼 | if 语句 |
| `<where>` | 自动处理 where 和 and/or | 智能 where |
| `<foreach>` | 遍历集合拼接 IN 条件 | for 循环 |
| `<set>` | 更新时自动处理逗号 | 智能 set |
| `<choose>/<when>/<otherwise>` | 多分支选择 | switch |

**分页查询的动态 SQL：**
```xml
<select id="pageQuery" resultType="DishVO">
    SELECT d.*, c.name AS categoryName
    FROM dish d LEFT JOIN category c ON d.category_id = c.id
    <where>
        <if test="name != null and name != ''">
            AND d.name LIKE CONCAT('%', #{name}, '%')
        </if>
        <if test="categoryId != null">
            AND d.category_id = #{categoryId}
        </if>
        <if test="status != null">
            AND d.status = #{status}
        </if>
    </where>
    ORDER BY d.create_time DESC
</select>
```

### 3. 删除菜品（foreach）

批量删除时，传进来的是 id 列表，要拼成 `IN (1,2,3)`：

```xml
<delete id="deleteBatch">
    DELETE FROM dish WHERE id IN
    <foreach collection="ids" item="id" open="(" close=")" separator=",">
        #{id}
    </foreach>
</delete>
```

**foreach 属性：**
- `collection`：传入的集合名
- `item`：遍历的每个元素
- `open`：开头拼什么
- `close`：结尾拼什么
- `separator`：分隔符

最终生成：`DELETE FROM dish WHERE id IN (1, 2, 3)`

---

## ⚠️ 避坑 Tips

### 1. 事务失效的坑（高频！）
- **同类中方法调用，事务会失效**。比如 A 方法里调用 B 方法（B 加了 @Transactional），因为 AOP 是代理对象调用，this 调用不走代理，事务就没了。
- **异常类型不对，事务不回滚**。默认只回滚 RuntimeException 和 Error，检查异常（如 IOException）不回滚。要回滚所有异常加 `@Transactional(rollbackFor = Exception.class)`。
- **方法不是 public，事务不生效**。private/protected 方法加 @Transactional 没用。
- **自己 new 的对象，事务不生效**。必须是 Spring 管理的 Bean 才有事务。
### 2. 动态 SQL 的坑
- **`<if>` 判断字符串为空，要同时判断 null 和空串**。`name != null and name != ''`，少一个都会出问题。
- **`<where>` 标签很智能**：它会自动去掉多余的 and/or，不用你自己判断第一个条件要不要加 and。
- **foreach 的 collection 名字**：如果参数是 List，默认叫 `list`；如果是数组，默认叫 `array`。用 @Param 指定名字就不会错。
### 3. 多表联查的坑
- **LEFT JOIN 和 INNER JOIN 搞混**。LEFT JOIN 左表全保留，INNER JOIN 只保留匹配上的。业务上要哪个？比如查菜品，分类可能被删了，但菜品还在，这时候要用 LEFT JOIN。
- **N+1 查询问题**。不要循环里查数据库，一次性 JOIN 查出来。100 个菜品循环查 100 次分类，性能直接崩。
- **字段冲突**。dish 表有 name，category 表也有 name，SELECT * 会冲突，一定要起别名。
---
## 🎯 面试高频题
### Q1：说一下事务的 ACID，以及 Spring 事务的原理？
> **回答框架：**
> 1. **ACID**：原子性（全做或全不做）、一致性（数据状态一致）、隔离性（事务间互不干扰）、持久性（提交后永久保存）
> 2. **Spring 事务本质是 AOP**：加了 @Transactional 的方法，Spring 会为它生成代理对象。方法执行前开启事务，正常执行完提交事务，抛异常就回滚
> 3. **底层依赖数据库事务**：Spring 只是封装了 JDBC 的事务 API，真正的事务是数据库做的
> 4. **失效场景（重点背）**：同类方法调用不生效、非 public 方法不生效、异常类型不匹配不回滚、自己 new 的对象不生效
> **一句话**：Spring 事务 = AOP 代理 + 数据库事务管理，方法前后自动开关事务，异常自动回滚。
### Q2：什么是动态 SQL？MyBatis 有哪些动态 SQL 标签？
> **回答框架：**
> 1. **是什么**：根据传入参数动态拼接 SQL 语句，不用写死 where 条件
> 2. **常用标签**：
>    - `<if>`：条件判断
>    - `<where>`：自动去 and/or
>    - `<foreach>`：遍历集合拼 IN 条件
>    - `<set>`：更新时自动去除末尾逗号
> 3. **项目里用在哪**：
>    - 菜品分页查询：name/categoryId/status 可选条件，用 `<if>` 动态拼
>    - 批量删除菜品：传 id 列表，用 `<foreach>` 拼 IN
> 4. **好处**：不用在 Java 里拼 SQL 字符串，逻辑清晰，不容易写错
### Q3：LEFT JOIN 和 INNER JOIN 的区别？
- **INNER JOIN（内连接）**：只返回两表匹配上的行。dish 和 category，category_id 找不到对应分类的菜品，不会返回。
- **LEFT JOIN（左外连接）**：左表全保留，右表没匹配上的补 null。分类被删了，菜品还能查出来，categoryName 是 null。
- **什么时候用 LEFT JOIN**：主表数据不能丢，关联表可能没匹配项。比如查菜品列表，分类可能被删了，但菜品还在卖，用 LEFT JOIN。
- **什么时候用 INNER JOIN**：只查两边都有的。比如查有库存的菜品和供应商，没供应商的不要。
### Q4：@Transactional 在什么情况下会失效？（高频）
> 1. **方法不是 public**：AOP 代理只能拦截 public 方法
> 2. **同类中自调用**：this.method() 不走代理对象，事务不生效
> 3. **异常类型不匹配**：默认只回滚 RuntimeException 和 Error，检查异常不回滚（要加 rollbackFor = Exception.class）
> 4. **数据库引擎不支持**：MyISAM 不支持事务，要用 InnoDB
> 5. **自己 new 的对象**：不是 Spring 管理的 Bean，没有代理，事务不生效

> **面试话术**："我知道事务失效的几个场景，最常见的是自调用和异常类型不匹配。项目里我们统一在 Service 层加 @Transactional(rollbackFor = Exception.class)，并且避免同类内自调用需要事务的方法。"
### Q5：事务的传播行为有哪些？（了解即可）

| 传播行为         | 含义                |
| ------------ | ----------------- |
| REQUIRED（默认） | 有事务就用当前事务，没有就新建   |
| REQUIRES_NEW | 不管有没有事务，都新建一个独立事务 |
| SUPPORTS     | 有事务就用，没有就非事务运行    |
| NESTED       | 嵌套事务，子事务回滚不影响父事务  |

> 项目里用默认的 REQUIRED 就够了。面试能说出来 REQUIRED 和 REQUIRES_NEW 的区别就行。