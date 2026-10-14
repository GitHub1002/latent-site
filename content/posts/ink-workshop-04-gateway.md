---
title: "墨坊源码解读 · 04 | 浏览器只能看见网关：限流、熔断、两份店铺"
date: 2026-10-14
slug: ink-workshop-04-gateway
draft: false
weight: 933
tags: ["墨坊", "源码解读", "网关", "Nacos", "Resilience4j"]
summary: "对外只暴露 8080。网关用 lb://ink-workshop 轮转两份店铺，限流每秒 20 可从 Nacos 热更新，店铺连续失败后熔断返回 503。店面 Vite 只打网关。配边缘图。"
ShowToc: true
---

03 把库存进程讲清楚了。这篇证明一件更乏味的事：**后来加上的浏览器、第二份店铺、限流和熔断，没有再给内部端口开一扇门。**

> 一句话：`web/src/api.js` 的 `gateway` 写死 `http://localhost:8080`。8081–8084 是进程边界，不是产品边界。

![流量只从 8080 进来](/images/ink-workshop-04-gateway.svg)

## 一、路由：五条路，一个熔断器

`gateway.yml` 把 `/users`、`/sessions`、`/orders`、`/products`、`/inventory` 都指到 `lb://ink-workshop`。每条都挂同一个 CircuitBreaker `shop`，失败转发 `/fallback/shop`。

```java
// ink.gateway.ShopFallbackController
return ResponseEntity.status(503)
    .body(new GatewayError("SHOP_UNAVAILABLE", "店铺暂时不可用，请稍后再试"));
```

默认：窗口 10、最少 5 次调用、失败率 80%、打开后等 10 秒。测试里把阈值改小，连续打 `/products` 就能看见 forwardedUrl 变成 fallback——`ShopCircuitBreakerTest` 不需要真的店铺进程。

网关自己 **不向 Nacos 注册**（`register-enabled: false`），也排除了 DataSource 和 Rabbit 自动配置。它是门口，不是又一个店铺。

## 二、两份店铺：看见负载，而不是相信它

两份 `InkWorkshopApplication` 用同一个服务名注册。`InstancePortFilter` 在响应里写 `X-Instance-Port`。店面把这个头读出来，航行记录里能看到请求打到了哪一份。

商品库和库存不注册，是故意的：它们各一份，没必要被网关直接负载。店铺用 `ink.catalog.base-url` / `ink.inventory.base-url` 指过去。多出来的那份店铺只分担**面向用户的**读写，并和第一份抢同一张订单发件箱——02 里的 `SKIP LOCKED` 就是为这一段写的。

## 三、限流：每秒 20，改 Nacos 不用重启

`ShopRateLimitFilter` 只拦店铺那五个前缀。拿不到许可就 429，`{"code":"RATE_LIMITED",...}`。默认 `permits: 20`、`period: 1s`，超时时长 0——不够就立刻拒绝，不排队。

`nacos/ink-gateway.yaml` 里同样有这组数字。`EnvironmentChangeEvent` 一旦碰到 `ink.gateway.rate-limit*`，过滤器丢掉旧的 limiter 再建一个。README 的原话是：改完后网关会重新加载，不用重启。

这是阶段 9 的提交要演示的东西：洪水停在门口，店铺连续失败也停在门口。业务进程不自己写 Servlet 限流。

## 四、请求编号和店面

`RequestIdFilter` 认 `X-Request-Id`，没有或太长就生成 UUID，放进 MDC 和响应头。店铺调商品库时 `RequestIds.forward()` 把同一个号带过去。Zipkin 在 9411，本系列不展开采样配置，只要知道门口已经打了号。

店面是 Vite，开发端口锁死 5173，被占用就退出。CORS 只放 `localhost` / `127.0.0.1` 的 5173。登录 token 留在当前标签页。下单和付款都由页面生成 `Idempotency-Key`。这和 curl 练习是同一套门：少了钥匙，店铺直接拒。

## 五、测试和还没做的事

`mvn test` 用 Testcontainers 拉起 Postgres 和 RabbitMQ，显式关掉 Nacos。集成测试会再拉 `CatalogApplication` / `InventoryApplication` 进同一套用例——`PlaceOrderIntegrationTest` 是本仓库最值钱的说明书：下单、扣减、付款失败回补，都在这里用 HTTP 走一遍。

`k8s/platform.yaml` 和 `k8s/apps.yaml` 把同样的进程搬成 Pod，网关 NodePort 30080。README 写明：**这份清单还没有在集群里部署过。** 读到这里可以停。把未跑过的 YAML 写成「生产部署」会把 01 的论题作废。

## 六、小结

- 对外只有网关；内部端口不是 API。
- 两份店铺共享订单库，靠发件箱抢锁，不靠「希望不要并发」。
- 限流和熔断是门口的硬规则，数字可以热更新。
- 墨坊不是 Agent，但它和 PocketPilot 共用一句工程话：**副作用的许可不由调用方说了算。**

四篇读完，建议回到仓库跑一次 `mvn test`，再用四块油烟墨走一遍余额不够的回补。纸上的状态机，要被 202 和一次 429 顶过，才算读进去。
