---
title: "究竟多余的参数是哪来的，关于multipart的事情"
date: 2022-08-05T14:11:48+08:00
lastmod: 2022-11-01T14:11:48+08:00
categories: ["Spring Boot"]
tags: ["Spring Boot"]
keywords: ["Spring Boot", "multipart"]
draft: false
---

## 问题现象

最近项目框架升级，部署到测试环境后，有业务线反映调接口失败了，并且他们没有变更。

异常的现象就是，服务端收到的所有参数都重复了一遍，也就是一个key有两个相同的value。

## 排查问题

发现问题后，我第一时间把流量切回去了，然后，业务线调接口就正常了，而且业务方坚持参数只传了一遍。所以很容易想到是我的问题，但是，我没改代码逻辑呀，谁闲着没事多加一遍参数呀。

所以我仍然质疑是业务方多传了参数，于是我上服务器抓了个包。

1. 使用**tcpdump**抓包。

   ```shell
   tcpdump -tttt -s0 -X -vv tcp port 8080 -w captcha.cap
   ```

2. 将captcha.cap下载到本地，然后使用**Wireshark**分析。

   抓到的包大概如下：

   ```
   POST /account/login?phone=155****7115 HTTP/1.1
   ...
   content-type: multipart/form-data; boundary=1753c6a6-e133-4b30-9494-83e989249882
   content-length: 876
   accept-encoding: gzip
   user-agent: okhttp/3.9.1
   ...
   
   --1753c6a6-e133-4b30-9494-83e989249882
   Content-Disposition: form-data; name="phone"
   Content-Length: 11
   
   155****7115
   ```

   很显然，在请求行和请求体中都带了同样的参数。

所以，调用方确实传了两遍参数，但是为啥项目框架升级没有问题呢？

一开始我是按照日常大家都用的调用方式测试，发现问题无法复现。在抓包之后，我尝试更改请求的方式，发现必须使用**content-type: multipart/form-data**，问题才会出现。

排查到这，我们可以确定的是multipart类型请求的原因。那么我们需要知道springMVC对multipart类型请求的处理。

背了那么多次**DispatcherServlet**，终于派上了用场。

我们看`DispatcherServlet#checkMultipart`

```java
if (this.multipartResolver != null && this.multipartResolver.isMultipart(request))
```

显然，需要处理multipart类型请求，需要**multipartResolver**。

我们看`MultipartAutoConfiguration`

```java
@AutoConfiguration
@ConditionalOnClass({ Servlet.class, StandardServletMultipartResolver.class, MultipartConfigElement.class })
@ConditionalOnProperty(prefix = "spring.servlet.multipart", name = "enabled", matchIfMissing = true)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(MultipartProperties.class)
public class MultipartAutoConfiguration {

	private final MultipartProperties multipartProperties;

	public MultipartAutoConfiguration(MultipartProperties multipartProperties) {
		this.multipartProperties = multipartProperties;
	}

	@Bean
	@ConditionalOnMissingBean({ MultipartConfigElement.class, CommonsMultipartResolver.class })
	public MultipartConfigElement multipartConfigElement() {
		return this.multipartProperties.createMultipartConfig();
	}

	@Bean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
	@ConditionalOnMissingBean(MultipartResolver.class)
	public StandardServletMultipartResolver multipartResolver() {
		StandardServletMultipartResolver multipartResolver = new StandardServletMultipartResolver();
		multipartResolver.setResolveLazily(this.multipartProperties.isResolveLazily());
		return multipartResolver;
	}

}
```

`spring.servlet.multipart`默认开启。所以，原因就是：

- 框架升级前不支持multipart，所以取不到body中的参数。
- 框架升级后默认支持multipart，所以同时取到了url和body中的参数。

## 解决方法

1. 禁用multipart

   ```properties
   spring.servlet.multipart.enabled=false
   ```

2. 业务调用方修改错误的调用方式（只传一次参数）。

由于我们应该保证框架升级前和升级后一致，所以先禁用multipart，同时推进调用方修改请求方式。
