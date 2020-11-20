---
title: maven RSA premaster secret error SunTls12RsaPremasterSecret KeyGenerator not available
date: 2020-05-18 20:51:29
categories: ["Maven"]
tags: ["Maven"]
toc: false
---

今天使用Maven打包的时候，报了一个maven RSA premaster secret error SunTls12RsaPremasterSecret KeyGenerator not available这样的错。而前面的报错信息给我的原因是某个Jar包下载不下来。

一看到这个错我就去Maven中心仓库看了，Jar确实是存在的而且也可以下载下来。错误信息给我的帮助网址是这个https://cwiki.apache.org/confluence/display/MAVEN/PluginResolutionException。

但是，我点进去一看，根本没有符合我的这种情况的。

终于，我通过maven RSA premaster secret error SunTls12RsaPremasterSecret KeyGenerator not available找到了原因。

Eclipse里的`Window->Preferences->Java->Installed JREs`的问题，因为之前将这个由`JRE`**修改**成`JDK`了。但是JRE下的Jar包还是**修改之前**的Jar，并没有变成JDK下的Jar。所以，通过`Restore Default`，修改一下Jar的指向位置就好了。

这个坑踩得真的是，被下载不下来Jar的错误信息迷惑了。

