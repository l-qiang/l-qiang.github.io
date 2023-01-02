---
title: "SpringBoot拦截器怎样才能不拦截404请求"
date: 2022-06-21T20:05:36+08:00
lastmod: 2022-06-21T20:05:36+08:00
categories: ["Spring Boot"]
tags: ["Spring Boot"]
keywords: ["Spring Boot", "404", "拦截器"]
toc: false
---

正常情况下，请求一个没有Handler的url时，Spring MVC会返回一个404的**Whitelabel Error Page**。但是，最近给项目加了个拦截器，拦截器把404请求也给拦截住了，从DispatcherServlet的doDispatch的源码来看：

```java
// Determine handler for the current request.
mappedHandler = getHandler(processedRequest);
if (mappedHandler == null) {
  noHandlerFound(processedRequest, response);
  return;
}
```

找不到handler的时候，应该直接就返回了，不会走拦截器的逻辑才对。所以这是为啥呢？

<!--more-->

Spring MVC默认会在404的时候，转发到/error，再次执行DispatcherServlet的doDispatch，此时请求就有Handler了，所以就会走拦截器。那么怎么让404不走拦截器呢？

1. 不转发到/error。

   ```properties
   spring.mvc.throw-exception-if-no-handler-found=true
   ```

   此方式需要自定义实现404异常逻辑。

2. 拦截器排除/error路径。

   那么怎么排除呢？通过源码能跟踪到，我们的拦截器都被包装成了MappedInterceptor，然后在AbstractHandlerMapping的getHandlerExecutionChain方法中给对应的Handler添加拦截器。

   ```java
   protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
   		HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
   				(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
   
   		for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
   			if (interceptor instanceof MappedInterceptor) {
   				MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
   				if (mappedInterceptor.matches(request)) {
   					chain.addInterceptor(mappedInterceptor.getInterceptor());
   				}
   			}
   			else {
   				chain.addInterceptor(interceptor);
   			}
   		}
   		return chain;
   	}
   ```

所以只需要addInterceptors时excludePathPatterns即可。
```java
registry.addInterceptor(new MyInterceptor()).excludePathPatterns("/error")
```



{{% admonition warning "注意" %}}

默认情况下，会有ResourceHandlers对一些不存在的url进行处理，所以需要禁用它。

```properties
spring.web.resources.add-mappings=false
```

{{%  /admonition %}}



