---
layout: post
title: 单测全绿，功能全废：一次「集成测试反攻」的复盘
date: 2026-11-04 11:20:00 +0800
description: 两天 35 个提交、占整个仓库 31%，全在修同一类 bug：SQL 里的表名列名拼错。它们能编译、能启动、单测全绿，却在真实数据库前全线崩溃。这篇复盘缺陷类型学，以及为什么「mock 掉被测对象」是最危险的一种测试。
tags: [测试, Testcontainers, 工程实践]
categories: [后端]
featured: false
related_posts: false
toc:
  sidebar: left
---

有一段提交历史我印象很深：**两天里 35 个提交，占整个仓库提交量的 31%，全在修同一类 bug**。

不是架构问题，不是并发问题，是 SQL 写错了——列名拼错、表名拼错、关键字拼错。

---

## 一、现象：全绿的单测，跑不起的功能

那批 bug 长这样：

| 症状             | 真实原因                                                     |
| :--------------- | :----------------------------------------------------------- |
| 查询返回空       | 列名写成 `lgo_url`，实际是 `logo_url`                        |
| 报表永远 0 条    | `SELECT ... FROM produt_sku`（少了个 `c`）                   |
| 条件永远不生效   | `WHERE a = 1 ADN b = 2`（`AND` 打成 `ADN`）                  |
| 参数个数不匹配   | `UPDATE ... SET a=?, b=? WHERE id=?` 只传了 2 个参数         |
| 更新到了错误的行 | `UPDATE` 里用了 `p.status` 别名，但 `p` 没在 `UPDATE` 中声明 |

共同点：**编译期绝对发现不了**。SQL 是字符串，Java 编译器对它一无所知。

而单元测试也发现不了——因为单元测试里的 Repository 是 mock 的，SQL 根本不会被执行。

---

## 二、为什么单测抓不到：我 mock 掉了被测对象

假设给 outbox 分发任务写一个「漂亮」的单元测试：

```java
var dispatcher = mock(JdbcOutboxDispatcher.class);
when(dispatcher.dispatch()).thenReturn(1);

new OrderOutboxJob(dispatcher).dispatch();

verify(dispatcher).dispatch();
```

这个测试验证了什么？**验证了「定时任务会调用 dispatch()」**。

它没有验证 `dispatch()` 内部能不能跑通——因为 `dispatch()` 被 mock 掉了。这个测试即使在被测逻辑完全写错的情况下，也会 100% 通过。

更糟的是我在五个服务里都写了这种测试（每个服务的 outbox 配置一个），于是**五份「全绿」共同保护了一个根本跑不通的类**。

> `mock` 掉**依赖**（数据库、外部 HTTP、MQ）是合理的，那是隔离环境；
> `mock` 掉**被测对象本身**，测试就退化成了「我调用了我自己」。

---

## 三、修复：把真实数据库接回来

真正的转折点不是写更细的单测，而是用 Testcontainers 接上真库：

```java
@Container
static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.4")
    .withDatabaseName("ygh_order")
    .withUsername("test")
    .withPassword("test");
```

刻意不用 H2——它的 SQL 方言和 MySQL 差得远，很多问题它不报。用真的 MySQL 起容器，跑一次真的 Flyway 迁移建表，再让代码对着它跑。

这一步做完，那批 SQL 错误在**第一次运行**就全暴露了：Unknown column、Unknown table、语法错误，一条条非常明确。

顺带把这类 bug 归成四类，现在写新 SQL 会专门自查：

1. **标识符拼错**：表名、列名、别名。占比最高。→ 建表后跑一次真实查询兜底。
2. **关键字拼错**：`ADN` / `FORM`。→ 只有真实执行才会报。
3. **占位符与参数不匹配**。→ 集成测试一跑就报，单测永远发现不了。
4. **动态 SQL 拼接错误**：

```java
// 拼出来是 order_outboxSET status=... —— 少了一个空格
"UPDATE " + table + "SET status = 'PROCESSING' WHERE id = ?"
```

第 4 类最阴险：只在「表名来自变量」的代码里出现，拼出来的字符串在日志里看着「差不多对」。

---

## 四、代价

1. **测试变慢**：每个集成用例几秒到几十秒，全量跑一次要起 6 个容器。所以只能分层——纯单元测试覆盖逻辑（122 个），集成测试只覆盖每条 SQL 路径（22 个）。**22 个集成测试抓住的 bug，比 122 个单元测试加起来还多。**
2. **CI 变重**：GitHub Actions 里要能跑 Docker，构建时间从几分钟涨到十几分钟，为此给 job 设了 45 分钟超时。
3. **覆盖率不再等于质量**：行覆盖率 70% 的门禁是真的，但它保护不了字符串里的 SQL——覆盖率只统计「这行代码执行过」，不统计「这句 SQL 对不对」。
4. **仍然有漏网**：动态拼表名的分支如果没被集成测试覆盖到，错误依旧只能等到线上。这也是我把它列进「已知边界」的原因。

> 一句话：**能被编译器和类型检查的东西，交给人 review 是浪费；不能被检查的东西（SQL 字符串、配置、正则），才必须交给真实环境去跑。**
