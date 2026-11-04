---
layout: page
title: 交易中台后端
description: 把「下单 → 占库存 → 扣款 → 回调」做成不会超卖、不会重复扣款、状态可复盘的系统。14 个 Spring Boot 微服务 + 7 个公共库。
img: /assets/img/projects/trade-platform-cover.svg
importance: 1
category: work
toc:
  sticky: left
---

> 把一条交易链路做成在任意一环挂掉时都不会超卖、不会重复扣款、状态可复盘的系统。

**规模**：14 个 Spring Boot 微服务 · 7 个公共库 · 702 个 Java 文件（主代码 558 个 / 21,173 行有效代码）· 144 个测试文件（含 22 个集成测试）· 51 个 Flyway 迁移脚本。

---

### 01 · 我最想讲的一个决策：状态机 CAS，而不是悲观锁

下单要防重复提交、防并发改状态。最直觉的做法是 `SELECT ... FOR UPDATE` 加锁，我没这么做。

理由有两条：下单是短事务、冲突概率低；悲观锁会让长尾请求排队。而真正会冲突的只有「同一订单被重复操作」这一种情况——**这种冲突，失败才是正确语义**（用户连点两次取消，第二次不该成功）。

所以我把判断塞进 `WHERE`，用影响行数定成败：

```sql
UPDATE order_main
   SET status = ?, version = version + 1
 WHERE id = ? AND user_id = ? AND version = ? AND status = ?
```

影响行数 ≠ 1，直接抛业务冲突让前端刷新。

**如果当初用了悲观锁会怎样**：正常路径多一次加锁往返，冲突路径要处理锁超时，而且「重复取消」会变成「第二次也成功」，产生两条取消流水——问题从「要不要排队」变成「历史数据怎么修」。

这条链路上所有状态变更只有一个出口 `transition()`，「从哪个状态、到哪个状态、为什么」必须显式传入。**没有第二个地方能改订单状态。**

---

### 02 · 分层不是约定，是编译期约束

大部分「微服务项目」最后会变成分布式单体，原因是有人 `import` 了隔壁服务的 Service 类。我用 Maven 模块图把这条路焊死了：

- 12 个业务服务，每个强制拆成 `-api` 和 `-service` 两个模块；
- **服务之间只允许依赖对方的 `-api`，不允许依赖 `-service`**；
- 7 个公共库只依赖 `common-core`，不允许反向依赖任何业务模块。

59 个 `pom.xml` 逐一核对：**违规 0 处**。这意味着 `order` 服务在编译期就装配不出 `inventory` 的 Bean——想拿数据只能走 HTTP。

{% include figure.liquid loading="eager" path="assets/img/projects/trade-platform-arch.svg" title="分层与依赖方向：基础层不得反向引用，跨服务只能依赖对方的 -api" class="img-fluid rounded z-depth-1" %}

---

### 03 · 交易链路：五步，每步都有一个「不能那样写」

{% include figure.liquid path="assets/img/projects/trade-platform-flow.svg" title="下单到支付回调的五个环节" class="img-fluid rounded z-depth-1" %}

| 步骤 | 做什么 | 关键决策 |
|:--|:--|:--|
| 1 · 网关 | RS256 验签 → 查 Redis 会话黑名单 → 注入签名身份头 | 会话校验返回**三态**（有效 / 失效 / 依赖不可用），Redis 挂了返回 503 而不是放行 |
| 2 · 预览取价 | 只收 `skuId + quantity`，价格从商品服务重新读 | **价格永远不从客户端来**；同时校验商品已上架、库存可满足 |
| 3 · 预占库存 | 一条带 `available >= ?` 条件的 UPDATE 扣减 | 判断与修改在同一条语句里，**没有先查后改的窗口** |
| 4 · 落单 | 订单主表 + 明细 + 地址快照 + 状态历史 + outbox，一个事务 | 六次写同事务；失败则逐条释放已占库存 |
| 5 · 扣款与回调 | 悲观锁取余额 → 三段幂等 → 事件回写 | 幂等三重：`request_id`、`reference_id + type`、余额行锁 |

**幂等的最终担保不在 Java，在 DDL**：

```sql
UNIQUE KEY uk_order_user_request (user_id, request_id)
```

应用层「先查再插」只是快路径；并发双写时第二条 INSERT 被数据库拒绝，事务回滚，语义仍然正确。

---

### 04 · 没有分布式事务，一致性从哪来

四层机制，加一条对账旁路：

{% include figure.liquid path="assets/img/projects/trade-platform-consistency.svg" title="本地事务 → 至少一次投递 → 消费幂等 → 状态机 CAS，外加对账兜底" class="img-fluid rounded z-depth-1" %}

