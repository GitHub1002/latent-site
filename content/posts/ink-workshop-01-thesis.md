---
title: "墨坊源码解读 · 01 | 它不是演示店面：先收下订单，再让库存和余额各自失败"
date: 2026-10-14
slug: ink-workshop-01-thesis
draft: false
weight: 930
tags: ["墨坊", "源码解读", "分布式", "Spring", "Outbox"]
summary: "墨坊是一份可读的 Java 小店：网关只暴露 8080，商品库、库存、店铺各用各的库。下单先 202 收下，库存经 RabbitMQ 后扣；余额不够再发消息把库存加回去。本文钉死产品论点和分层，配架构图。仓库 https://github.com/GitHub1002/ink-workshop。"
ShowToc: true
---

本站前面把 Agent 写成循环、工具和策略。墨坊回答的是另一问：**没有模型的时候，一笔订单怎么在多个进程里失败，还不超卖、不把钱扣错。**

代码在 [GitHub1002/ink-workshop](https://github.com/GitHub1002/ink-workshop)。卖的是松烟墨、油烟墨、宣纸和朱砂印泥，金额用整数分。pom 里写到阶段 10：配置进 Nacos，付款失败后退回库存，并准备了尚未在集群里跑过的 Kubernetes 清单。

一句话定位：

> 浏览器和 `curl` 都只打网关。店铺先收下订单，库存后扣；扣得动才能付款，余额不够就把留下的库存加回去。

它不是又一个「注册登录下单」的单体演示。成功标准是：**超卖进不了库、同一把幂等钥匙得到同一张单、店铺挂了网关会熔断，而不是页面更好看。**

## 一、它到底在解决什么

同步扣库存的写法是：下单接口里 HTTP 调库存，成功再写订单。库存慢或店铺进程在提交前崩溃，用户刷新就会重下一单，或者钱扣了货没留。

墨坊把这两步拆开：

1. **先收下。** `POST /orders` 校验商品、写 `PENDING`、把扣库存请求放进店铺自己的发件箱，接口返回 **202**。库存还没动。
2. **再让库存说话。** 库存进程消费 RabbitMQ，按行 `update … where available >= quantity`。成功回 `RESERVED`，不够回 `REJECTED`。
3. **付款是第三步。** 只有 `RESERVED` 能付。余额够则 `PAID`；不够则 `RELEASING`，再发一条释放消息，库存加回去之后变成 `CANCELLED`。

商品库、库存、订单各用各的 Postgres。店铺用 HTTP 问商品价，用消息问库存——定价可以同步快照，扣库存必须能重试、能幂等。

## 二、分层：对外只有 8080

![墨坊进程与数据边界](/images/ink-workshop-01-architecture.svg)

| 进程 | 主类 | 端口 | 库 |
|------|------|------|----|
| 网关 | `ink.gateway.GatewayApplication` | 8080 | 无 |
| 库存 | `ink.inventory.InventoryApplication` | 8081 | `ink_inventory` |
| 商品库 | `ink.catalog.CatalogApplication` | 8082 | `ink_catalog` |
| 店铺 | `ink.workshop.InkWorkshopApplication` | 8083 | `ink_order` |
| 店铺第二份 | 同一个主类，`--server.port=8084` | 8084 | 同一份 `ink_order` |
| 店面 | `web/` Vite | 5173 | 无 |

两份店铺注册到 Nacos，服务名都是 `ink-workshop`。网关用 `lb://ink-workshop` 轮转。商品库和库存**不注册**，店铺用配置里的固定地址调它们。店面不访问 8081–8084。

响应头 `X-Instance-Port` 是实际处理请求的店铺端口，用来看见负载打到了 8083 还是 8084。

几个角色后面三篇都会撞上：

| 角色 | 文件 | 干什么 |
|------|------|--------|
| 网关 | `gateway.yml` · `ShopRateLimitFilter` | 限流、熔断、负载均衡；自己不注册 |
| 店铺订单 | `order/OrderService.java` | 收下、付款、写发件箱 |
| 发件箱 | `order/OutboxPublisher.java` · `OutboxClaims` | `FOR UPDATE SKIP LOCKED`，两份店铺抢同一张表 |
| 库存 | `inventory/DeductionService` · `ReleaseService` | 幂等扣减与回补，结果再进库存自己的发件箱 |
| 拓扑 | `messaging/StockTopology.java` | `ink.stock` 交换机 + 死信 |

> 钥匙：**订单状态写在店铺库里，库存数量写在库存库里。两边只通过消息对齐，不共享一张表。**

## 三、技术选型（以及故意不选什么）

| 层 | 选型 | 为什么 |
|----|------|--------|
| 运行时 | JDK 21 · Spring Boot 4.1 · Spring Cloud 2025.1 | 一个仓库五个主类，用 `PropertiesLauncher` 选入口 |
| 库 | Postgres 16 · Flyway · JPA `ddl-auto: validate` | 三套 migration 目录，各管各的 schema |
| 消息 | RabbitMQ · 发件箱 | 先提交本地事务，再投递；不是「接口里直接 `convertAndSend`」 |
| 发现 / 配置 | Nacos 3 | 只有店铺注册；限流数字可热更新 |
| 弹性 | Resilience4j 限流 + 熔断 | 网关挡洪水和店铺连续失败 |
| 观测 | `X-Request-Id` · Zipkin | 一条请求跨进程仍能对上 |
| 前端 | Vite，只打 `http://localhost:8080` | 店面不直连内部端口 |
| 测试 | Testcontainers Postgres + RabbitMQ | 不连本机 Nacos |

几个取舍要提前说死：

- **金额用分。** 没有 `BigDecimal` 进订单总额；`multiplyExact` / `addExact` 溢出直接炸。
- **密码走 BCrypt，拒绝超过 72 字节。** 这是阶段里单独修过的脚枪，不是装饰。
- **Kubernetes 清单在 `k8s/`，README 写明还没在集群里部署过。** 本系列不假装它已经生产就绪。
- **练习口令全是 `ink` / `nacos`。** 不要拿到正式环境。

## 四、怎么跑起来

先起基础设施，再起五个 Java 进程和店面。Windows 上多个 `mvn spring-boot:run` 会锁 `target/classes`，README 用 classpath 启动：

```bash
docker compose up -d postgres rabbitmq nacos zipkin
# Nacos 3 首次需要初始化管理员密码，见仓库 README
```

对外只访问 http://localhost:8080 。店面 http://localhost:5173 。新用户余额 50000 分；四件商品各 10 件。一笔 51200 分的单（四块油烟墨）会超过余额，正好走退库存。

```bash
mvn test
```

测试自己拉容器，不连本机 Nacos。

## 五、本系列怎么读（四篇，按论点切）

不另写 00 路线图。墨坊的核心也不大，按「逐文件九篇」拆会出水分。四篇各自只回答一个问题：

| 篇 | 只回答 | 核心代码 |
|----|--------|----------|
| **01（本篇）** | 它不是演示店面 | README、进程表、三库 |
| **02** | 一张订单怎么从 202 活到 PAID / CANCELLED | `OrderService` · 发件箱 · 状态机 |
| **03** | 库存进程为什么能重试 | `DeductionService` · `ReleaseService` · 条件更新 |
| **04** | 为什么浏览器只能看见网关 | 限流、熔断、Nacos、Vite |

建议 **01 → 04 按序读**。02 是脊柱，03 是「为什么拆进程」，04 是「失败怎么停在门口」。

这不是 Agent 系列。对照阅读（可选）：本站讲 Agent 时，策略门在工具调用前拦住副作用；墨坊的门在网关和库存 SQL 上——**副作用同样不准「调用方说了算」。**

## 六、小结

- 墨坊的产品论题是**边界清晰的失败**，不是更花的店面。
- 店铺收下订单，库存后扣；付款失败再回补。
- 三库 + 发件箱 + 两份店铺抢同一张 outbox。
- 本系列四篇按论点切，不按目录清单切。

下一篇 [**02 · 一张订单怎么活**]({{< relref "ink-workshop-02-order-life.md" >}}) 会沿着 `POST /orders` → `PENDING` → 发件箱 → 库存结果 → 付款走一遍。
