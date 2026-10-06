---
tags:
  - 苍穹外卖
  - 微信登录
  - JWT
  - 拦截器
  - 面试题
---
# Day07 - 微信登录 + 用户端拦截器

## 一、微信小程序登录流程

微信登录和普通的"账号密码登录"不一样，它的核心是**用授权码 code 换 openid**。

### 1. 完整流程

```
小程序                你的服务器              微信服务器
  |                     |                      |
  |--- wx.login() ----->|                      |
  |<--- 拿到 code ------|                      |
  |                     |                      |
  |--- 发送 code ------>|                      |
  |                     |--- appid+secret+code ->|
  |                     |                      |
  |                     |<-- openid+session_key-|
  |                     |                      |
  |<--- 返回 JWT token -|                      |
  |                     |                      |
  |--- 以后请求都带 JWT token --------------->|
  |<--- 返回业务数据 ----|                      |
```

**每一步干什么：**
1. 小程序调 `wx.login()` 拿到临时 code
2. 把 code 发给你的后端
3. 后端拿 code + appid + appsecret 去微信服务器换 openid + session_key
4. 后端生成自己的 JWT token 返回给小程序
5. 以后小程序每次请求都带 JWT，后端校验 JWT

### 2. openid 是什么？

- 每个用户在每个小程序里的**唯一标识**，相当于微信里的用户 id
- 同一个微信用户，在不同小程序里 openid 不一样
- 你的后端只要存 openid，就知道是哪个微信用户了

### 3. 我踩的坑

**appId 填错了，微信返回：**
```json
{"errcode": 40029, "errmsg": "invalid code"}
```

排查了半天才发现是微信小程序 appId 填错了。。。

> **教训：** 调第三方接口报错，先检查配置对不对（appId、secret、URL），别上来就怀疑代码。配置错了，代码再对也没用。

## 二、用户端 JWT 拦截器

### 1. 和 admin 端的区别

admin 端（管理后台）的拦截器我们之前写过了，用户端（小程序）也要写一个，因为：
- admin 端用的 token 是员工 id，用户端用的是用户 id
- 拦截路径不一样：admin 是 `/admin/**`，用户端是 `/user/**`
- 排除的路径也不一样

### 2. 代码

**JwtTokenUserInterceptor：**
```java
@Component
public class JwtTokenUserInterceptor implements HandlerInterceptor {

    @Autowired
    private JwtProperties jwtProperties;

    @Override
    public boolean preHandle(HttpServletRequest request, 
                             HttpServletResponse response, Object handler) {
        String token = request.getHeader("token");
        try {
            JwtUtil.parseJWT(jwtProperties.getUserSecret(), token);
            // 从 token 里解析出用户 id，存到 ThreadLocal
            Long userId = ...;
            BaseContext.setCurrentId(userId);
            return true;  // 放行
        } catch (Exception e) {
            response.setStatus(401);
            return false;  // 不放行
        }
    }

    @Override
    public void afterCompletion(...) {
        BaseContext.removeCurrentId();  // 记得清理！
    }
}
```

**注册拦截器：**
```java
@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {
    @Autowired
    private JwtTokenUserInterceptor jwtTokenUserInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(jwtTokenUserInterceptor)
                .addPathPatterns("/user/**")        // 拦截用户端所有请求
                .excludePathPatterns("/user/user/login")  // 登录接口不拦截
                .excludePathPatterns("/user/shop/status"); // 店铺状态不拦截
    }
}
```

### 3. 为什么登录接口和店铺状态不拦截？

- **登录接口**：用户还没登录呢，哪来的 token？肯定不能拦截
- **店铺状态**：用户还没登录就能看到店铺打烊了，这个不需要登录就能访问
## 三、商品浏览功能测试

导入用户端商品浏览代码，测试了一下，没问题。

这部分比较简单，就是：
- 按分类查商品套餐列表
- 查商品详情（带口味）
- 查套餐详情（带菜品）
---
## ⚠️ 避坑 Tips

### 1. 第三方接口报错的坑
- **先查配置，再查代码**。appId、secret、URL、请求方式、参数名，这些配错了，代码写得再对也没用
- **code 是一次性的**。微信给的 code 只能用一次，用第二次就失效了。别拿着同一个 code 反复试
---
## 🎯 面试高频题

### Q1：微信登录的流程是什么？和普通登录有什么区别？

> **回答框架：**
> 1. **普通登录**：用户名+密码，后端验证密码，对了就发 token
> 2. **微信登录**：
>    - 小程序调 wx.login() 拿临时 code
>    - 后端拿 code + appid + appsecret 去微信服务器换 openid
>    - 后端生成自己的 JWT token 返回
>    - 以后请求带 JWT
> 3. **区别**：微信不需要用户输密码，用微信的身份授权；openid 是微信给的，不是你自己存的
> 4. **关键点**：code 是临时的、一次性的；appsecret 只能后端用；openid 是用户的唯一标识
### Q2：JWT 拦截器怎么实现的？为什么要用 ThreadLocal？

> **回答框架：**
> 1. **拦截器做什么**：preHandle 里从请求头拿 token，解析 token，拿到用户 id
> 2. **为什么用 ThreadLocal**：Controller 里要用当前用户 id，但不能从参数一层层传，所以存在 ThreadLocal 里，哪里用哪里取
> 3. **为什么要 afterCompletion 清理**：Tomcat 用线程池，线程复用，不清理会导致下一个请求读到上一个用户的 id，数据串号
> 4. **本项目有两个拦截器**：admin 端存员工 id，user 端存用户 id，都用 BaseContext 存
### Q3：拦截器和过滤器有什么区别？

|                     | 过滤器 Filter   | 拦截器 Interceptor    |
| ------------------- | ------------ | ------------------ |
| 来源                  | Servlet 规范   | Spring MVC         |
| 能拦截什么               | 所有请求（包括静态资源） | 只拦截 Spring MVC 的请求 |
| 能不能拿到 Controller 信息 | 不能           | 能（Handler）         |
| 依赖                  | 不依赖 Spring   | 依赖 Spring          |

> **项目里为什么用拦截器不用过滤器**：因为我们要在 Spring 环境里用 @Autowired 注入 JwtProperties、BaseContext，过滤器里不好注入。