---
title: SpringBoot统一异常处理与404
date: 2020-05-21 14:40:29
categories:
 - Spring Boot
tags:
 - Spring Boot
---

几天前，我收到一封邮件，邮件是**Chatra**的消息。有人问了我一个问题。从消息里我猜是关于SpringBoot统一异常处理与404的问题。当时，没啥时间就没去自己试。现在，我来说说我的解决办法。

{% asset_img 1.jpg %}

<!-- more -->



我们的项目中可能会使用`@ControllerAdvice`来进行统一异常处理。比如，我在项目中所有Controller抛出的异常会返回同一个格式的JSON数据。但是，当我们请求一个不存在的页面的时候，我们看到的是这样的。

{% asset_img 2.png %}

当然并不是404都是显示**Whitelabel Error Page**页面，通过**BasicErrorController**处理，我们可以知道，只有请求的`MimeType`是`text/html`的时候才是，如果不是`text/html`会返回一个404的JSON。

{% codeblock lang:JSON %}
{
  "timestamp": "2020-05-25T06:35:10.859+00:00",
  "status": 404,
  "error": "Not Found",
  "message": "No message available",
  "path": "/aaa"
}
{% endcodeblock %}



那么，如何让404也走统一异常处理呢。

我们在网上可能一搜就能查到，通过下面这种方式来解决。

```properties
spring.mvc.throw-exception-if-no-handler-found=true
spring.resources.add-mappings=false
```

`spring.resources.add-mappings=false`这个在我的另一篇博客中也提到过，这样设置会让静态资源没有handler。这个方式的原理就是通过这个设置使得没有handler处理404的问题，然后通过`spring.mvc.throw-exception-if-no-handler-found=true`来抛出异常。

这样坐确实能达到我们想要的效果，但是，我们的静态资源没法访问了。

那么，我想404的时候抛出异常而且静态资源能访问该怎么办呢？

其实，那个**Whitelabel Error Page**已经告诉我们答案了。我们自己处理`/error`就好了，这里我直接抛出异常，让它走统一异常处理。

{% codeblock lang:java %}
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class MyErrorController implements ErrorController {
    @Override
    public String getErrorPath() {
        return null;
    }
    @RequestMapping
    public void error() throws Exception {
        throw new Exception("404");
    }
}
{% endcodeblock %}