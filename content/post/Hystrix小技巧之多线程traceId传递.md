---
title: "Hystrix小技巧之多线程traceId传递"
date: 2022-06-22T21:03:07+08:00
lastmod: 2022-06-22T21:03:07+08:00
categories: ["Hystrix"]
tags: ["Hystrix"]
keywords: ["Hystrix", "多线程", "traceId"]
draft: false
toc: false
---

<!--more-->

```java
public class MdcHystrixConcurrencyStrategy extends HystrixConcurrencyStrategy {

    @Override
    public <T> Callable<T> wrapCallable(Callable<T> callable) {
        return new MdcAwareCallable<>(callable, MDC.getCopyOfContextMap());
    }

    private static class MdcAwareCallable<T> implements Callable<T> {

        private final Callable<T> delegate;

        private final Map<String, String> contextMap;

        public MdcAwareCallable(Callable<T> callable, Map<String, String> contextMap) {
            this.delegate = callable;
            this.contextMap = contextMap != null ? contextMap : new HashMap<>();
        }

        @Override
        public T call() throws Exception {
            try {
                MDC.setContextMap(contextMap);
                return delegate.call();
            } finally {
                MDC.clear();
            }
        }
    }
}
```

```java
@Configuration
public class HystrixConfig {

    @PostConstruct
    public void init() {
        try {
            HystrixPlugins.getInstance().registerConcurrencyStrategy(new MdcHystrixConcurrencyStrategy());
        } catch (IllegalStateException e) {
            // ignore
        }
    }

}
```

原理：

1. 主线程使用MDC.getCopyOfContextMap复制一份数据。
2. 子线程使用MDC.setContextMap设置为主线程复制的数据。

适用于所有多线程traceId传递场景。
