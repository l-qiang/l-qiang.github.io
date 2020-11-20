---
title: 同一个Controller的@RequestMapping指向同一个value会发生什么？
date: 2019-08-08 17:03:22
categories: ["Java", "Spring MVC"]
tags: ["Java", "Spring MVC"]
toc: false
---

首先，是如下这种写法

```java
@RequestMapping(value="/a")
    public String a() {
        return "a";
    }
    @RequestMapping(value="/a")
    public String d() {
        return "a";
}
```

这个应该都知道了，肯定会报错的

> org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'requestMappingHandlerMapping' defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Invocation of init method failed; nested exception is java.lang.IllegalStateException: `Ambiguous mapping`.

<!--more-->

那是不是value的值一定不能一样呢，答案是否定的

```java
@RestController
public class TestController {
    
    @RequestMapping(value="/a")
    public String a() {
        return "a";
    }
    
    @RequestMapping(value="/a", method = RequestMethod.HEAD)
    public ResponseEntity<Void> b() {
        return ResponseEntity.ok().header("b", "b").build();
    }
    
    @RequestMapping(value="/a", method = RequestMethod.GET)
    public String c() {
        return "c";
    }
}
```

![这是一张图片](/image/同一个Controller的@RequestMapping指向同一个value会发生什么/1.png)
![这是一张图片](/image/同一个Controller的@RequestMapping指向同一个value会发生什么/2.png)
![这是一张图片](/image/同一个Controller的@RequestMapping指向同一个value会发生什么/3.png)
从结果可以看出，只要指定不同method就行。而且会优先匹配对应的方法，没有找到匹配的才会执行没有指定method的@RequestMapping对应方法

