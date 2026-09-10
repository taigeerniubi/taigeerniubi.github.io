---
layout: post
title: "博客开张：为什么把博客放在 GitHub 上"
date: 2026-09-09 09:00:00 +0800
categories: [随笔]
tags: [github, jekyll]
---
这是博客的第一篇文章。

写博客的方式很简单：在 `_posts/` 目录下新建一个 `YYYY-MM-DD-标题.md` 文件，顶部写好上面这段 front matter，正文用 Markdown，push 到 main 分支，一两分钟后就发布了。注意 `date` 不要写成未来的时间，Jekyll 默认会跳过还没到时间的文章。

GitHub Pages 会自动生成 `feed.xml`，我的 [GitHub 主页](https://github.com/taigeerniubi) 每天会从这个 RSS 拉最新文章展示出来。

接下来打算写的东西：

1. 用 MCP 给 LLM 接工具时踩过的坑
2. 一次 LoRA 微调的完整流水账
