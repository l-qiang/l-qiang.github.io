---
title: Docker Desktop安装后启动报Cannot enable Hyper-V service错误
date: 2020-05-18 20:26:46
categories: ["Docker"]
tags: ["Docker", "Docker Desktop"]
toc: false
---

今天在自己的Win10系统笔记本上装了个Docker Desktop，启动的时候报错Cannot enable Hyper-V service。

上网一查，很多都说的是在`控制面版->程序->程序和功能`的`启用或关闭Windows功能`里面把**Hyper-V**开启就好了。

但是，我的电脑的**Hyper-V**本来就是开着的呀。这确实让我头疼了好一会儿。究竟是怎么一回事呢？

<!--more-->

经过我的一番检查和上网查阅资料后发现是**BIOS**里没有开启**虚拟机**的设置。

例如：我的笔记本是`HP Elitebook 848 G4`，开机后按`F10`，然后在`Advanced`的`System Options`里开启

`Virtualization Technology(VTx)`和`Virtualization Technology for Directed(VTd)`就行了。