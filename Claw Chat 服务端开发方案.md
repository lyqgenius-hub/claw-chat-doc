# 速控侠（Claw Chat）服务端开发方案

## 一、方案概述

速控侠服务端是三端协同架构的核心调度与数据中枢。其核心目标是承担账号权限管控、设备组网协调、数据兜底中转、跨端数据同步、计费（免费/10元/20元）及广告变现支撑等核心职责。本方案基于 **JDK 21 LTS + Spring Boot 3.5.x + PostgreSQL** 构建，旨在为开发团队提供清晰的架构规范、接口定义与数据库落地标准。此外，服务端将配套提供统一的 Web 前端服务，涵盖软件官网展示页、普通用户自助后台以及管理员总控后台。

## 二、场景描述

系统的核心业务场景主要分为系统级自动交互与用户级主动操作两大类：

### 2.1 系统级交互场景 (System-to-System)

* **复杂网络下的穿透失败与自动兜底**：当双端处于极严的 NAT 隔离网络导致 P2P 穿透失败并上报状态后，服务端将自动分配 WebSocket 中转房间，安全转发控制指令并主动拦截视频流以保护带宽。
* **跨端资产的秒级漫游**：用户在 PC 端新建自动化脚本并落库成功后，服务端利用长连接向手机端推送同步信令，实现配置的静默拉取与无缝漫游。

### 2.2 用户级操作场景 (Human-to-System)

* **商业化变现与权限流转**：免费版用户下发指令前需观看广告，服务端接收穿山甲 Webhook 回调记录收益。用户支付 20 元开通金牌会员后，服务端向在线设备广播权限更新信令，即时点亮高级功能。
* **设备安全与客服管控**：用户可在自助后台强制丢失的手机下线，服务端通过 Redis 拉黑 Token 并发送死亡信令；B 端运营人员可在总控后台执行退款，并将用户权限降级回免费版。

## 三、功能列表

* **WebRTC/P2P 信令引擎**：协助多端交换 SDP 信令，建立直连通道。
* **设备心跳与在线管控**：基于 WebSocket 实时监控多端设备在线状态，支持单账号多端“挤占下线”逻辑。
* **兜底中转控制**：当 P2P 失败时，提供安全的数据中转服务，严格拦截大体积二进制视频流。
* **统一认证与 Token 管理**：支持邮箱验证码注册登录，提供双 Token 机制静默换发新凭证。
* **计费与权限调度**：控制免费、普通（10元）、金牌（20元）用户的权限下发与定时降级逻辑。
* **广告防作弊校验**：校验穿山甲回调签名，记录广告变现流水。
* **用户订单信息展示功能**：支持普通用户在手机端、PC 端及自助后台查看自己的费用订单列表，包括订单编号、价格、商品种类、支付状态及交易时间。
* **多角色 Web UI 服务**：提供官网展示页、普通用户自助后台及总控后台的数据接口支持。

## 四、功能设计

* **三端通信与组网模块**：这是解决设备连通性的核心。开发人员在处理 `CMD_RELAY` 中转处理器时，必须严格拦截二进制流（如视频流帧），仅放行 `JSON/Text` 控制指令，以严控云端带宽。
* **鉴权与权限流转模块**：下发 `ad_free` 标识控制客户端广告展示（免费版为 false，付费版为 true）。结合 Spring `@Scheduled` 每日凌晨 2:00 扫描到期用户，强制改回 `FREE` 状态并推送降级通知。
* **订单与防刷模块**：支付回调时需利用 Redis 分布式锁 `lock:order:{orderNo}` 防止并发；针对穿山甲回调，需拦截 IP 白名单并验证 `X-Pangle-Signature` 签名，处理透传的 userId 后记入系统日志。
* **Web 前端架构拆分**：官网建议使用 Next.js 或纯静态 HTML 优化 SEO；用户后台与总控后台建议合并为 Ant Design Pro V5 项目，通过路由权限（`ROLE_USER` / `ROLE_ADMIN`）动态控制菜单。

## 五、工程结构与技术规范

* **核心技术栈**：开发环境基于 IntelliJ IDEA，基础环境采用 JDK 21 和 Spring Boot 3.5.x。
* **数据与组件选型**：数据存储使用 PostgreSQL 16+ 和 Redis 7.2+，ORM 框架采用 MyBatis-Plus 3.5.x 配合 Flyway 管理版本，配置中心使用 Nacos 2.x，鉴权采用 Spring Security + jjwt。
* **项目代码包结构 (`com.clawchat.server`)**：
* `config`：全局配置（Redis, Nacos, WebSocket, MyBatis-Plus）。
* `security`：鉴权拦截器（JWT Filter, BCrypt 加密）。
* `controller`：RESTful API 入口（前端/移动端/PC端请求）。
* `websocket`：WebSocket 处理器（信令交换、消息中转、设备保活）。
* `service` / `mapper` / `entity`：标准业务层、数据库操作层及实体类。
* `dto`：数据传输对象（Req/Resp, 统一响应体）。
* `enums` / `exception`：全局枚举与异常处理。



