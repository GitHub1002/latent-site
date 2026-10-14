---
title: "墨坊源码解读 · 03 | 库存为什么能重试：条件更新、指纹、付款失败再加回去"
date: 2026-10-14
slug: ink-workshop-03-inventory
draft: false
weight: 932
tags: ["墨坊", "源码解读", "库存", "幂等", "RabbitMQ"]
summary: "库存进程不共享订单表。扣减是 update … where available >= quantity，同一把钥匙用 SHA-256 指纹挡住换行重放。付款失败走另一条释放队列，结果仍经库存自己的发件箱回到店铺。配扣减回补图。"
ShowToc: true
---

02 把店铺侧的 202 和发件箱讲完了。库存进程在 8081，库是 `ink_inventory`。它不认识用户，不认识余额，只认识「这一把钥匙、这几行货，扣过没有」。

> 一把钥匙：超卖用 SQL 挡住，重复消息用指纹挡住。HTTP 在这里太容易把「超时」理解成「没扣成」。

![库存只回答扣成没扣成](/images/ink-workshop-03-inventory.svg)

## 一、为什么库存必须是另一个进程

阶段历史上，库存先从店铺里拆出去，当时还是 HTTP 扣减。后来改成 RabbitMQ，提交词写得很直：*an order is accepted before the result arrives*。

同步 HTTP 的问题不是「微服务时髦」，而是语义：

- 店铺等 2 秒超时，不知道库存到底扣没扣。
- 重试可能扣两次，不重试可能漏扣。
- 店铺崩溃发生在收到 200 之前，用户刷新再下一单。

消息把「收下」和「结果」拆开之后，库存可以慢慢做，做完自己写结果发件箱。店铺只在 `applyStockResult` 里推进状态。两边的库都不借对方的表。

商品库仍走 HTTP：定价是只读快照，超时就让这张单失败，不必上消息。`CatalogClient` 把 `X-Request-Id` 转发出去，Zipkin 上还能对上同一条请求。

## 二、扣减：条件更新 + 指纹

`InventoryRepository.deductIfEnough` 不是先 `find` 再 `setAvailable`：

```java
update Inventory inventory
set inventory.available = inventory.available - :quantity
where inventory.productId = :productId
  and inventory.available >= :quantity
```

更新行数不是 1，就当库存不够。两个扣减并发时，数据库行锁决定谁赢，输的人走 `InsufficientStockException`，不会把 `available` 写成负数——表上还有 `check (available >= 0)`。

`DeductionService.accept` 在扣之前算 fingerprint：行项目按 `productId` 排序后做 SHA-256。`stock_deductions` 以 `idempotency_key` 为主键。

| 情况 | 行为 |
|------|------|
| 钥匙未见过，库存够 | 逐行扣减，记下 `APPLIED`，发件箱写 `RESERVED` |
| 钥匙未见过，库存不够 | 不扣（或事务回滚后记 `REJECTED`），发件箱写 `REJECTED` |
| 钥匙见过，指纹相同 | 不再扣，把当时的结果再发一遍 |
| 钥匙见过，指纹不同 | `IdempotencyConflictException` |
| 同一消息里两个相同 `productId` | 直接拒绝 |

店铺侧的钥匙是 `userId + ":" + idempotencyKey`。库存不需要知道用户是谁，只要这把钥匙在自己的表里唯一。

并发插入撞主键时，`accept` 会 `replay`：读已有行、核对指纹、重发结果。消息可以丢、可以重复，库存数量只按成功的那一次变。

## 三、回补：另一条队列，同一套纪律

付款失败并不是 `available += quantity` 写在店铺里。店铺没有库存表。它往发件箱放 `STOCK_RELEASE_REQUESTED`，钥匙是 `release:userId:paymentKey`。

`ReleaseService` 同样用指纹和主键。`inventoryService.restore` 走 `addAvailable`。成功后发件箱写 `RELEASED`，店铺 `markCancelled`。

释放队列和扣减队列分开（`ink.stock.release` / `ink.stock.deduct`），死信也分开。扣失败不该和退货抢同一条消费者逻辑。`StockTopology` 把 DLX 绑好：重试三次仍失败进死信，不在业务队列里无限打转。

种子数据很土：四件商品各 10 件。新上架的货库存先是 0，要在「舰务」里再补给——商品库插入和库存插入不是同一事务，这是拆库的老实代价。

## 四、库存进程故意不做什么

- **不读订单表。** 结果消息只带 `orderId` 和状态。
- **不在扣减接口里等店铺确认。** 它写自己的 outbox 就结束。
- **不把「查询库存」当成扣减。** 查询走店铺反代的 HTTP；扣减只走消息。

下一篇 [**04 · 浏览器只能看见网关**]({{< relref "ink-workshop-04-gateway.md" >}}) 把 8080 上的限流、熔断和两份店铺打开。
