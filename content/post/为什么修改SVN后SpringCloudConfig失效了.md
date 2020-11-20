---
title: 为什么修改SVN后Spring Cloud Config失效了？
date: 2019-08-21 19:20:03
categories: ["Java", "Spring Cloud"]
tags: ["Java", "Spring Cloud"]
toc: false
---

`spring.cloud.config.server.svn.uri`修改成`新的SVN地址`之后，访问配置中心获得的配置还是`修改前SVN`的。



在我百思不得其姐的时候，我看到了配置中`spring.cloud.config.server.svn.basedir`指定了一个服务器上的目录。



于是，我上服务器看了看`spring.cloud.config.server.svn.basedir`指定的目录有什么。



果然跟我猜测的一样，这个目录其实就是SVN上Check out下来的文件。平时我们在自己电脑上换SVN的时候都会`relocate`一下，但是Spring Cloud Config没有`relocate`。



所以，解决办法就是：

1. 手动`relocate` `spring.cloud.config.server.svn.basedir`指定的目录。
2. 直接删除`spring.cloud.config.server.svn.basedir`指定的目录。