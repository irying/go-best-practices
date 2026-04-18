---
marp: true
theme: default
paginate: true
size: 16:9
header: 'Go 错误处理最佳实践'
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

# Go 错误处理最佳实践

## 5 条结论 · 1 张分层图 · 1 份 Checklist

<br>

分享者 · 2026

---

## 核心结论（先讲完，再展开）

> **错误不是 `if err != nil` 的重复劳动，而是 API 的一部分。**

1. **错误在下层<span class="good">带上现场</span>，在上层<span class="good">变成日志和响应</span>。**
2. **下层用 <span class="kbd">%w</span> 包装，中层透传，入口层<span class="good">只打一次日志</span>。**
3. **对内用 <span class="kbd">errors.Is / As</span>，对外用<span class="good">稳定错误码</span>** —— 字符串匹配是脆弱的。
4. **error 管失败，context 管截止** —— 一起用才完整。
5. **规则越少越能被一致执行** —— 下面一页能记住的，才是真规范。

<br>

<span class="mute">剩下的 9 页，都是在给这 5 条加注脚。</span>

---

## 为什么要专门聊它 · 4 个痛点合在一起

<div class="two-col">

### <span class="bad">① 日志散落</span>
dao / service / handler 都打一遍
→ 一次错误 3 条日志，还是拼不出现场

### <span class="bad">② 堆栈丢失</span>
`fmt.Errorf("%v", err)` 把 error 变字符串
→ 类型没了、链路没了

</div>

<div class="two-col">

### <span class="bad">③ 错误码靠字符串匹配</span>
`strings.Contains(err.Error(), "not found")`
→ 文案一改就炸，包一层就废

### <span class="bad">④ 契约靠口口相传</span>
前端不知道 toast 什么、下游不知道能不能重试
→ 错误是 API 的一部分，却从没被设计过

</div>

---

## 本质：Go 为什么这样设计

<div class="two-col">

### error 是一个**值**
```go
type error interface {
    Error() string
}
```
没有 try/catch，没有跨栈跳跃
<span class="good">错误就是一个能检查、能包装、能比较的返回值</span>

### error 是 API 契约
```go
func GetUser(ctx, id) (*User, error)
//           ↑ happy   ↑ unhappy
```
两份契约一样重要
<span class="bad">大多数项目只认真设计了前者</span>

</div>

> 一次失败携带三层信息：**根因**（给 SRE）· **上下文**（给开发）· **语义**（给前端）。
> 好的错误处理 = 三层都不丢、各自送到对的人手上。

---

## 分层：error 在项目里怎么流动

```
 ┌───────────────────────────────────────────────────────┐
 │  入口层   Handler / gRPC / Job / Worker                │
 │   ✔ 统一打日志（唯一一次）                              │
 │   ✔ error → 业务错误码 → 响应客户端                     │
 └──────────────────────▲────────────────────────────────┘
                        │  (带上下文的 error)
 ┌──────────────────────┴────────────────────────────────┐
 │  业务层   Service / Logic                              │
 │   ✔ errors.Is/As 判断 → 转语义化 ecode                 │
 │   ✔ 同包透传：直接 return err                          │
 └──────────────────────▲────────────────────────────────┘
                        │
 ┌──────────────────────┴────────────────────────────────┐
 │  边界层   DAO / RPC Client / 外部 SDK                   │
 │   ✔ fmt.Errorf("...: %w", err)   加上下文             │
 │   ✔ 需要堆栈时用 pkg/errors.Wrap                       │
 └──────────────────────▲────────────────────────────────┘
                        │
                 外部世界（DB / HTTP / RPC / OS）
```

| 层 | <span class="good">该做</span> | <span class="bad">不要做</span> |
|---|---|---|
| 边界 | 包装 + 加上下文 | 打日志 |
| 业务 | 判断 + 透传 | 重复包装、打日志 |
| 入口 | 打日志 + 出错误码 | 继续向上抛 |

---

## 这么做能拿到什么

<div class="two-col">

### 🎯 一次故障，一条日志
从 *"4 条散落 ERR"* 变成 *"1 条带完整 wrap 链"*
排障时间从分钟级到秒级

### 🤝 错误码成为真正的 API
前端 <span class="good">switch code</span> 代替 <span class="bad">indexOf</span>
下游基于 code 决定是否重试
告警基于 code 分级

</div>

<div class="two-col">

### 📈 可观测性基本免费
Metrics / Tracing / Log 的 code 维度
不用每个人手动埋点

