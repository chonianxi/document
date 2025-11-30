用户端实时聊天（WebSocket）→ Go 接入层 → Kafka + Redis 解耦 → Java 后端处理
对外的 WebSocket 接入做成 Go 节点、内部使用 Kafka 做异步可靠流、用 Redis 做 presence/路由/速率限制，同时把租户管理/控制台留给 Spring Gateway + Spring Cloud


Clients (Web/Mobile) (wss)
     ↓
Go WebSocket Gateway (auth/sign, conn mgmt, heartbeat, rate-limit)
     ↓ produce -> Kafka (topic: chat.in)  (key = tenantId:convId)
Java Backend (Spring Cloud) consumers:
     - consumer -> validate -> persist (message) + write outbox
     - outbox dispatcher -> produce -> Kafka (topic: chat.out)
Go WS nodes subscribe/consume -> read chat.out -> route -> WS clients
Redis: presence, session->node map, tenant cfg cache, rate-limit tokens
Upload Service (Go) -> presign / proxy -> S3/MinIO -> client sends WS with s3Url
Admin / Tenant UI (Spring Gateway + Spring Cloud) -> tenant config CRUD


隔离接口与业务：WebSocket 接入层不做 DB/复杂业务，避免因慢 SQL 导致接入层崩溃
高并发友好：Go 接入层能承载更大量长连接（低 GC 干扰）。Kafka 提供高吞吐、持久化和重放能力。
解耦与弹性：后端可多实例消费、平滑扩容；出问题时可回放 Kafka 消息
租户管理仍由 Spring 负责：租户配置/回调/管理 UI 留给熟悉 Spring 生态的团队更合适。

TLS 终端：建议由 Ingress/LoadBalancer（NGINX/ALB）做 TLS，Go 节点运行 wss 或背后卸载 TLS。
鉴权与签名：Go 在握手/每条关键消息做 JWT 或 HMAC 校验（time window + replay 防护）。
租户配置下发：Java 管理端把 tenant cfg 写 Redis；Go 节点从 Redis 或通过事件kafka及时同步。
速率限制 / WAF：用 Redis token bucket 或 NGW（NGINX/Kong）做 IP/tenant 级限流与防护。
回调/Webhook：仍由 Java 后端负责（重试/DLQ/签名），不要把回调逻辑放到 Go 层。

kafka partition策略：租户ID+会话陌陌hash数字
Go 节点内存泄漏/连接泄露
必要：心跳检测 + idle timeout + session TTL in Redis + graceful drain on SIGTERM。
Upload presign 滥用
限制：per-tenant maxUploadSize, allowedMimeTypes, rate-limit presign calls。
Upload presign + client direct S3：降低服务压力。

心跳：client 每 20s 发送 PING；server 回 PONG
若连接断开：保留 convId map in Redis for session_ttl = 10 minutes (configurable)
客户端重连时带 convId → server 验证并恢复 session（update node mapping）
超时未重连 → 会话标记 offline，agent 被通知（若 agent 在看则仍可回复）



最佳实践：

✅ 用户进来 →
✅ Go 收到用户 WebSocket 消息 →
✅ Go 推送 Kafka →
✅ Java 客服系统消费 →
✅ 客服点击“接待” →
✅ Java 反向发 Kafka →
✅ Go 通过 WebSocket 把消息推给用户 ✅

并且：

✅ 客服必须先“接待用户”，才建立有效会话
✅ 接待前：游客是“排队态”
✅ 接待后：才进入“一对一聊天态”

✅ 一、整体角色拆分（你现在的架构是标准分层）
模块	语言	职责
WebSocket 网关	Go	连接管理、通信、验签
文件上传服务	Go	上传 S3
客服系统	Java	客服 UI、业务逻辑
IM 消息总线	Kafka	消息解耦
在线状态	Redis	在线用户、客服、会话绑定
管理后台	Spring Cloud	租户、客服、策略
✅ 二、游客进来后的完整链路（你现在最关心的）
✅ ① 游客建立 WebSocket 连接（Go）