## 六、接口与通信协议规范

### 6.1 HTTP RESTful API 规范

所有接口必须返回统一的 `RB<T>` 格式，以完美适配多端的链路追踪与错误提示规范：

```json
{
  "code": 200,                  // 状态码 (如 200 表示 SUCCESS)
  "msg": "操作成功",               // 提示信息
  "success": true,              // 成功标识布尔值
  "tdata": { ... },             // 单体业务数据负载 (泛型 T)
  "data": {                     // 分页业务数据负载 
    "list": [ ... ], "current": 1, "pageSize": 10, "total": 100
  },
  "traceId": "550e8400-e29b..." // 链路追踪ID (UUID)
}

```

### 6.2 手机端与 PC 端核心调用接口定义

为支撑 PC 端与手机端的功能流转，以下接口的 `URL` 均以 `/api/` 统一标识：

**1. 账号认证与登录接口**

* **路径**: `POST /api/auth/login`
* **功能**: 手机端/PC端提交账号密码换取通信凭证。
* **请求体**: `{"email": "user@test.com", "password": "加密字符串"}`
* **响应 `tdata**`: `{"accessToken": "jwt_xxx", "refreshToken": "jwt_yyy"}`

**2. 获取用户信息与权限接口**

* **路径**: `GET /api/v1/user/info`
* **功能**: 客户端获取当前账号的权限类型、到期时间，用于点亮高级功能。
* **请求头**: `Authorization: Bearer <accessToken>`
* **响应 `tdata**`: `{"email": "...", "versionType": "GOLD", "versionExpireTime": "2025-12-31T23:59:59"}`

**3. 用户订单信息展示接口（新增）**

* **路径**: `GET /api/v1/user/orders`
* **功能**: 手机端/PC端分页获取用户的历史充值订单明细。
* **请求参数**: `?current=1&pageSize=10`
* **响应 `data.list**`:
```json
[{
    "orderNo": "ORD202408010001",
    "amount": 20.00,
    "productType": "GOLD_MONTH",
    "payStatus": "SUCCESS",
    "payTime": "2024-08-01T14:30:00"
}]

```



**4. 设备绑定与状态上报接口**

* **路径**: `POST /api/v1/device/bind`
* **功能**: 设备登录后上报硬件指纹，供服务端进行单端挤占与多端管理。
* **请求体**: `{"deviceId": "手机/PC的唯一机器码", "deviceType": "ANDROID / WINDOWS"}`

**5. 跨端自动化脚本同步接口**

* **路径**: `POST /api/v1/template/sync`
* **功能**: PC 端上传本地录制的 RPA 脚本，手机端拉取执行。
* **请求体**: `{"name": "一键巡检", "shortcutNumber": "001", "scriptJsonPayload": "..."}`

### 6.3 WebSocket 实时通信协议规范

这是速控侠的核心命脉，用于设备状态监控、信令交换和指令中转。

* **连接地址**: `ws://server-ip:port/api/ws/connect?token=jwt_token&deviceId=xxx`
* **消息帧格式 (JSON Text Frame)**:
```json
{
  "type": "SIGNAL_OFFER",    // 消息类型：PING, PONG, SIGNAL_OFFER, CMD_RELAY等
  "targetDeviceId": "xxx",   // 目标设备ID
  "payload": { ... }         // 具体内容，如 SDP 或中转指令
}

```


* **保活机制 (Keep-Alive)**：客户端每 15 秒发送 `PING`，服务端返回 `PONG`。超过 45 秒未收到 PING，服务端主动断开连接，更新设备 `is_online = false`。

## 七、数据库设计

数据库开发需使用 Flyway 脚本管理，核心表结构约束如下：

* **用户核心表 (`sys_user`)**：记录邮箱、BCrypt 加密的密码、会员类型（FREE, STANDARD, GOLD）及到期时间。
* **财务与订单表 (`sys_order`)**：记录业务订单号、金额、商品种类、支付状态及交易时间；为提升订单列表展示的查询效率，需在 `user_id` 和 `pay_time` 上建立索引。
* **自动化脚本资产表 (`ai_task_template`)**：记录多端同步的 RPA 物理动作序列（JSON 格式）及版本时间戳。
* **系统辅表**：包括验证码审计表（`sys_verify_code`）、用于踢下线的在线设备管控表（`sys_device`）以及广告收益穿山甲对账表（`sys_ad_log`）。

## 八、部署与运维方案

* **部署架构**：推荐采用 Docker + Docker Compose (或 K8s) 的容器化部署方案。
* **网关与环境**：使用 Nginx 作为统一网关，暴露后端 API 接口（`/api/*`）并处理 SSL 证书，同时代理 Ant Design Pro 编译后的 Web 静态资源。
* **数据托管**：PostgreSQL 与 Redis 建议直接采用云厂商的 RDS 托管服务，保障数据高可用与每日自动备份。
* **动态扩展策略**：通过 Nacos 配置中心，保障未来业务量增长时，将“Web/计费模块”与“设备 WebSocket 信令模块”拆分为独立集群的平滑过渡。