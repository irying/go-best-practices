# Go Best Practices

Go 后端团队基础实践分享合集。每个主题一个独立目录，包含 slides 原文（Marp markdown）和导出的 pptx。

**表达风格：思维导图式** —— 从 *为什么* 到 *本质* 到 *业务分层* 到 *收益* 到 *怎么做*，
先给结论再展开，尽量一页一个论点，可以直接拿去团队内分享。

## 路线图

### 已完成

| # | 主题 | 原文 | Slides | 一句话 |
|---|------|------|--------|--------|
| 01 | [错误处理](./01-error-handling/) | [slides.md](./01-error-handling/slides.md) | [slides.pptx](./01-error-handling/slides.pptx) | 错误在下层带上现场，在上层变成日志和响应；对外用 proto 定义的错误码 |
| 02 | [日志](./02-logging/) | [slides.md](./02-logging/slides.md) | [slides.pptx](./02-logging/slides.pptx) | 新项目默认 `log/slog`；按读者分四层；应用只写 stdout，收集交给 agent |

### 规划中

按痛感和覆盖面排序，欢迎提 issue 调整优先级。

| # | 主题 | 为什么需要 |
|---|------|-----------|
| 03 | **Context** | 超时传递、goroutine 泄漏、ctx 塞 struct、Value 滥用、框架 ctx 下沉问题 |
| 04 | **优雅退出** | 信号处理、drain 长连接、超时兜底、goroutine 收尾、组件关闭顺序 |
| 05 | **可观测性（Metrics + Tracing）** | 和日志配套；基数爆炸、直方图桶、OTel 规范、trace / log / metric 打通 |
| 06 | **并发 & goroutine 生命周期** | 启动易退出难、panic 不 recover、worker pool、errgroup 坑、`-race` |
| 07 | **配置管理** | env / file / 远程配置混用、密钥管理、热更新、默认值约定 |
| 08 | **时间处理** | 时区、monotonic clock、零值、deadline 传递、DB 时区一致性 |
| 09 | **数据库访问** | 连接池参数、事务传 ctx、N+1、NULL 处理、migration 安全 |
| 10 | **HTTP Client** | 默认 `http.Client` 的坑、超时分层、连接复用、重试 + 幂等、熔断 |
| 11 | **测试** | 表驱动、mock vs fake vs real、集成测试、testcontainers、覆盖率误区 |
| 12 | **API 设计与版本化** | gRPC / HTTP 并存的契约、兼容性演进、分页、幂等键 |
| 13 | **项目结构** | cmd / internal / pkg、接口放消费侧、依赖方向、模块边界 |

## 目录组织约定

```
NN-topic-name/
├── slides.md         # Marp markdown 源文件
├── slides.pptx       # 导出的 PPT（入库方便下载）
└── README.md         # 主题简介、参考资料（可选）
```

## 本地预览 / 重新导出

Slides 使用 [Marp](https://marp.app/) 编写，源码是标准 Markdown。

```bash
# 浏览器实时预览
npx @marp-team/marp-cli slides.md --preview

# 导出 pptx
npx @marp-team/marp-cli slides.md --pptx --allow-local-files

# 导出 pdf
npx @marp-team/marp-cli slides.md --pdf --allow-local-files
```

也可以在 VS Code 装 **Marp for VS Code** 插件，侧边栏实时预览 + 一键导出。

## License

MIT
