---
title: Spring Cloud升级记录
date: 2020-09-24 13:22:26
categories: ["Spring Cloud", "Spring Boot"]
tags: ["Spring Cloud", "Spring Boot"]
---

 最近接到一个将所有项目的Spring Cloud升级到最新版的任务。一开始我觉得挺不容易的，比较目前我们的版本很低，真正做起来发现并没有想象中难。

`Spring Cloud`升级版本为~~Edgware.SR1~~ => **Hoxton.SR8**。

`Spring Boot`升级版本为~~1.5.20.RELEASE~~=> **2.3.3.RELEASE**。

<!--more-->

## Eureka Server

我们之前是使用**Eureka**做服务注册与发现，这次升级我继续使用**Eureka**。

我将Eureka Server项目的**pom.xml**里的`Spring Cloud`和`Spring Boot`的版本号改为目标版本号。

不出意外，此时，项目就报错了。由于Eureka Server的是没有业务代码的，所以需要修改的代码并不多。

- **pom.xml**

  改完版本号，发现**hystrix**和**eureka**的依赖都报错了，这里可以从Spring Cloud的[官方文档](https://docs.spring.io/spring-cloud-netflix/docs/2.2.5.RELEASE/reference/html/)了解到，这些依赖的jar包的**artifactId**都改了。

  `修改前`
  
  ```xml
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-eureka-server</artifactId>
  </dependency>
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-hystrix</artifactId>
  </dependency>
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
  </dependency>
  ```
  
  `修改后`
  
	```xml
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
  </dependency>
  <dependency>
  	<groupId>org.springframework.cloud</groupId>
  	<artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
  </dependency>
  ```

- **Eureka Server配置**

  ```properties
  eureka.client.serviceUrl.defaultZone=http://${service.center.username}:${service.center.password}@localhost:9999/eureka/
  ```

  这里添加了eureka.client.serviceUrl.defaultZone的配置，且指向自己。是因为控制台一直报错**Connect to localhost:8761 time out**，具体原因我们都可以再网上查到。

- **jar包升级**
  在我们修改完代码和配置文件，项目的代码没有红叉叉的时候。启动项目可能会有这样的信息。

  > <span id="error1">Description:</span>
  >
  > The bean 'multipartConfigElement', defined in class path resource [org/springframework/boot/autoconfigure/web/servlet/MultipartAutoConfiguration.class], could not be registered. A bean with that name has already been defined in class path resource [com/h3c/config/MultipartConfig.class] and overriding is disabled.
  >
  > Action:
  >
  > Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true

  如果bean是自己代码，则需要自己修改（[见后文](#filetemp)）。如果不是，可以尝试升级对应的jar包。

- **Spring Security**
  由于我们的Eureka Server使用了简单的`HTTP Basic`认证。所以，有关于Spring Security的配置。

  在上面的操作都完成后，就可以成功启动项目了。但升级还未结束，启动完后，访问`http://localhost:9999`发现，验证方式变成Form表单验证了，而且输入用户名密码告诉我错了。

  此时，需要进行两处修改。

  此时，需要进行两处修改。

  1. `application.properties`

     ~~security.user.name~~ =>**spring.security.user.name**

     ~~security.user.password~~=>**spring.security.user.password**

  2. `WebSecurityConfigurerAdapter`

     继承WebSecurityConfigurerAdapter，并覆盖对应方法。

     ```java
     import org.springframework.context.annotation.Configuration;
     import org.springframework.security.config.annotation.web.builders.HttpSecurity;
     import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
     import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
     
     @Configuration
     @EnableWebSecurity
     public class WebSecurityConfig extends WebSecurityConfigurerAdapter{
         @Override
         protected void configure(HttpSecurity http) throws Exception {
             http.csrf().disable(); // 不加这个也不行
             http.authorizeRequests().anyRequest().authenticated().and().httpBasic();
         }
      }
     ```

## Eureka Client

上面说过的就在这不说了，下面的一些问题都跟项目相关，是否会遇到取决于是否使用了相关的依赖。

- <span id="filetemp">**Spring Boot文件上传配置更新**</span>

  之前的Spring Boot需要配置**MultipartConfigElement**设置文件上传的临时目录，升级之后会[报错](#error1)。

  升级之后使用`spring.servlet.multipart.location`来设置文件上传的临时目录。更多相关配置见[官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html#web-properties)

- **flyway配置修改**

  ~~flyway.baseline-on-migrate~~=> **spring.flyway.baseline-on-migrate**

  ~~flyway.locations~~ => **spring.flyway.locations**

- **Spring Data JPA 相关接口名称及构造方法改变**

- **Logger**

  之前都是用的`org.apache.log4j.Logger`，现改为`org.slf4j.Logger`。这种遍布所有代码且不统一的话，最好还是自己再封一层，以后修改会方便很多，不用到处改。

- **This may be the result of an unspecified view, due to default view name generation**

  访问页面报错，这个问题网上也有解决办法，什么view和path不能同名啥的，引入thymeleaf依赖之类的。

  这些在我这可不好使，我典型的逆反心理，我就是要同名。而且我没用thymeleaf，我引什么这个依赖。

  直接一个**spring.freemarker.suffix=.ftl**解决，造成问题的原因就是找不到视图，由于默认freemarker视图后缀是`.ftlh`，但是我们项目中的都是`.ftl`，所以修改一下视图后缀就好了。
  
- **Feign**

  我们项目的Feign可以说是用的很老版本的了，当时的Feign Form甚至不支持多文件。基本不需要太大的改变。

  首先，是依赖包修改。

  `修改前`

  ```xml
  <dependency>     
    	<groupId>org.springframework.cloud</groupId>     
    	<artifactId>spring-cloud-starter-feign</artifactId> 
  </dependency> 
  <dependency>     
    	<groupId>io.github.openfeign.form</groupId>     
    	<artifactId>feign-form</artifactId>     
    	<version>3.2.2</version> 
  </dependency> 
  <dependency>     
    	<groupId>io.github.openfeign.form</groupId>    
    	<artifactId>feign-form-spring</artifactId>    
    	<version>3.2.2</version> 
  </dependency>
  ```

  `修改后`

  ```xml
  <dependency>     
    	<groupId>org.springframework.cloud</groupId>     
    	<artifactId>spring-cloud-starter-openfeign</artifactId> 
  </dependency> 
  ```

  然后，启动就报了一个错，**The bean '服务名.FeignClientSpecification' could not be registered. A bean with that name has already been defined and overriding is disabled.**

  这个错是因为`@FeignClient`有多个同`name`的，通过在`@FeignClient`中指定`contextId`即可解决。

- **Spring Security**

    升级Spring Boot后，Spring Security也升级成5.x，这对于我目前的项目来说，最大的影响就是PasswordEncoder。我们之前是使用MD5加密，升级之后是没有`Md5PasswordEncoder`这个类了。

    但是，我们可以使用`new MessageDigestPasswordEncoder("MD5")`，这个类也提示我们弃用了，所以最好的办法是升级我们的密码编码方式。

    为了能`兼容`以前的MD5的密码，我通过以下的方式来修改Spring Security配置的PasswordEncoder。

  ```java
  DelegatingPasswordEncoder encoder = (DelegatingPasswordEncoder) PasswordEncoderFactories.createDelegatingPasswordEncoder();
    encoder.setDefaultPasswordEncoderForMatches(new MessageDigestPasswordEncoder("MD5"));
  ```

- **Spring Session Data Redis**

  除了[官方文档](https://docs.spring.io/spring-session/docs/2.3.1.RELEASE/reference/html5/)的说明之外，再说三点。

  1. 需要加上Jedis的依赖。

     ```xml
     <dependency>
     	<groupId>redis.clients</groupId>
     	<artifactId>jedis</artifactId>
     </dependency>
     ```

  2. 不需要再自定义Bean`RedisOperationsSessionRepository`，且已弃用，直接注入使用`RedisIndexedSessionRepository`即可。

  3. ConfigureRedisAction的配置依然需要，一开始我是把所有配置全删了。

     ```java
     @Bean  
     public ConfigureRedisAction configureRedisAction() {  
         return ConfigureRedisAction.NO_OP;  
     }
     ```

## Spring Cloud Config

版本一改直接启动，毫无问题

## Zuul

~~spring-cloud-starter-zuul~~ => **spring-cloud-starter-netflix-zuul**