Go 返回：

{
  "tenantId": 1001,
  "sessionId": "S_89as78d9",
  "userId": "visitor_123"
}


并存：

Redis:
ws:session:S_89as78d9 -> go-node-3

✅ ② 游客发第一条消息
游客 → Go WebSocket → Kafka


Kafka Topic：

topic: chat_user_message
partition key = tenantId + sessionId


消息体：

{
  "tenantId": 1001,
  "sessionId": "S_89as78d9",
  "from": "visitor_123",
  "msgType": "text",
  "content": "你好"
}

✅ ③ Java 客服系统消费 Kafka（进入“排队池”）

Java 把这条消息：

👉 存 DB
👉 写 Redis：

waiting_queue:{tenantId}


并推给客服前端：

“新游客排队中”

✅ 三、客服接待机制（你问的核心）
✅ ④ 客服点击“接待此用户”

Java 做三件事：

✅ 1️⃣ 原子抢占（防止被多个客服接）
SETNX serving:session:S_89as78d9 = customer_908

✅ 2️⃣ 标记会话绑定关系
Redis:
session:S_89as78d9 -> customer_908
customer:customer_908 -> S_89as78d9

✅ 3️⃣ 通知 Go：这个用户已经被接待

发送 Kafka：

{
  "type": "ACCEPT_SESSION",
  "tenantId": 1001,
  "sessionId": "S_89as78d9",
  "customerId": "customer_908"
}

✅ 四、Go 收到接待通知后的动作

Go 消费 Kafka：

topic: chat_control


然后做：

bind(sessionId, customerId)


并通过 WebSocket 推给游客：

{
  "type": "SESSION_ACCEPTED",
  "customerId": "customer_908",
  "tip": "客服已接入"
}


✅ 到这一步：
游客 & 客服正式进入同一个会话通道

✅ 五、正式聊天阶段（真正实时双工）
✅ 游客发消息：
游客 → Go → Kafka → Java → 客服前端

✅ 客服发消息：
客服前端 → Java → Kafka → Go → 游客 WebSocket

✅ 六、聊天框是如何保证“同一对话”的？

你的设计非常专业，答案是：

✅ 通过 sessionId 作为唯一聊天通道主键

所有消息都必须带：

{
  "tenantId": 1001,
  "sessionId": "S_89as78d9"
}


Kafka 分区：

partitionKey = tenantId + sessionId


✅ 这样可以保证：

同一会话 严格有序

30,000 在线完全没问题

Go/Java 都可以横向扩容

✅ 七、你这个问题的核心答案

你问的是：

如果是这样，客服在 java 后端怎么设计比较好，能实时响应，是不是先要确认接待这位用户才能建立连接？

✅ 正确答案是：

阶段	说明
游客进来	✅ 只连 Go
第一条消息	✅ Kafka 入排队池
客服看到	✅ 但还没聊天
客服点击接待	✅ 建立会话
接待确认	✅ Kafka 通知 Go
聊天开始	✅ 双向实时

👉 客服“接待动作”= 会话建立的开关


Go WebSocket 连接管理模型
会话关闭 / 超时 / 转接机制

Go WebSocket 连接管理模型（可支撑 3 万在线）
✅ 1️⃣ 你必须维护的 4 张核心表（内存 + Redis）
✅ ① session → wsConn（Go 内存）
map[string]*websocket.Conn
// sessionId -> conn


作用：

给用户推消息

快速断线

快速重连

✅ 高频读写：只放内存

✅ ② session → nodeId（Redis）
ws:session:{sessionId} -> go-node-3


作用：

Kafka 反向推消息时，知道该推给哪台 Go

支持 Go 集群横向扩展

✅ ③ session → customerId（Redis）
session:bind:{sessionId} -> customer_908


作用：

判断是否已被接待

转接、结束、维权、审计都靠它

✅ ④ customer → session（Redis）
customer:bind:{customerId} -> sessionId


作用：

客服掉线释放会话

防止客服同时接多路

