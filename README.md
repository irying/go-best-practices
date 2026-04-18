# Go Best Practices

一个 Go 基础实践分享合集。每个主题一个独立目录，包含 slides 原文（Marp markdown）和导出的 pptx。

## 目录

| # | 主题 | 原文 | Slides |
|---|------|------|--------|
| 01 | 错误处理 Error Handling | [slides.md](./01-error-handling/slides.md) | [slides.pptx](./01-error-handling/slides.pptx) |

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

也可以在 VS Code 安装 **Marp for VS Code** 插件，侧边栏实时预览 + 一键导出。

## License

MIT
