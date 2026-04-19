---
marp: true
theme: default
paginate: true
size: 16:9
header: '日志分层的艺术'
footer: '分享 · 2026'
style: |
  section {
    font-family: 'PingFang SC', 'Helvetica', sans-serif;
    font-size: 23px;
    padding: 44px 60px;
  }
  section.lead {
    text-align: center;
    justify-content: center;
  }
  h1 { color: #00ADD8; }
  h2 { color: #007d9c; border-bottom: 2px solid #00ADD8; padding-bottom: 6px; margin-bottom: 18px; }
  h3 { color: #5C6BC0; margin: 8px 0; }
  code { background: #F5F5F5; padding: 2px 6px; border-radius: 3px; font-size: 0.9em; }
  pre { background: #1E1E1E; color: #D4D4D4; font-size: 0.7em; line-height: 1.35; padding: 14px; border-radius: 6px; }
  pre code { background: transparent; color: inherit; padding: 0; }
  table { font-size: 0.78em; }
  blockquote { border-left: 4px solid #00ADD8; color: #555; padding-left: 14px; margin: 10px 0; }
  .two-col { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
  .bad  { color: #D32F2F; }
  .good { color: #388E3C; }
  .mute { color: #888; }
  .kbd  { background: #263238; color: #fff; padding: 2px 8px; border-radius: 4px; font-size: 0.85em; }
---

<!-- _class: lead -->

# 日志分层的艺术

## 按**读者**切,不按 **level** 切 · 4 类日志全景

<br>

分享者 · 2026

---

## 核心结论

1. **4 类日志不是一类**:<span class="good">应用 / 接入 / 审计 / 业务事件</span>,读者、保留期、严格度全不同
2. **按读者切 > 按级别切** —— `DEBUG/INFO/ERROR` 只是严重度,不是"给谁看"
3. **流向不能混** —— 排障进 ES、审计进 WORM、业务事件进 Kafka → 数仓
4. **schema 严格度按类递增** —— 应用日志最宽松,业务事件最严(proto/avro 定义)
5. **成本差一个数量级** —— 不分层就按最贵的那类(审计年保留)算账单

<br>

<span class="mute">下面 11 页围绕这 5 条展开。</span>

---

## 问题定义:把 4 类混在一起打,会踩什么坑

<div class="two-col">

### <span class="bad">场景 1 · ES 账单爆炸</span>
应用 DEBUG 和 业务事件 / 审计 都往一个 index 里打,统一按最长保留
→ 一个月存储费翻 4 倍

### <span class="bad">场景 2 · 合规审计不过</span>
"谁在什么时候改了什么" 问不出来
actor / action / target 没规范字段,靠 grep 拼
→ 合规整改一个月

</div>

<div class="two-col">

### <span class="bad">场景 3 · 排障被业务日志淹没</span>
`订单创建成功` 每秒一万条
`db connection timeout` 混在里面,grep 半天看不见
→ 故障定位从 5 分钟拖到 1 小时

### <span class="bad">场景 4 · 业务事件进错地方</span>
"用户注册" 只进 ES,没进 Kafka
数仓 / 推荐系统 / 产品 BI 拿不到,要 SRE 导日志
→ 数据链路断裂,数据团队每周跟你吵

</div>

---

## 本质:为什么是按"读者"切

每一类日志,**读者的期望根本不同**:

| 读者 | 核心期望 | 导致的约束 |
|---|---|---|
| **开发 / SRE** | 详细 · 最近 · 随时加字段 | 保留短、schema 宽松 |
| **安全** | 完整 · 精确 · 可追溯 | 保留长、schema 严格、不可改 |
| **合规** | 权威 · 不可篡改 · 可举证 | WORM 存储、actor 真实、写失败=业务失败 |
| **产品 / 数据** | 结构化 · 可 join · 长跨度 | schema 严格、进数仓、版本兼容 |

**按 level 切不够**:

- `DEBUG/INFO/WARN/ERROR` 只描述"严重度"
- 一条 `ERROR` 既是开发想看(排障),也可能要进审计(支付失败)
- **必须加一个"日志类型"维度**,才能正确分流

---

## 分层:4 类日志全景表

| 维度 | **应用日志** | **接入日志** | **审计日志** | **业务事件** |
|---|---|---|---|---|
| 谁看 | 开发 / SRE | SRE / 安全 | 安全 / 合规 | 数据 / 产品 |
| 保留 | 7–30 d | 30–90 d | **1–7 y** | 看需求 |
| 严格度 | 宽松 | 严格 | 严格 + 不可改 | **schema 严格** |
| 产生时机 | 代码任何处 | 每次请求 | 关键操作发生 | 状态机变化 |
| 主要用途 | 排障 | 分析 + 安审 | 合规举证 | BI / 特征 / 推荐 |
| 传输 | stdout → agent | stdout → agent | stdout → agent | **Kafka**(不走日志) |
| 存储 | ES / Loki | ES / ClickHouse | 审计库(WORM) | 数仓 / 数据湖 |
| 字段代表 | level, msg, trace_id, err | method, path, status, latency, uid, ip, ua | actor, action, target, reason, ts | event_name, payload(proto/avro) |

> **记住一个直觉**:成本从左到右递增(审计最贵),schema 严格度从左到右递增。

---

## 内部两类:应用日志 + 接入日志

<div class="two-col">

### 应用日志 (Application Log)
代码里 `slog.InfoContext / ErrorContext` 打的

**字段**:
- `level` / `msg` / `trace_id` / `uid`
- 错误时加 `err` / `err_kind`

**保留**:7–30 天
**关键规则**:
- 结构化字段,不拼字符串
- trace_id 从 ctx 取
- <span class="good">只在入口层打 error</span>(和错误处理同源)

### 接入日志 (Access Log)
每个请求一条,由 middleware 统一生成

**字段**(严格集):
- `method` / `path` / `status` / `latency_ms`
- `uid` / `ip` / `ua` / `req_size` / `resp_size`

**保留**:30–90 天
**关键规则**:
- 每个协议一个 MW(HTTP / gRPC / Kafka)
- <span class="good">字段集固定</span>,业务不能加减
- 带 IP / UA —— 安全分析刚需

</div>

---

## 外部两类:审计日志 + 业务事件

<div class="two-col">

### 审计日志 (Audit Log)
"谁 · 什么时候 · 对什么 · 做了什么"

**字段**(必填):
- `actor` —— 真实操作者(不能 `system`)
- `action` —— 标准化动作 `USER_PASSWORD_CHANGE`
- `target` —— 对象 `user:123`
- `reason` / `ip` / `ts`

**保留**:1–7 年(依合规)
**关键规则**:
- <span class="bad">WORM 存储,不可篡改</span>
- <span class="bad">审计写失败 = 业务失败</span>
- 幂等 key,不重复记录

### 业务事件 (Business Event)
业务状态机变化的"事实记录"

**字段**(proto/avro 定义):
- `event_id` / `event_name` / `ts`
- 业务 payload(`order_id` / `amount` 等)

**载体**:Kafka / Pulsar
**下游**:数仓 / 推荐 / BI
**关键规则**:
- <span class="bad">不走日志系统,走消息队列</span>
- schema 演进有兼容规范(只加字段)
- 数据团队是上下游,PR 评审对齐

</div>

---

## 收益:4 类分开后你能拿到

<div class="two-col">

### 成本降一个数量级
应用 7d vs 全局 30d
→ ES 存储费 <span class="good">-75%</span>

### 合规一次过
审计独立归档 · WORM
→ "谁在几点改了什么" <span class="good">一查就有</span>

</div>

<div class="two-col">

### 排障清爽
ES 只看应用日志
→ 故障定位从<span class="good">分钟到秒</span>

### 数据 / 业务分家
事件进 Kafka · 产品直接消费
→ 数据团队不再找 SRE 要日志

</div>

---

## HOW-1 · 内部两类怎么打

```go
// ─── 应用日志:业务代码任何处 ───
slog.InfoContext(ctx, "order created",
    "type", "app",               // ★ 用于 agent 路由
    "order_id", id,
    "amount", amount,
)

// ─── 错误只在入口打一次 ───
if err := h.svc.Login(ctx, req); err != nil {
    slog.ErrorContext(ctx, "login failed",
        "type", "app",
        "err", err,
    )
    return err
}

// ─── 接入日志:HTTP middleware,字段集固定 ───
func AccessLogMW() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        slog.InfoContext(c.Request.Context(), "access",
            "type",       "access",
            "method",     c.Request.Method,
            "path",       c.FullPath(),
            "status",     c.Writer.Status(),
            "latency_ms", time.Since(start).Milliseconds(),
            "uid",        getUID(c),
            "ip",         c.ClientIP(),
            "ua",         c.Request.UserAgent(),
        )
    }
}
```

`type` 字段是 agent 路由分流的依据 —— 不同 `type` → 不同 index / 不同保留策略。

---

## HOW-2 · 外部两类怎么打

```go
// ─── 审计日志:独立 API,不是 slog.Info ───
package audit

type Event struct {
    Actor, Action, Target, Reason, IP string
    TS                                time.Time
}

func Record(ctx context.Context, e Event) error {
    e.TS = time.Now()
    // ★ 必须同步落盘,失败则业务失败
    if err := auditWriter.Sync(ctx, e); err != nil {
        return err
    }
    slog.InfoContext(ctx, "audit",
        "type", "audit",
        "actor", e.Actor, "action", e.Action, "target", e.Target,
        "reason", e.Reason, "ip", e.IP,
    )
    return nil
}

// 业务调用 —— 审计失败,业务也失败
if err := audit.Record(ctx, audit.Event{
    Actor: user.ID, Action: "USER_PASSWORD_CHANGE",
    Target: fmt.Sprintf("user:%d", user.ID), Reason: "self_service_reset",
}); err != nil {
    return err
}

// ─── 业务事件:走 Kafka,不走日志 ───
func PublishOrderCreated(ctx context.Context, o *Order) error {
    return producer.Publish(ctx, "order.created", &pb.OrderCreated{
        EventId:    uuid.NewString(),
        OrderId:    o.ID, UserId: o.UserID,
        AmountCent: o.Amount,
        TsMs:       time.Now().UnixMilli(),
    })
}
```

---

## 反模式 Checklist

| 反模式 | 正确做法 |
|---|---|
| 审计日志混在应用日志一个 index | 加 `type` 字段,agent 分流到审计库 |
| 业务事件只进日志不进 Kafka | 事件必须走 MQ,日志里只留副本(可选) |
| 接入日志由业务代码打 | middleware 统一,字段集固定,业务不加减 |
| 审计失败业务继续执行 | 审计写失败 = 业务失败,事务绑一起 |
| `actor=system` / `unknown` | actor 必须真实可查的操作者 |
| 应用日志按审计保留期存 | 应用 7–30d,别按 1 年那档算钱 |
| 业务事件 schema 随意加减 | proto 定义,兼容性走 PR 评审 |
| access log 缺 IP / UA | 安全分析、WAF 联动全靠这俩 |
| 用 `slog.Info("debug xxx")` 打 DEBUG | 用 `slog.Debug`,靠 level 过滤 |
| 一个 logger handler 打四类 | 按 `type` 分路由,或者干脆分 writer |
| error 日志在 DAO / SDK / middleware 各打一次 | **只在入口打一次**(和错误处理同源) |
| 业务事件字段 `data: json.RawMessage` | 强类型 proto,不要"万能 JSON" |

---

<!-- _class: lead -->

## 回到开头那 5 条

1. **4 类日志不是一类**:应用 / 接入 / 审计 / 业务事件
2. **按读者切,不按 level 切**
3. **流向不能混**:ES / ClickHouse / WORM / Kafka 各走各的
4. **schema 严格度按类递增**,业务事件最严
5. **不分层 = 按最贵那类算账单**

<br>

> **日志不是一堆字符串,是**给 4 类不同读者**的 4 种证据**。

<br>

# Q & A

<span class="mute">欢迎拍砖 · 分享你们 4 类混打的坑</span>
