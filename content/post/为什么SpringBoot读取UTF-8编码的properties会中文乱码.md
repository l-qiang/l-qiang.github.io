---
title: 为什么Spring Boot读取UTF-8编码的properties会中文乱码？
date: 2020-03-01 09:41:04
categories: ["Spring Boot"]
tags: ["Spring Boot", "乱码", "UTF-8"]
---

我们经常会在`properties`文件中使用中文。当然大多数时候中文都是写在注释里。我们也会在STS中安装[PropertiesEditor](https://java-properties-editor.com/index.html)插件显示中文。但是，在我升级STS4之后我没有安装`PropertiesEditor`，而是直接修改`properties`文件为`UTF-8`编码，然后在里面写中文。这样就发生了一个问题，应用程序读取出来的
`properties`属性里中文是乱码的。
<!--more-->

## 那么这是为什么呢？
我们可以追溯到Spring Boot加载`properties`文件的一段代码。

```java
/**
 * Load properties from the given resource (in ISO-8859-1 encoding).
 * @param resource the resource to load from
 * @return the populated Properties instance
 * @throws IOException if loading failed
 * @see #fillProperties(java.util.Properties, Resource)
 */
public static Properties loadProperties(Resource resource) throws IOException {
		Properties props = new Properties();
		fillProperties(props, resource);
		return props;
}
```

我们从注释中可以看到，使用`ISO-8859-1`编码加载资源文件。所以`UTF-8`编码的文件用`ISO-8859-1`编码读取出来所以中文乱码了。

那么，为什么使用`PropertiesEditor`就没有问题呢？

因为，`PropertiesEditor`只是让你看到中文，但是实际文件里**\u**xxxx的形式的中文（用文本编辑器打开properties文件看看）。

在这之前还有`native2ascii`这个东西，想了解的也可以去了解一下。



## 关于这个问题，我是怎么解决的呢？

我没有用`PropertiesEditor`，因为我希望我的一些配置不用`PropertiesEditor`仍然能看懂什么意思。

所以，我把我的配置文件分成了`两份`，属于系统的一些配置仍然放在`application.properties`里面使用`ISO-8859-1`编码，属于业务相关的配置放到了另一个配置文件`xxqg.properties`使用`UTF-8`编码。



然后，读取配置的地方使用`@PropertySource`注解。

例如，以下代码

```java
@Configuration
@ConfigurationProperties(prefix="xpath.agreement")
@PropertySource(value = "xxqg.properties", encoding = "UTF-8")
public @Data class AgreementXPath {
	private String agree;

}
```



