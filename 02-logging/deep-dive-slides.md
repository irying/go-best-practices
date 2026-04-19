---
marp: true
theme: default
paginate: true
size: 16:9
header: 'Go 日志库深度剖析'
footer: '从 log 到 slog 的十四年 · 分享 2026'
style: |
  section {
    font-family: 'PingFang SC', 'Helvetica', sans-serif;
    font-size: 22px;
    padding: 42px 56px;
  }
  section.lead {
    text-align: center;
    justify-content: center;
  }
  h1 { color: #00ADD8; }
  h2 { color: #007d9c; border-bottom: 2px solid #00ADD8; padding-bottom: 6px; margin-bottom: 16px; }
  h3 { color: #5C6BC0; margin: 6px 0; }
  code { background: #F5F5F5; padding: 2px 6px; border-radius: 3px; font-size: 0.88em; }
  pre { background: #1E1E1E; color: #D4D4D4; font-size: 0.66em; line-height: 1.3; padding: 12px; border-radius: 6px; }
  pre code { background: transparent; color: inherit; padding: 0; }
  table { font-size: 0.76em; }
  blockquote { border-left: 4px solid #00ADD8; color: #555; padding-left: 14px; margin: 8px 0; }
  .two-col { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
  .bad  { color: #D32F2F; }
  .good { color: #388E3C; }
  .mute { color: #888; }
  .kbd  { background: #263238; color: #fff; padding: 2px 8px; border-radius: 4px; font-size: 0.85em; }
---

<!-- _class: lead -->

# Go 日志库深度剖析

## 从 `log` 到 `slog` 的十四年
### 原理 · 源码 · 权衡 · 迁移

<br>

分享者 · 2026

---

## 今天的脉络

```
 Part 1 · 为什么日志库会迭代(3 组核心矛盾)
 Part 2 · 四代演进
            log → logrus → zap → zerolog → slog
            每一代解决什么、代价是什么、源码关键路径
 Part 3 · slog 的真正价值
            官方立项依据 · 8 个场景客观评估
 Part 4 · 反方意见
            什么情况下"不必"换 slog
 Part 5 · 迁移与收尾
            benchmark · 3 条迁移路径
```

> **读完能带走**:每代库的设计原理、slog 的 Handler/LogValuer/Record 内部机制、场景化的客观选型判断。

---

## Part 1 · 日志库的三组核心矛盾

所有日志库的演进,都在这三组矛盾里找不同的平衡点:

| 矛盾 | 典型表现 |
|------|---------|
| **可读性 vs 可机读性** | 人想看文本 · 机器想要 JSON |
| **表达力 vs 性能** | structured 字段越多越好查 · 分配越多越慢 |
| **易用性 vs 可控性** | 全局 logger 方便 · 依赖注入干净但样板多 |

```
 log       可读 ✓  机读 ✗  表达力 ✗  性能 ✓  易用 ✓
 logrus    可读 ✓  机读 ✓  表达力 ✓  性能 ✗  易用 ✓
 zap       可读 △  机读 ✓  表达力 ✓  性能 ✓  易用 △ (两套 API)
 zerolog   可读 △  机读 ✓  表达力 ✓  性能 ✓✓ 易用 △
 slog      可读 ✓  机读 ✓  表达力 ✓  性能 ✓  易用 ✓  (标准库 + 可插拔)
```

slog **不是性能冠军**,但第一次把 5 个维度同时做到"够好",且进了标准库。

---

## Part 2 · `log` (2012) —— 起点

```go
// 简化版 stdlib/log.Logger —— 就是加了锁的 io.Writer 包装
type Logger struct {
    outMu  sync.Mutex
    prefix atomic.Pointer[string]
    flag   atomic.Int32
    out    io.Writer
}

// 核心 Output 干的事:拼 header(时间+prefix+file:line)→ 加锁 → 写 Writer
```

**做得好**:零依赖、线程安全、够快。
**局限**:

- <span class="bad">无 Level</span> —— 区分 DEBUG/ERROR 要自己拼前缀
- <span class="bad">无结构化</span> —— 线上日志聚合后 grep 到怀疑人生
- <span class="bad">`Fatal` 跳过 defer</span> —— 资源泄漏
- <span class="bad">全局状态</span> —— `log.SetOutput` 改全局,测试难隔离

这些局限在 2014 催生了 logrus。

---

## Part 2 · `logrus` (2014) · 身临其境

2015 年春天,你给公司用户服务加日志,logrus API 特别顺手:

```go
log.WithFields(log.Fields{"user_id": uid, "action": "login"}).Info("user login")
```

code review 一片好评。直到压测:

- **QPS 上不去 3 万**,服务 CPU 跑满
- pprof 火焰图:**30%+ CPU** 在 `runtime.mapassign_faststr` 和 `reflect.Value.Interface`
- heap 采样:分配热点全是 `map[string]interface{}` 和 `*logrus.Entry`

你去 Uber 博客找优化办法,发现他们在造新轮子。那个轮子叫 **zap**。

---

## Part 2 · `logrus` · 为什么慢

```go
// ★ 性能瓶颈在数据结构
type Entry struct {
    Logger  *Logger
    Data    Fields            // ★ map[string]interface{}
    ...
}
type Fields map[string]interface{}

// WithFields 每次都新建一个 map,复制老 map 内容
func (e *Entry) WithFields(fields Fields) *Entry {
    data := make(Fields, len(e.Data)+len(fields))    // ① 新 map
    for k, v := range e.Data { data[k] = v }         // ② 复制
    for k, v := range fields { data[k] = v }         // ③ 再复制
    return &Entry{..., Data: data, ...}               // ④ 新 Entry
}

// Formatter 再走 json.Marshal(反射)
func (f *JSONFormatter) Format(entry *Entry) ([]byte, error) {
    ... b, err := json.Marshal(data) ...  // 反射 + type switch
}
```

链式 5 次 `WithField` = 5 个 map + 5 个 Entry + 反射 Marshal。logrus 已声明 "in maintenance mode",不接重大 feature。

---

## Part 2 · `zap` (2016) · 零分配三板斧

Uber 的目标是**"避免 hot path 的 `reflect` 和 `interface{}`"**。

#### 板斧 1 · `Field` 不用 `interface{}`

```go
// 关键:Field 是 struct 不是 interface
type Field struct {
    Key       string
    Type      FieldType   // 枚举,指示哪个字段有效
    Integer   int64       // int/bool/duration 都塞这里
    String    string      // 字符串
    Interface interface{} // 只有复杂类型才用
}

func Int(key string, val int) Field { return Field{Key: key, Type: Int64Type, Integer: int64(val)} }
```

**原生类型零装箱**,只有 `zap.Any(complexStruct)` 才走 `interface{}`。

#### 板斧 2 · Encoder 直写 buffer

每种类型一个 `AddX(key, val)` 方法,**直接往 `*buffer.Buffer` 追加字节**,不走 `json.Marshal` 反射。

#### 板斧 3 · `sync.Pool` 复用 buffer

单条日志几乎零堆分配。

---

## Part 2 · `zap` 的痛点 · 诚实盘点

做过一轮对比后,zap 真正站得住的"痛"其实**只有 1 条半**。很多常被说成"zap 痛"的,其实不是 zap 的问题。

### ✅ 真痛 —— 直白、API 层面的根本差距
**Context 不是一等公民** —— 没有 `InfoContext(ctx, ...)`,业务里到处 `L(ctx).Info(...)`,每家团队一套 helper。
<span class="mute">→ Part 3 场景详讲 1 展开</span>

### ⚠️ 真痛但场景窄 —— 只疼写库的少数人
**SDK 锁死 logger 版本** —— 下页故事。

### ❌ 常被说成 zap 痛,其实不是
| 常被说 | 真相 |
|---|---|
| 默认不带 stack | 错误处理规范的事,按 01 做就行(下一页) |
| 全量脱敏要写 Core | sed 200 处或 Core 40 行都行,非质变(再下一页) |
| 两套 API 撕裂 | 团队定规范就没事 |
| 全局静态字段要每处 With | `zap.ReplaceGlobals(root.With(...))` 一行,**根本不是痛** |

> **结论**:API 层面 zap 真正跟不上 slog 的只有 ctx 这条。
> 加上"标准库身份"这条根本性的生态优势,**真正硬的转投理由就 3 条**(本章最后总结)。

---

## "默认不带 stack" —— 是错误处理规范的事,不是 zap 的锅

### 凌晨 2 点那条日志
```json
{"error":"dial tcp 10.0.1.5:6379: connection refused"}
```
看起来是"zap 不给力,没 stack"。**其实不是**。

### 按 01 错误处理规范做,zap 自动就有 stack

**边界层用 `pkg/errors.Wrap`** —— 第一次和外部交互时捕获 `runtime.Callers`:
```go
// DAO 层
if err := db.QueryContext(ctx, ...); err != nil {
    return errors.Wrap(err, fmt.Sprintf("dao get user id=%d", id))
}
```

**层间用 `fmt.Errorf("%w")` 带参数** —— 上下文链本身就是"人类可读的 stack":
```go
return fmt.Errorf("login user id=%d: %w", uid, err)
```

**zap 自动输出 stack**:
```go
logger.Error("login fail", zap.Error(err))
// {
//   "error":"login user id=123: dao get user id=123: sql: no rows",
//   "errorVerbose":"...完整堆栈..."   ← ★ zap 识别 pkg/errors 自动加
// }
```

### 90% 场景下,wrap 链上下文比 stack 更好用

`"login user id=123: dao get user id=123: sql: no rows"` —— **一眼看出业务流向 + 出错数据**。纯 stack 只有函数名 + 行号,还要去读代码猜。

### 结论

- **错误处理规范做好 → zap 自动 OK**,不是 zap 痛点
- 规范做不好 → slog 也救不了
- 这页只为了呼应一个常见误解 —— 不是转投 slog 的理由
> <span class="mute">联动 01-error-handling 的 "错误在下层带上现场" 原则。</span>

---

## "全量脱敏" —— 勉强算痛,非质变

### 合规 3 天内要你全量脱敏,方案其实都能做

| 方案 | 代码量 | 一次工作量 |
|---|---|---|
| A · `sed` 改 200 处调用点 | 1 个脚本 | 半天 + regression test |
| B · zap 写 `RedactCore` | 30-40 行 | 半天,要搞 `Field.Type` 分支 |
| C · slog 写 `RedactHandler` | 10 行 | 一小时 |

**都能做完**。Core 也没那么难,新手半天也能写出来。

### 真正差距不在代码行数,在**生态可组合性**

```go
// slog 的 Handler 天然可以和别的 Handler 叠加
handler := slogmulti.Fanout(
    slogmulti.Pipe(
        &RedactHandler{}, &SamplingHandler{rate: 10},
    ).Handler(slog.NewJSONHandler(os.Stdout, nil)),
    slogsentry.Option{}.NewSentryHandler(),
    slogloki.Option{}.NewLokiHandler(),
)
// 脱敏 + 采样 + 多家输出,全用同一个 slog.Handler 接口
```

zap 的 Core 也能串,但每家第三方接入方式不统一(Hook / WriteSyncer / 自定义 Core),**组合成本高**。

### 结论

- "写 Core 难" 本身**不是质变**,算不上 zap 的硬痛
- 真正值得说的是 slog 的 `samber/slog-*` 生态更统一、可组合
- 但这个优势**只在接多家 / 做多种横切时**才有感
- 大多数团队:用不上,**不构成转投理由**

---

## 真痛(场景窄)· SDK 绑了 zap,你就动弹不得

> **这条只对写公共库 / 基础 SDK 的人疼**。只写业务应用的团队,go.mod 里多一条 zap 根本没感觉 —— 跳过也没关系。

**真实场景**:公司内部 `auth-sdk` 做统一鉴权,业务服务都在用它:

```go
import "internal.company.com/auth-sdk/v1"
// auth-sdk 的 go.mod 里:
//   require go.uber.org/zap v1.21.0
```

有一天你想:
- **升级 zap 到 v1.27** (新特性 / 修 CVE / 修 race) → sdk 还在 v1.21,你升不了。等 sdk 维护者发新版
- **换成 zerolog** (性能考虑,极致场景) → sdk 的 `auth.NewWithZap(logger *zap.Logger)` API 打破了,**30 个业务服务联动改**
- **本地单测 mock 掉日志** → sdk 内部用了全局 `zap.L()`,mock 不干净

**痛感定位**:这个痛**对写公共库的人疼到极点**,对只写业务的人没感觉。

### slog 方案

SDK 签名只依赖标准库:
```go
func NewClient(opts ...Option) *Client
func WithLogger(l *slog.Logger) Option
```

业务想换底层 logger?
```go
// 用 zap 底层
handler := slogzap.Option{Logger: zapLogger}.NewZapHandler()
auth := sdk.NewClient(sdk.WithLogger(slog.New(handler)))

// 换 zerolog 底层?换 adapter 就行,SDK 一行不动
```

**SDK 作者解脱,业务方也不再被迫选边**。这是**标准库身份**带来的解放。

---

## Part 2 · `zerolog` (2017) · 把零分配做到极致

关键想法:**把 JSON 字节直接拼在 Event 自己的 `[]byte` 里**,连 Encoder 分派都省。

```go
type Event struct { buf []byte; ... }

func (e *Event) Str(key, val string) *Event {
    if e == nil { return e }
    e.buf = enc.AppendKey(e.buf, key)
    e.buf = enc.AppendString(e.buf, val)
    return e                        // ★ 返回自己,零分配
}

// 链式:log.Info().Str("uid", "1").Int("age", 30).Msg("hi")
// → 一路 append 到 buf,.Msg() 时刷 Writer + 放回 pool
```

benchmark 常比 zap 快 10-30%。**代价**:

---

## Part 2 · `zerolog` · 凌晨 3 点的线上故障(真事)

某个 feature flag 下的分支,开发漏了 `.Msg()`:

```go
if featureEnabled(user) {
    log.Info().Str("uid", uid).Str("action", "bonus_credit").Int("amount", amt)
    // ↑ 忘了 .Msg("bonus credited")!
}
```

结果:

- **线上跑了两周没日志**
- 用户投诉"钱没到账",查 ELK 这个 path **搜索结果为空**
- 编译 OK · 单测 OK · code review 也没人发现

zerolog 的 `.Msg()` 是**触发写入**那一步,忘记 = 静默丢日志,**linter 抓不到**。其他局限:动态字段(来自 map)不友好、Context 派生语法啰嗦。

---

## Part 2 · `slog` (2023) · 五大核心类型

```go
// 1. 用户面向的 API
type Logger struct { handler Handler }

// 2. 可插拔后端 —— slog 的核心
type Handler interface {
    Enabled(context.Context, Level) bool
    Handle(context.Context, Record) error            // ★ ctx 是一等公民
    WithAttrs(attrs []Attr) Handler
    WithGroup(name string) Handler
}

// 3. 一条日志事件 —— inline 数组优化
type Record struct {
    Time time.Time; Message string; Level Level; PC uintptr
    front  [nAttrsInline]Attr    // 通常 5 —— 小量字段零 slice 分配
    nFront int
    back   []Attr                // 溢出才分配
}

// 4. 一个键值对
type Attr struct { Key string; Value Value }

// 5. kind-discriminated union,避免 interface{} 装箱
type Value struct {
    num uint64                    // 整型/bool/duration 塞这里
    any any                       // 字符串头 / 复杂类型
}
```

**和 zap 的 `Field` 设计思路一致**,但把 ctx 推到 Handler 接口里。

---

## Part 3 · slog 立项依据 · Go 官方给了 4 个公开论据

> 来源:[proposal #56345](https://github.com/golang/go/issues/56345) · [discussion #54763](https://github.com/golang/go/discussions/54763) · [Go blog](https://go.dev/blog/slog) · [GopherCon 2023 · Jonathan Amsterdam](https://www.youtube.com/watch?v=tC4Jt3i62ns)

| # | 论据 | 大意 |
|---|------|------|
| 1 | **生态分裂,库作者被迫选边** | "Library authors must either adopt one of these packages (forcing that choice on their users) or use the standard `log` package, which has no structure." |
| 2 | **目标是共通接口,不是替代实现** | "A key design principle is a clean separation of the front-end API from the back-end handler." |
| 3 | **性能不牺牲** | Handler 模式允许零分配实现(benchmark 与 zap 同量级) |
| 4 | **Context 是一等公民** | `Handle(ctx, Record)` 写进 Handler 接口 —— 让"从 ctx 动态取字段"天然可行 |

> zap / zerolog / logrus 的维护者都参与了设计讨论。`slog-zap` 不是逆向工程,是**设计阶段就预留的对接点**。

---

## Part 3 · slog vs zap 的 8 个场景 · 诚实评分

> 经过一轮自我攻防,真正站得住的差距**只有 2 条**。其他要么是规范问题,要么是边际优势。

| 场景 | 客观评分 |
|------|---------|
| **1 ctx 动态字段**(trace_id) | <span class="good">**✅ 真优势**</span> —— API 根本差距,不可回避 |
| 2 error 富化(stack + kind) | ❌ **不是 zap 痛**,按 01 错误处理规范做就行 |
| 3 动态 level | 打平(zap 甚至更顺手) |
| 4 敏感字段脱敏 | ⚠️ **勉强** —— Core 40 行 vs Handler 10 行非质变,差距在生态 |
| 5 接 Sentry/Loki/OTel | ⚠️ 窄 —— 接多家才有感,多数团队只接一家 |
| 6 单元测试 | ⚠️ 轻微 —— zaptest/observer 也够用 |
| **7 库不绑后端** | <span class="good">**✅ 真优势(对库作者)**</span> —— 标准库身份硬核 |
| 8 dev vs prod 格式 | 边际 —— 两边都能做 |

### 关键认识

**API 层面**:只有 **1(ctx)** 是 zap 不能优雅追上的硬伤。
**生态层面**:**7(库作者解耦)** + **标准库身份** 构成另外两条硬理由。

其他都是锦上添花 —— **有就更好,没有也活**。

---

## Part 3 · 场景详讲 1 · ctx → trace_id

**zap 方案**(主流):
```go
// middleware 注入 logger 进 ctx
l := zap.L().With(zap.String("trace_id", tid))
ctx := context.WithValue(r.Context(), loggerKey{}, l)
// 业务代码
L(ctx).Info("...")                              // L() 是团队自己写的 helper
```

**slog 方案**:
```go
type TraceHandler struct{ slog.Handler }
func (h *TraceHandler) Handle(ctx context.Context, r slog.Record) error {
    if sc := trace.SpanContextFromContext(ctx); sc.IsValid() {
        r.AddAttrs(slog.String("trace_id", sc.TraceID().String()))
    }
    return h.Handler.Handle(ctx, r)
}

// 业务代码直接 slog.InfoContext(ctx, "...")
```

| | zap(主流) | slog |
|---|---|---|
| 业务代码 | `L(ctx).Info(...)` | `slog.InfoContext(ctx, ...)` |
| 入口管道 | 每协议写 MW 注入 | 不需要,Handler 从 ctx 取 |
| helper | ~20 行 | 0 |

**slog 的优势**:Handle 第一参数就是 ctx。zap 也能做但需要管道配合。

---

## Part 3 · 场景详讲 2 · error 日志的 4 条最佳实践

| # | 实践 | zap | slog |
|---|------|-----|------|
| 1 | **err 作为结构化字段** | ✅ `zap.Error(err)` | ✅ `slog.Any("err", err)` |
| 2 | **保留 wrap 链文本**(`%w` 链) | ✅ `err.Error()` 自动 | ✅ 自动 |
| 3 | **带 stack** —— 定位第一现场 | ⚠️ `zap.Stack` 或 pkg/errors | ⚠️ 同上 |
| 4 | **根因类型可识别** —— 告警 group-by | ❌ 手动 `errors.Is/As` | ❌ 同上 |

**BP 1、2 两边都免费**。差距在 **BP 3、4 想不想"在日志管道里统一装"**:

- 手动加:`zap.Error(err), zap.Stack("stack"), zap.String("err_kind", classify(err))` 每处一遍,新人大概率忘
- 想**一次部署全局生效**?→ 需要写"日志中间件",回到场景痛 3 的 zap `Core` vs slog `Handler` 对比

### 和错误处理章节打通

```go
func classifyErr(err error) string {
    var e *ecode.Error
    if errors.As(err, &e) { return e.Reason }   // "USER_NOT_FOUND"
    return "internal"
}
```

→ **日志 `err_kind`、HTTP/gRPC 响应的 code、告警 dashboard 的 group-by,三处共用一套 `reason`**。这是错误处理规范 + slog 中间件的协同收益。

<span class="mute">（zap `ErrEnrichCore` 实现细节与场景痛 3 的脱敏 Core 类似，只是把分支换成 `Field.Type == zapcore.ErrorType`。代码差一半，核心是 slog 的 `Handle(ctx, Record)` 接口让拦截和改写更直白。）</span>

---

## Part 3 · Handler 真的好写吗 · 30 行模板

**90% 的 Handler 只需要定制 `Handle`**,其他三个是样板:

```go
type MyHandler struct {
    slog.Handler   // 嵌入现成的,白嫖 Enabled/WithAttrs/WithGroup 默认
}

// 只改这个 —— 你的定制逻辑
func (h *MyHandler) Handle(ctx context.Context, r slog.Record) error {
    r.AddAttrs(slog.String("my_field", "my_value"))  // 加字段/脱敏/转发/采样
    return h.Handler.Handle(ctx, r)
}

// 下面两个是样板:让 With 返回的还是 *MyHandler
func (h *MyHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return &MyHandler{h.Handler.WithAttrs(attrs)}
}
func (h *MyHandler) WithGroup(name string) slog.Handler {
    return &MyHandler{h.Handler.WithGroup(name)}
}
```

**就这样**。PodHandler / ErrEnrichHandler / recordCapture / 脱敏 —— 都是这个模板的变体,只改 `Handle`。

> 不想自己写?`samber/slog-multi`、`slog-zap`、`slog-sentry`、`slog-loki`、`phsym/console-slog` 已经把 90% 的常用场景包好了。

---

<!-- _class: lead -->

## 真正硬的只有 3 条

### ① 标准库身份(生态层 · 不可替代)
- 新项目**选型成本 = 0** —— 不用开会比 zap / logrus / zerolog
- 2024+ 新 SDK / 框架**默认接 `*slog.Logger`**,跟着走就享受红利
- 新人学一套就够,教程、官博、权威书都以 slog 为例
- <span class="bad">zap 永远追不上这一条</span>

### ② Context 一等公民(API 层 · 根本差距)
- `Handle(ctx, Record)` 写进 Handler 接口 —— ctx 天然到达日志管道
- `slog.InfoContext(ctx, ...)` 是标准库 API —— 不用自己造 `L(ctx)` helper
- <span class="bad">zap 靠管道配合能做到"一样好",但永远是补丁</span>

### ③ 写公共库 / SDK(硬要求 · 场景窄)
- 只依赖标准库 = **不绑架调用方**
- `slog-zap`、`slog-zerolog` 让调用方爱选啥选啥
- <span class="bad">zap 再快,基础库 import 它 = 所有下游被迫选边</span>

<br>

> 其他所谓"优势" —— **没它们也能活**。诚实面对这点,转投才不是盲目跟风。

---

## Part 4 · 反方意见 · 什么情况下**不必**换 slog

站到 zap 那边给自己打几拳:

| 声称的 slog 优势 | 反驳 |
|---|---|
| ctx → 动态字段 | 99% 团队 middleware + `L(ctx)` 足够,trace_id 请求内不变 |
| error 自动富化 | 多数团队**不应该**做(日志变肿、噪音大),error 观测交给 APM/Sentry |
| 接多家 | 大多数只接一家日志系统 |
| 库不绑后端 | 大多数团队**不写公共库** |
| 动态 level / 单测 / 格式 | zap 也能做,差别小 |

### 真正应该换的只有两条

1. **你写公共库/SDK** —— 不想绑架调用方,只能依赖标准库 → slog 唯一解
2. **新项目从零** —— 选 zap 还是 slog 的讨论成本 > 随便选的成本

**其他情况**:zap 用得好就继续用。迟早要迁? 先让业务代码接 `*slog.Logger` 接口,底层走 `slog-zap` adapter —— 迁接口不迁实现。

---

## Part 4 · 决策树

```
 ┌────────────────────────────────┐
 │ 新项目?                          │
 │   是 → slog(避免选型会议)        │
 │   否 ↓                           │
 │                                  │
 │ 写公共库/SDK?                    │
 │   是 → slog(硬要求,不绑架调用方) │
 │   否 ↓                           │
 │                                  │
 │ zap 用得好吗?                    │
 │   是 → 不换,业务代码抽象接口      │
 │   否 ↓                           │
 │                                  │
 │ 有多种 Handler 富化需求?          │
 │   是 → slog(Handler 短一半)      │
 │   否 → 其实 logrus 都够用          │
 └────────────────────────────────┘
```

> **slog 的价值是"标准",不是"更好"**。生态收敛本身就是最大红利。

---

## Part 5 · 横向对比 · benchmark 数字

10 个字段,JSON 输出,单次调用(量级参考,±20% 机器波动):

| 库 | ns/op | allocs/op | 备注 |
|---|---|---|---|
| `log` (stdlib) | ~1500 | 8 | 拼文本,本就不做 structured |
| `logrus` | ~3000 | 23 | map + reflect |
| `zap.Logger` | ~200 | 0 | 类型化 Field |
| `zap.Sugared` | ~500 | 1 | interface{} 装箱一次 |
| `zerolog` | ~150 | 0 | Event 直接 append |
| `slog.LogAttrs` | ~300 | 0 | 和 zap 同量级 |
| `slog.Info`(kv 可变参) | ~500 | 1 | 变参 type switch |

**结论**:
- slog 和 zap 在一个量级,日常业务差距可忽略
- logrus 比 slog 慢 **10 倍**,高并发项目升级有实质 QPS 收益
- 极致性能场景:`slog-zap` 底层,业务接 slog 接口

---

## Part 5 · 迁移路径 · 3 条

**路线 1:logrus → slog(推荐)**
```
log.WithFields(log.Fields{"k": v}).Info("msg")
           ↓
slog.Info("msg", "k", v)
```
过渡期用 `slog-logrus` Hook 做影子双写,稳定后切主。

**路线 2:zap → slog(保留底层)**
```go
zapLogger, _ := zap.NewProduction()
h := slogzap.Option{Level: slog.LevelInfo, Logger: zapLogger}.NewZapHandler()
slog.SetDefault(slog.New(h))
// 业务代码从此只用 slog API,性能保 zap
```

**路线 3:库层彻底解耦**
```go
// 旧:type Client struct { logger *zap.Logger }   ← 绑架调用方
// 新:type Client struct { logger *slog.Logger }  ← 谁都能接
```

### 风险点

- 字段命名不一致 → 统一 `logger.With` 规范
- 格式变化 → 过渡期 Handler 按老格式
- 性能回退 → logrus→slog 涨 · zap→slog(adapter)持平 · 裸 JSONHandler 约 -30%

---

## 诚实版总结

- **log (2012)**:文本时代终点 —— 没 level、没结构化
- **logrus (2014)**:structured 开端,`map + reflect` 慢 10 倍
- **zap (2016)**:零分配达成。**API 层面真正跟不上 slog 的只有 ctx 一等公民这一条**;其他常说的痛(stack、脱敏、两套 API)其实不是 zap 的锅
- **zerolog (2017)**:极致零分配 + 链式 API 陷阱
- **slog (2023)**:真正硬的只有 3 条 —— **标准库身份 · ctx 一等公民 · 库作者零绑架**

### 所以该不该换

| 情况 | 做什么 |
|---|---|
| 新项目从零 | <span class="good">slog</span>(避免选型会议) |
| 写公共库 / SDK | <span class="good">slog</span>(硬要求) |
| 老项目 zap 用得好 | **不换**,业务代码慢慢往 `*slog.Logger` 接口靠 |
| 老项目 logrus | 有性能压力就换,否则可等 |

> **slog 的价值是"标准",不是"更好"**。承认这点,转投才不盲目跟风。

---

<!-- _class: lead -->

# Q & A

<span class="mute">欢迎拍砖 · 你踩过哪代日志库的坑?</span>

<br>

> 完整源码级剖析 → [deep-dive.md](./deep-dive.md)
> 主分享 deck → [slides.md](./slides.md)
> 分层专题 → [layering-slides.md](./layering-slides.md)
