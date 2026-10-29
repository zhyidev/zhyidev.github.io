---
layout: page
title: AI 智慧校园平台
description: 基于 Tool Use 的 AI 助手，6 个只读业务工具，回答强制带引用溯源。
importance: 2
category: work
---

传智杯 Web 开发大赛全国二等奖项目，负责 AI 助手模块。

### 工具调用

基于大模型 Tool Use 注册 6 个只读业务工具，用 JSON Schema 定义工具契约，实现「模型决策 → 工具执行 → 结果回填」的多轮循环，最多 3 轮防死循环。

### 边界与可信

system prompt 固化只读边界并显式拒绝写操作；回答强制携带 references 引用溯源，降低「看起来对」的风险。

### 上下文与鉴权

会话级上下文管理 + 12 条滑动窗口裁剪控制 Token 开销，支持多轮历史回溯；JWT 无状态认证、Redisson 分布式锁防重复提交，配合 429 限流控制请求频率。

技术栈：Java、Spring Boot 3、MyBatis、MySQL、Redis、Redisson、JWT、LLM Tool Use、Vue 3。
