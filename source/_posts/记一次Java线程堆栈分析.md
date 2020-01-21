---
title: 记一次Java线程堆栈分析
date: 2019-10-30 13:44:41
categories:
 - Java
tags:
 - Java
 - fastThread
 - 线程堆栈
 - Druid
---

最新某系统经常出现无法登录，不能响应请求的情况。一开始找上我的时候，他们已经通过重启恢复了，然后让我查原因。我表示一脸懵逼，啥都没有就让我解决了，最起码得让我有东西分析吧。还好这个问题经常发生，于是就在一次故障发生的时候。

<!-- more -->

我使用`jps`查到`进程pid`，然后用`jstack`将堆栈信息保存到一个文件里，然后就用上了我不知道啥时候收藏的网站[fastThread](https://fastthread.io/)，将文件上传上去。

{% asset_img 1.png %}

嚯，这么多线程再`WAITING`状态。点进去一个看看。

{% asset_img 2.png %}

大多数线程的堆栈信息都跟上面差不多，都是在等从Druid数据库连接池等连接。



所以，原因就是数据库连接不够用了。然后我看了一眼数据库连接池的配置。

```
spring.datasource.maxActive=80
spring.datasource.removeAbandoned=false
```

以我对这个系统的了解，这个配置完全够了。



于是，我打开了`druid`的监控。

```
spring.datasource.druid.web-stat-filter.enabled=true
spring.datasource.druid.stat-view-servlet.enabled=true
```

监控页面的部分功能需要设置这个。

```
spring.datasource.druid.removeAbandoned=true
```

果然，`removeAbandoned`设置为`true`之后就有地方出现问题了，抛出了`connection holder is null`的异常错误。

这说明，有地方长时间占用数据库连接了，由于超时回收，导致事务提交的时候出现问题。找到出错的代码后发现是因为定时任务的所有操作都在一个事务里，一个定时任务`几十分钟`。其实，这个定时任务完全没必要加一个这样的大粒度事务，它的大多数操作都不必与数据库交互。所以，只要把事务粒度控制好了问题就解决了。



另外，还有一个原因是下面这个配置并不会生效。我其实挺不理解的，Spring Boot的文档写得很清楚了，为什么还有人把这些配置弄错了，而且还不验证有没有生效。

```
spring.datasource.maxActive=80
spring.datasource.removeAbandoned=false
```

因为系统用的Druid，所以`正确的配置`是下面这样的

```
spring.datasource.druid.maxActive=80
spring.datasource.druid.removeAbandoned=false
```





