---
title: Jackson序列化字段脱敏处理
date: 2019-10-11 20:25:55
categories: ["Java", "Jackson"]
tags: ["Java", "Jackson", "JSON"]
toc: false
---

例如，我有如下类A，我需要A序列化为JSON是name字段值为***。

```java
public class A{

	@JsonProperty
	private String name;

}
```

<!--more-->

我们可以这么做。

```java
public class NameDesensitizeConverter extends StdConverter<String, String>{
    @Override
    public String convert(String value) {
        return "***";
    }
}
```

```java
public class A{
    @JsonProperty
    @JsonSerialize(converter = NameDesensitizeConverter.class)
    private String name;
}
```

当然，网上还有[其他的方法](https://blog.csdn.net/liufei198613/article/details/79009015)