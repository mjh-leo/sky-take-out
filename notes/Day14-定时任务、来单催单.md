---
tags:
  - 苍穹外卖
  - SpringTask
  - cron
  - Websocket
  - 面试题
---
# Day14 - 定时任务、来单、催单

## 一、Spring Task

### 1. 场景

- 用户下单后一直未支付 → 自动取消订单（15分钟超时）
- 派送完成后 → 自动点击完成订单

这类"到点自动做某件事"的需求，用定时任务。

### 2. 使用步骤（3步）

1. **导入依赖**：spring-context（Spring Boot 自带，不用额外加）
2. **启动类加 @EnableScheduling**：开启任务调度
3. **自定义任务类**：方法上加 `@Scheduled(cron = "...")` 指定执行时间

### 3. cron 表达式

cron 有 6~7 位，从秒开始：`秒 分 时 日 月 周 年`

```
"0 * * * * ?"      → 每分钟的第0秒执行一次
"0 0 1 * * ?"      → 每天凌晨1点执行一次
"0 0/5 * * * ?"    → 每5分钟执行一次
"0 0 12 * * ?"     → 每天中午12点执行
```

- `*` 任意值，`?` 只用于"周"表示不指定，`/` 步长，`0/5` 从0开始每5
- 不用自己写，直接问 AI 或者用在线 cron 生成器

### 4. 两个定时任务

**① 处理超时未支付订单（每分钟）：**
```java
@Scheduled(cron = "0 * * * * ?")  // 每分钟执行一次
public void processTimeoutOrder() {
    // 找 15 分钟前还没付款的订单
    LocalDateTime time = LocalDateTime.now().plusMinutes(-15);
    List<Orders> ordersList = orderMapper.selectByStatusAndOrderTimeLT(
            Orders.PENDING_PAYMENT, time);
    if (ordersList != null && !ordersList.isEmpty()) {
        ordersList.forEach(order -> {
            order.setStatus(Orders.CANCELLED);
            order.setCancelReason("订单超时未付款，自动取消");
            order.setCancelTime(LocalDateTime.now());
        });
        orderMapper.updateBatch(ordersList);
    }
}
```

**② 处理一直处于派送中的订单（每天凌晨1点）：**
```java
@Scheduled(cron = "0 0 1 * * ?")  // 每天凌晨1点
public void processDeliveryOrder() {
    // 找前一天还在派送中的订单
    LocalDateTime time = LocalDateTime.now().plusMinutes(-60);
    List<Orders> ordersList = orderMapper.selectByStatusAndOrderTimeLT(
            Orders.DELIVERY_IN_PROGRESS, time);
    ...
}
```
## 二、WebSocket

### 1. 是什么

**WebSocket 是基于 TCP 的新协议，一次握手建立持久连接，浏览器和服务器双向实时通信。**

**和 HTTP 对比：**

| | HTTP | WebSocket |
|---|---|---|
| 连接 | 短连接，请求一次断一次 | 长连接，握手后一直保持 |
| 通信方向 | 单向（客户端请求 → 服务端响应） | 双向（两边随时发） |
| 底层 | TCP | TCP（协议不同） |
| 适合 | 普通接口调用 | 实时推送：弹幕、聊天、行情、来单提醒 |

**简单理解：** HTTP 是"你问我答"，服务端不能主动找你；WebSocket 是"拉起一根电话线"，服务端随时能主动给你打电话（推消息）。

### 2. 应用场景

视频弹幕、网页聊天、体育实况更新、股票基金报价实时更新——**凡是"服务端要主动推给客户端"的场景**。
## 三、来单提醒、客户催单

### 1. 需求分析

- 管理端页面和服务器保持**长连接**（打开页面就连上 WebSocket）
- 客户支付后 → 服务端**主动推送**消息
- 浏览器解析消息 → 判断是来单(1)还是催单(2) → **弹提示 + 语音播报**
- 消息格式 JSON：`{type, orderId, content}`
### 2. nginx 转发

```nginx
location /ws/ {
    proxy_pass http://webservers/ws/;
}
```

WebSocket 握手时浏览器发的是 `ws://localhost/ws/xxx`，nginx 把 `/ws/` 开头的请求转发到后端。

**抓包看到的握手过程：**
```
请求 URL: ws://localhost/ws/jxw35jp24f
请求方法: GET
状态码: 101 Switching Protocols
Upgrade: websocket
Connection: upgrade
```

**状态码 101 = 协议切换成功**，HTTP 升级成了 WebSocket，长连接建立。

### 3. 服务端推送消息

```java
Map map = new HashMap();
map.put("type", 1);        // 1来单提醒，2客户催单
map.put("orderId", orders.getId());
map.put("content", "订单号：" + orders.getNumber());

String json = JSON.toJSONString(map);
webSocketServer.sendToAllClient(json);  // 推给所有连接的管理端页面
```

**抓包收到的消息：**
```json
{"orderId": 25, "type": 1, "content": "订单号: 1790844190456"}
```

客户催单逻辑一样，`type` 改成 2 就行。

## 四、WebSocket 交互完整过程

以"客户支付 → 管理端响铃"为例，走一遍完整链路：

