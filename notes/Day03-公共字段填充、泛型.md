---
tags:
  - 苍穹外卖
  - AOP
  - 自定义注解
  - 泛型
  - 面试题
  - IOC
---
# Day03 - 公共字段填充、泛型

## 一、公共字段自动填充

### 1. 遇到什么问题？

每张业务表都有 4 个公共字段：

| 字段名 | 含义 | 数据类型 | 什么时候填 |
|--------|------|----------|-----------|
| create_time | 创建时间 | datetime | insert |
| create_user | 创建人 id | bigint | insert |
| update_time | 修改时间 | datetime | insert、update |
| update_user | 修改人 id | bigint | insert、update |

**问题：** 每次新增都要手动 `employee.setCreateTime(LocalDateTime.now())`、`setCreateUser(BaseContext.getCurrentId())`……每张表都写一遍，**代码冗余、改个字段名要改一堆地方**。

### 2. 解决思路

用 **AOP 切面 + 自定义注解**，把重复的代码抽到一个地方：

1. **自定义注解 `@AutoFill`**：标记哪些 Mapper 方法需要自动填充，注解里指定是 INSERT 还是 UPDATE
2. **切面类 `AutoFillAspect`**：拦截所有带 `@AutoFill` 的 Mapper 方法，通过反射自动给公共字段赋值
3. **在 Mapper 方法上加注解**：不用改业务代码，加个注解就行

### 3. 核心代码

**自定义注解：**
```java
@Target(ElementType.METHOD)      // 只能加在方法上
@Retention(RetentionPolicy.RUNTIME)  // 运行时保留
public @interface AutoFill {
    OperationType value();       // 指定操作类型：INSERT 还是 UPDATE
}
```

**枚举（操作类型）：**
```java
public enum OperationType {
    INSERT, UPDATE
}
```

**切面类：**
```java
@Aspect
@Component
@Slf4j
public class AutoFillAspect {

    // 切入点：匹配 sky.mapper 包下所有类的所有方法，且方法上有 @AutoFill 注解
    @Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")
    public void autoFillPointcut() {}

    // 前置通知：在目标方法执行前填充公共字段
    @Before("autoFillPointcut()")
    public void autoFill(JoinPoint joinPoint) {
        // 1. 获取方法上的 @AutoFill 注解，拿到操作类型
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        AutoFill autoFill = signature.getMethod().getAnnotation(AutoFill.class);
        OperationType operationType = autoFill.value();

        // 2. 获取方法参数（约定第一个参数就是实体对象）
        Object[] args = joinPoint.getArgs();
        if (args == null || args.length == 0) return;
        Object entity = args[0];

        // 3. 准备赋值数据
        LocalDateTime now = LocalDateTime.now();
        Long currentId = BaseContext.getCurrentId();

        // 4. 根据操作类型，用反射给字段赋值
        if (operationType == OperationType.INSERT) {
            // INSERT 填 4 个字段
            setField(entity, "setCreateTime", now);
            setField(entity, "setUpdateTime", now);
            setField(entity, "setCreateUser", currentId);
            setField(entity, "setUpdateUser", currentId);
        } else if (operationType == OperationType.UPDATE) {
            // UPDATE 填 2 个字段
            setField(entity, "setUpdateTime", now);
            setField(entity, "setUpdateUser", currentId);
        }
    }

    // 反射调用 setter 方法
    private void setField(Object obj, String methodName, Object value) {
        try {
            Method method = obj.getClass().getDeclaredMethod(methodName, value.getClass());
            method.invoke(obj, value);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**Mapper 上使用：**
```java
@Insert("insert into employee ...")
@AutoFill(OperationType.INSERT)
void insert(Employee employee);
```

> **本质：** 这就是"切面"的典型应用——把多个地方重复的横切逻辑（公共字段填充）抽到一个地方，业务代码只关心自己的事。AOP 还能用在日志记录、事务控制、权限校验、性能监控上。
---
## 二、泛型 - Result 统一返回结果

### 1. 为什么要用泛型？

后端所有接口都要返回统一格式：
```json
{ "code": 1, "msg": null, "data": {...} }
```

但 data 的类型不一样：
- 登录接口：data 里是 JWT token
- 分页查询：data 里是 PageResult 对象
- 新增员工：data 是 null
**不用泛型的问题：** data 只能写 Object，调用方拿到后要强转，容易出类型转换错误。
### 2. Result 类
```java
@Data
public class Result<T> {
    private Integer code;    // 1成功，0失败
    private String msg;      // 错误信息
    private T data;         // 返回数据，类型由调用时决定
    
