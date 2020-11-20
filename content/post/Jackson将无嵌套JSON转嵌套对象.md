---
title: Jackson将无嵌套JSON转嵌套对象
date: 2019-12-19 19:04:52
categories: ["Java", "Jackson"]
tags: ["Java", "Jackson", "JSON"]
toc: false
---

<!--more-->

例如，我有以下JSON

```json
{age: 18, lastname: "Liu", firstname: "Ryan"}
```

我需要将这个JSON装成Person对象

```java
class Person {
    private int age;
    private Name name;
}
class Name {
    private String lastname;
    private String firstname;
}
```

只需要使用Jackson的注解`@JsonUnwrapped`

```java
class Person {
    private int age;
    @JsonUnwrapped
    private Name name;
}
```

