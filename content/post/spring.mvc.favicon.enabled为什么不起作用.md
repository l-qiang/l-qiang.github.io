---
title: spring.mvc.favicon.enabled为什么不起作用？
date: 2019-09-05 18:01:25
categories: ["Java", "Spring Boot"]
tags: ["Java", "Spring Boot"]
toc: false
---

偶然发现不管我将spring.mvc.favicon.enabled设置成true还是false，系统表面上都没有什么变化。favicon该显示还是显示。

那么这个配置究竟有什么用呢？

<!--more-->

我在github的[issue](https://github.com/spring-projects/spring-boot/issues/17925)找到了答案。

> It only disables serving a favicon.ico from the root of the classpath. A favicon.ico that's placed in one of the static resource locations will still be served.

也就是说将`spring.mvc.favicon.enabled`设置成`false`，只是让`classpath`下的`favicon`图标不能用。而`static`下的`favicon`仍然可以用。



`WebMvcAutoConfiguration`的`FaviconConfiguration`也可以验证这一点。



favicon的请求是由浏览器发起的，所以如果要禁用需要让系统不再请求这个图标。