### 🧑‍💻 新人心智负担下降
3 条规则就能上手
规则越少，越容易被一致执行

</div>

---

## HOW-1 · 包装 & 判断

```go
// ❌ 丢类型、丢链路
return fmt.Errorf("query user: %v", err)

// ✅ 标准库原生方案（Go 1.13+）
return fmt.Errorf("query user id=%d: %w", id, err)

// 上层判断 —— 不要再用字符串比较
if errors.Is(err, sql.ErrNoRows)    { ... }   // 哨兵错误
var ve *ValidationError
if errors.As(err, &ve)              { ... }   // 带字段的错误类型
```

**工具选型（2026）**：

| 需求 | 选择 |
|---|---|
| 包装 / 判断 | 标准库 `fmt.Errorf` + `%w` + `errors.Is/As` |
| 需要堆栈 | `pkg/errors.Wrap` 或自研 |
| 业务错误码 | 自研 `ecode` 包，映射 HTTP / gRPC status |
| 并发聚合 | `golang.org/x/sync/errgroup`（注意坑，见下页） |

---

## HOW-2 · 错误码 & 日志

```go
// ecode/ecode.go —— 最小设计
type Code struct {
    Code    int         // 数字码，稳定对外
    Message string      // 英文，给开发者
    Details interface{} // 前端提示 / 重试建议
}
func (c *Code) Error() string { return c.Message }

// 公共码复用 HTTP 语义
var (
    BadRequest   = &Code{400, "bad request",    nil}
    Unauthorized = &Code{401, "unauthorized",   nil}
    NotFound     = &Code{404, "not found",      nil}
)
// 业务码：命名空间自维护，如 100xxx 用户域 / 200xxx 订单域
var UserNotFound = &Code{100001, "user not found", nil}
```

**日志只在入口打一次**（middleware / interceptor / worker 顶层）：
```go
log.Ctx(c).WithFields(...).Errorf("%+v", err)   // %+v 打印完整 wrap 链
```
<span class="bad">DAO / SDK / 工具库一律不打日志，只返回 error。</span>

---

## HOW-3 · Context 搭档 & errgroup 三个坑

```go
// error 管失败，context 管截止 —— 一起用才完整
func (d *DAO) Query(ctx context.Context, ...) error {
    rows, err := d.db.QueryContext(ctx, ...)   // 业务库全部吃 ctx
    // 显式传递 > 隐式传递：ctx 永远是第一个参数，不要塞进 struct
    // 不要把 gin.Context 直接灌到 service，在 handler 里转成 context.Context
}
```

**errgroup 三个坑：**
```go
g, ctx := errgroup.WithContext(ctx)
for _, id := range ids {
    id := id                         // ① 闭包捕获，必须 rebind
    g.Go(func() error {
        defer func() {               // ② goroutine panic 不会变成 err，自己 recover
            if r := recover(); r != nil { /* 转 error */ }
        }()
        return doWork(ctx, id)
    })
}
if err := g.Wait(); err != nil { }   // ③ 只返回第一个 error，必要时用 errors.Join 聚合
```

---

## 反模式 Checklist · 开 PR 前自己过一遍

| 反模式 | 正确做法 |
|---|---|
| `fmt.Errorf("xx: %v", err)` | 用 `%w` |
| `err.Error() == "xxx"` / `strings.Contains` | `errors.Is` / `errors.As` |
| 每层都 `log.Error(err)` | 入口层统一打一次 |
| `return errors.New("failed")` 无上下文 | 带参数 / 用 `%w` 包装下层 |
| `context` 存进 struct | 第一个参数显式传 |
| `_ = doSomething()` 吞 err | 显式处理或至少注释说明 |
| panic 代替 return err | Go 不是 Java，仅用于真正不可恢复 |
| 裸 `errors.New` 对外 | 对外一律 `ecode.*` |

---

<!-- _class: lead -->

## 回到开头那 5 条

1. 错误在下层<span class="good">带上现场</span>，在上层<span class="good">变成日志和响应</span>
2. <span class="kbd">%w</span> 包装 · 中层透传 · 入口只打一次日志
3. 对内 <span class="kbd">errors.Is/As</span>，对外 <span class="good">稳定错误码</span>
4. error 管失败，context 管截止
5. 规则越少，越能被一致执行

<br>

> **把失败路径当成一等公民来设计。**

<br>

# Q & A

<span class="mute">欢迎拍砖 · 分享你们项目里的 CaseStudy</span>
