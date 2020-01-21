---
title: Jackson将无嵌套JSON转嵌套对象
date: 2019-12-19 19:04:52
categories:
 - Java
 - Jackson
tags:
 - Java
 - Jackson
 - JSON
---

例如，我有以下JSON

```
{age: 18, lastname: "Liu", firstname: "Ryan"}
```

我需要将这个JSON装成Person对象

```
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

```
class Person {
    private int age;
    @JsonUnwrapped
    private Name name;
}
```

