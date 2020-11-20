---
title: 踩坑SpringBoot单元测试与spring.profiles.active
date: 2020-11-03 15:45:19
categories: ["Spring Boot"]
tags: ["Spring Boot", "单元测试"]
toc: false
---

今天运行单元测试的时候，突然就报错了，大概意思就是无法解析${spring.profiles.active}，但是之前的跑完全没问呀，而且单测的类上已经加了`@ActiveProfiles`注解了。为什么呢？

<!--more-->

根据输出的错误信息，原因是使用了@Value("${spring.profiles.active}")获取activeProfiles。那为什么正常启动的时候没有问题呢？

因为加了启动参数spring.profiles.active。

但是我不想每个单测都加启动参数，太麻烦了。

那么，该怎么解决呢？

应该使用

```java
applicationContext.getEnvironment().getActiveProfiles()[0]
```

来获取activeProfiles。



使用@Value("${spring.profiles.active}")只是单纯获取他的值，如果没有设置这个值就会报错。

`Environment`除了取spring.profiles.active还可以取注解`@ActiveProfiles`的值来设置ActiveProfiles，所以应该避免使用@Value("${spring.profiles.active}")。