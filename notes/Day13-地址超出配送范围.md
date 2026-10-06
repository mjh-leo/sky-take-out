---
tags:
  - 苍穹外卖
  - 百度地图
  - 面试题
---
# Day13 - 地址超出配送范围

## 一、环境准备

注册百度账号，进入控制台创建应用获取 ak，里面让填一个 IP 白名单，我们自己写项目本地开发，直接填`0.0.0.0/0`提交就行，上线一定要改成服务器真实公网 IP。

**IP 白名单是什么？不改会怎么样？安全性和成本问题。**

- 百度地图 API 校验请求来源 IP：**只有白名单里的 IP 才能调用**
- 本地开发 IP 不固定（可能是 192.168.x.x、10.x.x.x），填 `0.0.0.0/0` = 允许所有 IP，本地才能调通
- **上线不改的后果**：服务器公网 IP 不在白名单里 → API 调用失败 → 用户下单就报"地址解析失败"，整个配送校验功能废掉
- **改了和没改的区别**：白名单只放行服务器 IP，防止别人盗用你的 ak（ak 有调用量限制，被盗用会耗尽你的额度，还要花钱）
## 二、代码开发

在 yml 文件里配置好外卖商家店铺地址和百度地图的 ak，结构 sky.shop，sky.baidu

```yml
shop:  
  address: ${sky.shop.address}  
baidu:  
  ak: ${sky.baidu.ak}
```

在`OrderServiceImpl`注入配置项
```java
 @Value("${sky.shop.address}")
    private String shopAddress;

    @Value("${sky.baidu.ak}")
    private String ak;
```

在用户下单业务里调用这个方法。
```java
// 检查用户的收货地址是否超出配送范围  
checkOutOfRange(addressBook.getCityName()+addressBook.getDistrictName()+addressBook.getDetail());
```

## 三、checkOutOfRange 方法梳理

这个方法整体做一件事：**算一下"店铺到用户收货地址"的驾车距离，超过 5000 米就拒绝下单**。

怎么算距离？百度地图不会直接告诉你"这俩地址相距多少米"，所以要分三步：

```
① 店铺地址 → 百度地理编码API → 店铺经纬度
② 用户地址 → 百度地理编码API → 用户经纬度
③ 两个经纬度 → 百度路线规划API → 驾车距离 → 和 5000 米比较
```

### 第 1 步：店铺地址 → 经纬度

```java
// 1.1 准备请求参数：地址 + 输出格式 + 你的ak密钥
Map map = new HashMap();
map.put("address", shopAddress);   // 店铺地址，从yml配置读的
map.put("output", "json");         // 返回json格式
map.put("ak", ak);                 // 你的百度密钥

// 1.2 发HTTP请求，调用百度"地理编码"接口（把文字地址转成经纬度）
String shopCoordinate = HttpClientUtil.doGet("https://api.map.baidu.com/geocoding/v3", map);

// 1.3 把返回的JSON字符串解析成对象
JSONObject jsonObject = JSON.parseObject(shopCoordinate);

// 1.4 检查返回状态：status=0 表示成功，不是0说明地址解析失败
if (!jsonObject.getString("status").equals("0")) {
    throw new OrderBusinessException("店铺地址解析失败");
}

// 1.5 从结果里取出经纬度：result.location.lat / result.location.lng
JSONObject location = jsonObject.getJSONObject("result").getJSONObject("location");
String lat = location.getString("lat");
String lng = location.getString("lng");

// 1.6 拼成"纬度,经度"格式
String shopLngLat = lat + "," + lng;
```

> **HttpClientUtil.doGet 是什么？** 就是"用代码发一个 HTTP GET 请求"的工具类，相当于你在浏览器地址栏输入这个 URL。百度返回一段 JSON 字符串，我们解析它。

### 第 2 步：用户地址 → 经纬度

和第 1 步一样，只是把 `address` 参数从店铺地址换成了用户收货地址：

```java
map.put("address", address);   // 换成用户地址
String userCoordinate = HttpClientUtil.doGet("https://api.map.baidu.com/geocoding/v3", map);

jsonObject = JSON.parseObject(userCoordinate);
if (!jsonObject.getString("status").equals("0")) {
    throw new OrderBusinessException("收货地址解析失败");
}

location = jsonObject.getJSONObject("result").getJSONObject("location");
lat = location.getString("lat");
lng = location.getString("lng");
String userLngLat = lat + "," + lng;   // 用户经纬度
```

