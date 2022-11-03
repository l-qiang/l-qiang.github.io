---
title: "SpringBoot实现ResponseBody动态JSON字段过滤"
date: 2022-09-05T20:42:52+08:00
lastmod: 2022-09-05T20:42:52+08:00
categories: ["Spring Boot"]
tags: ["Spring Boot"]
keywords: ["Spring Boot", "JSON", "ResponseBody", "动态过滤"]
draft: false
---

怎么优雅地做到N个接口根据请求参数决定返回值是否包含某个字段呢？

<!--more-->

我们知道Spring MVC返回JSON数据的时候，序列化使用的是Jackson。而且，Jackson提供了自定义的字段Filter。因此，我们只需要修改序列化JSON的**ObjectMapper**。

总结就是：

1. 自定义Filter
2. 修改ObjectMapper
3. 为返回结果的类添加@JsonFilter

## 自定义Filter

```java
public static class MyFilter extends SimpleBeanPropertyFilter {

        @Override
        protected boolean include(PropertyWriter writer) {
            return include0(writer);
        }

        @Override
        protected boolean include(BeanPropertyWriter writer) {
            return include0(writer);
        }

        private boolean include0(PropertyWriter writer) {
            // 过滤逻辑，需要过滤的字段返回false
            // 可使用自定义注解标识字段
            // 例如获取request的参数进行过滤
            // request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
          	return true;
        }
    }
```



## 修改ObjectMapper

### 自定义ObjectMapper未生效

我的第一想法就是寻找SpringBoot自定义**ObjectMapper**的方法。一般SpringBoot的自动配置都会提供某某`Customizer`来自定义。

找到**JacksonAutoConfiguration**，查看源码可得知，只需要自定义一个**Jackson2ObjectMapperBuilderCustomizer**的bean即可。

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
  return builder -> {
    final SimpleFilterProvider simpleFilterProvider = new SimpleFilterProvider();
    simpleFilterProvider.addFilter("myFilterId", new MyFilter());
    builder.filters(simpleFilterProvider);
  };
}
```

然而，现实是残酷的，定义完这个之后，压根没生效。而且debug跟了好多代码，发现**ObjectMapper**确实改变了。那为啥不生效呢？

那说明没有使用我们认为的**ObjectMapper**，原因就是我自定义了**WebMvcConfigurationSupport**。

看**WebMvcAutoConfiguration**的源码，发现类上有如下注解：

```java
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
```

所以，当我们自定义**WebMvcConfigurationSupport**时，那么mvc的自动配置压根就没生效。

而**WebMvcConfigurationSupport**中的如下代码，则是罪魁祸首：

```java
@Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcValidator") Validator validator) {

		RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
		adapter.setContentNegotiationManager(contentNegotiationManager);
		adapter.setMessageConverters(getMessageConverters());
		adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer(conversionService, validator));
		adapter.setCustomArgumentResolvers(getArgumentResolvers());
		adapter.setCustomReturnValueHandlers(getReturnValueHandlers());

		if (jackson2Present) {
			adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
			adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
		}

		AsyncSupportConfigurer configurer = getAsyncSupportConfigurer();
		if (configurer.getTaskExecutor() != null) {
			adapter.setTaskExecutor(configurer.getTaskExecutor());
		}
		if (configurer.getTimeout() != null) {
			adapter.setAsyncRequestTimeout(configurer.getTimeout());
		}
		adapter.setCallableInterceptors(configurer.getCallableInterceptors());
		adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

		return adapter;
	}
```

```java
protected final List<HttpMessageConverter<?>> getMessageConverters() {
		if (this.messageConverters == null) {
			this.messageConverters = new ArrayList<>();
			configureMessageConverters(this.messageConverters);
			if (this.messageConverters.isEmpty()) {
				addDefaultHttpMessageConverters(this.messageConverters);
			}
			extendMessageConverters(this.messageConverters);
		}
		return this.messageConverters;
	}
```

这里会生成8个默认的HttpMessageConverter。跟踪代码可以看到生成**MappingJackson2HttpMessageConverter**使用的**Jackson2ObjectMapperBuilder**是新生成的，不是我们使用Jackson2ObjectMapperBuilderCustomizer自定义的。

### 自定义MappingJackson2HttpMessageConverter

覆盖**WebMvcConfigurationSupport**的extendMessageConverters方法。

```java
@Override
    protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        super.extendMessageConverters(converters);
        for (final HttpMessageConverter<?> httpMessageConverter : converters) {
            if (httpMessageConverter instanceof MappingJackson2HttpMessageConverter) {
                final MappingJackson2HttpMessageConverter converter = (MappingJackson2HttpMessageConverter) httpMessageConverter;
                final SimpleFilterProvider simpleFilterProvider = new SimpleFilterProvider();
                simpleFilterProvider.addFilter("myFilterId", new MyFilter());
                converter.getObjectMapper().setFilterProvider(simpleFilterProvider);
                break;
            }
        }
    }
```



这里需要注意的是，我选择修改MappingJackson2HttpMessageConverter而不是替换。

一开始，我替换为SpringBoot自动配置的MappingJackson2HttpMessageConverter。发现Date字段由原来的long类型的时间戳，变为格式化的字符串。原因是JacksonAutoConfiguration有如下配置：

```java
static {
		Map<Object, Boolean> featureDefaults = new HashMap<>();
		featureDefaults.put(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
		featureDefaults.put(SerializationFeature.WRITE_DURATIONS_AS_TIMESTAMPS, false);
		FEATURE_DEFAULTS = Collections.unmodifiableMap(featureDefaults);
	}
```

## 为返回结果的类添加@JsonFilter

```java
@JsonFilter("myFilterId")
```

@JsonFilter需要加在序列化的类上面，且value值为前面定义的filterId值，此处为`myFilterId`。
