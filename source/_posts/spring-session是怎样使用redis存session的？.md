---
title: spring-session是怎样使用redis存session的？
date: 2019-10-13 19:07:11
categories:
 - [Java, Spring Session]
 - Redis
tags:
 - Java
 - Spring Session
 - Redis
---

其实，我的直觉告诉我使用Hash。

我在项目里找到[RedisOperationsSessionRepository](https://docs.spring.io/spring-session/docs/current/api/org/springframework/session/data/redis/RedisOperationsSessionRepository.html)然后在注释中找到了答案

> Each session is stored in Redis as a [Hash](http://redis.io/topics/data-types#hashes). Each session is set and updated using the [HMSET command](http://redis.io/commands/hmset). 

<!-- more -->

```
HMSET spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe creationTime 1404360000000 maxInactiveInterval 1800 lastAccessedTime 1404360000000 sessionAttr:attrName someAttrValue sessionAttr:attrName2 someAttrValue2
```

- `33fdd1b6-b496-4b33-9f7d-df96679d32fe`是session id
- `creationTime 1404360000000`是session的创建时间
- `maxInactiveInterval 1800`是过期时间
- `lastAccessedTime 1404360000000`是最后访问时间
- 后面的就是一些自定义的属性了

当然，除了上面的这些，在Redis中还存了一些过期时间相关的数据

```
EXPIRE spring:session:sessions:33fdd1b6-b496-4b33-9f7d-df96679d32fe 2100
APPEND spring:session:sessions:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe ""
EXPIRE spring:session:sessions:expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe 1800
SADD spring:session:expirations:1439245080000 expires:33fdd1b6-b496-4b33-9f7d-df96679d32fe
EXPIRE spring:session:expirations1439245080000 2100

```

原因是Spring Session依赖Redis的过期键的删除触发`SessionDestroyedEvent`事件来释放资源。但是，Redis的键过期之后不能保证立马删除，所以就会有后台任务不断地访问session过期键来触发Redis过期键删除。

这个可以在`RedisOperationsSessionRepository`的注释或者[Spring Session的文档](https://docs.spring.io/spring-session/docs/2.0.0.M4/reference/html5/#api-redisoperationssessionrepository-expiration)查看详细介绍。

