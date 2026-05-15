# 实时数据推送基础设施（Realtime Push Infrastructure）

> 本文讨论的是应用级实时数据推送（SSE / WebSocket / Pub-Sub），不是移动端通知推送（FCM/APNs）。两者解决不同问题：前者管"用户在线时如何高质量实时传输数据"，后者管"用户离线时如何唤醒设备"。

## 推荐结论

**托管服务首选：Ably。**

核心理由（3 条，每条有硬依据）：

1. **在主流托管服务中，唯一在协议层面设计了 exactly-once 投递机制**。Pusher 和 PubNub 均无此设计。对于聊天、交易、协同编辑等场景，这是决定性差异。注意：Ably 自己的文档也承认实践中是"mostly-once"——正常情况下 exactly-once，故障时可能降级为 at-most-once 或 at-least-once。"唯一"的范围是 Ably/Pusher/PubNub/Firebase 对比，非穷举全球所有服务。（依据：Ably idempotency 文档原文 + websocket.org 对比，Pusher/PubNub 官方文档均无此保证。）

2. **可靠性数据最好**。24 个月公开事故对比：客户受影响总分钟 Ably 1,711 vs Pusher 9,661（5.6x）vs PubNub 7,903（4.6x）。（依据：各平台公开状态页收集 133 个事故、其中 102 个被分析。注意：分析者为 Ably 联合创始人，但数据源公开可验证。）

3. **全球多区域主动-主动架构 vs 竞品单集群/app**。Ably 700+ PoP、多区域 P2P 复制、自动故障转移。Pusher 虽有 9 个区域集群可选，但每个 app 只能部署在单个集群中，集群间不互联、不复制（Pusher 官方博客原文："Our clusters are not connected to each other, and this is by design"）。全球用户访问同一 app 必须路由到同一集群，无自动地理故障转移。

**何时不选 Ably：**
- 只需单向推送、不需要 Ably 级别的协议保证 → SSE 自建（几乎零成本，SSE 自身有 Last-Event-ID 重连机制，实现得当也可做到基本不丢）
- 需要服务端逻辑与连接在同一位置 → Cloudflare Durable Objects
- 预算极度敏感、工程能力强 → Centrifugo（开源 Go，自托管）

---

## 问题定义

当服务端有新数据需要**立即**送达客户端时，传统的 HTTP 请求-响应模型（pull）无法满足需求——客户端不知道什么时候该去拉。实时推送（push）架构让服务端主动将数据送达客户端，无需客户端轮询。

典型场景：聊天消息、协同编辑、实时仪表盘、AI 流式输出、体育赛事比分、在线游戏状态同步、IoT 遥测数据。

## 核心难题

在**单台服务器**上做实时推送很简单——开一个 WebSocket，发消息。难的是在**分布式环境、全球用户、生产级可靠性**下做对：

1. **有状态连接与无状态基础设施的矛盾**：WebSocket 是长连接，绑定到特定服务器进程。而现代基础设施（CDN、负载均衡、Serverless）全部假设无状态。
2. **断线恢复**：移动网络频繁断连，断线期间的消息谁存、重连后怎么补发、怎么不重复。
3. **消息有序**：多个发布者、多个区域，消息到达顺序可能不同。
4. **水平扩展**：连接状态要在节点间迁移，channel 数据要在节点间分布，不能丢。
5. **全球低延迟**：需要多区域部署，但多区域又加剧了上述所有问题。

---

## Ably

### 一句话定位

全球分布式的托管实时消息基础设施。通过原生 WebSocket（降级支持 SSE、MQTT、HTTP 长轮询）在服务端和客户端之间传递消息，提供协议级的消息有序、不丢、不重复保证。

### 解决什么问题

Ably 的核心价值是：开发者不需要自己构建分布式实时基础设施，即可获得消息有序、不丢、不重复、全球低延迟的实时消息传输层。Ably 自己的类比是"像依赖 TCP/IP 一样依赖实时传输层"（来源：Ably 联合创始人 websocket.org 文章）。

对比自建的工程成本：据 Ably 联合创始人在 websocket.org 撰文称，HubSpot 评估自建等价系统需要约 20% 的工程团队。据 Ably 官方案例页称，HubSpot 选择 Ably 后节省了 60% 前期 CAPEX 和每年 $300,000 运维成本。（注：这两个数据均来自 Ably 侧的营销材料，未找到 HubSpot 独立发布的验证。）

### 如何解决：架构四层

Ably 的架构由四层组成，自底向上：

#### Gossip Layer（集群感知层）

采用受 Scuttlebutt 启发的 gossip 协议（Ably 架构文档原文："Scuttlebutt-inspired"），在所有区域的所有节点间维护一份叫 netmap 的集群拓扑数据结构。每个节点通过周期性心跳交换状态，确保"谁活着谁挂了"达成最终一致。没有单点协调者——任何节点都能通过 gossip 获得全局视图。

这一层是所有上层容错机制的基础。如果节点连集群状态都达不成共识，故障转移就无法工作。

#### Routing Layer（路由层）

