# 速控侠（Claw Chat）服务端开发方案

## 一、方案概述

速控侠服务端是三端协同架构的核心调度与数据中枢，承担账号权限管控、设备组网协调、数据兜底中转、跨端数据同步、计费（免费/10元/20元）及广告变现支撑等核心职责。
本方案基于 **JDK 21 LTS + Spring Boot 3.5.x + PostgreSQL** 构建，为开发团队提供清晰的架构规范、接口定义与数据库落地标准。
此外，服务端还将配套提供统一的 Web 前端服务，涵盖软件官网展示页、普通用户自助后台以及管理员总控后台。

## 二、场景描述 (多维业务场景推演)

为指导开发、测试人员理解业务上下文及上下游流转关系，特梳理以下核心场景闭环：

### 2.1 系统级交互场景 (System-to-System)

* 🎬 **场景一：复杂网络下的穿透失败与自动兜底 (网络中转场景)**
  双端处于极严的 NAT 隔离网络中无法 P2P 直连。双端上报 `penetrate_failed` 状态后，服务端自动分配 WebSocket 中转房间。手机端控制指令先发给服务端，服务端安全转发给 PC 端并回传结果，同时主动拦截视频流以保护云服务器带宽。
* 🎬 **场景二：跨端资产的秒级漫游 (数据同步场景)**
  用户在 PC 端新建自动化脚本并存入数据库，落库成功的瞬间，服务端利用长连接向该用户的手机端推送 `sync_notify` 信令。手机端静默拉取增量数据，实现配置无缝漫游。

### 2.2 用户级操作场景 (Human-to-System)

* 🎬 **场景一：免费用户的体验与平台商业变现 (C端使用场景)**
  免费版用户下发指令前唤起广告 SDK，观看完毕后平台前端放行指令；同时穿山甲服务器向速控侠服务端发送 `csj-notify` Webhook，服务端记录广告收益。
* 🎬 **场景二：会员购买与权益即时生效 (C端充值场景)**
  用户在自助后台支付 20 元开通金牌会员。支付回调更新用户 `version_type` 为 `GOLD`。服务端向该用户在线设备广播 `permission.updated` 信令，手机端的灰色高级功能瞬间点亮。
* 🎬 **场景三：设备安全管控 (C端安全场景)**
  用户手机遗失。登录 Web 用户自助后台的【设备管理】点击“强制下线”。服务端在 Redis 中拉黑该设备 Token，并发送死亡信令，遗失的手机瞬间退出登录。
* 🎬 **场景四：运营人员的客服处理 (B端管理场景)**
  运营人员登录 Web 管理总控后台，搜索客诉用户邮箱，核实订单后执行退款并手动修改账号权限，系统将其降级回 `FREE` 免费版状态。

## 三、功能设计

本章节将服务端的庞大业务拆解为四大核心功能模块，详细阐述其业务边界与底层处理逻辑。

### 3.1 三端通信与组网中转功能

这是速控侠的核心命脉，解决设备连通性问题。

* **WebRTC/P2P 信令引擎**：协助手机端与 PC 端交换 SDP 信令，建立直连通道。
* **设备心跳与在线管控**：基于 WebSocket 的 `PING/PONG` 机制，实时监控多端设备在线状态，支持单账号多端“挤占下线”逻辑。
* **兜底中转控制 (底层拦截机制)**：当 P2P 失败触发中转时，开发人员在 `CMD_RELAY` 处理器中必须**严格拦截大体积的二进制流（如视频流帧）**，仅放行 `JSON/Text` 控制指令，严控云端带宽。

### 3.2 统一账号与鉴权中心功能

* **注册登录与 Token 续签**：支持邮箱验证码注册/登录。基于双 Token 机制（Access + Refresh），静默换发新凭证实现长久免密。
* **资产云端同步**：负责跨设备的 RPA 脚本数据持久化与下发流转。

### 3.3 商业化与计费变现功能 (含防刷机制)

承担产品的营收流水、权限计算及防作弊审核。

* **计费与权限升降级逻辑**：
* `FREE` (免费版)：下发 `ad_free = false`，禁止调用看屏与极速回放接口。
* `STANDARD` (普通会员-10元/月)：下发 `ad_free = true`（客户端彻底屏蔽穿山甲 SDK）。
* `GOLD` (金牌会员-20元/月)：下发 `ad_free = true`，拥有所有高阶 API 调用权限。
* **降级流转**：通过 Spring `@Scheduled` 每日凌晨 2:00 扫描用户到期时间，过期强制改回 `FREE` 并推送降级通知；支付回调时利用 Redis 分布式锁 `lock:order:{orderNo}` 防止并发。


