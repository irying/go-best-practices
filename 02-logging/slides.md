---
marp: true
theme: default
paginate: true
size: 16:9
header: 'Go 日志最佳实践'
footer: '分享 · 2026'
style: |
  section {
    font-family: 'PingFang SC', 'Helvetica', sans-serif;
    font-size: 24px;
    padding: 48px 64px;
  }
  section.lead {
    text-align: center;
    justify-content: center;
  }
  h1 { color: #00ADD8; }
  h2 { color: #007d9c; border-bottom: 2px solid #00ADD8; padding-bottom: 6px; margin-bottom: 18px; }
  h3 { color: #5C6BC0; margin: 8px 0; }
  code { background: #F5F5F5; padding: 2px 6px; border-radius: 3px; font-size: 0.9em; }
  pre { background: #1E1E1E; color: #D4D4D4; font-size: 0.72em; line-height: 1.35; padding: 14px; border-radius: 6px; }
  pre code { background: transparent; color: inherit; padding: 0; }
  table { font-size: 0.82em; }
  blockquote { border-left: 4px solid #00ADD8; color: #555; padding-left: 14px; margin: 10px 0; }
  .two-col { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; }
  .bad  { color: #D32F2F; }
  .good { color: #388E3C; }
  .mute { color: #888; }
  .kbd  { background: #263238; color: #fff; padding: 2px 8px; border-radius: 4px; font-size: 0.85em; }
---

<!-- _class: lead -->

# Go 日志最佳实践

## 从 `log` 到 `slog` 的十四年 · 一份 2026 指南

<br>

分享者 · 2026

---

## 核心结论（先讲完，再展开）

> **日志不是"多写几行 fmt.Printf"，而是给未来的自己和下游系统留证据。**

1. **新项目默认 <span class="kbd">log/slog</span>** —— 标准库统一了接口，生态已收敛到它。
2. **按读者分层**：<span class="good">应用 / 接入 / 审计 / 业务事件</span> 四类，保留期和严格度都不一样。
3. **只在入口层打应用日志** —— 和错误处理同源：DAO / SDK 不打，入口统一打。
4. **结构化 + ctx logger + trace_id** —— 能 grep、能关联、能追溯。
5. **应用只写 stdout，收集交给 agent** —— 不要让 logger 直连远端存储。

<br>

<span class="mute">下面 11 页都是在给这 5 条加注脚。</span>

---

## 为什么要专门聊它 · 4 个痛点合在一起

<div class="two-col">

### <span class="bad">① 组件分裂</span>
`log` / `logrus` / `zap` / `zerolog` / `slog`
不同项目一套风格，跨项目排障靠硬背字段名

### <span class="bad">② 到处都打，重复打</span>
DAO 打一次、service 打一次、handler 再打一次
一次错误 N 条日志，磁盘爆炸，关键信息反而被淹没

</div>

<div class="two-col">

### <span class="bad">③ 敏感信息裸奔</span>
`log.Printf("user=%+v", user)` 把手机号 / 身份证 / token 直接写进去
合规事故 + 安全审计红灯

### <span class="bad">④ 关键操作没留痕 + debug 刷屏</span>
登录、改权限、支付没审计日志 → 真出事查不到
Prod 忘关 DEBUG → 日志费用翻倍、ES 扛不住

</div>

---

## 本质：Go 日志的十四年演进

```
 2012  log (stdlib 1.0)        文本 · 无 level · 无结构化 · 全局状态
 2014  logrus                  第一代 level + structured (Entry-based · 反射慢)
 2016  zap (Uber)              零分配 · 高性能 · API 稍冗长
 2017  zerolog                 chained API · 零分配 · JSON-first
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 2023  log/slog (stdlib 1.21)  官方标准化 · Handler 可插拔 · LogValuer 脱敏
 2024  zap / zerolog 实现 slog.Handler         生态向 slog 收敛
 2025  Go 1.24 继续打磨 slog                   新项目事实默认
```

**slog 的三个设计要点**（为什么它能成为默认）：

| 要点 | 含义 | 对团队意义 |
|---|---|---|
| `Logger` ↔ `Handler` 解耦 | 接口 vs 实现 | 库只依赖 `*slog.Logger`，不绑死后端 |
| `LogValuer` 接口 | 字段懒求值 | <span class="good">优雅实现脱敏 / 按需渲染</span> |
| 标准库内置 | 零依赖 | 库作者可以放心写 `slog` |

---

## Go 版本速查表：什么时候升级日志栈

| Go 版本 | 年份 | 和日志相关的变化 | 团队动作 |
|---|---|---|---|
| 1.0 | 2012 | `log` 包 | —— |
| 1.13 | 2019 | `errors.Unwrap/Is/As` | error log 开始有完整链路 |
| 1.19 | 2022 | `any` 别名、slog 提案 | 观望 |
| **1.21** | **2023** | **`log/slog` 进入标准库** | <span class="good">新项目开始用</span> |
| 1.22 | 2024 | slog 性能 / 内存优化 | 老项目可评估迁移 |
| 1.23 | 2024 | `slog.DiscardHandler`、`LogValuer` 生态成熟 | 库层统一接口 |
| 1.24 | 2025 | 继续打磨 | <span class="good">2026 默认栈</span> |

> <span class="good">**结论**</span>：Go ≥ 1.21 的新项目直接 slog；老项目（logrus/zap）不急着改，先统一接口。

---

## 分层：**内**部日志 vs **外**部日志

按**读者**切，而不是按级别切：

| 分类 | 类型 | 读者 | 保留期 | 严格度 | 典型字段 |
|---|---|---|---|---|---|
| <span class="good">**内部**</span> | 应用日志 | 开发 / SRE | 7–30d | 宽松 | level, msg, trace_id, err |
| <span class="good">**内部**</span> | 接入日志 | SRE / 安全 | 30–90d | 严格 | method, path, status, latency, uid |
| <span class="bad">**外部**</span> | 审计日志 | 安全 / 合规 | 1–7y | 严格 + 不可篡改 | actor, action, target, reason, ts |
| <span class="bad">**外部**</span> | 业务事件 | 数据 / 产品 | 依需求 | schema 严格 | event_name, payload（定义走 proto / avro） |

**流向不同，别混在一起打：**
```
  应用日志 ─► stdout ─► agent ─► ES / Loki           排障
  接入日志 ─► stdout ─► agent ─► ES / ClickHouse     分析 + 安审
  审计日志 ─► stdout ─► agent ─► 审计库 (WORM)       合规
  业务事件 ─► Kafka  ────────► 数仓                  产品 / BI
```

---

## 这么做能拿到什么

<div class="two-col">

### 故障排障从分钟到秒
trace_id 串起全链路，一次搜索定位现场
<span class="mute">之前：5 个服务 grep 拼时间线</span>

### 合规能过
审计日志单独归档、单独保留期
<span class="mute">真的能回答"谁在几点改了什么"</span>

</div>

<div class="two-col">

### 日志成本可控
分层 + 采样 + agent 过滤
<span class="mute">高频路径打 1%，ES 费用降一个数量级</span>

### 新人心智负担低
一个 `L(ctx)` + 统一字段名
<span class="mute">不用学 3 种 logger 的 3 套 API</span>

</div>

---

## HOW-1 · 组件选型（2026 版）

| 场景 | 选择 | 理由 |
|---|---|---|
| **新项目** | <span class="good">`log/slog` + JSONHandler</span> | 标准库、生态默认、零依赖 |
| **性能极致敏感** | `slog` API + `zap` 或 `zerolog` 作为 Handler | 保留 slog 接口，底层走高性能库 |
| **老项目 logrus** | 渐进迁移到 slog | logrus 不再积极维护 |
| **老项目 zap/zerolog** | 保留，包一层 slog API | 业务代码用 slog，底层不动 |
| **库 / SDK 层** | <span class="good">**只依赖 `*slog.Logger`**</span> | 不要在库里 import zap/logrus |

> **原则**：业务代码 import `log/slog`，<span class="bad">不要</span> import 具体实现。
> 换后端（本地 console ↔ 线上 JSON ↔ 性能版 zap handler）只改入口一行。

---

## HOW-2 · slog 实操：10 行够用

```go
import "log/slog"

// 入口初始化（main.go）
h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level:     levelFromEnv(),     // dev=Debug, prod=Info
    AddSource: true,                // 自动带 file:line
})
slog.SetDefault(slog.New(h))

// 打日志 —— 结构化字段，不要 Sprintf
slog.Info("user login",
    "uid", 123,
    "ip", ip,
    "duration_ms", d.Milliseconds(),
)

// With：预绑定字段，派生子 logger
reqLogger := slog.Default().With("trace_id", traceID, "path", path)
reqLogger.Error("db query failed", "err", err)
```

<div class="two-col">

**级别策略**
- `Debug`：开发 / 按需开关
- `Info`：关键业务节点（少！）
- `Warn`：降级 / 重试命中
- `Error`：真需要人看

**Handler 切换**
- 本地：`NewTextHandler`（人看）
- 线上：`NewJSONHandler`（机读）
- 测试：`NewTextHandler(io.Discard)` 或自定义捕获

</div>

---

## HOW-3 · Context 注入 + Trace 联动

**原则**：logger 从 <span class="good">context</span> 取，不要全局到处用。

```go
// logger/ctx.go
type ctxKey struct{}

func WithLogger(ctx context.Context, l *slog.Logger) context.Context {
    return context.WithValue(ctx, ctxKey{}, l)
}
func L(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(ctxKey{}).(*slog.Logger); ok {
        return l
    }
    return slog.Default()
}

// middleware/log.go —— 入口注入一次，全链路带着
func LogMW() gin.HandlerFunc {
    return func(c *gin.Context) {
        traceID := c.GetHeader("X-Trace-ID")
        if traceID == "" { traceID = uuid.NewString() }

        // 和 OpenTelemetry 联动：从 span 取 trace_id / span_id
        sc := trace.SpanContextFromContext(c.Request.Context())
        attrs := []any{"trace_id", traceID, "path", c.Request.URL.Path}
        if sc.IsValid() {
            attrs = append(attrs, "otel_trace", sc.TraceID().String())
        }

        l := slog.Default().With(attrs...)
        c.Request = c.Request.WithContext(WithLogger(c.Request.Context(), l))
        c.Next()
    }
}

// 业务代码
func (s *UserSvc) Login(ctx context.Context, ...) error {
    logger.L(ctx).Info("login start", "uid", uid)  // 自动带 trace_id
    ...
}
```

---

## HOW-4 · 脱敏 + 采样

**脱敏** —— 用 `slog.LogValuer`，类型自带脱敏语义：

```go
type Phone string
func (p Phone) LogValue() slog.Value {
    s := string(p)
    if len(s) < 7 { return slog.StringValue("***") }
    return slog.StringValue(s[:3] + "****" + s[7:])
}

slog.Info("send sms", "phone", Phone(user.Phone))
// 输出: "phone":"138****1234"
```

或者在 **Handler 层统一** `ReplaceAttr` 兜底:
```go
var sensitive = map[string]bool{"password": true, "token": true, "id_card": true}
slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
        if sensitive[a.Key] { return slog.String(a.Key, "***") }
        return a
    },
})
```

**采样** —— 高频路径降频，error 级别保证不丢：
```go
type SamplingHandler struct {
    slog.Handler
    rate int // 1/N 采样
}
func (h *SamplingHandler) Handle(ctx context.Context, r slog.Record) error {
    if r.Level >= slog.LevelError { return h.Handler.Handle(ctx, r) }
    if rand.IntN(h.rate) == 0    { return h.Handler.Handle(ctx, r) }
    return nil
}
```

---

## HOW-5 · 日志收集架构：应用只写 stdout

```
 ┌──────────────────────┐
 │   应用进程           │
 │   slog → stdout JSON │        只做这一步！
 └─────────┬────────────┘
           │
 ┌─────────▼────────────────────────────────────────────┐
 │  Host / Pod 级收集 agent                              │
 │  filebeat / fluent-bit / vector                       │
 │   · 加字段（host / pod / env）                         │
 │   · 按 type 路由                                       │
 │   · 本地缓冲 · 重启续传                                │
 └─────────┬────────────────────────────────────────────┘
           │
           ├──► Kafka  ─► ES / Loki          (应用+接入)
           ├──► Kafka  ─► 审计库 WORM         (审计)
           └──► Kafka  ─► 数仓                (业务事件)
```

**为什么应用不直连远端？**
- 存储抖动<span class="bad">反压应用</span> → 业务请求被日志卡住
- 多副本部署缓冲零散，丢日志风险高
- Agent 统一加元数据、过滤、路由，变更不动应用
- 应用重启不丢日志（agent 基于文件续传）

---

## 反模式 Checklist · 开 PR 前自己过一遍

| 反模式 | 正确做法 |
|---|---|
| `log.Printf("%+v", err)` 当日志 | `slog.Error("...", "err", err)` 结构化 |
| 日志里拼 password / token / 完整 body | `LogValuer` 脱敏 + `ReplaceAttr` 兜底 |
| DAO / SDK 里打日志 | 只返回 error，入口层统一打 |
| 关键字段用 `Sprintf` 拼 | 结构化字段 `"uid", 123` |
| 全局 `slog.Default()` 到处直接调用 | 从 ctx 取带 trace 的 logger：`L(ctx)` |
| 同步写远端存储 | 只写 stdout，agent 收集 |
| Prod 开 `DEBUG` | 级别按环境 + <span class="good">动态可调</span>（信号 / 接口） |
| 业务事件混在应用日志里 | 业务事件单独 topic，proto / avro 定义 schema |
| 用日志做统计 / 告警 | 用 Prometheus counter / histogram |
| 日志文件不 rotate | logrotate / k8s 托管 |
| 库里 import `zap`、`logrus` | 库只依赖 `*slog.Logger` |
| `panic` 时日志丢失 | `recover` + 同步 flush handler |

---

<!-- _class: lead -->

## 回到开头那 5 条

1. 新项目默认 <span class="kbd">log/slog</span>
2. 按读者分层：应用 / 接入 / 审计 / 业务事件
3. 只在入口层打应用日志（和错误处理同源）
4. 结构化 + ctx logger + trace_id
5. 应用只写 stdout，收集交给 agent

<br>

> **日志不是给现在的你看的，是给未来的你和下游系统留证据。**

<br>

# Q & A

<span class="mute">欢迎拍砖 · 分享你们踩过的日志坑</span>
