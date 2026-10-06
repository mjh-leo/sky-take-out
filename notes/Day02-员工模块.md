---
tags:
  - 属性拷贝
  - 线程局部变量
  - 异常
---
# Day02 - 员工模块

## 一、新增员工功能

### 1. 属性拷贝 BeanUtils.copyProperties

新增员工时，前端传过来的是 EmployeeDTO，数据库表对应的是 Employee 实体。两个类字段名基本一致，不需要手动 set 每一个字段，用 `BeanUtils.copyProperties(来源,目标)` 拷贝。

```java
Employee employee = new Employee();
BeanUtils.copyProperties(employeeDTO, employee);
```

**注意：**
- 拷贝是按**字段名匹配**的，名字不一样的字段不会拷贝
- 是**浅拷贝**，引用类型只拷地址不拷对象
- 拷贝完别忘手动设置：状态、密码（MD5）、创建时间、创建人 id
### 2. ThreadLocal 线程局部变量

**问题：** 怎么在 Service 层拿到当前登录用户的 id？

- 拦截器在 preHandle 里校验了 JWT，拿到了用户 id
- 但 Controller → Service → Mapper 一层层往下传 id 太麻烦
- 解决：用 ThreadLocal 把用户 id 存在当前线程里，即用即取

**BaseContext 工具类：**
```java
public class BaseContext {
    private static ThreadLocal<Long> threadLocal = new ThreadLocal<>();

    public static void setCurrentId(Long id) {
        threadLocal.set(id);
    }

    public static Long getCurrentId() {
        return threadLocal.get();
    }

    public static void removeCurrentId() {
        threadLocal.remove();
    }
}
```

  >**思考：业务中调用为什么不用 new BaseContext()？**
>  BaseContext 里面维护的是**静态 ThreadLocal 变量**，getCurrentId ()、setCurrentId ()、removeCurrentId () 都是**静态方法**。 静态方法属于类本身，**不需要 new 实例**，直接`类名.方法()`调用。

> **在拦截器的 afterCompletion 里清理 ThreadLocal**：
> ```java
> @Override
> public void afterCompletion(HttpServletRequest request, 
>     HttpServletResponse response, Object handler, Exception ex) {
>     BaseContext.removeCurrentId();
> }
> ```
> **为什么？** Tomcat 用线程池，请求处理完线程不会销毁，会复用给下一个请求。如果不清理，下一个请求用了同一个线程，就会读到上一个请求的用户 id，导致**数据串号**——A 用户的操作变成 B 用户的了。

> **思考：**
> 本项目里可能会出现串号问题，不会出现内存泄漏情况，因为BaseContext里静态ThreadLocal实例不会被GC回收，那么key不会为null，不会产生脏Entry（key = null）。本地测试，单账号，不写remove，看不出bug。
## 二、全局异常处理器 GlobalExceptionHandler

### 1. 两个地方的异常类有比较

| 位置                              | 作用         | 内容                                                                                     |
| ------------------------------- | ---------- | -------------------------------------------------------------------------------------- |
| `common/exception/`             | **定义异常类型** | 自定义异常类，比如 `AccountNotFoundException`、`PasswordErrorException`，都是继承自 `RuntimeException` |
| `server/GlobalExceptionHandler` | **处理异常**   | 用 `@RestControllerAdvice` 拦截所有 Controller 抛出的异常，统一返回友好提示                               |

**一句话：** common 里是"定义有哪些异常"，server 里是"异常来了怎么处理"。
### 2. 处理 SQL 唯一约束异常

新增员工时，如果用户名重复，数据库会抛 `SQLIntegrityConstraintViolationException`，错误信息里有 `Duplicate entry 'xxx' for key 'employee.idx_username'`。

```java
@ExceptionHandler
public Result exceptionHandler(SQLIntegrityConstraintViolationException ex) {
    String message = ex.getMessage();
    if (message.contains("Duplicate entry")) {
        String[] split = message.split(" ");
        String username = split[2];  // 取出重复的用户名
        return Result.error(username + " 已存在"); // 写好常量替代提示信息，不写死
    } else {
        return Result.error("未知错误");
    }
}
```

**为什么要这样处理？** 不然前端直接看到数据库原始报错，用户根本看不懂，这样处理对客户友好。
## 三、分页查询 PageHelper

### 1. 用法

```java
PageHelper.startList(page, pageSize);  // 从第几页开始，每页几条
List<Employee> list = employeeMapper.list(name);  // 正常查询
// 返回 PageResult，包含 total 和 records
```
### 2. 底层原理？

**PageHelper 底层是动态拼接 SQL 的 limit 语句。**

它用 MyBatis 的拦截器（Interceptor）机制：
1. `PageHelper.startPage()` 把分页参数存到 ThreadLocal 里
2. 执行下一条 SQL 时，MyBatis 插件拦截 Executor.query()
3. 从 ThreadLocal 取出分页参数，在原 SQL 后面拼上 `limit offset, size`
4. 同时执行一条 `count(*)` 查询总条数
5. 把结果包装成 Page 对象返回
## 四、日期统一格式化

**问题：** LocalDateTime 序列化成 JSON 时，默认格式不好看，每个接口都加 `@JsonFormat` 太麻烦。

**解决方法：** 扩展 Spring MVC 的消息转换器，全局统一处理 LocalDateTime 的序列化格式。