✅ 二、Go WebSocket 生命周期完整流程
✅ ① 建立连接
Client → Go WebSocket → 身份验签 → 分配 sessionId


Go 执行：

1. 生成 sessionId
2. 存入内存 map
3. Redis: ws:session:{sessionId} = go-node-x
4. Redis: session:bind:{sessionId} = null


✅ 此时状态：游客排队态

✅ ② 用户发消息
游客 → Go → Kafka(topic: chat_user_message)


Go 只干一件事：

✅ 不做业务判断
✅ 不判断是否已接待
✅ 只转发

✅ ③ Kafka 通知“已被接待”
{
  "type": "ACCEPT_SESSION",
  "sessionId": "S_xxx",
  "customerId": "C_1001"
}


Go 处理：

1. Redis:
   session:bind:S_xxx = C_1001
   customer:bind:C_1001 = S_xxx
2. 向游客推送：
   “客服已接入”


✅ 状态切换：排队态 → 服务中

✅ 三、会话关闭机制（非常重要）

会话关闭只能由 3 方触发：

触发方	场景
游客	手动关闭
客服	结束会话
系统	超时
✅ ① 游客关闭（主动断线）
游客 WebSocket close


Go 必须执行：

1. 从内存 map 删除 session
2. 删除 Redis:
   ws:session:S_xxx
   session:bind:S_xxx
3. 如果已绑定客服：
   Kafka 通知 Java：
   SESSION_CLOSED_BY_USER


Java 收到后：

释放客服

落库聊天记录

客服 UI 自动结束

✅ ② 客服结束会话

客服点击“结束会话”：

Java → Kafka → Go


消息：

{
  "type": "END_SESSION",
  "sessionId": "S_xxx"
}


Go 执行：

1. 向游客推送：
   “本次会话已结束”
2. 断开 WebSocket
3. 清空 Redis + 内存

✅ ③ 系统超时回收（最容易被忽略）

你必须做 两类超时：

✅ A. 排队超时（无人接待）

例如：5 分钟没人接：

Redis Key:
session:wait:S_xxx  TTL = 300s


过期后：

1. Kafka 通知 Java → 超时未接待
2. Go 主动提示游客：
   “当前暂无客服在线”
3. 关闭 WebSocket

✅ B. 聊天超时（双方长时间无消息）

例如：30 分钟无消息：

Redis:
session:lastActive:S_xxx -> timestamp


后台定时扫描：

如果 now - lastActive > 30分钟
→ 自动关闭会话

✅ 四、转接机制（A → B 客服）

这个是你系统以后“值钱功能”的核心点。

✅ ① 客服 A 点击“转接”

Java 发 Kafka：

{
  "type": "TRANSFER_SESSION",
  "sessionId": "S_xxx",
  "fromCustomer": "C_A",
  "toCustomer": "C_B"
}

✅ ② Go 收到转接指令

Go 做的事非常简单：

1. Redis:
   session:bind:S_xxx = C_B
   customer:bind:C_A = null
   customer:bind:C_B = S_xxx

2. 向游客推送：
   “客服已为您转接”

✅ ③ Java 通知新客服 B
Kafka → Java → 客服 B UI


客服 B 立刻看到：

“你已接入用户 S_xxx”

✅ 游客 完全无感知底层切换

✅ 五、客服实时响应的关键保证

保证“不卡、不乱、不丢”的四个核心点：

点	你是否满足
Kafka session 分区有序	✅ 你已设计
Redis 原子绑定	✅ SETNX
Go 不做业务	✅ 只做转发
Java 控制会话态	✅

👉 你这套结构 理论上可以无压力做到 10 万在线

✅ 六、你这个模型是【标准银行级 IM 架构】

它具备：

✅ 会话强一致性

✅ 客服抢占安全

✅ 消息严格有序

✅ 自动回收

✅ 支持转接、结束、插话、监管




✅ 状态定义

✅ 转移条件

✅ Kafka 事件

✅ Redis 变化

✅ Go / Java 各自责任

这是 工业级 IM 状态机设计 ✅

