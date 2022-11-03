---
title: "Hystrix小技巧之Feign动态超时时间配置"
date: 2022-06-22T20:53:37+08:00
lastmod: 2022-06-22T20:53:37+08:00
categories: ["Hystrix"]
tags: ["Hystrix"]
keywords: ["Hystrix", "Feign", "动态超时时间"]
draft: false
---

为应对服务上线后一些二方接口的响应时间长和无法预估的问题，需要将Feign调用远程接口时的超时时间设置成动态的，指定时间内无法返回则进行降级。

二方接口优化之后能动态调整超时时间，而不需要重新发布。

<!--more-->

## 动态配置中心

超时时间的配置使用公司搭建的Etcd配置中心。使用jetcd就可以实现监听，以实现动态配置。

## 设置Feign超时时间

按正常的想法，监听到配置变动后重新使用HystrixFeign.Builder构建一个新的实例不就行了吗？

然而并不行，Feign Hystrix实际使用的是HystrixCommand，它有着一套自己的动态配置。

> Hystrix uses [Archaius](https://github.com/Netflix/archaius) for the default implementation of properties for configuration.
>
> The documentation below describes the default [`HystrixPropertiesStrategy`](http://netflix.github.io/Hystrix/javadoc/index.html?com/netflix/hystrix/strategy/properties/HystrixPropertiesStrategy.html) implementation that is used unless you override it by using a [plugin](https://github.com/Netflix/Hystrix/wiki/Plugins).

没错，正是[Archaius](https://github.com/Netflix/archaius) 。

## [Archaius](https://github.com/Netflix/archaius) 

那么，怎么使用[Archaius](https://github.com/Netflix/archaius) 实现Hystrix的动态配置呢。

从github的[wiki](https://github.com/Netflix/archaius/wiki/Users-Guide)我们能找到答案，甚至提供了动态配置的类[DynamicWatchedConfiguration](https://github.com/Netflix/archaius/blob/23222691e904a8c79020ab5a2e93ae740a1db96e/archaius-core/src/main/java/com/netflix/config/DynamicWatchedConfiguration.java)。

看这个类的构造函数：

```java
public DynamicWatchedConfiguration(WatchedConfigurationSource source, boolean ignoreDeletesFromSource,
            DynamicPropertyUpdater updater) {
        this.source = source;
        this.ignoreDeletesFromSource = ignoreDeletesFromSource;
        this.updater = updater;

        // get a current snapshot of the config source data
        try {
            Map<String, Object> currentData = source.getCurrentData();
            WatchedUpdateResult result = WatchedUpdateResult.createFull(currentData);

            updateConfiguration(result);
        } catch (final Exception exc) {
            logger.error("could not getCurrentData() from the WatchedConfigurationSource", exc);
        }

        // add a listener for subsequent config updates
        this.source.addUpdateListener(this);
    }
```

其实我们要做的就是：

1. 建立一个与我们配置中心关联的configuration。
2. 执行ConfigurationManager.install(configuration);

重点在第1步，我们可以使用DynamicWatchedConfiguration，只要提供构造函数所需的参数就行。其实选择实现WatchedConfigurationSource接口就好了。

但是，我当时没想到，选择直接继承ConcurrentMapConfiguration。在配置更新的时候调用ConcurrentMapConfiguration的**setProperty**方法（我当时咋就脑子抽风了选择这种不优雅的方式呢）。

## Hystrix Configuration

说了这么多，怎么配置超时时间呢？

从[wiki](https://github.com/Netflix/Hystrix/wiki/Configuration)里我又知道了。

| Default Value               | `1000`                                                       |
| --------------------------- | ------------------------------------------------------------ |
| Default Property            | `hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds` |
| Instance Property           | `hystrix.command.*HystrixCommandKey*.execution.isolation.thread.timeoutInMilliseconds` |
| How to Set Instance Default | `HystrixCommandProperties.Setter()   .withExecutionTimeoutInMilliseconds(int value)` |

Feign Hystrix默认是使用类加方法名作为HystrixCommandKey，我们可以使用SetterFactory自定义。

就像这样：

```java
public class HystrixSetterFactory implements SetterFactory {
    @Override
    public HystrixCommand.Setter create(Target<?> target, Method method) {
        String groupKey = target.name();
        String commandKey = Feign.configKey(target.type(), method);
        String threadPoolKey = null;
        if (method.isAnnotationPresent(HystrixCommandKeyName.class)) {
            HystrixCommandKeyName annotation = method.getAnnotation(HystrixCommandKeyName.class);
            commandKey= annotation.value();
            threadPoolKey = annotation.threadPoolKey();
        }
        return setter(groupKey, commandKey, threadPoolKey);
    }
}
```

然后调用HystrixFeign.Builder.setterFactory设置一下。

我们只要在etcd配置如下配置即可实现动态超时时间了：

```properties
hystrix.command.自定义key.execution.isolation.thread.timeoutInMilliseconds=3000
```
