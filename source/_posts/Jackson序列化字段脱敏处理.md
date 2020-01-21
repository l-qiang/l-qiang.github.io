---
title: Jackson序列化字段脱敏处理
date: 2019-10-11 20:25:55
categories:
 - Java
 - Jackson
tags:
 - Java
 - Jackson
 - JSON
---

例如，我有如下类A，我需要A序列化为JSON是name字段值为***。

{% codeblock A.java lang:java %}
public class A{

​    @JsonProperty
​    private String name;

}
{% endcodeblock %}

<!-- more -->

我们可以这么做。

{% tabs code %}

<!-- tab StdConverter-> -->
{% codeblock NameDesensitizeConverter.java lang:java %}
public class NameDesensitizeConverter extends StdConverter<String, String>{
    @Override
    public String convert(String value) {
        return "***";
    }
}
{% endcodeblock %}
<!-- endtab -->

<!-- tab A -->

{% codeblock A.java lang:java %}
public class A{
    @JsonProperty
    @JsonSerialize(converter = NameDesensitizeConverter.class)
    private String name;
}
{% endcodeblock %}

<!-- endtab -->

{% endtabs %}

当然，网上还有[其他的方法](https://blog.csdn.net/liufei198613/article/details/79009015)