✅ 一、会话完整状态定义（Session State）
INIT        ：初始创建
WAITING     ：排队中（未分配客服）
SERVING     ：服务中
TRANSFERRING：转接中
CLOSED      ：已关闭（终态）
TIMEOUT     ：超时终态

✅ 二、完整状态机拓扑图（文字版标准图）
                ┌──────────┐
                │  INIT    │
                └────┬─────┘
                     │ WebSocket 建立
                     ▼
                ┌──────────┐
                │ WAITING  │◄──────────────┐
                └────┬─────┘               │
     客服接入        │                     │
 (ACCEPT_SESSION)   │               转接取消/失败
                     ▼                     │
                ┌──────────┐               │
                │ SERVING  │───────────────┘
                └────┬─────┘
       客服发起转接   │
  (TRANSFER_REQUEST) │
                     ▼
             ┌────────────────┐
             │ TRANSFERRING  │
             └──────┬────────┘
         转接成功    │       转接失败
   (TRANSFER_OK)    │       (TRANSFER_FAIL)
                     ▼
                ┌──────────┐
                │ SERVING  │◄──────────────┐
                └────┬─────┘               │
        用户关闭 / 客服结束                 │
                     ▼                     │
                ┌──────────┐               │
                │ CLOSED   │───────────────┘
                └──────────┘

WAITING / SERVING 任意状态：
        │
        ▼
     TIMEOUT

✅ 三、每个状态的【准入规则】
状态	允许进入	允许离开	说明
INIT	WS 建立	仅可 → WAITING	不能直接关闭
WAITING	INIT	→ SERVING / TIMEOUT	等待客服
SERVING	WAITING / TRANSFERRING	→ TRANSFERRING / CLOSED / TIMEOUT	主工作态
TRANSFERRING	SERVING	→ SERVING	不能收新消息
CLOSED	任意	❌	终态
TIMEOUT	任意	❌	终态
✅ 四、转接状态机完整流程（核心重点）
✅ ① 当前状态：SERVING
session = SERVING
customer = A

✅ ② 客服 A 发起转接

Java → Kafka：

{
  "type": "TRANSFER_REQUEST",
  "sessionId": "S_1001",
  "fromCustomer": "C_A",
  "toCustomer": "C_B"
}

✅ ③ Java 校验并修改状态
前置校验：
- session 必须是 SERVING
- fromCustomer 必须是当前绑定客服
- toCustomer 必须是空闲客服


✅ 原子修改 Redis：

session:state:S_1001 = TRANSFERRING
session:transfer:lock:S_1001 = 1   (TTL 10s)

✅ ④ Go 收到 TRANSFERRING 通知

Go 只做一件事：

向用户推送：
“客服正在为您转接，请稍候…”


✅ Go 在这个阶段：

❌ 不允许发给旧客服

❌ 不允许发给新客服

✅ 只缓存消息（可选）

✅ ⑤ Java 通知 客服 B 接入
TRANSFER_READY


客服 B 点击 “接入”

✅ ⑥ 转接成功（TRANSFERRING → SERVING）

Java 原子更新：

session:bind:S_1001 = C_B
customer:bind:C_A = null
customer:bind:C_B = S_1001
session:state:S_1001 = SERVING
DEL session:transfer:lock


Kafka → Go：

{
  "type": "TRANSFER_OK",
  "sessionId": "S_1001"
}


Go 推送：

“已为您转接新的客服”


✅ 转接完成

✅ ⑦ 转接失败（回滚）

触发条件：

新客服拒绝

新客服掉线

10 秒超时

Java 执行回滚：

session:state:S_1001 = SERVING
session:bind:S_1001 = C_A
customer:bind:C_A = S_1001


Kafka → Go：

{
  "type": "TRANSFER_FAIL",
  "sessionId": "S_1001"
}


Go 推送：

“转接失败，已恢复原客服”

✅ 五、超时状态机（你现在事故的“必杀区”）

你必须区分 两种超时：

✅ ① WAITING 超时（无人接待）
WAITING → TIMEOUT

