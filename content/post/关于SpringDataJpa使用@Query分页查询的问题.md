---
title: 关于Spring Data Jpa使用@Query分页查询的问题
date: 2019-08-20 16:25:33
categories: ["Java", "Spring Data JPA"]
tags:  ["Java", "Spring Data JPA", "分页"]
toc: false
---

原本没有想过要用@Query来查询的，毕竟JpaRepository提供的方法已经基本够用了。但是今天这个sql比较特殊可能要用正则匹配，然后看到了ExampleMatcher里的`StringMatcher`：

```java
public static enum StringMatcher {

        /**
         * Store specific default.
         */
        DEFAULT,
        /**
         * Matches the exact string
         */
        EXACT,
        /**
         * Matches string starting with pattern
         */
        STARTING,
        /**
         * Matches string ending with pattern
         */
        ENDING,
        /**
         * Matches string containing pattern
         */
        CONTAINING,
        /**
         * Treats strings as regular expression patterns
         */
        REGEX;

}
```

有正则REGEX，我以为这样就很好办了（讲道理sql用正则真的非常慢）。然而事情并不是这么简单。因为会报一个异常`Unsupported StringMatcher REGEX`。
上面的枚举时有REGEX，但是`QueryByExamplePredicateBuilder`类的这部分代码:

```java
switch (exampleAccessor.getStringMatcherForPath(currentPath)) {

                    case DEFAULT:
                    case EXACT:
                        predicates.add(cb.equal(expression, attributeValue));
                        break;
                    case CONTAINING:
                        predicates.add(cb.like(expression, "%" + attributeValue + "%"));
                        break;
                    case STARTING:
                        predicates.add(cb.like(expression, attributeValue + "%"));
                        break;
                    case ENDING:
                        predicates.add(cb.like(expression, "%" + attributeValue));
                        break;
                    default:
                        throw new IllegalArgumentException(
                                "Unsupported StringMatcher " + exampleAccessor.getStringMatcherForPath(currentPath));
}
```

可以看到并没有REGEX，这是什么骚操作？



所以，我打算使用@Query。在官网文档找到如下示例：

```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query(value = "SELECT * FROM USERS WHERE LASTNAME = ?1",
    countQuery = "SELECT count(*) FROM USERS WHERE LASTNAME = ?1",
    nativeQuery = true)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

按照示例写好，然后一运行。非常好，报错了`Cannot use native queries with dynamic sorting and/or pagination in method...`
看下报错地方的代码：

```java
public NativeJpaQuery(JpaQueryMethod method, EntityManager em, String queryString,
            EvaluationContextProvider evaluationContextProvider, SpelExpressionParser parser) {

        super(method, em, queryString, evaluationContextProvider, parser);
    
        Parameters<?, ?> parameters = method.getParameters();
        boolean hasPagingOrSortingParameter = parameters.hasPageableParameter() || parameters.hasSortParameter();
        boolean containsPageableOrSortInQueryExpression = queryString.contains("#pageable")
                || queryString.contains("#sort");
    
        if (hasPagingOrSortingParameter && !containsPageableOrSortInQueryExpression) {
            throw new InvalidJpaQueryMethodException(
                    "Cannot use native queries with dynamic sorting and/or pagination in method " + method);
        }
}
```

从这段代码里看出，报异常的原因时我们的sql里没有`#pageable`
所以，加上这个就好了，`修改后`的代码如下：

```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query(value = "SELECT * FROM USERS WHERE LASTNAME = ?1 \n#pageable\n",
    countQuery = "SELECT count(*) FROM USERS WHERE LASTNAME = ?1",
    nativeQuery = true)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

