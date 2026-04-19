# Go 日志库深度剖析 · 从 `log` 到 `slog` 的十四年

> 读完你能得到什么
>
> 1. 每一代日志库的**设计目标**、**核心数据结构**、**性能关键路径**
> 2. 为什么 logrus 慢、为什么 zap 快、为什么 zerolog 还能更快
> 3. slog 的三个要点:**Logger ↔ Handler 解耦**、**LogValuer**、**零依赖** —— 这三点为什么让它成为 2026 的默认选择
> 4. 从 logrus / zap 迁移到 slog 的可操作路径

文档按时间顺序讲,每一代都会涉及"设计 → 源码原理 → 实战代码 → 局限"四层。slog 单独拎出来深讲。

---

## 目录

- [第 0 章 · 为什么日志库会迭代](#第-0-章--为什么日志库会迭代)
- [第 1 章 · `log` (2012,stdlib)](#第-1-章--log-2012stdlib)
- [第 2 章 · `logrus` (2014) — structured 时代开端](#第-2-章--logrus-2014--structured-时代开端)
- [第 3 章 · `zap` (2016) — 零分配标杆](#第-3-章--zap-2016--零分配标杆)
- [第 4 章 · `zerolog` (2017) — 把零分配做到极致](#第-4-章--zerolog-2017--把零分配做到极致)
- [第 5 章 · `log/slog` (2023) — 标准库大一统](#第-5-章--logslog-2023--标准库大一统)
- [第 6 章 · slog 到底解决了什么 —— 8 个场景下的对比](#第-6-章--slog-到底解决了什么--8-个场景下的对比)
- [第 7 章 · 横向对比 + 基准数据](#第-7-章--横向对比--基准数据)
- [第 8 章 · 迁移路径](#第-8-章--迁移路径)
- [附录](#附录)

---

## 第 0 章 · 为什么日志库会迭代

日志库要解决的核心矛盾,其实就 3 条:

| # | 矛盾 | 典型表现 |
|---|------|---------|
| 1 | **可读性 vs 可机读性** | 人想看文本,机器想要 JSON |
| 2 | **表达力 vs 性能** | structured 字段越多越好查,但分配越多越慢 |
| 3 | **易用性 vs 可控性** | 全局 logger 方便但难注入;依赖注入干净但样板代码多 |

每一代日志库,都是在这三个维度上做不同权衡:

```
log       可读 ✓  机读 ✗  表达力 ✗  性能 ✓  易用 ✓
logrus    可读 ✓  机读 ✓  表达力 ✓  性能 ✗  易用 ✓
zap       可读 △  机读 ✓  表达力 ✓  性能 ✓  易用 △ (两套 API)
zerolog   可读 △  机读 ✓  表达力 ✓  性能 ✓✓ 易用 △
slog      可读 ✓  机读 ✓  表达力 ✓  性能 ✓  易用 ✓  (标准库 + 可插拔)
```

slog 不是性能冠军(zap/zerolog 还是略快),但它第一次把五个维度同时做到"够好",且进了标准库。

---

## 第 1 章 · `log` (2012,stdlib)

### 设计目标

Go 1.0 的 `log` 包,目标非常朴素:**"替代 `fmt.Printf`,多加个时间戳和 file:line"**。

### 核心数据结构

`log.Logger` 就是一个加了锁的 `io.Writer` 包装:

```go
// 简化版,真实代码在 src/log/log.go
type Logger struct {
    outMu  sync.Mutex          // 保护输出
    prefix atomic.Pointer[string]
    flag   atomic.Int32
    out    io.Writer
    // ...
}
```

核心方法 `Output` 就干一件事:**把 header(时间 + 前缀 + file:line)和消息拼进一个 buffer,加锁写 Writer**。

```go
func (l *Logger) Output(calldepth int, s string) error {
    now := time.Now()
    var file string
    var line int
    if l.flag.Load()&(Lshortfile|Llongfile) != 0 {
        _, file, line, _ = runtime.Caller(calldepth)
    }
    buf := getBuffer()
    defer putBuffer(buf)
    formatHeader(buf, now, l.prefix.Load(), l.flag.Load(), file, line)
    *buf = append(*buf, s...)
    if len(s) == 0 || s[len(s)-1] != '\n' {
        *buf = append(*buf, '\n')
    }
    l.outMu.Lock()
    defer l.outMu.Unlock()
    _, err := l.out.Write(*buf)
    return err
}
```

### API 一瞥

```go
log.Print("server started")                     // INFO 类
log.Printf("listen on %s", addr)
log.Fatalf("db fail: %v", err)                  // 打完调 os.Exit(1)
log.Panicf("bug: %v", err)                      // 打完 panic

// 自定义 logger
l := log.New(os.Stdout, "[worker] ", log.LstdFlags|log.Lshortfile)
l.Println("job done")
```

### 局限 —— 为什么 2014 之后就不够用了

1. **没有 Level** —— 想区分 debug/info/error,只能拼前缀
2. **没有结构化** —— 全是文本,线上日志聚合后 grep 肉眼累死
3. **`Fatal` 不走 `defer`** —— 直接 `os.Exit`,资源泄漏
4. **全局状态** —— `log.SetOutput` 改一个全局 Logger,测试隔离难

这些局限,催生了 logrus。

---

## 第 2 章 · `logrus` (2014) — structured 时代开端

### 身临其境:2015 年你正在踩的坑

你给公司用户服务加日志，刚把 logrus 用起来，API 特别顺手：

```go
log.WithFields(log.Fields{
    "user_id": uid, "action": "login", "ip": ip,
}).Info("user login")
```

code review 一片好评。直到压测那天：

- **QPS 上不去 3 万**，服务 CPU 跑满
- 打开 pprof 火焰图：**30%+ CPU** 消耗在 `runtime.mapassign_faststr` 和 `reflect.Value.Interface`
- heap 采样：分配热点全是 `map[string]interface{}` 和 `*logrus.Entry`

你去 Uber 的技术博客找优化办法，发现他们已经在自己造轮子了。那个轮子后来叫 **zap**。

### 设计目标

**"给 stdlib log 加上 level 和 structured fields，API 尽量兼容"**。

logrus 的 tagline 至今还是 `Structured, pluggable logging for Go`。它在 2014-2018 之间几乎是 Go 后端的事实标准。

### 核心数据结构:`Entry` 和 `Logger`

```go
// 简化版
type Logger struct {
    Out          io.Writer
    Hooks        LevelHooks
    Formatter    Formatter
    Level        Level
    mu           MutexWrap
    entryPool    sync.Pool
    // ...
}

type Entry struct {
    Logger  *Logger
    Data    Fields            // map[string]interface{} ← ★ 性能瓶颈
    Time    time.Time
    Level   Level
    Caller  *runtime.Frame
    Message string
    Buffer  *bytes.Buffer
    Context context.Context
}

type Fields map[string]interface{}
```

`Entry` 是"一条日志事件",`Logger` 是全局配置。

### 关键机制:`WithFields` 怎么工作

```go
logger.WithField("user_id", 123).
       WithField("action", "login").
       Info("user login")
```

源码逻辑:

```go
func (entry *Entry) WithFields(fields Fields) *Entry {
    // 每次 With 都新建一个 map,并把老 map 的内容复制过来
    data := make(Fields, len(entry.Data)+len(fields))
    for k, v := range entry.Data {
        data[k] = v
    }
    for k, v := range fields {
        // fields 的 value 是 interface{},可能是 error
        if errVal, ok := v.(error); ok {
            data[k] = errVal.Error()
        } else {
            data[k] = v
        }
    }
    return &Entry{Logger: entry.Logger, Data: data, Time: entry.Time, ...}
}
```

**两次分配**:新 map + 新 Entry。 链式调用 5 次 `WithField` = 5 个 map + 5 个 Entry。

### Formatter:序列化那一步

```go
type Formatter interface {
    Format(*Entry) ([]byte, error)
}

// JSONFormatter 简化实现
func (f *JSONFormatter) Format(entry *Entry) ([]byte, error) {
    data := make(Fields, len(entry.Data)+4)
    for k, v := range entry.Data {
        switch v := v.(type) {
        case error:
            data[k] = v.Error()
        default:
            data[k] = v
        }
    }
    data["time"] = entry.Time.Format(f.TimestampFormat)
    data["level"] = entry.Level.String()
    data["msg"] = entry.Message
    // ...
    b, err := json.Marshal(data)  // ← ★ encoding/json 走反射
    return append(b, '\n'), err
}
```

**性能瓶颈**:
1. 每条日志分配至少一个 `map[string]interface{}`
2. `json.Marshal` 走反射,每个字段 type switch + reflect Call
3. `interface{}` 装箱,整数/bool 也要走堆

### Hook 机制

logrus 提供 Hook 扩展点,可以在日志落地之前做异步操作(发 Kafka、接入 Sentry):

```go
type Hook interface {
    Levels() []Level
    Fire(*Entry) error
}

logger.AddHook(&SentryHook{dsn: "..."})
```

Hook 是 logrus 生态的亮点,但现在更常见的做法是把采集/分发推到日志 agent 层,不在应用里做。

### 实战代码

```go
import log "github.com/sirupsen/logrus"

func init() {
    log.SetFormatter(&log.JSONFormatter{})
    log.SetLevel(log.InfoLevel)
}

func main() {
    log.WithFields(log.Fields{
        "user_id": 123,
        "action":  "login",
        "ip":      "1.2.3.4",
    }).Info("user login")
}
```

### 局限

- **性能差**:基准测试里比 zap 慢 ~10 倍,map + 反射是主因
- **API 双栈**:`logger.WithField` 链式 vs `logger.Info` 直接
- **维护节奏慢**:sirupsen/logrus 已明确声明 "in maintenance mode",不接重大新 feature

这两个原因,让 zap 在 2016 诞生时一炮而红。

---

## 第 3 章 · `zap` (2016) — 零分配标杆

### 设计目标

Uber 内部日志量巨大,需要**"结构化 + 极致性能 + 工程级可配置"**。zap 的设计哲学是:

> **"Avoid `reflect` and `interface{}` in the hot path."**

### 核心洞察 1:`Field` 不用 `interface{}`

logrus 的 `Fields` 是 `map[string]interface{}`,每个值都要装箱。zap 观察到:**大部分字段其实是原生类型**(string/int/bool/time),只要为它们各开一个 struct 字段,就能绕开 `interface{}`。

```go
// zap.Field —— 关键:它是 struct 不是 interface
type Field struct {
    Key       string
    Type      FieldType   // 枚举,指示哪个字段有效
    Integer   int64       // int/int64/bool/duration 都塞这里
    String    string      // 字符串
    Interface interface{} // 只有复杂类型(slice/struct)才用
}

// 构造函数
func Int(key string, val int) Field {
    return Field{Key: key, Type: Int64Type, Integer: int64(val)}
}
func String(key, val string) Field {
    return Field{Key: key, Type: StringType, String: val}
}
func Bool(key string, val bool) Field {
    return Field{Key: key, Type: BoolType, Integer: b2i(val)}
}
```

调用方:

```go
logger.Info("user login",
    zap.Int("user_id", 123),
    zap.String("ip", "1.2.3.4"),
    zap.Bool("first_login", true),
)
```

**原生类型零装箱**。只有 `zap.Any("foo", complexStruct)` 才走 `interface{}` + 反射。

### 核心洞察 2:Encoder 直接写 buffer

logrus 先构造 `map` 再 `json.Marshal`,zap 的 Encoder 一次写 buffer:

```go
type Encoder interface {
    AddBool(key string, val bool)
    AddInt(key string, val int)
    AddString(key, val string)
    AddTime(key string, val time.Time)
    AddDuration(key string, val time.Duration)
    AddObject(key string, val ObjectMarshaler) error
    // ... 每个原生类型一个方法
    EncodeEntry(Entry, []Field) (*buffer.Buffer, error)
}

// jsonEncoder.AddString 的简化实现
func (e *jsonEncoder) AddString(key, val string) {
    e.addKey(key)
    e.buf.AppendByte('"')
    e.safeAddString(val)
    e.buf.AppendByte('"')
}
```

**按类型分派,直接往 `*buffer.Buffer` 里追加字节**,不走 `json.Marshal` 的反射路径。

### 核心洞察 3:buffer.Pool 复用

```go
// buffer 是 []byte 的封装,通过 sync.Pool 复用
var _pool = buffer.NewPool()

func (enc *jsonEncoder) EncodeEntry(ent Entry, fields []Field) (*buffer.Buffer, error) {
    final := _pool.Get()                 // ← 从池里拿
    // ... 写入字段
    return final, nil                    // 调用方用完再 final.Free()
}
```

**单条日志几乎零堆分配**。

### Core 管道

zap 把日志流程抽象为 Core:

```go
type Core interface {
    Enabled(Level) bool
    With([]Field) Core
    Check(Entry, *CheckedEntry) *CheckedEntry
    Write(Entry, []Field) error
    Sync() error
}
```

- `Check` 决定是否写(level + sampling)
- `Write` 做实际写入
- `With` 派生带固定字段的 Core

可以组合 Core 实现:
- **Tee**:同时写多处(file + stdout)
- **Sampling**:高频日志按百分比采样
- **Level gate**:只放过某个级别

```go
core := zapcore.NewTee(
    zapcore.NewCore(jsonEncoder, fileWS, zap.InfoLevel),
    zapcore.NewCore(consoleEncoder, stdoutWS, zap.DebugLevel),
)
logger := zap.New(core)
```

### Logger vs SugaredLogger

zap 两套 API,对应两种场景:

```go
// 1. 类型化 Logger —— 快,但啰嗦
logger := zap.NewProduction()
logger.Info("user login",
    zap.Int("user_id", 123),
    zap.String("ip", ip),
)

// 2. SugaredLogger —— 方便,稍慢(interface{})
sugar := logger.Sugar()
sugar.Infow("user login", "user_id", 123, "ip", ip)   // 类似 slog
sugar.Infof("user login %d", 123)                      // printf 风格
```

SugaredLogger 内部也是转成 `[]Field` 再交给 Core,多一次 `interface{}` → `Field` 的 switch。

### zap 的真实痛点(分场景客观评估)

性能问题 zap 解决得很好。下面列出的是**团队转 slog 时会提到的理由**，但每条都应该放到具体场景下评估。为了避免"强塞痛点"，下面按**共性痛 / 场景痛 / 习惯痛**分级：

- **共性痛**：绝大多数团队都会遇到
- **场景痛**：只有在做某类事情时才疼
- **习惯痛**：换种使用方式就能缓解，不是 zap 的"原罪"

#### 痛点 1 · 两套 API 撕裂，新人选择困难

```go
// Team A 写法（快，啰嗦）
logger.Info("user login",
    zap.Int("uid", uid),
    zap.String("ip", ip),
)

// Team B 写法（像 slog，但慢）
sugar.Infow("user login", "uid", uid, "ip", ip)
```

体感：SugaredLogger 内部要把 `interface{}` 做 type switch 转成 `Field`，benchmark 上比 Logger 慢 2-3 倍（大致 ~500ns vs ~200ns，仍然比 logrus 的 ~3000ns 快一个量级，不是"接近 logrus"，只是"放弃了一半性能红利"）。团队一旦混用两种写法，code style 难统一，新人常困惑选哪个。

**是否是痛**：如果团队有明确规范（统一 Logger 或统一 Sugared），其实不痛。混用才是痛。

#### 痛点 2 · error 日志默认信息量不足

```go
err := fmt.Errorf("get user id=%d: %w", id, sql.ErrNoRows)
logger.Error("login fail", zap.Error(err))
```

输出：
```json
{"level":"error","msg":"login fail","error":"get user id=123: sql: no rows in result set"}
```

**先澄清一个常见误解**：`zap.Error(err)` 会通过 `err.Error()` 把 `%w` wrap 链的完整字符串保留下来 —— 这点没问题。真正缺的是另外两件事：

1. **没有 stack** —— 不知道错误在哪个 file:line 产生。除非 err 来自 `pkg/errors`（这时 zap 会识别并自动加 `errorVerbose` 字段），否则要每处手动 `zap.Stack("stack")`。
2. **根因类型无法识别** —— 日志里看不出这是 `sql.ErrNoRows` / `context.DeadlineExceeded` / 某个业务 sentinel。想按根因告警 group-by，只能文本匹配，脆弱。

两件事都能做，但 zap 要**每处手动加**：

```go
logger.Error("login fail",
    zap.Error(err),
    zap.Stack("stack"),
    zap.String("err_kind", classify(err)),   // 自己写 classify
)
```

规范靠自觉，新人大概率忘。想让所有 error 日志**在 Handler 层自动富化**（加 stack + kind）？那就要写一个 `zapcore.Core` —— 直接对应**痛点 5**。

> error 进日志的四条最佳实践和 zap/slog 满足度，会在第 6 章场景 2 里展开。

#### 痛点 3 · Context 不是一等公民 —— 轻微，写个 helper 就解决

zap 没有 `logger.InfoContext(ctx, ...)` 方法。想把 ctx 里的 trace_id 带进日志，要自己写 helper：

```go
func L(ctx context.Context) *zap.Logger {
    if l, ok := ctx.Value(loggerKey{}).(*zap.Logger); ok { return l }
    return zap.L()
}
L(ctx).Info("...")
```

**10 行代码解决**。slog 把它写进标准库（`InfoContext`、`LogAttrs` 首参 ctx）是好事，统一了写法，但这不是"痛"，只是"官方化带来的规范性"。

真正的麻烦是如果团队没有统一 helper，5 个微服务 5 种写法 —— 但这是**团队纪律问题**，不是 zap 的问题。

#### 痛点 4 · 全局静态字段 —— **其实 zap 一行就能搞定，不算痛**

K8s 部署要求所有日志带 `pod_name`。这类**和请求无关的静态字段**（pod、env、region、build_sha），zap 的标准做法是 `ReplaceGlobals`，一行生效：

```go
pod := os.Getenv("HOSTNAME")
zap.ReplaceGlobals(rootLogger.With(zap.String("pod", pod)))

// 之后业务代码都用 zap.L()，自动带 pod 字段
zap.L().Info("user login", zap.Int("uid", 123))
```

**这不是 zap 的痛点**。slog 做同样的事也是 `slog.SetDefault(slog.New(h).With("pod", pod))`，代码差不多。

**真正有差距的是另一类需求**：字段不是静态的，而是**每条日志动态产生**，比如：

- 从 `ctx` 里取本次请求的 `trace_id`
- 给每条 error 日志自动加 stack + 根因分类
- 按日志路径采样 1%

这类"要在 Handle 时拦截 Record 做点事"的场景，zap 要写 `zapcore.Core`，slog 写个 Handler 就行（对应**痛点 5**）。

> **修正**：早先版本这里说 "每个入口都要 With 一次"，对静态字段不成立。真正的差距在动态字段/富化逻辑上，见痛点 5。

#### 痛点 5 · 想加个中间件（比如脱敏），要写整个 Core

你想让**所有日志**里的 `password`、`token` 字段自动变成 `***`。zap 的方案是实现一个 `zapcore.Core`：

```go
type RedactCore struct{ zapcore.Core }

func (c *RedactCore) Write(ent zapcore.Entry, fields []zapcore.Field) error {
    for i := range fields {
        if sensitive[fields[i].Key] {
            // 注意：不同 FieldType 存储在不同字段里
            switch fields[i].Type {
            case zapcore.StringType:
                fields[i].String = "***"
            case zapcore.ReflectType, zapcore.ErrorType:
                fields[i].Interface = "***"
            case zapcore.Int64Type, zapcore.Uint64Type:
                fields[i].Integer = 0
            // 还有十几种 Type...
            }
        }
    }
    return c.Core.Write(ent, fields)
}

func (c *RedactCore) With(fields []zapcore.Field) zapcore.Core {
    return &RedactCore{c.Core.With(fields)}
}

func (c *RedactCore) Check(ent zapcore.Entry, ce *zapcore.CheckedEntry) *zapcore.CheckedEntry {
    if c.Enabled(ent.Level) { return ce.AddCore(ent, c) }
    return ce
}
```

**三十多行，且 Field 的 Type 分支要照顾全** —— `Field` struct 里字符串、整数、复杂类型存在不同字段。第一次写容易漏分支，线上才发现某个类型没被脱敏。

#### 痛点 6 · 库依赖污染

你公司基础 SDK 里 `import "go.uber.org/zap"`。这意味着：

- 所有业务项目 `go.mod` 都多一条 `go.uber.org/zap`
- 升级 zap 版本要 30+ 服务同步
- 有服务卡在 zap v1.21，有的升到 v1.27，**SDK 发新版本时得小心兼容**
- 想测试用内存 logger 替换掉，单测要引入 `zap/zaptest`

**一个基础库的日志选择，绑架整条依赖链**。

---

#### 小结:zap 到底"值不值得"换掉

| 痛点 | 级别 | 能不能在 zap 内解决 |
|---|---|---|
| 1 两套 API | 共性（但可控） | 团队定规范就行 |
| 2 error 信息量 | 场景（要 Handler 统一富化才疼） | 能，但要写 Core |
| 3 Context 不一等 | 习惯 | 10 行 helper |
| 4 全局静态字段 | **不算痛** | `ReplaceGlobals` 一行 |
| 5 写 Core 富化 | 场景（真有需求才疼） | 能写，代码 30+ 行 |
| 6 库依赖污染 | 场景（对 SDK/基础库作者） | 无解 |

**转 slog 的真实理由通常是两条**：
- **你在写公共库/SDK**（痛点 6 无解）
- **你需要频繁写 Handler/Core 做富化**（痛点 5，代码成本差一半）

**如果你只是应用层写业务、有明确规范、不接多家第三方、日志不做额外富化 —— zap 本来就够用，不用换**。

---

## 第 4 章 · `zerolog` (2017) — 把零分配做到极致

### 设计目标

zap 已经很快了,zerolog 的 tagline 是:**"Zero Allocation JSON Logger"**。

它用了一个小而巧的想法:**把 JSON 字符串直接拼在 Event 自己的 `[]byte` 里**,不走 Encoder 接口分派。

### 核心数据结构:`Event`

```go
type Event struct {
    buf       []byte              // ← 日志 JSON 字节直接写这里
    w         LevelWriter
    level     Level
    done      func(msg string)
    stack     bool
    ch        []Hook
    skipFrame int
}

// 每个类型一个方法,直接 append
func (e *Event) Str(key, val string) *Event {
    if e == nil {
        return e
    }
    e.buf = enc.AppendKey(e.buf, key)
    e.buf = enc.AppendString(e.buf, val)
    return e
}

func (e *Event) Int(key string, i int) *Event {
    if e == nil {
        return e
    }
    e.buf = enc.AppendKey(e.buf, key)
    e.buf = strconv.AppendInt(e.buf, int64(i), 10)
    return e
}
```

**注意三点:**

1. 每个方法返回 `*Event`,支持链式:`log.Info().Str("k", "v").Int("i", 1).Msg("hello")`
2. 返回值**固定是 `e` 自己**,所以链式调用不分配
3. `e.buf` 是直接 `append`,连 buffer pool 都省了(Event 自己通过 sync.Pool 复用)

### 链式 API 的巧思:编译时类型分派

```go
log.Info().                               // *Event(从 pool 拿)
    Str("user_id", "123").                // 追加 `"user_id":"123"`
    Int("age", 30).                       // 追加 `,"age":30`
    Err(err).                             // 追加 `,"error":"..."`
    Msg("user registered")                // 追加 `,"message":"user registered"}` + 刷 Writer + 放回池
```

**运行时没有反射,没有 type switch,没有 interface{}** —— 链式每一环都是确定的 static call,编译器能完美内联。

实测 benchmark 里 zerolog 经常比 zap 快 10–30%。代价是:

- `if e == nil { return e }` 要写在每个方法开头(disabled logger 返回 nil)
- Event 有状态,`.Msg()` 之前的每一步都有可能忘调用,**忘了 `.Msg()` 整条日志不输出**

### Context:派生 logger 的方式

```go
logger := zerolog.New(os.Stdout).With().
    Str("service", "user").
    Timestamp().
    Logger()

// 带着 service=user 和时间戳的 sub-logger
logger.Info().Str("uid", "1").Msg("login")
```

`With()` 返回一个 Context,Context 上还能继续链式,最后 `.Logger()` 封回 Logger。

### 局限与真实坑

#### 凌晨 3 点的线上故障（真事）

某个 feature flag 下的分支，开发漏了 `.Msg()`：

```go
if featureEnabled(user) {
    log.Info().
        Str("uid", uid).
        Str("action", "bonus_credit").
        Int("amount", amt)                   // ← 忘了 .Msg("bonus credited")!
}
```

这段代码**在线上跑了两周**，直到有用户投诉"我钱咋没到账"，运维去查日志：

- 这个 feature 完全没日志
- ELK 里这个 path 搜索结果为空
- 编译 OK、单测 OK、code review 也没人发现

zerolog 的 `.Msg()` 是**触发写入**的那一步，忘记 = 静默丢日志。go vet 和主流 linter 目前都抓不到。

#### 其他局限

- **动态字段不友好**：字段来自 map 或 slice 时，得手写 for 循环 append
  ```go
  e := log.Info()
  for k, v := range fields { e = e.Interface(k, v) }   // 啰嗦
  e.Msg("...")
  ```
- **Context 派生语法**：`.With().Str(...).Str(...).Logger()` 比 slog 的 `logger.With(...)` 啰嗦
- 社区规模比 zap 小，中文资料少，招人时熟练用过的不多

---

## 第 5 章 · `log/slog` (2023) — 标准库大一统

### 设计目标

Go 团队在 2022 年立项 [Proposal: structured logging](https://github.com/golang/go/discussions/54763),目标:

> **"一个官方的、可插拔的、不牺牲性能的、不绑定具体实现的结构化日志接口。"**

2023 年 8 月随 Go 1.21 发布。

### 五个核心类型

```go
package slog

// 1. Logger —— 用户面向的 API
type Logger struct {
    handler Handler
}

// 2. Handler —— 可插拔后端接口
type Handler interface {
    Enabled(context.Context, Level) bool
    Handle(context.Context, Record) error
    WithAttrs(attrs []Attr) Handler
    WithGroup(name string) Handler
}

// 3. Record —— 一条日志事件
type Record struct {
    Time    time.Time
    Message string
    Level   Level
    PC      uintptr       // 调用点,用于拿 file:line

    // attrs 用 inline 数组 + overflow slice 存(避免小量字段时分配 slice)
    front  [nAttrsInline]Attr  // 通常 5
    nFront int
    back   []Attr
}

// 4. Attr —— 一个键值对
type Attr struct {
    Key   string
    Value Value
}

// 5. Value —— kind-discriminated union,避免 interface{} 装箱
type Value struct {
    _   [0]func()       // 阻止比较
    num uint64          // 整型/bool/duration/时间 都塞这里
    any any             // 字符串头 / 复杂类型
}

type Kind int
const (
    KindAny Kind = iota
    KindBool
    KindDuration
    KindFloat64
    KindInt64
    KindString
    KindTime
    KindUint64
    KindGroup
    KindLogValuer
)
```

**和 zap 的 `Field` 设计很像**:原生类型走 `num`,复杂类型走 `any`,避免装箱。

### Logger 的典型使用

```go
// 初始化
h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level:     slog.LevelInfo,
    AddSource: true,
})
logger := slog.New(h)
slog.SetDefault(logger)   // 设为默认,slog.Info 之类直接用

// 三种打日志姿势
slog.Info("user login", "uid", 123, "ip", ip)                              // key-value 可变参
slog.LogAttrs(ctx, slog.LevelInfo, "user login",                           // 强类型版,最快
    slog.Int("uid", 123), slog.String("ip", ip))
logger.With("trace_id", traceID).Info("login", "uid", 123)                 // 派生 logger
```

三种的区别:

| 方法 | 性能 | 可读性 | 使用场景 |
|---|---|---|---|
| `slog.Info("...", "k", v)` | 中(可变参 + 反射判断类型) | 高 | 一般业务 |
| `slog.LogAttrs(ctx, lvl, "...", slog.Int("k", v))` | 高(全类型化) | 中 | 性能敏感 |
| `logger.With(...).Info(...)` | 高(固定字段预编码) | 高 | 入口派生 |

### Record 的攻击性优化

```go
type Record struct {
    // ...
    front  [nAttrsInline]Attr  // 数组,inline 存储
    nFront int
    back   []Attr              // 溢出才分配 slice
}
```

小 ≤5 个字段时,**零 slice 分配**。比 logrus 的"每条日志一个 map"便宜得多。

### JSONHandler 的写路径

```go
// 简化
func (h *JSONHandler) Handle(ctx context.Context, r Record) error {
    buf := newBuffer()                    // sync.Pool
    defer buf.Free()

    buf.WriteByte('{')
    appendJSONAttr(buf, slog.Time("time", r.Time))
    appendJSONAttr(buf, slog.String("level", r.Level.String()))
    appendJSONAttr(buf, slog.String("msg", r.Message))
    r.Attrs(func(a Attr) bool {
        appendJSONAttr(buf, a)
        return true
    })
    buf.WriteByte('}')
    buf.WriteByte('\n')

    _, err := h.w.Write(buf.Bytes())
    return err
}
```

和 zap 的 Encoder 几乎一致:**pool + 直接拼字节,不走反射**。

### 性能:slog 和 zap 的差距

参考 [官方 benchmark](https://github.com/golang/go/blob/master/src/log/slog/benchmarks):

- `slog.LogAttrs`:~300 ns/op, 0 allocs/op
- `zap.Logger.Info`:~200 ns/op, 0 allocs/op
- `logrus.Info`:~3000 ns/op, 20+ allocs/op

slog 比 zap 慢约 30%,但**仍然在零分配区间**,对 99% 业务完全够用。想要 zap 的速度?下一节。

---

## 第 6 章 · slog 到底解决了什么 —— 8 个场景下的对比

上一章把 zap 的 6 个痛点摊开了。这一章不讲抽象的"三要点"，直接用团队日常的场景，看 **zap 怎么写、slog 怎么写**。你会看到 slog 的三个设计决策（Logger ↔ Handler 解耦、LogValuer、零依赖）在每个场景里自然体现。

> 阅读建议：每个场景都配了 "zap 怎么做" 和 "slog 怎么做" 两段代码。对比着看体感最强。

---

### 场景 1 · 给每条日志自动注入 `trace_id` (来自 ctx)

> 之前版本举的 `pod_name` 例子其实两个库都一行搞定（`zap.ReplaceGlobals(root.With(...))` / `slog.SetDefault(slog.New(h).With(...))`），那是**静态全局字段**，不是 slog 的真正优势场景。真正凸显差距的是**动态字段** —— 每条日志的值不一样，比如从 ctx 取 trace_id。

**需求**：每条日志自动带本次请求的 `trace_id`（从 OTel span 或自定义 ctx 值取）。**每条日志 trace_id 不同**，所以不能靠启动时 With 一次。

#### zap 方案 A · 入口 middleware 派生 logger 存 ctx

```go
// middleware
func LogMW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        tid := extractTraceID(r)
        l := zap.L().With(zap.String("trace_id", tid))
        ctx := context.WithValue(r.Context(), loggerKey{}, l)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
// 业务代码
L(ctx).Info("...")   // L() 是团队自己写的 ctx→logger helper
```

**能用**。代价：
- 业务代码里**每处都要调 `L(ctx)`**，不能直接 `zap.L()`（后者拿不到 trace_id）
- 每个入口（HTTP / gRPC / Cron / Kafka 消费者）都要写自己的 middleware，管 context 注入

#### zap 方案 B · 写 zapcore.Core 拦截 ctx 提取

`zapcore.Core.Write` **不拿 ctx**（zap 的 API 设计里 ctx 不到 Core 层）。所以这条路 zap 走不通。

#### slog 方案 · 写 Handler 在 Handle 里从 ctx 取

```go
type TraceHandler struct{ slog.Handler }

func (h *TraceHandler) Handle(ctx context.Context, r slog.Record) error {
    if sc := trace.SpanContextFromContext(ctx); sc.IsValid() {
        r.AddAttrs(slog.String("trace_id", sc.TraceID().String()))
    }
    return h.Handler.Handle(ctx, r)
}
// + WithAttrs / WithGroup 样板两行
```

业务代码直接 `slog.InfoContext(ctx, "...")`，**不需要入口 middleware，不需要 helper 函数**。Handler 直接从 ctx 里掏。

#### 对比

| 维度 | zap(方案 A)| slog |
|---|---|---|
| 业务代码 | `L(ctx).Info(...)` 每处 | `slog.InfoContext(ctx, ...)` |
| 入口管道 | 每个协议写 middleware 注入 logger | 不需要，Handler 自己从 ctx 取 |
| helper 代码量 | ~20 行 ctx + Logger 封装 | 0 |
| ctx 到达 logger | 靠 middleware 传递 | Handler 原生拿 ctx |

zap 的方案 A 很主流，能用，但 slog 把 **"Handle 方法的第一参数是 ctx"** 写进了接口，让"从 ctx 动态取字段"天然可行。这才是**动态字段场景的真正差距**。

---

### 场景 2 · error 日志自动富化:stack + 根因类型

> **先澄清一个常见误解**：`zap.Error(err)` 和 `slog.Any("err", err)` 都会通过 `err.Error()` 把 `%w` wrap 链的完整字符串保留下来 —— 这点两个库都天然做到。真正容易缺失的不是 wrap 链，而是**stack 和根因类型**。

#### error 进日志的 4 条最佳实践

| # | 实践 | 目的 | 常见做法 |
|---|------|------|---------|
| 1 | **err 作为结构化字段** | 可 grep / 可按字段告警 / 不拼字符串 | `zap.Error(err)` / `slog.Any("err", err)` |
| 2 | **保留 wrap 链文本上下文** | 一眼看清传播路径 | `fmt.Errorf("dao get user id=%d: %w", id, err)` 每层带参数 |
| 3 | **带 stack** | 定位第一现场 file:line | `pkg/errors.WithStack` / 自己写 stack helper |
| 4 | **根因类型可识别** | 告警 / metrics 按类型 group-by | `errors.Is/As` 后打稳定 label 或 `ecode.Reason` |

**BP 1 和 2 两个库都免费做到**。真正的分水岭是 **BP 3、4 要不要在团队里一致执行** —— 这就要 Handler 层统一富化。

#### zap vs slog 满足度对比

| 实践 | zap 业务代码 | zap Handler 统一 | slog 业务代码 | slog Handler 统一 |
|---|---|---|---|---|
| BP 1 结构化字段 | ✅ `zap.Error(err)` | — | ✅ `slog.Any("err", err)` | — |
| BP 2 wrap 链文本 | ✅ 自动 | — | ✅ 自动 | — |
| BP 3 stack | ⚠️ `pkg/errors` 或 `zap.Stack` | 🚧 写 Core(30+ 行) | ⚠️ `pkg/errors` 或手写 | 🟢 写 Handler(10 行) |
| BP 4 根因类型 | ❌ 每处手动 | 🚧 写 Core | ❌ 每处手动 | 🟢 写 Handler(10 行) |

业务代码层面两个库打平。**关键差距在 Handler/Core 层的统一富化成本**。

#### zap 方案:实现 ErrEnrichCore

```go
type ErrEnrichCore struct{ zapcore.Core }

func (c *ErrEnrichCore) Write(ent zapcore.Entry, fields []zapcore.Field) error {
    for _, f := range fields {
        if f.Type == zapcore.ErrorType {               // ★ 只有这个 Type 才是 error
            err := f.Interface.(error)
            fields = append(fields,
                zap.String(f.Key+"_kind",  classifyErr(err)),
                zap.Stack(f.Key+"_stack"),
            )
            break
        }
    }
    return c.Core.Write(ent, fields)
}

func (c *ErrEnrichCore) Check(ent zapcore.Entry, ce *zapcore.CheckedEntry) *zapcore.CheckedEntry {
    if c.Enabled(ent.Level) { return ce.AddCore(ent, c) }
    return ce
}
func (c *ErrEnrichCore) With(fields []zapcore.Field) zapcore.Core {
    return &ErrEnrichCore{c.Core.With(fields)}
}
```

坑点:
- 要 `Field.Type == zapcore.ErrorType` 判断(其他 Type 不对)
- Check 里涉及 `CheckedEntry` 引用计数
- With/Sync 要配齐

#### slog 方案:实现 ErrEnrichHandler

```go
type ErrEnrichHandler struct{ slog.Handler }

func (h *ErrEnrichHandler) Handle(ctx context.Context, r slog.Record) error {
    r.Attrs(func(a slog.Attr) bool {
        if err, ok := a.Value.Any().(error); ok {     // ★ 直接类型断言,不用认 Type 枚举
            r.AddAttrs(
                slog.String(a.Key+"_kind",  classifyErr(err)),
                slog.String(a.Key+"_stack", fmt.Sprintf("%+v", err)),
            )
        }
        return true
    })
    return h.Handler.Handle(ctx, r)
}

func (h *ErrEnrichHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return &ErrEnrichHandler{h.Handler.WithAttrs(attrs)}
}
func (h *ErrEnrichHandler) WithGroup(name string) slog.Handler {
    return &ErrEnrichHandler{h.Handler.WithGroup(name)}
}
```

一个 `Handle` 搞定。`a.Value.Any()` 直接拿回原 error 接口,不用认 Type 枚举、不用理 CheckedEntry。

#### 和「错误处理」章节打通

团队按 [01-error-handling](../01-error-handling/) 里 proto + `ecode.Error` 的做法,`classifyErr` 不用自己手写:

```go
func classifyErr(err error) string {
    var e *ecode.Error
    if errors.As(err, &e) { return e.Reason }    // "USER_NOT_FOUND" 之类
    return "internal"
}
```

→ **日志 `err_kind`、HTTP/gRPC 响应的 error code、告警 dashboard 的 group-by,三者共用一套 `reason`**。这是错误处理规范 + slog Handler 的协同收益。

#### 结论

- BP 1、2:天然满足,业务代码一行搞定
- BP 3、4:都要手动,但**让所有 error 日志自动带** —— slog Handler 比 zap Core 代码短一半,且无 CheckedEntry / Field.Type 等认知坑
- "结构化的 wrap 链(每层单独一个字段)"**不是公认最佳实践**,用 `err.Error()` 的字符串足够做 grep 和人读

---

### 场景 3 · Prod 动态打开 DEBUG 10 分钟排障 —— **两个库都能做，差距不大**

**需求**：线上默认 INFO，出事想临时开 DEBUG 排查，不想重启。

**zap** 用 `zap.AtomicLevel`：

```go
lvl := zap.NewAtomicLevelAt(zap.InfoLevel)
logger := zap.New(zapcore.NewCore(encoder, ws, lvl))
// 改 level
lvl.SetLevel(zap.DebugLevel)
// zap 甚至直接提供了 HTTP handler：
http.Handle("/admin/log-level", lvl)   // GET 查/ PUT 改
```

**slog** 用 `slog.LevelVar`：

```go
var levelVar = new(slog.LevelVar)      // 默认 Info
levelVar.Set(slog.LevelInfo)

h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: levelVar,                   // Level 接受 Leveler 接口，LevelVar 实现了它
})
slog.SetDefault(slog.New(h))

// HTTP 管理接口动态改
http.HandleFunc("/admin/log-level", func(w http.ResponseWriter, r *http.Request) {
    switch r.URL.Query().Get("level") {
    case "debug": levelVar.Set(slog.LevelDebug)
    case "info":  levelVar.Set(slog.LevelInfo)
    case "warn":  levelVar.Set(slog.LevelWarn)
    case "error": levelVar.Set(slog.LevelError)
    }
    w.WriteHeader(200)
})
```

一行 `curl /admin/log-level?level=debug` 生效。

**客观**：这条 zap 甚至做得比 slog 顺手（现成的 HTTP handler），slog 要自己写一小段。**不算 slog 的优势**，只是能做。

---

### 场景 4 · 敏感字段防漏 —— 双层防御

**痛点**：`password`、`token`、`id_card`、手机号到处传递，团队规范要求日志脱敏。靠人记是记不住的，code review 也会漏。

**slog 方案 1 · 类型自律（LogValuer）**

```go
type Phone string

func (p Phone) LogValue() slog.Value {
    s := string(p)
    if len(s) < 7 { return slog.StringValue("***") }
    return slog.StringValue(s[:3] + "****" + s[7:])
}

// 业务代码里用起来和普通 string 一样
user := User{Phone: Phone("13800001234")}
slog.Info("registered", "phone", user.Phone)
// 输出: "phone":"138****1234"
```

`LogValuer` 接口就一个方法：

```go
type LogValuer interface {
    LogValue() Value
}
```

**只要类型实现它，整个代码库都自动脱敏**，且有两个额外收益：

- **懒求值**：LogValue 只在日志级别通过时调用。`slog.Debug("state", "s", state)` 如果 level=Info，`LogValue` 根本不执行，里面的昂贵计算都省了。
- **递归解开结构体**：

  ```go
  type User struct {
      ID    int64
      Name  string
      Phone Phone     // Phone 自己也是 LogValuer
  }
  func (u User) LogValue() slog.Value {
      return slog.GroupValue(
          slog.Int64("id", u.ID),
          slog.String("name", u.Name),
          slog.Any("phone", u.Phone),   // 递归走 Phone.LogValue
      )
  }
  slog.Info("login", "user", user)
  // 输出: "user":{"id":123,"name":"foo","phone":"138****1234"}
  ```

  slog 递归 resolve，但有深度限制（10 层）防死循环。

**slog 方案 2 · 全局兜底（ReplaceAttr）**

即使有人没用 `Phone` 类型，直接 `slog.Info("...", "password", rawPwd)`，Handler 也能拦住：

```go
sensitive := map[string]bool{
    "password": true, "token": true, "secret": true,
    "id_card":  true, "bank_card": true,
}

h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
        if sensitive[a.Key] {
            return slog.String(a.Key, "***")
        }
        return a
    },
})
```

**两层防御 = 类型自律（LogValuer）+ 全局兜底（ReplaceAttr）**，一层漏另一层能接住。

**对比 zap**：要做同样的事，类型级要实现 `MarshalLogObject(ObjectEncoder)`，全局级要实现 `Core`（见 zap 痛点 5 那 30 行）——而且 zap 的 `Field.Type` 有十几种分支，脱敏代码容易漏分支。

---

### 场景 5 · 接 Sentry / Datadog / Loki，不改业务代码

**痛点**：日志要同时写 stdout（给 SRE）、错误级别发 Sentry（告警）、业务事件发 Loki（长期存储）。

**zap 方案**：
- Sentry 要实现 Hook 或自定义 WriteSyncer
- Datadog 用官方 zap provider，但配置项和 Sentry 的不一样
- Loki 找社区包，有的包装成 WriteSyncer，有的包装成 Core

接 3 家看 3 份文档，每家 API 风格不同，版本升级得一起测。

**slog 方案**：所有接入方**都实现 `slog.Handler` 统一接口**：

```go
import (
    slogmulti  "github.com/samber/slog-multi"
    slogsentry "github.com/samber/slog-sentry/v2"
    slogloki   "github.com/samber/slog-loki/v3"
)

handler := slogmulti.Fanout(
    slog.NewJSONHandler(os.Stdout, nil),                                     // 本地 stdout
    slogsentry.Option{Level: slog.LevelError}.NewSentryHandler(),            // error 以上发 Sentry
    slogloki.Option{Endpoint: "...", BatchWait: 5 * time.Second}.NewLokiHandler(),
)
slog.SetDefault(slog.New(handler))
```

**一个接口、统一生态**。下次加 OTel exporter，再插一行 Fanout。

---

### 场景 6 · 单元测试断言 "是否打了某条日志" —— 两边都能做，差别小

**需求**：业务要求某分支必须打 error，想写单测保证不会悄悄不打。

**zap 方案** —— 用官方 `zaptest/observer`，开箱即用：

```go
import "go.uber.org/zap/zaptest/observer"

core, logs := observer.New(zap.InfoLevel)
logger := zap.New(core)

Login(logger, user)

assert.Equal(t, 1, logs.FilterMessage("user login").Len())
```

**客观评估**：zap 的 observer 已经很好用，`ObservedLogs` API 也不复杂，"认知负担" 的说法偏主观。

**slog 方案** —— 自己写 20 行的 Handler（**和生产用同一份 `slog.Handler` 接口**）：

```go
type recordCapture struct {
    records []slog.Record
}

func (h *recordCapture) Enabled(context.Context, slog.Level) bool { return true }
func (h *recordCapture) Handle(_ context.Context, r slog.Record) error {
    h.records = append(h.records, r)
    return nil
}
func (h *recordCapture) WithAttrs([]slog.Attr) slog.Handler { return h }
func (h *recordCapture) WithGroup(string) slog.Handler      { return h }

func TestLogin_logs(t *testing.T) {
    cap := &recordCapture{}
    logger := slog.New(cap)

    Login(logger, user)

    if got := cap.records[0].Message; got != "user login" {
        t.Fatalf("expected login log, got %q", got)
    }
}
```

区别在于：slog 的 testing handler 和生产 handler 用同一个 `slog.Handler` 接口，不需要学 zap 专属的 observer。是**轻微**优势。

---

### 场景 7 · 库只依赖 `*slog.Logger`，不绑定后端

**痛点**（对应 zap 痛点 6）：你写公司内部 Redis SDK，一旦 `import "go.uber.org/zap"`：

- 所有业务 `go.mod` 多一条 zap 依赖
- 升级 zap 要 30 个服务联动
- 业务想用 logrus 也得陪你改

**slog 方案**：

```go
package myredis

import "log/slog"

type Client struct {
    logger *slog.Logger
}

func New(opts ...Option) *Client {
    c := &Client{logger: slog.Default()}   // 调用方不传也能跑
    for _, o := range opts { o(c) }
    return c
}

func WithLogger(l *slog.Logger) Option {
    return func(c *Client) { c.logger = l }
}

func (c *Client) Get(ctx context.Context, key string) (string, error) {
    c.logger.DebugContext(ctx, "redis GET", "key", key)
    ...
}
```

- `import "log/slog"` 是标准库，**`go.mod` 不会多一行**
- 调用方想换 logger：`myredis.New(myredis.WithLogger(slog.New(anyHandler)))`，**库零改动**
- 调用方想不给 logger：直接用默认，跑得起来

> **这让基础库真正"人畜无害"**。没有人被"我用了 X 库所以你也得用 Y logger"绑架。

---

### 场景 8 · 本地看人读格式，线上看机读 JSON —— 两边都很成熟

**需求**：本地开发想看彩色 + key=value 文本；线上要 JSON 给 ELK。

**zap 方案**：`zap.NewDevelopment()` / `zap.NewProduction()` 两个预设 config 开箱即用，或者自己拼 `Encoder + WriteSyncer + Core`。

**slog 方案**：`NewTextHandler` / `NewJSONHandler` 两个内置 Handler：

```go
var h slog.Handler
if os.Getenv("ENV") == "dev" {
    h = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
        Level:     slog.LevelDebug,
        AddSource: true,
    })
} else {
    h = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
}
slog.SetDefault(slog.New(h))
```

**客观评估**：两个库都做得好。slog 的 API 表面统一（都是 Handler），zap 是两个 config preset，组合自定义选项时 slog 稍微灵活。是**边际优势**，**不是转投理由**。

---

### 小结：八个场景客观评估

| 场景 | slog 相对 zap 的优势 | 评分 |
|------|--------------------|------|
| 1 动态字段（ctx→trace_id） | Handle 拿 ctx，免 middleware + helper | **真实优势** |
| 2 error 自动富化（stack + kind） | Handler 代码比 Core 短一半，无 Type/CheckedEntry 坑 | **真实优势** |
| 3 动态 level | zap 甚至更顺手（现成 HTTP handler） | **打平 / zap 略强** |
| 4 敏感字段（LogValuer + ReplaceAttr）| LogValuer 比 zap `ObjectMarshaler` 轻 | **真实优势** |
| 5 接 Sentry/Loki/OTel | 生态 `slog.Handler` 统一，加一家改一行 | **真实优势（要接多家才有感）** |
| 6 单元测试 | 自定义 Handler 用生产同一接口 | **轻微优势** |
| 7 库不绑后端 | 标准库零依赖 | **真实优势（对库作者）** |
| 8 dev vs prod 格式 | API 表面更统一 | **边际优势** |

**真实差距集中在**：动态/ctx 关联字段、自动富化、多家生态接入、库层解耦 —— 这几个场景确实 slog 更顺。**动态 level、单测、格式切换** 是两边都能做、差别小的地方。

> 换句话说：**如果你不写库、不接多家第三方、不在 Handler 层做富化，zap 和 slog 的实际差距不大**。slog 的核心收益是**"横切能力"在 Handle 接口里天然存在**（拿到 ctx + Record 直接改），这点 zap 要走 Core 才行。

---

## "Handler 真的好写吗?" —— 30 行模板足够用

很多人听到 "自定义 Handler 要实现 4 个方法" 就犹豫。真实情况是：**90% 的 Handler 只需要定制 `Handle` 这一个方法**，其他三个是机械样板。

### 最小模板

```go
type MyHandler struct {
    slog.Handler   // 嵌入一个现成的，白嫖 Enabled/WithAttrs/WithGroup 的默认行为
}

// 只要改这个 —— 做你的定制逻辑
func (h *MyHandler) Handle(ctx context.Context, r slog.Record) error {
    // 在这里：加字段 / 脱敏 / 转发 / 采样 ...
    r.AddAttrs(slog.String("my_field", "my_value"))
    return h.Handler.Handle(ctx, r)
}

// 下面两个是样板：让 With 返回的还是 *MyHandler 而不是内嵌的 slog.Handler
func (h *MyHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return &MyHandler{h.Handler.WithAttrs(attrs)}
}
func (h *MyHandler) WithGroup(name string) slog.Handler {
    return &MyHandler{h.Handler.WithGroup(name)}
}
```

**就这样**。本章前面讲过的 `PodHandler`、`ErrChainHandler`、`recordCapture`、脱敏 Handler —— 都是这个模板的变体，区别只在 `Handle` 里的逻辑。

### 对比 zap 写等价功能

```go
type MyCore struct { zapcore.Core }

func (c *MyCore) Write(ent zapcore.Entry, fields []zapcore.Field) error {
    fields = append(fields, zap.String("my_field", "my_value"))
    return c.Core.Write(ent, fields)
}

// 要处理 CheckedEntry 引用计数 —— 新手第一次写容易错
func (c *MyCore) Check(ent zapcore.Entry, ce *zapcore.CheckedEntry) *zapcore.CheckedEntry {
    if c.Enabled(ent.Level) {
        return ce.AddCore(ent, c)
    }
    return ce
}

func (c *MyCore) With(fields []zapcore.Field) zapcore.Core {
    return &MyCore{c.Core.With(fields)}
}
```

行数差不多，但：

- `Check` 涉及 `CheckedEntry`（zap 的"已匹配 core 列表"引用计数），第一次写 10 个有 8 个搞错
- slog 的 `Enabled` 只返回 `bool`，**没有这个坑**

### 不想写？用现成的

| 包 | 用途 |
|---|---|
| `samber/slog-multi` | 扇出 / 管道 / 过滤组合 |
| `samber/slog-zap` / `slog-zerolog` | zap/zerolog 当底层 Handler（性能党）|
| `samber/slog-logrus` | 渐进迁移 logrus |
| `samber/slog-sentry` / `slog-datadog` / `slog-loki` / `slog-otel` | 接第三方 |
| `phsym/console-slog` | 比官方 Text 好看的彩色 console |

**90% 的"自定义 Handler"其实是组合现成的**，真要自己写也就 20-30 行。

---

## 第 7 章 · 横向对比 + 基准数据

### 性能 benchmark(参考官方 + 作者 benchmark,数值是量级,不同机器会有 ±20% 波动)

```
 10 个字段,JSON 输出,单次调用 ns/op    allocs/op
 ─────────────────────────────────────────────
 log         ~1500 ns          8        (拼文本,本就不做结构化)
 logrus      ~3000 ns          23       (map + reflect)
 zap Logger  ~200 ns           0        (类型化 Field)
 zap Sugared ~500 ns           1        (interface{})
 zerolog     ~150 ns           0        (直接 append)
 slog.LogAttrs ~300 ns         0
 slog.Info (kv variadic) ~500 ns 1
```

**要点**:
- slog 和 zap 在一个量级,实际业务差距可以忽略
- logrus 比 slog 慢 10 倍,**高并发项目升级 slog 有实质 QPS 收益**

### API 复杂度

| 库 | 学习曲线 | 代码量 | 坑 |
|---|---|---|---|
| log | 5 分钟 | 最少 | 没 level 没结构化 |
| logrus | 10 分钟 | 少 | 慢、维护模式 |
| zap | 1 小时 | 中 | 两套 API、Field 写法啰嗦 |
| zerolog | 30 分钟 | 中 | 忘了 `.Msg()`、动态字段不友好 |
| slog | 20 分钟 | 少 | `With` 类型不安全(变参)、`LogValue` 递归陷阱 |

### 生态

| 库 | Stars (2026) | 第三方 Handler/Hook | 主流框架支持 |
|---|---|---|---|
| logrus | ~25k | 多但老 | 历史项目为主 |
| zap | ~22k | 多 | kratos/hertz/gozero 老版 |
| zerolog | ~11k | 中 | —— |
| slog | **标准库** | **爆发式增长** | 所有新框架 |

---

## 第 8 章 · 迁移路径

### 路线 1:logrus → slog(推荐)

**步骤**:

1. **定义公共 logger 包**,封装 slog + 和业务字段绑定
2. **在最外层替换初始化**:`log.SetOutput` → `slog.SetDefault`
3. **批量替换调用**:
   ```
   log.WithFields(log.Fields{"k": v}).Info("msg")
   ↓
   slog.Info("msg", "k", v)
   ```
4. 过渡期用 [slog-logrus](https://github.com/samber/slog-logrus) 做 Hook 双写

### 路线 2:zap → slog(保留底层)

业务代码里的 `*zap.Logger` 全换成 `*slog.Logger`,底层用 `slog-zap`:

```go
import (
    "log/slog"
    slogzap "github.com/samber/slog-zap/v2"
    "go.uber.org/zap"
)

zapLogger, _ := zap.NewProduction()
handler := slogzap.Option{Level: slog.LevelInfo, Logger: zapLogger}.NewZapHandler()
slog.SetDefault(slog.New(handler))

// 业务代码从此只用 slog API
slog.Info("hi", "foo", "bar")
```

**好处**:保留 zap 的性能 + 现有 WriteSyncer/Core 生态,业务代码统一到标准库。

### 路线 3:库层彻底解耦

```go
// 旧:库里 import 具体实现
type Client struct {
    logger *zap.Logger      // 调用方被绑定到 zap
}

// 新:只依赖 slog
type Client struct {
    logger *slog.Logger     // 调用方爱用啥用啥
}
```

这是最有价值的重构,建议所有 internal library、SDK、middleware 都走这一步。

### 迁移风险点

| 风险 | 缓解 |
|---|---|
| 字段名不一致 | 统一 `logger.With(...)` 规范,code review 把关 |
| 日志格式变化 | 过渡期 Handler 按老格式输出,和日志收集约定 |
| 性能回退 | logrus → slog 是涨,zap → slog(走 adapter)≈ 持平,直接换 JSONHandler 有 30% 回退 —— 量大再换 handler |
| 遗漏的 `panic(log.Fatal...)` | grep 全局 `log.Fatal` / `logrus.Fatal`,改成 `slog.Error` + return / exit |

---

## 附录

### 关键源码地址

| 项目 | 路径 |
|---|---|
| stdlib log | `src/log/log.go`([GitHub](https://github.com/golang/go/blob/master/src/log/log.go)) |
| stdlib log/slog | `src/log/slog/`([GitHub](https://github.com/golang/go/tree/master/src/log/slog)) |
| logrus | `github.com/sirupsen/logrus` |
| zap | `github.com/uber-go/zap` |
| zerolog | `github.com/rs/zerolog` |
| slog-multi | `github.com/samber/slog-multi` |
| slog-zap / slog-logrus / slog-sentry / ... | `github.com/samber/slog-*` |

### 推荐阅读

1. [Go Blog: Structured Logging with slog](https://go.dev/blog/slog) · 官方介绍
2. [slog design proposal](https://github.com/golang/go/discussions/54763) · 设计讨论
3. zap 作者 Ben Johnson 写的 [Zero Allocation JSON Logging](https://dave.cheney.net/2015/11/05/lets-talk-about-logging) · 零分配思想
4. Uber Engineering: [How we built zap](https://eng.uber.com/go-zap-logging/)

### 速查表:场景 → 选择

| 你现在的状态 | 做什么 |
|---|---|
| 新项目从零 | `log/slog` + `JSONHandler`,完事 |
| 老项目 logrus 慢 | 先上 `slog-logrus` Hook 做影子双写,稳定后换 slog 为主 |
| 老项目 zap 用得好 | 业务代码迁 slog API,底层保 zap(slog-zap)|
| 库 / SDK 作者 | 只依赖 `*slog.Logger`,不 import 具体实现 |
| 性能极致场景(>100k QPS 的日志路径) | `slog-zap` 或直接 zap,业务代码还是 slog 接口 |
| 要接 OTel / Sentry / Loki | 用 `slog-otel` / `slog-sentry` / `slog-loki`,不要自己实现 |

---

## 一页话总结

- **log (2012)** 是文本时代的终点 —— 没 level、没结构化
- **logrus (2014)** 带来 structured，代价是 `map[string]interface{}` + 反射让它慢 10 倍
- **zap (2016)** 用 `Field` struct + Encoder + buffer pool 把性能做到零分配，代价是：两套 API、error 链丢失、ctx 不原生、全局字段要逐入口 With、脱敏要自己写 Core、库依赖污染 —— **6 个日常痛点**
- **zerolog (2017)** 把零分配做到极致，但 "忘记 `.Msg()` 静默丢日志" 是真实线上坑
- **slog (2023)** 不用讲"三要点"，直接看场景：
  1. 加 pod_name → 一次 Handler 包装
  2. err chain 自动展开 → Handler 拦截 Record
  3. 动态 level → `LevelVar`
  4. 敏感字段 → LogValuer + ReplaceAttr 双层防御
  5. 接 Sentry/Loki → 统一 `slog.Handler` 接口
  6. 单测 → 20 行自定义 Handler
  7. 库不绑后端 → 只依赖 `*slog.Logger`
  8. dev/prod 格式 → Handler 一行切换

> **如果 2026 你还在为新项目选日志库犹豫：选 slog，性能极致场景再套 `slog-zap` 做底层。业务代码只认 `*slog.Logger` 接口，换谁都一行。**