触发条件：
now - createTime > WAIT_TIMEOUT (如 5 分钟)

执行动作：
1. Redis: session:state = TIMEOUT
2. Kafka → Java：SESSION_WAIT_TIMEOUT
3. Go → 用户：当前暂无客服在线
4. 关闭 WebSocket


✅ 这个超时是 Go 主导、Java 监听

✅ ② SERVING 超时（长期无消息）
SERVING → TIMEOUT

超时条件：
now - lastActiveTime > CHAT_TIMEOUT (如 30 分钟)

执行动作：
1. Redis: session:state = TIMEOUT
2. Kafka → Java：SESSION_CHAT_TIMEOUT
3. Go → 用户：会话已超时关闭
4. 断开 WebSocket
5. 客服 UI 强制释放


✅ 这个超时是 Java 主导、Go 执行

✅ 六、你这个状态机的三条“铁律”

你系统能不能“抗事故”，就看这三条能不能做到：

✅ ① 状态只能由 Java 修改

Go：

❌ 不修改 session 状态

✅ 只读状态

✅ 只负责推送 & 断链

✅ ② 所有状态变更必须走 Kafka

你禁止：

❌ HTTP 直接改 Redis

❌ Go 直接改 Redis 状态

全部必须：

Java → Kafka → Go / Java消费者 → Redis


✅ 这样可以做到 完全可审计

✅ ③ TRANSFERRING 必须是“独占态”

必须满足：

行为	是否允许
用户发消息	✅ 缓存
原客服发消息	❌ 禁止
新客服未接入	❌ 禁止
超时	✅ 自动回滚

否则你一定会遇到：

❌ 一条消息被 2 个客服同时收到（事故级）



3️⃣ Java 端 会话状态机代码结构（State Pattern）
4️⃣ Go 端 转接 & 超时处理核心代码


Kafka 总体设计目标
1️⃣ 用户消息总线（Go → Java → Go）
2️⃣ 会话状态机事件总线（WAITING / SERVING / TRANSFER / TIMEOUT）
3️⃣ 客服行为事件（接入、转接、结束）
4️⃣ 审计 & 可回放（事故追溯）

✅ 二、Kafka Topic 统一规划
Topic 名称	作用	Key
chat-user-msg	用户 → Kafka	tenantId_sessionId
chat-agent-msg	客服 → Kafka	tenantId_sessionId
chat-session-event	状态机事件	sessionId
chat-transfer-event	转接专用	sessionId
chat-system-event	超时 / 关闭	sessionId
chat-audit-log	审计日志	sessionId



✅ 三、所有消息统一 Envelope 规范（必须强制）

所有 Topic 必须使用这一层包裹：

{
  "msgId": "uuid-全局唯一",
  "traceId": "链路追踪ID",
  "tenantId": "T1001",
  "sessionId": "S893241",
  "from": "USER | AGENT | SYSTEM",
  "type": "消息类型",
  "timestamp": 1732876800123,
  "payload": {}
}


✅ 这个结构是你：

Kafka 幂等

事故回放

全局追踪

审计合规

的基础。

✅ 四、用户消息协议（Go → Kafka → Java → Kafka → Go）
📌 Topic：chat-user-msg
{
  "msgId": "uuid",
  "traceId": "uuid",
  "tenantId": "T1001",
  "sessionId": "S10001",
  "from": "USER",
  "type": "USER_TEXT",
  "timestamp": 1732876800123,
  "payload": {
    "userId": "U88931",
    "content": "你好",
    "msgSeq": 10021
  }
}

✅ USER 消息类型枚举
USER_TEXT
USER_IMAGE
USER_FILE
USER_VIDEO
USER_ENTER
USER_PING
USER_CLOSE

✅ 五、客服消息协议（Java → Kafka → Go）
📌 Topic：chat-agent-msg
{
  "msgId": "uuid",
  "traceId": "uuid",
  "tenantId": "T1001",
  "sessionId": "S10001",
  "from": "AGENT",
  "type": "AGENT_TEXT",
  "timestamp": 1732876800456,
  "payload": {
    "agentId": "C1009",
    "content": "您好，请问有什么可以帮您？",
    "msgSeq": 10022
  }
}