### 第 3 步：两个经纬度 → 距离 → 比较

```java
// 3.1 准备参数：起点（店铺）、终点（用户）、不开路线详情
map.put("origin", shopLngLat);        // 起点：店铺经纬度
map.put("destination", userLngLat);   // 终点：用户经纬度
map.put("steps_info", "0");           // 只要距离，不要详细步骤

// 3.2 调用百度"驾车路线规划"接口
String json = HttpClientUtil.doGet("https://api.map.baidu.com/directionlite/v1/driving", map);

jsonObject = JSON.parseObject(json);
if (!jsonObject.getString("status").equals("0")) {
    throw new OrderBusinessException("配送路线规划失败");
}

// 3.3 解析出距离（单位：米）
JSONObject result = jsonObject.getJSONObject("result");
JSONArray jsonArray = (JSONArray) result.get("routes");       // 路线数组，取第一条
Integer distance = (Integer) ((JSONObject) jsonArray.get(0)).get("distance");

// 3.4 距离超过5000米（5公里）就拒绝下单
if (distance > 5000) {
    throw new OrderBusinessException("超出配送范围");
}
```

### 为什么用"驾车距离"不用"直线距离"？ （真实行驶距离）

- 地理编码拿到的是经纬度，如果直接用经纬度算，算出来的是**直线距离**（穿楼的那种）
- 实际配送走的是马路，有红绿灯、绕路，**驾车路线规划算的是真实行驶距离**，更接近配送员实际跑的距离
---
## ⚠️ 避坑 Tips

### 1. 百度地图 API 的坑
- **ak 是密钥，别写死在前端**。ak 要放后端（yml 配置），前端暴露 ak 就是把免费额度给别人刷
- **IP 白名单上线必须改**。本地 `0.0.0.0/0` 能跑，上线不改就 403 拒绝访问，配送校验直接挂
- **每个接口都要判断 status**。地理编码、路线规划都可能失败（地址太偏、格式不对），必须抛异常而不是继续解析，否则空指针
### 2. HTTP 调用外部 API 的坑
- **外部请求有超时风险**。百度地图慢的话，下单接口也会跟着慢。生产环境要设超时时间 + 降级方案（比如百度挂了就直接放行，别让用户下不了单）
- **JSON 解析要判空**。`getJSONObject("result")` 如果 result 不存在会返回 null，再 `.get()` 就空指针了
- **请求参数用 Map 而不是硬编码 URL**。这样方便复用同一个 API 传不同参数（店铺/用户地址）

### 3. 业务校验的坑
- **5000 米是硬编码**。最好抽成常量配置，以后配送范围改了只改配置不改代码
- **校验放在下单事务里**。超出范围直接抛异常，事务回滚，不会产生脏数据
- **地址拼接顺序**：城市 + 区县 + 详细地址，少了"城市"百度可能解析出别的地方的地址
---
## 🎯 面试高频题

### Q1：怎么判断收货地址是否超出配送范围？

> **回答框架：**
> 1. **思路**：算出店铺到用户地址的实际距离，超过阈值就拒绝
> 2. **实现三步**：
>    - 店铺地址 → 百度地理编码 API → 店铺经纬度
>    - 用户地址 → 百度地理编码 API → 用户经纬度
>    - 两个经纬度 → 百度驾车路线规划 API → 行驶距离 → 和 5000 米比较
> 3. **为什么用驾车距离不用直线距离**：配送走的是马路，驾车路线规划算的是真实行驶距离
> 4. **异常处理**：每个 API 都要判断 status，解析失败抛业务异常
> 5. **配置化**：店铺地址、ak、配送范围阈值都放 yml 配置，不写死

### Q2：项目里怎么调用第三方 API（比如百度地图）？

> **回答框架：**
> 1. **准备**：注册账号、创建应用拿密钥（ak）、配置 IP 白名单
> 2. **配置**：密钥放 yml，用 @Value 注入，不写死在代码里
> 3. **调用**：HttpClient 工具类发 GET 请求，参数用 Map 组装，请求 URL 拼 query
> 4. **解析**：返回 JSON 字符串，用 Fastjson/Jackson 解析
> 5. **健壮性**：判断业务状态码（status != 0 抛异常）、判空防空指针、考虑超时降级

今天第十三天，加一个配送范围的判断。苍穹外卖day13，今天优化了用户下单的功能，加入校验逻辑，用户的收货地址和商家门店地址距离超出配送范围，就下单失败。调用百度地图api，一顿请求解析拿到结果。