    public static <T> Result<T> success() {
        Result<T> result = new Result<>();
        result.code = 1;
        return result;
    }
    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.code = 1;
        result.data = data;
        return result;
    }
    public static <T> Result<T> error(String msg) {
        Result<T> result = new Result<>();
        result.code = 0;
        result.msg = msg;
        return result;
    }
}
```
---
## 三、Spring 两大核心：IOC 与 AOP
这是 Spring 最核心的两个概念，面试必问，项目里到处都在用。
### 1. IOC（控制反转）
#### 是什么？
**以前怎么写代码：**
```java
// 自己 new 对象，自己管理依赖
public class EmployeeController {
    private EmployeeService employeeService = new EmployeeServiceImpl();  // 我自己 new
}
```

问题：Controller 依赖 Service，Service 依赖 Mapper，Mabber 依赖数据库连接……层层 new，改一个实现类要改一堆地方，耦合严重。

**Spring 怎么做：**
```java
@RestController
public class EmployeeController {
    @Autowired
    private EmployeeService employeeService;  // 我不 new，Spring 给我注入
}
```

**控制反转 = 把"谁创建对象、谁管理依赖"这件事，从你手里交给 Spring 容器。**
- 以前：你自己 new 对象，你控制对象的生命周期
- 现在：Spring 容器帮你 new、帮你管理，你只管"要"就行
#### DI（依赖注入）是什么？

**DI 就是 IOC 的实现方式。** IOC 是思想，DI 是具体做法。
Spring 创建好对象之后，通过反射把它注入到你声明的字段里，这就叫依赖注入。你不用关心对象怎么来的，Spring 塞给你就用。
#### 项目里哪里用了 IOC？

| 注解 | 作用 | 项目里的例子 |
|------|------|-------------|
| `@Component` | 通用组件，交给 Spring 管理 | BaseContext 这种工具类 |
| `@Service` | 业务层组件 | EmployeeServiceImpl、CategoryServiceImpl |
| `@Repository` | 持久层组件 | Mapper 接口 |
| `@Controller` / `@RestController` | 控制层组件 | EmployeeController |
| `@Autowired` | 自动注入依赖 | Controller 里注入 Service，Service 里注入 Mapper |
| `@Configuration` | 配置类 | WebMvcConfiguration、RedisConfiguration |

> 你写的每个 `@Service`、`@RestController`、`@Autowired`，都是在用 IOC。整个项目的对象图都是 Spring 帮你拼起来的，你根本没 new 过一次 Service。
#### IOC 解决了什么问题？

- **解耦**：A 类不直接依赖 B 类的具体实现，只依赖接口，换实现不用改 A
- **易测试**：单元测试时可以注入 mock 对象，不依赖真实数据库
- **易扩展**：加一个新实现类，Spring 自动管理，业务代码不用动
### 2. AOP（面向切面编程）

#### 是什么？

**AOP = 在不修改源码的情况下，给一批方法统一加功能。**

比如公共字段填充：所有 Mapper 的 INSERT/UPDATE 都要填 createTime、updateTime。如果每个方法里手动 set，重复又难维护。AOP 就是把这段重复逻辑抽出来，在方法执行前自动织入。
#### 项目里哪里用了 AOP？

- **@AutoFill 公共字段填充**：拦截 Mapper 方法，自动填 createTime/updateTime/createUser/updateUser
- **@Transactional 事务**：Spring 自己的 AOP，方法执行前开启事务，正常提交，异常回滚
- **日志记录**：可以做一个 @Log 注解，自动记录接口入参出参
- **权限校验**：拦截需要登录的接口，没登录就报错
#### AOP 的关键概念

| 术语 | 解释 | 项目例子 |
|------|------|---------|
| 切面（Aspect） | 横切逻辑的模块 | AutoFillAspect |
| 连接点（JoinPoint） | 可以被拦截的方法 | Mapper 里的 insert/update |
| 切入点（Pointcut） | 实际被拦截的方法集合 | execution(* com.sky.mapper.*.*(..)) |
| 通知（Advice） | 拦截后做什么 | @Before 前置通知，填公共字段 |
| 目标对象（Target） | 被拦截的对象 | EmployeeMapper 实现类 |

AOP 底层是**动态代理**：
- Spring 在运行时为目标类生成一个代理对象
- 你调用的其实是代理对象的方法
- 代理对象在调用真实方法前后，插入切面逻辑（填公共字段、开启事务等）
- 最后才调用真实方法
### 3. IOC 和 AOP 的关系

**IOC 是基础，AOP 是在 IOC 之上的增强。**

- 没有 IOC，你没法管理对象，AOP 就不知道该代理谁
- 有了 IOC，所有对象都在 Spring 容器里，Spring 就可以在创建 Bean 时给它套上代理，实现 AOP
- **你写的代码里用 @Autowired 是在用 IOC，Spring 在背后给你注入的对象其实是代理对象（AOP 生效时）**

一句话：**IOC 管对象，AOP 管行为。IOC 把对象装进容器，AOP 给对象的方法加 buff。**

---
## ⚠️ 避坑 Tips
### 1. 反射的坑
- **字段名写错不会编译报错**。`setCreateTime` 拼错了，编译期不报错，运行时才抛 `NoSuchMethodException`。所以项目里把字段名做成常量（AutoFillConstant），避免手写出错。
- **getDeclaredMethod 和 getMethod 的区别**：getDeclaredMethod 能拿所有方法（包括 private），getMethod 只能拿 public 的。这里 set 方法是 public 的，两个都行。
- **基本类型 vs 包装类型**：方法参数是 `long` 和 `Long` 是两个不同的方法，getDeclaredMethod 传错类型会找不到。这里用 `Long.class` 就对了。
### 2. 公共字段填充的业务坑
- **如果业务代码手动 set 了 createTime，AOP 会覆盖它**。因为前置通知在 Mapper 方法执行前就把字段设好了，你在 Service 层 set 的值会被覆盖。想跳过自动填充就不加 @AutoFill 注解。
- **多参数方法要注意参数顺序**。项目里约定好 `args[0]` 是实体对象，如果 Mapper 方法有多个参数，第一个不是实体，就会填错对象。
---
## 🎯 面试高频题

### Q1：AOP 的实现原理是什么？JDK 动态代理和 CGLIB 有什么区别？

> **回答框架：**
> 1. **AOP 是什么**：面向切面编程，不修改源码的情况下给方法统一增强（日志、事务、权限、公共字段填充）
> 2. **底层是动态代理**：运行时为目标类生成代理对象，在方法调用前后插入增强逻辑
> 3. **两种代理方式**：
>    - **JDK 动态代理**：基于接口，目标类必须实现接口，用 `Proxy.newProxyInstance()` 创建代理
>    - **CGLIB 代理**：基于继承，目标类不用实现接口，用子类化的方式生成代理
> 4. **Spring 怎么选**：目标类实现了接口就用 JDK 动态代理，没实现接口就用 CGLIB。Spring Boot 2.x 默认用 CGLIB
> 5. **本项目**：AutoFillAspect 就是 AOP 的实际应用——拦截所有带 @AutoFill 的 Mapper 方法，自动填充公共字段
### Q2：反射有什么优缺点？

- **优点**：运行时动态操作对象，不用在编译期知道类长什么样（AOP、ORM、JSON 序列化都靠反射）
- **缺点**：
  - 性能比直接调用慢（要做安全检查、方法查找）
  - 破坏封装性（能调用 private 方法）
  - 编译期不检查，字段名写错运行时才报错
- **本项目为什么用反射**：切面不知道具体是哪个实体类（Employee、Category、Dish 都可能），只能运行时通过方法名找到 set 方法调用，这就是反射的典型场景
### Q3：什么是 IOC？什么是 DI？它们的关系？

> **回答框架：**
> 1. **IOC（控制反转）**：把"谁创建对象、谁管理依赖"这件事，从程序代码反转给 Spring 容器。以前自己 new 对象，现在 Spring 帮你创建和管理。
> 2. **DI（依赖注入）**：是 IOC 的具体实现方式。Spring 创建好对象之后，通过反射把它注入到你声明的字段里，这就叫依赖注入。
> 3. **关系**：IOC 是思想，DI 是实现。
> 4. **好处**：解耦（不依赖具体实现）、易测试（可以注入 mock）、易扩展（换实现不用改业务代码）