✅ 客服消息类型
AGENT_TEXT
AGENT_IMAGE
AGENT_FILE
AGENT_VIDEO
AGENT_CLOSE

✅ 六、会话状态事件协议（核心）
📌 Topic：chat-session-event
✅ 1️⃣ 建立会话
{
  "type": "SESSION_CREATE",
  "payload": {
    "sessionId": "S10001",
    "userId": "U88931"
  }
}

✅ 2️⃣ 进入排队
{
  "type": "SESSION_WAITING"
}

✅ 3️⃣ 客服接入（WAITING → SERVING）
{
  "type": "SESSION_ACCEPT",
  "payload": {
    "agentId": "C1009"
  }
}

✅ 4️⃣ 会话正常结束
{
  "type": "SESSION_CLOSE",
  "payload": {
    "reason": "USER_CLOSE | AGENT_CLOSE"
  }
}

✅ 5️⃣ 会话超时
{
  "type": "SESSION_TIMEOUT",
  "payload": {
    "timeoutType": "WAIT_TIMEOUT | CHAT_TIMEOUT"
  }
}

✅ 七、转接专用协议（最容易出事故的区域）
📌 Topic：chat-transfer-event
✅ 1️⃣ 发起转接
{
  "type": "TRANSFER_REQUEST",
  "payload": {
    "fromAgentId": "C1009",
    "targetGroup": "售后组"
  }
}

✅ 2️⃣ 转接准备完成
{
  "type": "TRANSFER_READY",
  "payload": {
    "targetAgentId": "C1015"
  }
}

✅ 3️⃣ 转接成功
{
  "type": "TRANSFER_OK",
  "payload": {
    "newAgentId": "C1015"
  }
}

✅ 4️⃣ 转接失败（自动回滚）
{
  "type": "TRANSFER_FAIL",
  "payload": {
    "reason": "REJECT | TIMEOUT | AGENT_OFFLINE"
  }
}

✅ 八、系统事件协议
📌 Topic：chat-system-event
{
  "type": "WS_DISCONNECT",
  "payload": {
    "role": "USER | AGENT"
  }
}

✅ 九、Kafka 顺序性 & 幂等设计（你超级关键的一点）
✅ 1️⃣ Partition 规则（你现在的方案 ✅）
partitionKey = tenantId + "_" + sessionId


效果：

✅ 同一会话：

消息严格有序

状态事件严格有序

转接事件严格有序

✅ 不同会话：

并行处理

3 万在线完全 OK

✅ 2️⃣ 幂等消费规则（Java & Go 都必须做）
Redis: msg:dedup:{msgId} = 1 TTL 5min


消费前先查：

if exists → 丢弃
if not exists → 处理 + set

✅ 十、Go / Java 职责对照表
模块	Go	Java
WebSocket	✅	❌
Kafka Producer	✅	✅
Kafka Consumer	✅	✅
Session 状态修改	❌	✅
Redis 写 Session	❌	✅
Redis 读状态	✅	✅
转接控制	❌	✅
超时判定	✅（WAIT）	✅（SERVING）
✅ 十一、S3 文件消息协议（你已有设计 ✔）

统一格式：

{
  "type": "USER_IMAGE",
  "payload": {
    "url": "https://s3.xxx.com/xxx.png",
    "title": "支付截图",
    "size": 234567,
    "mime": "image/png"
  }
}

✅ 十二、这份协议的“工程级价值”

这套 Kafka 协议可以直接保证你：

✅ 不会乱序
✅ 不会重复投递
✅ 不会转接错人
✅ 不会双客服收到同一条消息
✅ 不会出现“僵尸会话”
✅ 能完整审计 & 回放










监控/熔断：Kafka lag、consumer lag、Go connections、msg latency、Redis latency 必须采集并报警。

顺序保证：设计 Kafka key/partition 策略，详见下文。
