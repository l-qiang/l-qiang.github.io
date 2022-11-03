---
title: "Hystrix小技巧之返回CompletableFuture"
date: 2022-06-22T20:58:32+08:00
lastmod: 2022-06-22T20:58:32+08:00
categories: ["Hystrix"]
tags: ["Hystrix"]
keywords: ["Hystrix", "CompletableFuture"]
toc: false
draft: false
---

<!--more-->

```java
final class ObservableCompletableFuture<T> extends CompletableFuture<T> {

  private final Subscription sub;

  ObservableCompletableFuture(final HystrixCommand<T> command) {
    this.sub = command.toObservable().single().subscribe(ObservableCompletableFuture.this::complete,
        ObservableCompletableFuture.this::completeExceptionally);
  }


  @Override
  public boolean cancel(final boolean b) {
    sub.unsubscribe();
    return super.cancel(b);
  }
}
```

此方式从**HystrixInvocationHandler**获得，由于feign-hystrix支持返回CompletableFuture，那么只需查看其源码即可完美复制。

由于**ObservableCompletableFuture**属于非public类，想要使用，我们自定义一个一样的即可。
