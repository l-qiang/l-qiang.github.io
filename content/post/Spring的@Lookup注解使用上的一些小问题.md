---
title: "Spring的@Lookup注解使用上的一些小问题"
date: 2021-04-13T18:42:43+08:00
lastmod: 2021-04-13T18:42:43+08:00
categories: ["Spring"]
tags: ["Spring"]
keywords: ["Spring", "Lookup"]
toc: false
draft: false
---

关于为什么要使用`@Lookup`的原因，在我之前的博客[Spring注解之@Lookup]({{<ref "Spring注解之@Lookup">}})中有写到。

<!--more-->

先看一段有问题的代码。

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class AHandler extends Handler{

    public AHandler(Handler next) {
        super(next);
    }

    @Override
    public void d0() {
        System.out.println("A");
    }
}

@Component
public class HandlerFactory {

    private Handler handler;

    public HandlerFactory() {
        this.handler = aHandler();
    }

    @PostConstruct
    protected void init() {

    }

    @Lookup
    private AHandler aHandler() {
        return new AHandler(null);
    }
}
```

这段代码里有**3**个问题。3个什么问题，暂且不说，下面先简述一下`@Lookup`注解的原理。

以上面的错误代码为例：

- Spring通过**Cglib**创建*HandlerFactory*的增强**代理**子类，然后通过代理类处理`@Lookup`。
- `LookupOverrideMethodInterceptor`通过**BeanFactory**的**getBean**方法获取*AHandler*实例。



问题在哪呢？

1. 不能再*HandlerFactory*的`构造函数`中调用aHandler()方法。

   上面的代码这么调用，相当于**没有使用**`@Lookup`。因为想要`@Lookup`生效，必须要先有代理类，而执行构造方法的时候还没有。

2. 不能使用`private`修饰aHandler()方法。

   **Cglib**不能代理`private`修饰的方法，所以`@Lookup`也会失效。

3. aHandler()方法应该传入需要的参数。

   这里，从aHandler()传入一个null和直接使用new AHandler(null)使用区别的。前者使用BeanFactory的**getBean(Class<T> requiredType, Object... args)**，后者使用**getBean(Class<T> requiredType)**。

   详见`LookupOverrideMethodInterceptor`的**intercept**。



所以，可以将错误的代码改成这样。

```
@Component
public class HandlerFactory {

    private Handler handler;

    public HandlerFactory() {

    }

    @PostConstruct
    protected void init() {
        this.handler = aHandler(null);
    }

    @Lookup
    protected AHandler aHandler(Handler next) {
        return new AHandler(next);
    }
}
```









