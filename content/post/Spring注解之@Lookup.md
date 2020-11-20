---
title: 'Spring注解之@Lookup'
date: 2019-10-17 16:12:49
categories: ["Java", "Spring"]
tags: ["Java", "Spring"]
toc: false
---

有时候我们在某个单例Bean中要用到原型Bean。那么我们怎么获取原型Bean呢？

- 用@Autowired，@Resource注解注入
- 用BeanFactory的getBean方法

使用`@Autowired`，`@Resource`的话我们就没法达到原型Bean的效果。我想在一个单例Bean中多次获取原型Bean该怎么做，而且我不想用BeanFactory。

<!--more-->

`@Lookup`就能满足我的需要。

```java
@Service
public class ServiceA {
    
    public void print(String msg) {
        System.out.println(msg);
    }
    
    public void test(String msg) {
        testBean(msg).print();
    }
    
    @Lookup
    protected TestBean testBean(String msg) {
        return new TestBean(msg);
    }
}
```

上面的test方法每次调用都需要一个新的TestBean。

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class TestBean {
    
    @Autowired
    private ServiceA serviceA;
    
    private String msg;

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
    
    public void print() {
        serviceA.print(msg);
    }

    public TestBean(String msg) {
        super();
        this.msg = msg;
    }
}
```

@Lookup并不神奇，它的就是通过BeanFactory的getBean实现的