通过 DNS 延迟路由将客户端连接到物理距离最近的数据中心。多 DNS 提供商冗余，自动将不健康区域从解析中摘除。客户端 SDK 连接失败时自动尝试最多 5 个备用端点（1 个主端点 + 5 个备用 = 6 个全球分布端点，两处文档表述一致）。

#### Frontend Layer（前端连接层，无状态）

维护与客户端的实际连接（WebSocket/SSE/MQTT/Comet），处理 REST 请求，执行认证和限速。这一层无状态——节点挂了，客户端重连到另一个节点即可，容错通过冗余 + 自动替换实现。

所有 channel 流量**复用一条 WebSocket 连接**（Connection Multiplexing），减少带宽、电量消耗和浏览器连接数占用。

#### Core Layer（核心处理层，有状态）

这是最关键的一层。通过一致性哈希将 channel 分配到 Core 节点，每个 channel 在区域内有 Primary + Secondary 两个副本（不同可用区）。

消息处理的关键路径：

```
Publisher SDK → Frontend Node → Core Node (Primary)
                                    │
                                    ↓
                   Primary 存储 + 幂等性检查（单个原子操作）
                   检查消息 ID 是否已存在，不存在则存入 Redis
                                    │
                                    ↓
                   复制到 Secondary（另一个可用区，也存入 Redis）
                                    │
                                    ↓
                             双副本都存好
                                    ↓
                        发送 ACK 给 Publisher
                                    ↓
               ┌────────────┼────────────┐
               ↓            ↓            ↓
          扇出给本区域     P2P 复制到      写入 Cassandra
          所有订阅者      其他活跃区域    （如果开了持久化）
```

> 注：幂等检查与 Primary 存储是原子操作（Ably idempotency 文档原文："persisted at the primary location and checked for uniqueness in a single atomic operation"），不是独立步骤。

**ACK 的含义**：只有当消息在两个不同可用区都存储之后（底层是 Redis 内存存储，通过多副本冗余而非磁盘写入实现 durability），才向发布者发送 ACK。收到 ACK = Ably 保证该消息会被投递给所有订阅者。但端到端的 exactly-once 还依赖订阅侧应用层正确处理消息（Ably idempotency 文档原文："the client processing ensures that any error or exception in processing the message is handled without loss or duplication"）。

**故障恢复**：Core 节点宕机 → Gossip 层检测（心跳丢失）→ 更新 netmap → 一致性哈希计算新归属 → Secondary 接管 → 8 秒内完成迁移。

### 三大协议级保证

这些保证内置在 Ably 协议中，使用官方 SDK 时自动生效，开发者不需要额外代码：

#### 1. Exactly-Once 投递

**发布侧（幂等发布）**：每条消息有唯一 ID（SDK 自动生成或开发者指定）。消息到达 Core 节点时，检查该 ID 是否已存在——检查与存储是原子操作，杜绝并发窗口。跨区域复制时每个区域也做同样的 ID 检查。结果：无论 Publisher 重试多少次、重试到哪个区域，消息只处理一次。

**订阅侧（Serial Number 恢复）**：每条消息携带递增 serial number。客户端 SDK 记录每个 channel 最后收到的 serial。断线重连时，SDK 告诉服务端"我最后收到 serial=N"，服务端从 N+1 开始精确重放缓存消息（2 分钟窗口）。

#### 2. 消息有序

同一个 Publisher 在同一个 WebSocket 上发布的消息，全球所有订阅者看到相同顺序（Ably message ordering 文档原文："subscribers on that channel anywhere in the world will receive those messages in the same order they were originally published"）。实现方式是每条消息在 Realtime 连接上携带唯一递增 serial number，结合区域内的消息到达顺序保证分发一致性。

设计取舍：不同区域的不同 Publisher 之间不保证全局顺序——这是有意为之，为了让不同区域的客户端能并发高吞吐发布，避免跨区域锁带来的延迟。每个 Publisher 保持自己的因果一致性（causal consistency）。

Ably 定义了两套顺序：Realtime Order（每个区域的实时到达顺序）和 Canonical Global Order/CGO（全局一致的规范顺序，用于 History API）。

#### 3. 断线自动恢复

SDK 自动管理连接状态机（initialized → connecting → connected → disconnected → suspended → closed / failed）。断线后每 15 秒自动重连，2 分钟内重连成功则无感知恢复（消息补发 + 有序），超过 2 分钟连接状态释放，需通过 History API 补数据。

页面刷新场景通过 recover 机制处理：beforeunload 时将 recoveryKey 存入 sessionStorage，新实例加载后可恢复上一个连接的状态。

### 前端使用方式

```js
import Ably from 'ably';

// 建立连接（一个实例 = 一条 WebSocket，所有 channel 复用）
const ably = new Ably.Realtime({
  authUrl: '/api/ably-token',  // 生产环境用 token auth，不要在前端暴露 API Key
  clientId: 'user-123'
});
await ably.connection.once('connected');

// 订阅
const channel = ably.channels.get('chat-room-42');
await channel.subscribe((message) => {
  console.log(message.data);
});

// 发布
await channel.publish('msg', 'Hello World');

// 不用了关掉
ably.close();
```

