---
title: "JMolecules究竟是啥,竟然Spring Data JDBC都为它单独给了example"
date: 2021-04-22T10:22:27+08:00
lastmod: 2021-04-22T10:22:27+08:00
categories: ["领域驱动设计"]
tags: ["领域驱动设计", "Spring Data JDBC", "JMolecules"]
keywords: ["领域驱动设计", "Spring Data JDBC", "JMolecules", "DDD"]
draft: false
---

前两天在选择Demo工程的框架的时候，选了从未用过的`Spring Data JDBC`。在官方给出的*Examples*里，发现有个单独的模块[**jmolecules-example**](https://github.com/spring-projects/spring-data-examples/tree/main/jdbc/jmolecules)。

我感觉挺惊讶的，Spring官方竟然给一个在Github上**star**不超过`400`（截止到2021-4-22）的项目给了单独的使用示例。

但是，我看到这个项目的第一眼就**star**上了。用注解和接口表达`领域驱动设计中`的概念，实在是太`coooool`了。

所以，我就决定将这个示例[**jmolecules-example**](https://github.com/spring-projects/spring-data-examples/tree/main/jdbc/jmolecules)，clone下来研究一下。

<!--more-->

## 为啥测试用例通过不了？

所有Spring Data示例在一个repository里简直太折磨人了，clone下来下载jar包都得需要好长时间。



下载完之后，运行测试用例就报错了。

> org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'example.springdata.jdbc.jmolecules.customer.Customers' available

一脸懵逼，官方示例不应该有问题。

但是，这个错误很明显，没找到**Customers**这个`Repository`。

让我们来看看**Customers**的代码。

```java
public interface Customers extends Repository<Customer, CustomerId>, AssociationResolver<Customer, CustomerId> {
	Customer save(Customer customer);
}
```

用过Spring Data JPA的都知道，我们自定义一个接口然后继承`org.springframework.data.repository.Repository`就行了。但是这里，继承的是`org.jmolecules.ddd.types.Repository`。所以上面的错误，肯定跟这个有关系。所以，就必须了解这个`jMolecules`了。

### jMolecules

> A set of libraries to help developers work with architectural concepts in Java. Goals:
>
> - Express that a piece of code (package, class, method…) implements an architectural concept.
> - Make it easy for the human reader to determine what kind of architectural concepts a given piece of code is.
> - Allow tool integration (to do interesting stuff like generating persistence or static architecture analysis to check for validations of the architectural rules.)

从Github上的介绍以及源码可以看出来，`jMolecules`只是提供了一些**标志性**的`接口`和`注解`而已。

比如： 使用`@ValueObject`来标志`领域驱动设计`中的**值对象**

如果这个`jMolecules`只有标识作用，是不是显得没有那么实用？因此，还有另一个叫[jmolecules-integrations](https://github.com/xmolecules/jmolecules-integrations)的项目。

### jMolecules — Technology integrations

正是这个项目，可以将`jMolecules`转换成`Spring Data`中的接口及注解。

从示例的pom.xml中可以看到就有这些依赖。

而重点就是下面这个。

```xml
<plugin>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy-maven-plugin</artifactId>
    <version>${byte-buddy.version}</version>
    <executions>
        <execution>
            <goals>
                <goal>transform</goal>
            </goals>
        </execution>
    </executions>
    <dependencies>
        <dependency>
            <groupId>org.jmolecules.integrations</groupId>
            <artifactId>jmolecules-bytebuddy</artifactId>
            <version>${jmolecules-integration.version}</version>
        </dependency>
    </dependencies>
</plugin>
```

使用这个插件在`target`目录下生成class文件，测试用例就可以通过了。

生成的`Customers`代码如下：

```java
public interface Customers extends Repository<Customer, CustomerId>, AssociationResolver<Customer, CustomerId>, org.springframework.data.repository.Repository<Customer, CustomerId> {
    Customer save(Customer customer);
}
```

可以，看到生成的class文件里多了`org.springframework.data.repository.Repository`接口。

## Spring Data JDBC示例分析

示例里，更多的还是关于`Spring Data JDBC`的内容，而`jMolecules`相当于起了一个辅助的作用。

下面，分析一下这个示例的代码。

```
--customer
  |--Address
  |--Customer
  |--Customers
--order
  |--LineItem
  |--Order
  |--Orders
```

`Customers`和`Orders`很简单，分别是聚合根`Customer`和`Order`的**Repository**。

所以重点还是`Customer`和`Order`。

### 一对一

`Customer`是一个典型的**一对一**关系示例。

```java
@Getter
@ToString
@AllArgsConstructor(access = AccessLevel.PRIVATE, onConstructor = @__(@PersistenceConstructor))
public class Customer implements AggregateRoot<Customer, CustomerId> {

	private final CustomerId id;
	private @Setter String firstname, lastname;
	private List<Address> addresses;

	public Customer(String firstname, String lastname, Address address) {

		Assert.notNull(address, "Address must not be null!");

		this.id = CustomerId.of(UUID.randomUUID().toString());

		this.firstname = firstname;
		this.lastname = lastname;

		this.addresses = new ArrayList<>();
		this.addresses.add(address);
	}

	@Value(staticConstructor = "of")
	public static class CustomerId implements Identifier {
		private final String id;
	}
}
```

```java
@Value
@ValueObject
public class Address {
	private final String street, city, zipCode;
}
```

{{% admonition info %}}

这里有Lombok的注解中的`@PersistenceConstructor`，其实意思就是在构造方法上加上这个`@PersistenceConstructor`注解，它的作用可以在`Spring Data JDBC`文档上了解到。

```java
@AllArgsConstructor(access = AccessLevel.PRIVATE, onConstructor = @__(@PersistenceConstructor))
```



{{% /admonition %}}

虽然，代码里使用的是List，但是这里`Customer`和`Address`确实是一对一的。这里`jMolecules`通过AggregateRoot接口，让我们不再需要加上`@Id`注解（org.springframework.data.annotation.@Id）。

而且，对于`Spring Data JDBC`只需要`@Id`注解就够了，甚至不需要setter和getter方法。`Spring Data`不愧是贯彻`领域驱动设计`概念的框架。有时候，有些框架强制使用setter和getter方法就让我觉得很别扭。

`Customer`和`Address`的建表语句可就有点意思了。

```sql
CREATE TABLE IF NOT EXISTS customer (
	id VARCHAR(100) PRIMARY KEY,
	firstname VARCHAR(100),
	lastname VARCHAR(100)
);

CREATE TABLE IF NOT EXISTS address (
	street VARCHAR(100),
	city VARCHAR(100),
	zip_code VARCHAR(100),
	customer VARCHAR(100),
	customer_key VARCHAR(100)
);
```

`Address`是一个值对象，它随customer产生也随之消亡。

我们不需要做任何操作，在`customer`保存的时候，会将`customer`的**id**保存到`address`的**customer**字段。

而`customer_key`，以我的理解对应的是`customer`中List的index。

`customer`保存后的数据如下：

| ID                                   | FIRSTNAME | LASTNAME |
| ------------------------------------ | --------- | -------- |
| 52eb4e05-4371-4009-a03e-2fbdc3a5963d | Carter    | Matthews |

`address`的数据如下：

| STREET        | CITY          | ZIP_CODE | CUSTOMER                             | CUSTPMER_KEY |
| ------------- | ------------- | -------- | ------------------------------------ | ------------ |
| 41 Greystreet | Dreaming Tree | 2731     | 52eb4e05-4371-4009-a03e-2fbdc3a5963d | 0            |

### 一对多

`Order`与`LineItem`就是一对多的关系了。

```java
@Table("MY_ORDER")
@Getter
@ToString
@RequiredArgsConstructor
public class Order implements AggregateRoot<Order, Order.OrderId> {

	private final OrderId id;
	private @Column("FOO") List<LineItem> lineItems;
	private Association<Customer, CustomerId> customer;

	public Order(Customer customer) {

		this.id = OrderId.of(UUID.randomUUID());
		this.customer = Association.forAggregate(customer);
		this.lineItems = new ArrayList<>();
	}

	public Order addLineItem(String description) {

		LineItem item = new LineItem(description);

		this.lineItems.add(item);

		return this;
	}

	@Value(staticConstructor = "of")
	public static class OrderId implements Identifier {

		private final UUID orderId;

		public static OrderId create() {
			return OrderId.of(UUID.randomUUID());
		}
	}
}
```

这其实跟一对一是非常类似的。唯一的点就是使用了`@Table`指定了表名和`@Column`指定了列名，而且是`line_item`表的列名。

```sql
CREATE TABLE IF NOT EXISTS my_order (
	id VARCHAR(100),
	customer VARCHAR(100)
);

CREATE TABLE IF NOT EXISTS line_item (
	my_order_key VARCHAR(100),
	description VARCHAR(100),
	foo VARCHAR(100)
);
```

示例执行后`my_order`表的数据如下：

| ID                                   | CUSTOMER                             |
| ------------------------------------ | ------------------------------------ |
| 1a02ef53-db71-47ef-8985-88f907800dc8 | 52eb4e05-4371-4009-a03e-2fbdc3a5963d |

示例执行后`line_item`表的数据如下：

| MY_ORDER_KEY | DESCRIPTION | FOO                                  |
| ------------ | ----------- | ------------------------------------ |
| 0            | Foo         | 1a02ef53-db71-47ef-8985-88f907800dc8 |
| 1            | Bar         | 1a02ef53-db71-47ef-8985-88f907800dc8 |

### 多对一

`Customer`与`Order`就是`多对一`。

上面的一对一，一对多。都是聚合根被删除了，其包含的值对象也应该被删除，这是合理的。但是`Customer`与`Order`都是聚合根。一个顾客可以有多个订单，订单删除了，顾客不能删除。

这种情况在`Spring Data JDBC`该怎么办呢。

在Spring官网的博客中有篇文章中讲到过处理方式。[Spring Data JDBC, References, and Aggregates](https://spring.io/blog/2018/09/24/spring-data-jdbc-references-and-aggregates)

对应于，这个示例，就是让`Order`持有`Customer`的**ID**，而非对象。

所以，这里`jMolecules`就发挥作用了。使用`Association`来封装这个**ID**。

如果是这样，`Spring Data JDBC`怎么知道`jMolecules`的封装，又该怎么解析`Association`呢？

那就是，`JMolecules Spring integration`的自定义转换器**PrimitivesToIdentifierConverter**和**PrimitivesToAssociationConverter**。在`Spring Data JDBC`的[Custom Conversions](https://docs.spring.io/spring-data/jdbc/docs/2.2.0/reference/html/#jdbc.custom-converters)有自定义转换器的介绍。

{{% admonition info "H2" %}}

示例中使用的`H2`数据库。所以除了只需要一个`schema.sql`外，不需要什么配置信息。

如果需要，查看`H2`数据库的数据。需要添加**application.properties**配置

```properties
spring.h2.console.enabled=true
```

另外，需要加入web的依赖。

然后在项目启动的时候，可以看到访问路径为`/h2-console`。

类似下面的这种连接的`url`也可以看到

> jdbc:h2:mem:0dbb7bce-452c-4d12-b9e3-1ca04479f6f5;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=false

{{% /admonition %}}

## 总结

不管是`Spring Data JDBC`, 还是`jMolecules`都是非常优秀的项目了。就连官方的给示例项目的很经典，可谓是以最少的代码表达出了你想知道的内容。

之前在还在网上看到有博客的作者说，`Spring Data JDBC`很鸡肋，我不清楚当时的作者为什么会有这样的言论，是不是真的了解过`Spring Data JDBC`。

我只想说，**Spring Data JDBC, yyds（永远滴神）**。