1. **本地事务**：业务数据和要发的消息（outbox 表）在同一个事务里落库；
2. **至少一次投递**：独立任务扫 outbox 表投递到 RocketMQ，带租约、重试与死信；
3. **消费幂等**：`INSERT IGNORE INTO order_inbox(event_id)` 返回 0 就说明处理过，直接返回；
4. **补偿**：跨服务写失败即逆序回滚（占库存失败就释放已占的）；
5. **对账兜底**：订单侧不依赖 MQ——直接问钱包要支付结果，事件链路整体失效也能把订单收敛到「已支付」。

**为什么不用 Seata**：评估过，没用。AT 模式要每个参与方建 `undo_log`、依赖全局锁，在「三个服务四张表、QPS 不高」的场景里，运维复杂度远大于收益。用本地事务 + Outbox + 对账是同一问题的更轻解。代价是中间态会对外可见（订单待支付但库存已占），而这在电商里本来就是正常语义。

---

### 05 · 另一条线：企业知识库与 RAG（同一套网关、认证、可观测）

文档入库 → Tika 解析 → 1200 字窗口 / 200 字重叠分块 → Elasticsearch 关键词与 pgvector 向量**双路召回** → RRF `1 / (60 + rank)` 融合 → 带文档版本引用的回答（SSE 流式）。

两个细节值得说：

- **两路各自独立降级**：ES 挂了只告警，向量那一路仍能出结果；反之亦然。检索质量下降和检索完全不可用是两个量级的事故。
- **每次回答落 RAG trace**：召回证据、prompt 版本、模型名、检索与生成耗时、token 估算。LLM 的输出不可复现，没有 trace 就无法评估、无法追责。

---

### 06 · 工程质量：能核对的部分

| 项 | 数字 |
|:--|:--|
| Flyway「仅向前」迁移脚本 | **51 个**，启动前校验命名、版本单调、路径穿越 |
| 测试 | 144 个测试文件 / **22 个集成测试**（Testcontainers 起真实 MySQL / Redis / PostgreSQL） |
| 覆盖率门禁 | 行 ≥ 70%、分支 ≥ 60%（网关分支 ≥ 90%） |
| CI | `mvn verify` → JaCoCo 门禁 → CycloneDX SBOM → Trivy（HIGH/CRITICAL 阻断）→ compose 编排合法性校验 |
| 已修复缺陷 | 40 个 `fix` 提交，集中在一次「集成测试反攻」里（2 天 35 个提交） |

**最能说明问题的一个 bug**：RocketMQ 消费偶发并发失败，根因是消费组 OWNER 生成器的正则漏了 Base64 URL 安全字符 `-` 和 `_`——只在特定 ID 下复现，靠日志比对才定位到。

---

### 07 · 已知边界与下一步

诚实列三条，也是我接下来要做的事：

1. **Outbox 分发的测试覆盖不足**：现有测试 mock 掉了分发器本身，只验证了「定时任务会调用」。需要补连真实数据库的集成测试，让 SQL 这一层真正被测到。
2. **持久层是手写 SQL**：换来对执行计划的完全控制，代价是 SQL 拼写错误编译期发现不了（我们集中修过一批表名列名拼写）。计划引入 SQL 静态检查。
3. **没做压测与容量评估**：当前只有功能与集成测试，链路 QPS 上限未知。这是 Roadmap 第一条。

---

### 08 · 一张图看全貌

{% include figure.liquid path="assets/img/projects/trade-platform-mindmap.svg" title="项目全景：定位 / 技术栈 / 分层 / 核心链路 / 关键设计 / 质量与风险 / 演化脉络" class="img-fluid rounded z-depth-1" %}

---

### 09 · 技术栈

Java 25 · Spring Boot 4.0.7 · Spring Cloud 2025.1.2（Nacos / Gateway / Sentinel）· MyBatis-Plus（仅审计字段自动填充基建，业务层手写 SQL）· MySQL · PostgreSQL 17 + pgvector · Elasticsearch · Redis · RocketMQ · Flyway · LangChain4j（embedding）· Vue 3.5 + TypeScript + Vite 7 · Prometheus / Grafana / Loki

### 10 · 代码与文档

- 仓库：[github.com/zhyidev/yuegang-zhihui-ai](https://github.com/zhyidev/yuegang-zhihui-ai)
- 接口契约：14 份 OpenAPI + Bruno 集合
- 本地运行：JDK 25 / Maven 3.9.16 / Docker（六件套中间件），详见 README