```
客户支付成功
    ↓
后端下单/支付业务代码里：构造 JSON 消息
    ↓
webSocketServer.sendToAllClient(json)   ← 服务端主动推送
    ↓
（长连接早已建立：管理端页面打开时 ws = new WebSocket(...)）
    ↓
管理端浏览器 ws.onmessage 收到消息
    ↓
前端解析 JSON，判断 type
    ↓
type=1 → 播放来单提示音 + 弹窗提示
```

**关键点：**
- **连接什么时候建立的？** 管理端页面加载时就 `new WebSocket()`，握手成功后一直挂着。不是支付那一刻才连的
- **服务端怎么找到前端？** `sendToAllClient` 把消息发给**所有**连接的客户端（管理端页面），前端自己判断 type 决定怎么处理
- **WebSocketServer 怎么存连接？** 每个连接进来时把 session 存进一个 Map/Set，发消息时遍历所有 session 逐个发

## 五、音频是怎么发出声音的？

### 1. 前端怎么播放？

从抓包看到页面加载了两个 mp3：`reminder.0a3849af.mp3`（来单提示音）、`preview.3f1fe127.mp3`。前端代码大概是这样：

```javascript
// 1. 页面加载时连接 WebSocket
const ws = new WebSocket('ws://localhost/ws/' + sessionId);

// 2. 收到服务器推送的消息
ws.onmessage = function(event) {
    const data = JSON.parse(event.data);
    
    if (data.type === 1) {
        // 来单提醒：播放来单提示音
        new Audio('/audio/reminder.mp3').play();
        alert('您有新的订单！');
    } else if (data.type === 2) {
        // 客户催单：播放催单提示音
        new Audio('/audio/preview.mp3').play();
        alert('客户催单，请尽快处理！');
    }
};
```

### 2. 原理拆解

| 步骤 | 代码 | 干什么 |
|------|------|--------|
| 创建音频对象 | `new Audio('reminder.mp3')` | 在内存里创建一个音频播放器，指向 mp3 文件 |
| 开始播放 | `.play()` | 浏览器加载并播放这个 mp3，扬声器出声 |
| 收到消息 | `ws.onmessage` | WebSocket 把后端推的 JSON 回调给前端 |
| 判断类型 | `data.type === 1` | 决定播放哪个音频、显示什么提示 |

---
## ⚠️ 避坑 Tips

### 1. WebSocket 的坑

- **nginx 要配置 upgrade 头**。只配 proxy_pass 不够，还要配 `proxy_set_header Upgrade $http_upgrade;` 和 `Connection "upgrade";`，不然握手 400
- **断线重连**。网络断了 WebSocket 会自动断开，前端要监听 onclose 然后重连，不然页面就"静音"了
- **不要给无关用户推**。正式项目一般按 sessionId/用户维度定向推送，`sendToAllClient` 是所有人
- **长连接占用资源**。每个连接占一个 TCP 连接，连接数多了要处理（心跳、超时释放）

### 2. 音频播放的坑

- 在写这个任务前我们写了个入门案例，写了个WebSocketTask，写得每个5秒发一次消息，一定要注掉！
- 没有声音了可能WebSocket 连接断开或缓存问题，重新连接清缓存重新登录就可以了。
- **浏览器自动播放限制**。Chrome 等浏览器禁止页面一打开就自动播放声音（用户没交互过）。这里因为是用户点了"支付"触发的后续消息，一般没问题；纯页面加载就响铃会被拦，设置一下就可以。
---
## 🎯 面试高频题

### Q1：HTTP 和 WebSocket 有什么区别？什么场景用 WebSocket？

> **回答框架：**
> 1. **连接**：HTTP 短连接，一次请求响应就断；WebSocket 长连接，握手后一直保持
> 2. **方向**：HTTP 单向（客户端请求→服务端响应）；WebSocket 双向，服务端能主动推
> 3. **底层**：都基于 TCP，WebSocket 是在 HTTP 握手后升级协议（101）
> 4. **场景**：需要服务端主动推送的——来单提醒、弹幕、聊天、行情、在线人数
> 5. **项目例子**：来单提醒，客户支付后服务端通过 WebSocket 主动推消息给管理端页面，前端收到后播提示音
### Q2：WebSocket 的握手过程是怎么样的？

> **回答框架：**
> 1. 客户端发 HTTP 请求，带 `Upgrade: websocket` 头
> 2. 服务端返回 **101 Switching Protocols**，协议切换成功
> 3. 之后连接变成全双工长连接，双方随时互发消息
> 4. 项目里：管理端页面 `new WebSocket('ws://localhost/ws/xxx')`，nginx 转发 `/ws/` 到后端，后端 WebSocketServer 处理握手
### Q3：来单提醒功能怎么实现的？

> **回答框架：**
> 1. **连接**：管理端页面加载时建立 WebSocket 长连接
> 2. **推送**：客户支付成功后，后端构造 `{type:1, orderId, content}` JSON，通过 WebSocket 推给管理端
> 3. **前端**：`ws.onmessage` 收到消息，解析 type，type=1 播放来单提示音 + 弹窗，type=2 播放催单音
> 4. **声音**：`new Audio('reminder.mp3').play()`，mp3 是前端静态资源，浏览器播放

第十四天，继续继续！苍穹外卖day14，今天写了定时任务和催单接单，了解了corn表达式。来单提醒这种确实又见识到了，开始跟着视频写了个入门案例，后面写完功能测试，声音就一直响，一定要注掉这个案例。