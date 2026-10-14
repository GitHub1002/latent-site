---
title: "墨坊源码解读 · 02 | 一张订单怎么活：202、发件箱、再决定能不能付款"
date: 2026-10-14
slug: ink-workshop-02-order-life
draft: false
weight: 931
tags: ["墨坊", "源码解读", "订单", "Outbox", "幂等"]
summary: "下单接口先写 PENDING 和发件箱，返回 202。两份店铺用 SKIP LOCKED 抢同一行再投递 RabbitMQ。库存结果把订单推进 RESERVED 或 REJECTED；付款只接受 RESERVED，余额不够则 RELEASING。配状态机图。"
ShowToc: true
---

01 把边界钉死了。这篇顺着一张单走：从 `Idempotency-Key` 进门，到库存结果回来，再到扣余额或把货退回去。

> 一把钥匙：接口返回 202 只表示「店铺记下了」，不表示货已经留下。页面和 `curl` 都要再查，直到状态离开 `PENDING` 或 `RELEASING`。

![订单状态只在店铺库里推进](/images/ink-workshop-02-order-life.svg)

## 一、下单：先定价，再进本地事务

`OrderController.place` 拒绝空的或超过 80 字符的幂等钥匙。然后 `OrderService.place`：

1. 用行项目算出 fingerprint。同一用户同一把钥匙若已有订单，指纹对得上就原样返回（HTTP 200），对不上抛冲突。
2. `catalogService.get` 按商品库现价做快照。店铺库里的 `order_lines` 存的是当时的名字和单价，后面改价不影响这张单。
3. 本地事务里 `Order.place(..., PENDING)` + 写入 `STOCK_DEDUCTION_REQUESTED` 发件箱。新单返回 **202**。

真正的实现是：

```java
// ink.workshop.order.OrderService.place
Optional<Order> existing = orderRepository.findByUserIdAndIdempotencyKey(userId, idempotencyKey);
if (existing.isPresent()) {
  return new PlaceResult(reuse(existing.get(), fingerprint), false);
}
List<LineSnapshot> snapshots = price(request.lines());
Order saved = transactions.execute(status -> savePending(...));
return new PlaceResult(response(saved, snapshots), true);
```

`savePending` 在**同一笔事务**里写订单和 outbox。崩溃发生在提交前，两边都没有；发生在提交后、消息发出前，发件箱还在，定时任务会再投。这就是不在接口里直接 `rabbitTemplate.send` 的原因。

唯一索引 `(user_id, idempotency_key)` 兜并发：两个请求同时插，一个成功，另一个吃 `DataIntegrityViolationException`，再读已有行当重放。

## 二、两份店铺抢同一张发件箱

8083 和 8084 共用 `ink_order`。如果两个进程都把同一行发到 RabbitMQ，库存会看到重复扣减请求——后面靠库存侧幂等也能挡住，但店铺不该先制造噪声。

`OutboxClaims.claimNext` 用原生 SQL：

```sql
select id from outbox
where published_at is null
  and (claimed_at is null or claimed_at < :staleBefore)
order by id
for update skip locked
limit 1
```

抢到的进程标记 `claimed_at`，发出去再标 `published_at`。claim 超过 30 秒视为过期，另一份店铺可以接手。这是两实例才出现的问题；单进程演示店写个 `findTop20ByPublishedAtIsNull` 就够了，墨坊偏要把这份痛留在代码里。

路由很窄：`STOCK_DEDUCTION_REQUESTED` 走 `deduct`，`STOCK_RELEASE_REQUESTED` 走 `release`。发件箱里出现别的类型直接抛错——不默默丢掉。

库存进程的发件箱更简单：只有一份库存，`findTop20ByPublishedAtIsNullOrderByIdAsc` 即可。同一套「先落库再投递」，实现按进程数减配。

## 三、状态机：PENDING 只接受两种结果

`Order.applyStockResult` 只允许 `PENDING → RESERVED | REJECTED`。已经离开 `PENDING` 的结果消息是空操作。`StockResultListener` 听到 `RELEASED` 则走 `markCancelled`，而且只从 `RELEASING` 进去。

| 状态 | 怎么进来 | 用户能做什么 |
|------|----------|--------------|
| `PENDING` | 下单事务提交 | 等；再查 |
| `RESERVED` | 库存 APPLIED | 付款 |
| `REJECTED` | 库存不够 | 结束 |
| `PAID` | 余额扣成功 | 结束 |
| `RELEASING` | 余额不够，已写释放消息 | 等退库存 |
| `CANCELLED` | 库存已加回 | 结束 |

付款同样要钥匙。`pay` 先 `lockByIdAndUserId`，同一把 `payment_key` 直接返回当前状态。不是 `RESERVED` 就 `OrderNotPayableException`。`user.debit` 成功则 `PAID`；失败则 `markReleasing`，发件箱多一行 `STOCK_RELEASE_REQUESTED`，接口再给 **202**。

店面 `web/src/api.js` 只认网关，轮询自己的订单列表。刷新之后，还在扣库存或退库存的单会继续等到结果——因为真相在库里，不在浏览器内存里。

## 四、这条生命故意不做什么

- **不在下单时扣余额。** 货没留下就扣钱，补偿更脏。
- **不在下单时同步等库存。** 接口时限绑死在库存进程上，店铺无法 202。
- **不让第二份店铺再插一张「看起来一样」的单。** 钥匙 + 指纹 + 唯一索引，三道。

下一篇 [**03 · 库存为什么能重试**]({{< relref "ink-workshop-03-inventory.md" >}}) 把镜头转到 8081：条件更新、fingerprint、回补。
