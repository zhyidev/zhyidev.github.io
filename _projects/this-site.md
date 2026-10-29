---
layout: page
title: 本站
description: al-folio + GitHub Pages。简历、项目与写作放在同一个仓库里。
importance: 3
category: fun
---

用 [al-folio](https://github.com/alshedivat/al-folio) 搭的个人主页，Jekyll 构建、GitHub Actions 发布到 GitHub Pages。

- 首页是简介与动态，`/cv/` 是从 `_data/cv.yml` 渲染的简历
- 项目与笔记按 collection 管理，正文写成 Markdown
- 部署走 `gh-pages` 分支，push 到 `main` 自动构建

之所以不用模板默认的演示内容，是因为一个学生的个人站点没必要背三十几篇占位文章——留下自己真正写过的就够了。
