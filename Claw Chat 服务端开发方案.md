# 速控侠（Claw Chat）服务端开发方案

# 一、 方案概述

速控侠服务端是三端（Android、Windows、Web）协同架构的核心调度与数据中枢。其核心目标是承担账号权限管控、设备会话调度、数据兜底中转、以及商业化（广告/阶梯计费）支撑。本方案基于 **JDK 21 LTS + Spring Boot 3.5.x + PostgreSQL** 构建，利用虚拟线程处理高并发长连接，旨在提供稳定、安全的跨端控制体验。

# 二、 场景描述

* **核心业务场景**：用户通过官网或客户端进行身份认证。登录后，用户的一台 PC 端和一台手机端分别与服务端建立 WebSocket 长连接。
* **典型流程**：用户在 PC 端下发 RPA 控制指令，服务端通过 WebRTC 信令或 WebSocket 兜底中转发往手机端执行。用户可通过管理后台实时监控登录设备 IP、类型，并能远程强制踢出异常会话。

# 三、 功能列表

* **用户账户系统**：支持用户名/密码登录、邮箱绑定、重置密码、个人资料管理。
* **VIP 订阅体系**：分为免费版（看广告）、10元/月（基础版）、20元/月（高级版）的权限分级。
* **设备会话管理**：
* **多端挤占**：严格限制**单账号仅允许 1 台 PC 和 1 台手机**同时在线。
* **会话监控**：展示登录设备的 IP 地址、设备类型、最后活跃时间。
* **远程登出**：用户可在后台手动断开指定设备的登录状态。


* **指令中转与保活**：
* **WebRTC 信令中转**：辅助 P2P 穿透。
* **WS 指令转发**：穿透失败时的 JSON 指令兜底转发。
* **心跳保活**：每 60 秒一次的双向心跳检测。


* **管理后台**：基于 Ant Design Pro，提供用户审计、订单流水、广告回调（穿山甲 Webhook）对账。

# 四、 功能设计

* **单端挤占逻辑**：服务端在用户登录时校验当前在线会话。若已有同类型设备（如已有一台 PC 在线，现在又登录一台 PC），服务端通过 Redis 获取旧设备会话并发送 `FORCE_LOGOUT` 信号，随后断开旧连接。
* **心跳与连接管理**：**客户端每分钟（60s）发送一次 PING**。服务端收到后即刻回 PONG。若连续 150s（约 2.5 次心跳周期）未收到信号，服务端自动清理该会话并更新数据库状态。

# 五、 工程结构与技术规范

* **技术选型**：JDK 21 (Virtual Threads), Spring Boot 3.5, Redis (Pub/Sub + Cache), PostgreSQL，Ant Design Pro。
* **编码规范**：所有 API 遵循 RESTful 风格；WebSocket 处理层必须使用虚拟线程以避免长连接阻塞系统线程。

# 六、 接口与通信协议规范

### 6.1 接口格式

严格执行以下 JSON 结构：

```json
{
  "code": 200,                  // 状态码 (如 200 表示 SUCCESS)
  "msg": "操作成功",               // 提示信息
  "success": true,               // 成功标识
  "tdata": { ... },              // 单体业务负载
  "data": {                      // 分页业务负载
    "list": [ ... ], "current": 1, "pageSize": 10, "total": 100
  },
  "traceId": "550e8400-e29b..."  // 链路追踪ID
}

```

### 6.2 通信规则与权限校验

* **通信协议**：WebSocket + JWT 鉴权（连接时通过 URL 参数传递 Token）。
* **交互方式**：双向异步 JSON 帧。下发控制指令前，服务端实时检索 Redis 中的 VIP 权限位，若权限失效则拦截指令。

# 七、 数据库设计

### 7.1 sys_user (用户核心表)

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint | 主键 |
| username | varchar(50) | 用户名（唯一） |
| password_hash | varchar(255) | 登录密码密文 |
| email | varchar(100) | 邮箱（用于重置密码） |
| vip_level | int | 0:免费, 1:10元, 2:20元 |
| expire_time | timestamp | 订阅到期时间 |

### 7.2 sys_device (登录设备会话表)

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint | 主键 |
| user_id | bigint | 关联用户 ID |
| device_id | varchar(100) | 硬件唯一标识/机器码 |
| device_type | varchar(20) | PC / MOBILE |
| last_ip | varchar(50) | 登录时的 IP 地址 |
| is_online | boolean | 是否在线 |
| last_heartbeat | timestamp | 最后心跳活跃时间 |

### 7.3 sys_order (支付与广告流水表)

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | bigint | 主键 |
| user_id | bigint | 用户 ID |
| order_no | varchar(100) | 支付单号或穿山甲回调 ID |
| amount | decimal | 金额（广告为 0） |
| status | int | 0:处理中, 1:成功, 2:失败 |

# 八、 部署与运维方案

* **环境准备**：基于 Docker 容器化部署；PostgreSQL 开启索引优化；Redis 启用持久化。
* **运维监控**：利用 Prometheus 监控虚拟线程活跃数与每分钟 PING/PONG 成功率。
* **链路追踪**：通过 `traceId` 贯穿 Web 接口日志与 WebSocket 指令日志，方便排查信令丢失问题。