```java
@Override
protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
    converter.setObjectMapper(new JacksonObjectMapper());  // 自定义的 ObjectMapper
    converters.add(0, converter);  // 加到最前面，优先级最高
}
```

**JacksonObjectMapper** 里配置了 LocalDateTime 的序列化/反序列化格式：
```java
public class JacksonObjectMapper extends ObjectMapper {
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";

    public JacksonObjectMapper() {
        SimpleModule module = new SimpleModule();
        module.addSerializer(LocalDateTime.class, 
            new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)));
        module.addDeserializer(LocalDateTime.class, 
            new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)));
        this.registerModule(module);
    }
}
```

> **对比：** 方式一（每个字段加 `@JsonFormat`）简单但重复；方式二（全局消息转换器）一次配置全局生效，更优雅，项目里推荐用方式二。

## 五、三个常用注解对比

| 注解 | 用于 | 用法 | 什么时候用 |
|------|------|------|-----------|
| `@PathVariable` | 路径中的参数 | `@GetMapping("/emp/{id}")` → `@PathVariable Long id` | RESTful 风格，参数在 URL 路径里 |
| `@RequestParam` | 查询参数 | `@GetMapping("/emp?page=1")` → `@RequestParam Integer page` | GET 请求的查询参数 |
| `@RequestBody` | 请求体 | `@PostMapping("/emp")` → `@RequestBody EmployeeDTO dto` | POST/PUT 请求的 JSON 数据 |

---
## ⚠️ 避坑 Tips

### 1. ThreadLocal 必须清理
- **线程池复用导致数据串号**是最经典的坑。拦截器 preHandle 存了 ThreadLocal，afterCompletion 一定要 remove。不清理的后果：A 用户登录后创建的订单，可能算到 B 用户头上。
- 不只是 ThreadLocal，任何用 ThreadLocal 存东西的场景（事务、数据源路由、灰度发布），用完都要清理。
### 2. BeanUtils 拷贝的坑
- **字段名不一致就拷不过去**。比如 DTO 里叫 `userName`，Entity 里叫 `name`，copyProperties 不会报错，但就是拷不过去，最后这个字段是 null。
- **浅拷贝坑**：如果属性是引用类型（比如 List），拷贝的是地址，改一个另一个也变。
- 别用 BeanUtils 做深拷贝，真要深拷贝用序列化或 JSON 转换。
### 3. PageHelper 的坑
- **startPage() 只对下一条 SQL 生效**。如果中间你手贱又执行了别的查询，分页就跑到那条上去了。
- **不支持嵌套查询**。如果你的 SQL 里有子查询，PageHelper 拼 limit 可能会出问题。
- 分页参数要做边界校验：page 不能小于 1，pageSize 不能太大（不然一次查十万条就崩了）。
### 4. 全局异常处理的坑
- **自定义异常要继承 RuntimeException**，别继承 Exception，不然 Spring 事务不会回滚。
- 异常处理器里要打日志！不然线上出了问题你都不知道是怎么回事。
---
## 🎯 面试题

### Q1：ThreadLocal 用过吗？它的原理是什么？

> **回答框架：**
> 1. **是什么**：ThreadLocal 是线程局部变量，每个线程有自己独立的副本，线程之间互不影响
> 2. **原理**：每个 Thread 对象内部有一个 ThreadLocalMap，key 是 ThreadLocal 实例，value 是存的值。线程隔离的本质是数据存在自己的线程对象里。
> 3. **场景**：当前登录用户信息（本项目里的 BaseContext）、事务管理器、数据源路由、SimpleDateFormat 线程安全问题
> 4. **坑**：**必须用完 remove()**，线程池复用线程会导致数据串号；另外 ThreadLocal 用 key 是弱引用，可能内存泄漏

> **简单理解**：ThreadLocal 就是给每个线程发了一个独立的小抽屉，你往里放东西，只有你这个线程能拿到。

### Q2：Spring 中 BeanUtils 和 Apache 的 BeanUtils 有什么区别？

- **Spring 的 BeanUtils**：基于反射，拷贝快，直接 get/set
- **Apache Commons BeanUtils**：带类型转换，功能多但慢，而且有坑（默认会把空字符串转成 null，容易出问题）
- **项目里用哪个？** 用 Spring 的 `org.springframework.beans.BeanUtils`，性能好，够用了

> 导错包了行为完全不一样。

### Q3：PageHelper 的原理是什么？有没有用过其他分页插件？

> **回答框架：**
> 1. **原理**：MyBatis 插件机制（Interceptor），拦截 Executor 的 query 方法，在 SQL 后面拼 limit 语句，同时执行 count 查询总条数
> 2. **流程**：startPage() 把分页参数存 ThreadLocal → 拦截 SQL → 拼 limit → 执行 → 从 ThreadLocal 清理
> 3. **优缺点**：优点是无侵入，业务代码不用改；缺点是不支持复杂嵌套查询、不支持 Oracle 特殊分页语法
> 4. **替代方案**：MyBatis-Plus 的分页插件（更现代）、自己写 RowBounds 分页

### Q4：@RestControllerAdvice 和 @ControllerAdvice 有什么区别？

- `@ControllerAdvice`：全局异常处理，默认返回视图
- `@RestControllerAdvice`：`@ControllerAdvice + @ResponseBody`，返回 JSON 数据
- 现在前后端分离项目，一律用 `@RestControllerAdvice`