* **广告防作弊与回调处理**：
* 拦截穿山甲回调请求的 IP 白名单，验证 Header 中的 `X-Pangle-Signature` 签名，防止第三方伪造请求，处理透传的 `userId` 后记入 `sys_ad_log`。



### 3.4 服务端 Web UI 体系功能

提供 Web 端的多角色操作界面，共分为三个视图：

> **前端架构技术栈拆分建议：**
> * **官网展示页**：注重 SEO 优化与极速首屏加载，建议前端团队使用 **Next.js (React) 或纯静态 HTML** 构建。
> * **用户/总控后台**：重交互的数据管理系统，建议统一合并在一个 **Ant Design Pro V5** 项目中，通过路由权限（如 `ROLE_USER` / `ROLE_ADMIN`）动态控制菜单展示。
>
>

* **官网与展示页 (无需登录)**：展示产品介绍功能亮点；提供 PC/移动端安装包下载链接及应用商店跳转；提供图文并茂的网络连通性指南及使用帮助 FAQ。
* **普通用户后台 (User Portal)**：提供 C 端用户查看自己的基本账户信息、当前账号权限及到期时间；提供**费用信息列表**（数据表格展示订单编号、价格、商品种类、支付状态及交易时间）；提供已绑定的 PC 和手机列表及【踢下线】功能。
* **总控后台 (Admin UI)**：提供 B 端运营人员数据大盘（在线人数/系统负载）；支持邮箱检索用户、手动修改 `version_type` 操作列；展示所有订单流水与广告变现监控大盘；提供 Redis 拉取的在线终端监控列表。

## 四、工程结构与技术规范

### 4.1 核心技术栈版本限定

* **开发环境**：IntelliJ IDEA
* **基础环境**：JDK 21, Spring Boot 3.5.x
* **数据存储**：PostgreSQL 16+, Redis 7.2+
* **ORM 框架**：MyBatis-Plus 3.5.x (配合 Flyway 管理数据库版本)
* **配置与注册中心**：Nacos 2.x
* **鉴权框架**：Spring Security + jjwt (JWT 令牌)

### 4.2 项目代码包结构设计 (Package Structure)

建议采用标准的领域驱动（DDD）或按层划分结构：

```text
com.clawchat.server
 ├── config        // 全局配置：Redis, Nacos, WebSocket, MyBatis-Plus
 ├── security      // 鉴权拦截器：JWT Filter, 密码加密(BCrypt)
 ├── controller    // RESTful API 入口 (前端/移动端/PC端请求)
 ├── websocket     // WebSocket 处理器：信令交换、消息中转、设备保活
 ├── service       // 业务逻辑层：User, Device, Order, Ad回调处理
 ├── mapper        // 数据库操作层 (MyBatis-Plus Mapper)
 ├── entity        // 数据库实体类 (PO)
 ├── dto           // 数据传输对象：Req/Resp, ResPageData, RB统一响应体
 ├── enums         // 全局枚举：VersionTypeEnum(FREE, STD, GOLD), PayStatusEnum
 └── exception     // 全局异常处理：GlobalExceptionHandler

```

## 五、接口与通信协议规范

### 5.1 HTTP RESTful API 规范

所有接口必须返回统一的 `RB<T>` 格式，完美适配 Ant Design Pro 的链路追踪与错误提示规范：

```json
{
  "code": 200,                  // 状态码 (如 200 表示 SUCCESS)
  "msg": "操作成功",               // 提示信息
  "success": true,              // 成功标识布尔值
  "errorCode": "200",           // 字符串格式错误码
  "errorMessage": "操作成功",      // 错误提示详情
  "showType": 2,                // 前端错误展示类型 (对应 Antd Pro 规范)
  "tdata": { ... },             // 单体业务数据负载 (泛型 T)
  "data": {                     // 分页业务数据负载 (对应 ResPageData)
    "list": [ ... ], 
    "current": 1, 
    "pageSize": 10, 
    "total": 100
  },
  "traceId": "550e8400-e29b...",// 链路追踪ID (UUID)
  "host": "server-node-1"       // 处理请求的服务器主机名
}

```

**核心接口定义列表：**

1. **公共展示模块 (`/api/public/**`)**：
* `GET /software-info`：获取软件的下载地址、软件描述及使用帮助信息。


2. **认证模块 (`/api/auth/**`)**：
* `POST /send-code`：发送邮箱验证码。
* `POST /login`：账号/邮箱登录，返回 `{ accessToken, refreshToken }`。


3. **业务模块 (`/api/v1/**` - 需 JWT)**：
* `GET /user/info`：获取当前用户信息及到期时间。
* `GET /user/orders`：获取普通用户的费用订单列表。
* `POST /device/bind`：绑定并上报设备信息。
* `POST /template/sync`：同步 PC 端录制的 RPA 自动化脚本。


