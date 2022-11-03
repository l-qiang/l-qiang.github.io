---
title: "SpringBoot匹配url忽略空格"
date: 2022-11-02T15:23:24+08:00
lastmod: 2022-11-02T15:23:24+08:00
categories: ["Spring Boot"]
tags: ["Spring Boot"]
keywords: ["url", "空格", "%20", "忽略"]
toc: false
---

最近项目框架升级，发现原本支持url带有`%20`这种编码后的空格，升级后却报404。

例如：有个Controller处理路径为`/hello/world`的请求。

升级前（**spring core 4.2.3.RELEASE**）：`//hello/world`和`/%20/hello/world`都能正常返回。

升级后（**spring core 5.3.23**）：`//hello/world`正常返回，但是`/%20/hello/world`却报**404**。

<!--more-->

我们知道Spring MVC是通过url查找匹配的handler来处理请求的。

通过**DispatcherServlet#doDispatch**方法层层跟进，我们能定位到**org.springframework.util.AntPathMatcher**是最终确定路径是否能匹配的地方。

那么，想要找出问题的原因所在，我们只需要对比升级前和升级后的版本**AntPathMatcher**有什么区别即可。

```java
// spring core 4.2.3.RELEASE
// pattern为controller的路径，path为url中的路径
String[] pattDirs = tokenizePattern(pattern); // /hello/world => [hello, world]
String[] pathDirs = tokenizePath(path); // /%20/hello/world => [hello, world]
```

```java
// spring core 5.3.23
String[] pattDirs = tokenizePattern(pattern);
// 这是新版特有的代码，就是这里直接返回false。
// 这里涉及到trimTokens变量，如果trimTokens为true，相同参数的情况下，升级前后的匹配结果就一致了
// 而且刚好5.3.23默认为false，4.2.3默认为true
if (fullMatch && this.caseSensitive && !isPotentialMatch(path, pattDirs)) {
  return false;
}

String[] pathDirs = tokenizePath(path);
```

所以，只需要将**doDispatch**时使用的**AntPathMatcher**的trimTokens设置为true即可。

那么问题来了，怎么设置呢？

一种方法是，在代码里一层层找能设置AntPathMatcher的扩展点。另一种当然就是直接搜索引擎搜了。

然后我就搜到了这个，正好与我的需求相反。[Disable trimming of whitespace from path variables in Spring 3.2](https://stackoverflow.com/questions/21047104/disable-trimming-of-whitespace-from-path-variables-in-spring-3-2)。



## 解决方法

```java
@Configuration
public class WebMvcConfiguration extends WebMvcConfigurationSupport {
  
  @Override
    protected void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setPathMatcher(antPathMatcher());
    }

    @Bean
    public AntPathMatcher antPathMatcher() {
        final AntPathMatcher antPathMatcher = new AntPathMatcher();
        antPathMatcher.setTrimTokens(true); //此为重点
        return antPathMatcher;
    }
}
```



