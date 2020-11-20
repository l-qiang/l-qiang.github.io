---
title: STS4启动报错Missing tools.jar
date: 2020-02-17 12:42:52
categories: ["Spring", "Spring Tool Suite"]
tags: ["Spring Tool Suite", "STS4"]
toc: false
---

今天将我的STS3升级到了STS4，升级完启动STS4的时候发现报`Missing tools.jar`的错误。

![这是一直图片](/image/STS4启动报错Missingtools.jar/1.png)

<!--more-->
从图中我们可以看出STS去C盘找`tools.jar`了。但是我的JDK并不是装在这个目录。我想大多数人的JDK都不在图上的目录吧。

那么这个路径从哪来的呢？

我们通过`Help -> About Spring Tool Suite 4 -> Installation Details -> Configuration`可以看到：

![这是一直图片](/image/STS4启动报错Missingtools.jar/2.png)

那么，这个配置该怎么修改呢？

我们打开STS4的安装目录，找到`SpringToolSuite4.ini`增加一下两行

```
-vm
D:\soft\java8\bin\javaw.exe
```

将`D:\soft\java8\bin`修改为自己电脑上的JDK目录即可。注意，这两行要加在`-vmargs`上面。



{{% admonition tip "tip" %}}
u1s1，IDEA真的太好用了，用什么STS
{{% /admonition %}}

