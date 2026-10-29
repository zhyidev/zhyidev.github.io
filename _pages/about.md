---
layout: about
title: 关于
permalink: /
subtitle: Java 后端开发 · AI 应用开发 · 广州

profile:
  align: right
  image: prof_pic.jpg
  image_circular: true # crops the image to make it circular
  more_info: >
    <p>计算机科学与技术 · 本科</p>
    <p>2023 – 2027 · 广州</p>

selected_papers: false # includes a list of papers marked as "selected={true}"
social: true # includes social icons at the bottom of the page

announcements:
  enabled: true # includes a list of news items
  scrollable: true # adds a vertical scroll bar if there are more than 3 news items
  limit: 3 # leave blank to include all the news in the `_news` folder

latest_posts:
  enabled: false
  scrollable: false # adds a vertical scroll bar if there are more than 3 new posts items
  limit: 3 # leave blank to include all the blog posts
---

我是 **zhyi**，计算机科学与技术专业 2023 级本科生，方向是 **Java 后端** 与 **AI 应用开发**。

过去一年主要在做两件事：一是把一个 14 微服务 + 7 公共库的后端写完——下单、库存预占、钱包扣款、支付回调的闭环，以及用 Outbox 加消费侧幂等状态机解决「落库成功、消息丢失」；二是给校园 AI 助手做基于 Tool Use 的工具调用，让模型在只读边界内完成「决策 → 执行 → 回填」。

比起「把功能跑通」，我更在意「为什么这么设计」：为什么订单要用状态机 CAS 而不是悲观锁，为什么消息要用 Outbox 而不是直接在事务里发 MQ。这些判断在代码里看不出来，所以试着写下来。

写代码之外，也看点动漫和哲学。这个站点用来放项目、简历，和那些写下来的思考。
