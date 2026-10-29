---
layout: page
title: 交易中台后端
description: 14 微服务 + 7 公共库，下单、库存预占、钱包扣款到支付回调的完整闭环。
importance: 1
category: work
---

实习期间负责的后端整体实现，Java 25 / Spring Boot 4 / Spring Cloud 2025（Nacos、Gateway、Sentinel）/ MyBatis-Plus / MySQL / Redis / RocketMQ / Elasticsearch / PostgreSQL + pgvector / Flyway。

### 架构与治理

14 个 Spring Boot 4 微服务 + 7 个公共库的 Maven 多模块工程，BOM 统一依赖版本；Nacos 注册配置、Gateway 统一路由 / JWT 校验 / Sentinel 限流 / 链路 ID；14 个服务接入统一的响应与错误码契约。

### 交易链路

实现「下单（服务端快照）→ 库存预占 / 确认 / 释放 → 钱包扣款 → 支付回调」闭环：

- `request_id` 幂等 + 状态机 CAS + `version` 乐观锁，防重复下单、重复扣款与超卖
- 30 分钟未支付自动关单并释放预占库存，库存台账与余额定期对账

这里选状态机 CAS 而不是悲观锁，是因为下单是短事务、冲突概率低，悲观锁会让长尾请求排队；真正会冲突的是同一商品的库存行，用版本号重试成本更低。

### 消息可靠性

本地消息表（Outbox）+ 消费侧幂等状态机（claim 租约、重试、死信）封装 RocketMQ，解决「业务落库成功、消息发送失败」的丢消息问题；订单、钱包、知识库事件按可恢复任务调度。

### RAG 检索

Tika 解析 → 1200 / 200 重叠分块 → 向量化 → Elasticsearch + pgvector 双路召回 → RRF（1/(60+rank)）融合 → 带文档版本引用的回答（SSE）；评估集校验证据覆盖率与拒答正确性。

### 工程质量

46 个 Flyway「仅向前」迁移脚本 + 启动前命名与版本校验；GitHub Actions 执行 `mvn verify`、JaCoCo 覆盖率门禁、CycloneDX SBOM 与 Trivy；维护 144 个测试类 / 22 个集成测试，修复 50+ 处缺陷并定位 RocketMQ 消费并发失败的根因。