React 有官方 hooks（`useChannel`、`usePresence`），Next.js 有 `AblyProvider`。25+ 客户端 SDK（Ably 定价页标注；websocket.org 文章称 30+，取保守值）。

连接管理、断线恢复、消息去重、有序投递全部由 SDK + 服务端协作完成，开发者只需关注 subscribe 回调里的业务逻辑。

### 关键指标

| 指标 | 数值 | 来源 |
|------|------|------|
| 数据中心内 p99 往返延迟 | < 30ms | Ably 官方 |
| 全球 PoP p99 往返延迟 | < 65ms（限接收 ≥1% 全球流量的 PoP） | Ably Four Pillars 页面 |
| 所有 PoP 平均往返延迟 | < 99ms（含偏远节点） | Ably Four Pillars 页面 |
| 消息可用性（实例故障） | 99.999999%（8 个 9） | Ably 架构文档 |
| 持久化数据可用性 | 99.99999999%（10 个 9） | Ably Four Pillars 页面 |
| SLA | 99.999% | Ably 商业承诺 |
| 实例故障迁移时间 | 8 秒内 | Ably 架构文档 |
| 全球边缘节点 | 700+ PoP | Ably 官方 |
| 区域（regions） | 11 个 | Ably 架构文档、定价页 |
| 物理数据中心 | 15-17 个（Four Pillars 页写 17，博客写 15。一个区域可含多个数据中心，与 11 regions 不矛盾） | Ably 官方 |
| 单 Channel 吞吐上限 | 200 msg/s，13 MiB/s | Ably 官方 |

24 个月公开事故数据对比（来源：各平台公开状态页，分析者为 Ably 联合创始人 Matthew O'Riordan）：

| 指标 | Ably | PubNub | Pusher |
|------|------|--------|--------|
| 重大事故数 | 7 | 26（3.7x） | 17（2.4x） |
| 客户受影响总分钟 | 1,711 | 7,903（4.6x） | 9,661（5.6x） |

### 适用场景

- 消息不能丢、不能重复（聊天、交易、协同）
- 用户全球分布
- 不想自建实时基础设施
- 需要消息有序保证

### 不适用场景

- 纯单向推送且不需要 Ably 级别的协议保证（SSE 自建更简单更便宜）
- 只是 AI 流式输出（标准 SSE 就够了）
- 预算极敏感且工程能力强 → Cloudflare DO 或 Centrifugo 自建（成本显著更低，但具体倍数取决于消息量和用法）
- 需要服务端业务逻辑与连接在同一位置 → Cloudflare Durable Objects
- 数据主权要求 EU 管辖 → Ably 是英国公司，基础设施在 AWS（US Cloud Act）

### 诚实的局限

1. **Exactly-once 的完整保证依赖 Realtime SDK 的持久连接**——发布侧的幂等性通过 REST API 也可实现（消息 ID 去重），但订阅侧的 serial number 断线恢复需要持久连接。通过 Webhook/AMQP 等外部集成接入时降级为 at-least-once。
2. **连接恢复窗口 2 分钟**——超过 2 分钟断线，连接状态释放，需应用层自行补数据。
3. **跨区域不同 Publisher 不保证全局强序**——设计取舍，不是缺陷。
4. **极端断线恢复场景可能乱序**——Ably 消息排序文档原文承认：如果断线期间维护连接状态的服务器恰好也被回收（"rare situation"），重放消息可能乱序。
5. **单 Channel 有吞吐上限**（200 msg/s），高吞吐需做 Channel 分片。
6. **消费计费模型**——成本随用量变化，难以精确预测，高量级需联系销售。
7. **关键指标均为自报**——可用性（8 个 9）等数字来自 Ably 自身，无独立第三方审计。延迟数据在不同 Ably 材料中表述不一致（websocket.org 文章称 "6.5ms median API latency"，博客称 "99th percentile transmit latency of 6.5ms"，Four Pillars 页面写 < 30ms / < 65ms），本文采用 Four Pillars 页面的分层数据。事故数据基于公开状态页，可验证但分析者有利益关联。物理数据中心数量在 Ably 不同页面间存在 15/17 的差异（但 11 regions 在多处一致）。
8. **Exactly-once 是设计目标而非绝对保证**——Ably idempotency 文档原文承认："In practice, many distributed systems can truly guarantee only 'mostly-once' delivery... when failures occur, some messages may revert to at-most-once or at-least-once semantics."

### 定价

免费层：6M 消息/月 + 200 并发连接 + 200 并发 channel。
付费：消费计费（按消息数 + 连接分钟 + channel 分钟），有量级折扣。具体费率需查官网或联系销售。

### 生产案例

- **HubSpot**：跨 120 国的企业 live chat，26.8 万+企业客户，每月 50 万+关键业务对话，每天 5 亿个 channel。
- **NASCAR**：赛车遥测数据实时推送，每场比赛 1.3TB 数据（120 更新/秒降采样到 2 次/秒广播）。NASCAR 称其 Drive 平台面向 8000 万全球赛车粉丝群体（注：这是 NASCAR 总粉丝数，非 Ably 同时在线数）。
