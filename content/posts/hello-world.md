---
title: "Hello World — 博客启航"
date: 2026-02-28
draft: false
tags: ["博客", "Hugo", "AI"]
categories: ["教程"]
summary: "这是博客的第一篇文章，介绍博客的定位、技术栈选型，以及未来的内容方向。"
ShowToc: true
TocOpen: true
---

## 博客定位

欢迎来到我的技术博客！这里将聚焦三个核心方向：

- **AI 工程** — LLM 应用开发、AI Agent 架构、Prompt Engineering
- **DevOps / SRE** — 基础设施即代码、可观测性、CI/CD 最佳实践
- **量化交易** — 加密货币量化策略、回测框架、风险管理

## 技术栈

本博客基于以下技术栈构建：

| 组件 | 技术选型 |
|------|---------|
| 静态站点生成器 | [Hugo](https://gohugo.io/) |
| 主题 | [PaperMod](https://github.com/adityatelange/hugo-PaperMod) |
| 托管 | GitHub Pages |
| CI/CD | GitHub Actions (Self-hosted Runner) |
| 内容运营 | OpenClaw AI Agent |

### 为什么选择 Hugo + PaperMod？

**Hugo** 是目前最快的静态站点生成器之一，构建速度通常在毫秒级别。对于技术博客来说，它的优势在于：

1. **极速构建** — 即使上百篇文章也能秒级完成
2. **Markdown 原生支持** — 写作体验流畅
3. **丰富的主题生态** — PaperMod 简洁优雅，自带暗色模式和搜索功能

**PaperMod** 主题则提供了开箱即用的：

- 暗色/亮色模式自动切换
- 内置全文搜索 (Fuse.js)
- 代码高亮与一键复制
- SEO 优化 (Open Graph, Twitter Cards)
- 阅读时间估算

## AI 自动运营

这个博客的一个有趣特点是**半自动化运营**。通过 OpenClaw AI Agent，实现了：

```
AI 生成草稿 → Telegram 通知审核 → 确认后自动发布
```

整个流程中，AI 负责选题建议和初稿生成，人类负责质量把关和最终确认，达到效率与质量的平衡。

## 未来计划

- 每周至少一篇技术文章
- 建立完整的量化交易系列教程
- 分享 AI Agent 开发实战经验
- 记录 SRE 实践中的经验教训

---

欢迎关注，一起探索技术的边界！