4. **商业化模块 (`/api/pay/**` & `/api/ad/**`)**：
* `POST /pay/create-order`：创建订阅订单（10元或20元），返回支付参数。
* `POST /pay/notify`：接收微信/支付宝异步支付回调（需严格验签与幂等处理）。
* `POST /ad/csj-notify`：穿山甲广告服务端 Webhook 回调接收。



### 5.2 WebSocket 实时通信协议规范

这是速控侠的核心命脉，用于设备状态监控、信令交换和指令中转。

* **连接地址**：`ws://server-ip:port/ws/connect?token=jwt_token&deviceId=xxx`
* **消息帧格式 (JSON Text Frame)**：

```json
{
  "type": "SIGNAL_OFFER",    // 消息类型：PING, PONG, SIGNAL_OFFER, SIGNAL_ANSWER, CMD_RELAY, EVENT_PUSH
  "targetDeviceId": "xxx",   // 目标设备ID（如手机发给PC，填PC的设备ID）
  "payload": { ... }         // 具体内容：如 WebRTC 的 SDP，或中转的文字指令
}

```

* **保活机制 (Keep-Alive)**：客户端每 15 秒发送 `PING`，服务端返回 `PONG`。超过 45 秒未收到 PING，服务端主动断开连接，更新数据库中该设备的 `is_online = false`。

## 六、数据库设计 (PostgreSQL 落地版)

数据库开发需使用 Flyway 脚本（如 `V1.0.0__init.sql`）管理。以下为带字段约束的开发级建表语句：

### 6.1 用户核心表

```sql
CREATE TABLE sys_user (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(100) NOT NULL UNIQUE, 
    password VARCHAR(100) NOT NULL, -- BCrypt加密
    version_type VARCHAR(20) NOT NULL DEFAULT 'FREE', -- 枚举: FREE, STANDARD, GOLD
    version_expire_time TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    is_deleted SMALLINT NOT NULL DEFAULT 0
);

```

### 6.2 财务与订单表 (支撑普通用户与总控后台)

```sql
CREATE TABLE sys_order (
    order_no VARCHAR(64) PRIMARY KEY, -- 前端展示的业务订单编号
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,    -- 价格 (10.00 或 20.00)
    product_type VARCHAR(20) NOT NULL,-- 商品种类 (STANDARD_MONTH, GOLD_MONTH)
    pay_status VARCHAR(20) NOT NULL DEFAULT 'PENDING', 
    third_trade_no VARCHAR(100),      
    pay_time TIMESTAMP,               
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_order_userid ON sys_order(user_id);
CREATE INDEX idx_order_paytime ON sys_order(pay_time); -- 加速订单列表按时间倒序查询

```

### 6.3 跨端自动化脚本资产表

```sql
CREATE TABLE ai_task_template (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    shortcut_number VARCHAR(20) NOT NULL,
    script_json_payload TEXT, -- 物理动作序列
    version_timestamp BIGINT NOT NULL, 
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

```

### 6.4 系统辅表 (设备管控、验证码审计、广告日志)

```sql
-- 验证码发送审计表
CREATE TABLE sys_verify_code (
    id BIGSERIAL PRIMARY KEY, 
    target VARCHAR(100) NOT NULL, 
    code VARCHAR(10) NOT NULL,
    business_type VARCHAR(20) NOT NULL, 
    send_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 在线设备管控表 (用于踢下线)
CREATE TABLE sys_device (
    id BIGSERIAL PRIMARY KEY, 
    user_id BIGINT NOT NULL, 
    device_id VARCHAR(100) NOT NULL UNIQUE,
    device_type VARCHAR(20) NOT NULL, 
    is_online BOOLEAN NOT NULL DEFAULT FALSE, 
    last_login_time TIMESTAMP
);

-- 广告收益穿山甲对账表
CREATE TABLE sys_ad_log (
    id BIGSERIAL PRIMARY KEY, 
    user_id BIGINT NOT NULL, 
    trans_id VARCHAR(100) NOT NULL UNIQUE, 
    reward_amount INT, 
    notify_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

```

## 七、部署与运维方案

1. **容器化架构**：推荐采用 **Docker + Docker Compose (或 K8s)** 的容器化部署方案。
2. **网关与静态托管**：使用 **Nginx** 作为统一网关，暴露后端 API 接口（`/api/*`），同时处理 SSL 证书，并将 Ant Design Pro 编译后的 Web 前端（官网、用户后台、管理后台）静态资源进行代理。
3. **中间件托管**：PostgreSQL 与 Redis 建议直接采用云厂商的 RDS 托管服务，保障数据高可用与每日自动备份。
4. **动态扩展**：Nacos 配置中心保障后续将“Web/计费模块”与“设备 WebSocket 信令模块”拆分为独立集群时的平滑过渡。