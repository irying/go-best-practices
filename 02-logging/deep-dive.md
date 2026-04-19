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
- [第 6 章 · slog 三要点深讲](#第-6-章--slog-三要点深讲)
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

### 设计目标

**"给 stdlib log 加上 level 和 structured fields,API 尽量兼容"**。

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

### 局限

- **两套 API** 学习曲线陡,新人经常在 `Info` / `Infow` / `Infof` 之间犹豫
- **依赖侵入**:库里一旦 `import "go.uber.org/zap"`,调用方换后端就难
- **Handler 模型相对封闭**:自定义输出(比如 OTel log exporter)要写 Encoder + WriteSyncer + Core 三层

这些在 2023 年 slog 出现后成了迁移动机。

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

### 局限

- **必须记得 `.Msg()`**,忘了就没日志(线上坑过)
- **链式 API 不适合动态字段**:字段来自循环时,你得手写 for 循环往 Event 加
- 社区规模比 zap 小,中文资料少

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

## 第 6 章 · slog 三要点深讲

### 要点 1 · Logger ↔ Handler 解耦

**问题**:logrus 和 zap 都把"打日志的 API"和"写日志到什么后端"耦合在 Logger 内部。想换后端(JSON ↔ console ↔ OTel ↔ Sentry)得改初始化代码。

**slog 的方案**:

```go
type Logger struct {
    handler Handler    // 只持有一个接口
}

type Handler interface {
    Enabled(context.Context, Level) bool
    Handle(context.Context, Record) error
    WithAttrs(attrs []Attr) Handler
    WithGroup(name string) Handler
}
```

Logger 本身很薄,所有具体行为都通过 Handler 实现。

#### 四个方法各自的意义

| 方法 | 目的 | 调用时机 |
|---|---|---|
| `Enabled` | 提前判断 level,避免构造 Record | 每次打日志前 |
| `Handle` | 真正输出 | 级别通过 Enabled 后 |
| `WithAttrs` | 派生带固定字段的 Handler | `logger.With(...)` 触发 |
| `WithGroup` | 派生带 group 前缀的 Handler | `logger.WithGroup(...)` 触发 |

#### 为什么 `WithAttrs` / `WithGroup` 很重要

如果没有这两个方法,`logger.With(...)` 就只能在每条日志时把固定字段拼进去,重复开销很大。

**有了 WithAttrs**,Handler 实现可以**预编码**:

```go
// 伪代码:JSONHandler.WithAttrs 的优化
func (h *JSONHandler) WithAttrs(attrs []Attr) Handler {
    newH := h.clone()
    // 关键:把 attrs 预先编码成 JSON 字节串,存到新 Handler 里
    newH.preformattedAttrs = encodeAttrs(attrs)
    return newH
}

// Handle 时直接拼预编码的字节,不重新序列化
func (h *JSONHandler) Handle(ctx context.Context, r Record) error {
    buf := newBuffer()
    buf.WriteByte('{')
    // ... 时间/level/msg
    buf.Write(h.preformattedAttrs)   // ★ 直接写字节
    // ... record 自己的 attrs
    buf.WriteByte('}')
    ...
}
```

**入口 middleware 注入 trace_id 一次,后续每条日志 0 开销**。

#### 写一个自定义 Handler:最小可用版

```go
// 把日志转发到 Kafka 的 Handler
type KafkaHandler struct {
    producer *kafka.Producer
    topic    string
    level    slog.Level
    preAttrs []slog.Attr
    groups   []string
}

func (h *KafkaHandler) Enabled(_ context.Context, lvl slog.Level) bool {
    return lvl >= h.level
}

func (h *KafkaHandler) Handle(ctx context.Context, r slog.Record) error {
    m := make(map[string]any, r.NumAttrs()+len(h.preAttrs)+3)
    m["time"]  = r.Time.UTC().Format(time.RFC3339Nano)
    m["level"] = r.Level.String()
    m["msg"]   = r.Message

    for _, a := range h.preAttrs {
        m[a.Key] = a.Value.Any()
    }
    r.Attrs(func(a slog.Attr) bool {
        m[a.Key] = a.Value.Any()
        return true
    })

    payload, _ := json.Marshal(m)
    return h.producer.Produce(h.topic, payload)
}

func (h *KafkaHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    nh := *h
    nh.preAttrs = append(slices.Clone(h.preAttrs), attrs...)
    return &nh
}

func (h *KafkaHandler) WithGroup(name string) slog.Handler {
    nh := *h
    nh.groups = append(slices.Clone(h.groups), name)
    return &nh
}
```

用法:

```go
// 同时写 stdout 和 Kafka —— 组合 Handler
logger := slog.New(slogmulti.Fanout(
    slog.NewJSONHandler(os.Stdout, nil),
    &KafkaHandler{producer: p, topic: "audit"},
))
```

#### Handler 组合的生态

社区已经有成熟的 Handler 组合库:

- `samber/slog-multi` · 扇出 / 管道 / 过滤
- `samber/slog-zap`、`samber/slog-zerolog` · 把 zap/zerolog 包成 slog.Handler(性能党)
- `samber/slog-sentry`、`slog-datadog`、`slog-loki` · 各家接入

**这就是"解耦"的真正价值**:生态可以正交组合。

---

### 要点 2 · LogValuer 接口

**问题**:
- 日志里常要写敏感字段(phone / id_card / token),每处都手动脱敏容易漏
- 日志里常要写昂贵计算(大对象 `.String()`、远程查询),一旦级别没通过就白算了

**slog 的方案**:

```go
type LogValuer interface {
    LogValue() Value
}
```

**任何实现了 `LogValue() Value` 的类型,在写日志时会被 Handler 调用一次该方法,拿到实际要记录的 Value**。

#### 实战 1:脱敏

```go
// 定义一个脱敏类型
type Phone string

func (p Phone) LogValue() slog.Value {
    s := string(p)
    if len(s) < 7 {
        return slog.StringValue("***")
    }
    return slog.StringValue(s[:3] + "****" + s[7:])
}

// 用法:业务代码保留原值,只在日志里脱敏
user := User{Phone: Phone("13800001234")}
slog.Info("user registered", "phone", user.Phone)
// 输出: {"...","phone":"138****1234"}
```

**类型自带脱敏语义**,谁 import 谁用,不会漏。

#### 实战 2:懒求值避免昂贵计算

```go
type ExpensiveState struct {
    conns map[string]*Conn
}

func (s *ExpensiveState) LogValue() slog.Value {
    // 只在日志级别通过时才调用这里
    return slog.GroupValue(
        slog.Int("conn_count", len(s.conns)),
        slog.Int("active", s.countActive()),   // 昂贵
    )
}

slog.Debug("state dump", "state", state)
// ↑ 如果 level = Info,LogValue 根本不会被调用,countActive 不执行
```

对比 zap:zap 的对应能力是 `zap.Object(key, ObjectMarshaler)`,需要实现 `MarshalLogObject(ObjectEncoder) error`,代码更啰嗦。

#### 实战 3:返回 Group 把结构体拆开

```go
type User struct {
    ID    int64
    Name  string
    Phone Phone
}

func (u User) LogValue() slog.Value {
    return slog.GroupValue(
        slog.Int64("id", u.ID),
        slog.String("name", u.Name),
        slog.Any("phone", u.Phone),   // 会递归走 Phone.LogValue
    )
}

slog.Info("login", "user", user)
// 输出: {"...","user":{"id":123,"name":"foo","phone":"138****1234"}}
```

**递归求值**:slog 在处理 Value 时,如果发现它是 `LogValuer`,会反复 resolve,但有**深度限制**(当前是 10 层)防止死循环。

#### Handler 侧的兜底:`ReplaceAttr`

即使有人忘了用脱敏类型,Handler 还可以在全局兜底:

```go
sensitive := map[string]bool{
    "password": true, "token": true, "secret": true,
    "id_card": true, "bank_card": true,
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

`ReplaceAttr` 在 Handler 写每个 Attr 之前调用,可以**修改、屏蔽或 drop** 字段。

**两层防御 = LogValuer(类型自律)+ ReplaceAttr(全局兜底)**。

---

### 要点 3 · 零依赖

**问题**:库作者写基础库时,打日志是刚需。但库一旦 `import "github.com/sirupsen/logrus"`,所有调用方都被迫装一个 logrus。zap 同理。

**结果**:大型项目 `go.mod` 里能同时看到 logrus、zap、zerolog、其他 SDK 带进来的 logrus v1.8……,互相不兼容,日志分裂。

#### slog 是标准库,没有这个问题

库代码:

```go
package myredis

import "log/slog"

type Client struct {
    logger *slog.Logger
}

func New(opts ...Option) *Client {
    c := &Client{logger: slog.Default()}   // 默认不给 logger 也能跑
    for _, o := range opts { o(c) }
    return c
}

func WithLogger(l *slog.Logger) Option {
    return func(c *Client) { c.logger = l }
}

func (c *Client) Get(ctx context.Context, key string) (string, error) {
    c.logger.DebugContext(ctx, "redis GET", "key", key)
    // ...
}
```

**调用方 `go.mod` 不会多一行依赖**。调用方想把 logger 路由到 zap,只需一行:

```go
zapHandler := slogzap.Option{Level: slog.LevelInfo, Logger: zapLogger}.NewZapHandler()
redis := myredis.New(myredis.WithLogger(slog.New(zapHandler)))
```

#### 生态收敛表现

2023 以来可观察的变化:

1. **大厂 SDK 改用 slog**:`aws-sdk-go-v2` 从 2024 开始接受 `*slog.Logger`
2. **框架默认 slog**:Kratos v2.8+、Hertz v0.9+ 都把 slog 作为一等公民
3. **老库过渡 Adapter**:`slog-zap`、`slog-logrus` 让你保留旧后端
4. **新手教程**:Go 官方博客、The Go Programming Language book 新版都以 slog 为例

**2026 的事实:新项目用 slog,老项目保留底层换 API 层**。

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

- **log (2012)** 是文本时代的终点 —— 没 level,没结构化
- **logrus (2014)** 带来 structured,但 map + reflect 让它慢 10 倍
- **zap (2016)** 用 `Field` struct 和 Encoder 解决了性能,代价是两套 API 和强依赖
- **zerolog (2017)** 把零分配做到极致,但链式 API 的"忘记 `.Msg()`"是线上坑
- **slog (2023)** 是官方的终局:**Logger ↔ Handler 解耦**(可插拔)+ **LogValuer**(脱敏懒求值)+ **零依赖**(标准库)—— 性能够用,生态收敛

> 如果 2026 你还在为新项目选日志库犹豫:**选 slog,把性能留给 `slog-zap` Adapter 做兜底**。
