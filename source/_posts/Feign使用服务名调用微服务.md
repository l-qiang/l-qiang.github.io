---
title: Feign使用服务名调用微服务
date: 2019-05-22 14:47:28
categories:
 - Java
 - Spring Cloud
tags:
 - Java
 - Spring Cloud
 - Feign
---

在这之前都是使用@FeignClient，在@FeignClient中的name指定服务名。

{% codeblock lang:java %}
@FeignClient(
    name = "file-upload-service",
    configuration = FileUploadServiceClient.MultipartSupportConfig.class
)
public interface FileUploadServiceClient extends IFileUploadServiceClient {
    public class MultipartSupportConfig {
        @Autowired
        private ObjectFactory<HttpMessageConverters> messageConverters;
    
        @Bean
        public Encoder feignFormEncoder () {
        return new SpringFormEncoder(new SpringEncoder(messageConverters));
    }
  }
}
{% endcodeblock %}

这种方式有地方不能满足我的要求，于是我就采用了这种方式

{% codeblock lang:java %}
public class BankService {
  public static void main(String[] args) {
    Bank bank = Feign.builder().decoder(
        new AccountDecoder())
        .target(Bank.class, "https://api.examplebank.com");
  }
}
{% endcodeblock %}
一开始，我从github上Feign的示例看到使用服务名代替url的Host部分有个示例
{% codeblock lang:java %}
public class Example {
  public static void main(String[] args) {
    MyService api = Feign.builder()
          .client(RibbonClient.create())
          .target(MyService.class, "https://myAppProd");
  }
}
{% endcodeblock %}
刚好我这项目也用到了Ribbon，但是在RibbonClient的介绍中看到了这个
> This integration relies on the Feign Target.url() being encoded like https://myAppProd where myAppProd is the ribbon client or loadbalancer name and myAppProd.ribbon.listOfServers configuration is set.

这个意思是需要设置.ribbon.listOfServers，如果设置这个的话不还是得在配置文件中写IP和端口吗？我觉得不合理，然后查看spring cloud的文档，我看到了

{% blockquote Spring Cloud 文档 https://cloud.spring.io/spring-cloud-static/Greenwich.SR4/single/spring-cloud.html#spring-cloud-ribbon-without-eureka How to Use Ribbon Without Eureka %}

Eureka is a convenient way to abstract the discovery of remote servers so that you do not have to hard code their URLs in clients. However, if you prefer not to use Eureka, Ribbon and Feign also work. Suppose you have declared a `@RibbonClient` for "stores", and Eureka is not in use (and not even on the classpath). The Ribbon client defaults to a configured server list. You can supply the configuration as follows:

**application.yml.** 

```
stores:
  ribbon:
    listOfServers: example.com,google.com
```

{% endblockquote  %}

不使用Eureka的时候才需要设置.ribbon.listOfServers，但是项目是使用Eureka的，应该不需要设置才对，但是不设置的话，用这个方式调用是不行的，根据服务名找不到服务。

{% codeblock lang:java %}
MyService api = Feign.builder()
          .client(RibbonClient.create())
          .target(MyService.class, "https://myAppProd");
{% endcodeblock %}

但是使用`RestTemplate`加上`@LoadBalanced`是没有问题的，通过debug源码，发现使用@LoadBalanced是有个地方会加载服务列表，但是使用Feign的RibbonClient不会。



因为使用`@FeignClient`的时候是可以找到服务的，说明Feign完全可以支持使用服务名调用，于是我查看Feign中Client的实现有哪些。

```
Client - feign
--Default - feign.Client
--LoadBalancerFeignClient - org.springframework.cloud.netflix.feign.ribbon
--RibbonClient - feign.ribbon
```

很好，有个LoadBalancerFeignClient，我猜使用这个就可以达到服务名调用，于是查看怎么实例化这个Client，一搜构造方法的调用就找到了这个

{% codeblock lang:java %}
@Configuration
class DefaultFeignLoadBalancedConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
                              SpringClientFactory clientFactory) {
        return new LoadBalancerFeignClient(new Client.Default(null, null),
                cachingFactory, clientFactory);
    }
}
{% endcodeblock %}

这都实例化好了，所以直接注入就可以使用。



所以，最终解决方法就是：

{% codeblock lang:java %}
@Autowired
    private Client feignClient;
    
    public Result test() {
        TestApi test= Feign.builder()
                               .encoder(new FormEncoder(new JacksonEncoder()))
                               .decoder(new JacksonDecoder())
                               .client(feignClient)
                               .target(TestApi .class, "http://service-name/");
    return test.test();
}
{% endcodeblock %}