---
title: Spring Boot注解之@ConditionalOnProperty
date: 2019-09-04 18:09:42
categories:
 - Java
 - Spring Boot
tags:
 - Java
 - Spring Boot
---
{% codeblock lang:java %}

@ConditionalOnProperty(value = "spring.mvc.favicon.enabled",
				matchIfMissing = true)

{% endcodeblock %}

- 当配置文件中**没有**`spring.mvc.favicon.enabled`时，由于`matchIfMissing = true`**，属于条件**匹配**。

- 当配置文件中**有**`spring.mvc.favicon.enabled`时，此时配合`havingValue`，如下表。`havingValue`默认为`“”`，`spring.mvc.favicon.enabled=true`时属于条件**匹配**(yes)， `spring.mvc.favicon.enabled=false`时属于条件**不匹配**(no)

| Property Value | havingValue="" | havingValue="true" | havingValue="false" | havingValue="foo" |
| -------------- | -------------- | ------------------ | ------------------- | ----------------- |
| "true"         | yes            | yes                | no                  | no                |
| "false"        | no             | no                 | yes                 | no                |
| "foo"          | yes            | no                 | no                  | yes               |

