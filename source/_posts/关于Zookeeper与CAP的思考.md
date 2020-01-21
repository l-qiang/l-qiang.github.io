---
title: 关于Zookeeper与CAP的思考
date: 2019-10-24 19:55:22
categories:
 - Zookeeper
tags:
 - Zookeeper
 - CAP
---

我看到很多博客都说`Zookeeper`满足了`CAP`中的`CP`。但是Zookeeper的`Follower`和`Observer`是可以处理非事务请求的。



那么，`如果一个读请求到了未同步的Follower和Observer上，那读到的数据不就是旧数据吗？那不就不一致了吗？`

<!-- more -->

带着这个疑问，我在`Zookeeper`的官方文档中找到了这么一段话。

> Sometimes developers mistakenly assume one other guarantee that ZooKeeper does not in fact make. This is: * Simultaneously Consistent Cross-Client Views* : ZooKeeper does not guarantee that at every instance in time, two different clients will have identical views of ZooKeeper data. Due to factors like network delays, one client may perform an update before another client gets notified of the change. Consider the scenario of two clients, A and B. If client A sets the value of a znode /a from 0 to 1, then tells client B to read /a, client B may read the old value of 0, depending on which server it is connected to. If it is important that Client A and Client B read the same value, Client B should should call the sync() method from the ZooKeeper API method before it performs its read. So, ZooKeeper by itself doesn't guarantee that changes occur synchronously across all servers, but ZooKeeper primitives can be used to construct higher level functions that provide useful client synchronization. (For more information, see the ZooKeeper Recipes. [tbd:..]).

所以，数据不一致的问题确实是存在的，但是可以通过sync方法来获取最新数据。