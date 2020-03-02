---
title: Java8漏洞-使用Collectors.toMap时，value为null报空指针异常
date: 2019-08-19 16:47:17
categories:
 - Java
tags:
 - Java
---

罪魁祸首就是`HashMap的merge`方法了，它的第一行就是这个：

{% codeblock lang:java %}
if (value == null)
    throw new NullPointerException();
{% endcodeblock %}
谁调`merge`了？
{% codeblock lang:java %}
public static <T, K, U, M extends Map<K, U>>
    Collector<T, ?, M> toMap(Function<? super T, ? extends K> keyMapper,
                                Function<? super T, ? extends U> valueMapper,
                                BinaryOperator<U> mergeFunction,
                                Supplier<M> mapSupplier) {
        BiConsumer<M, T> accumulator
                = (map, element) -> map.merge(keyMapper.apply(element),
                                              valueMapper.apply(element), mergeFunction);
        return new CollectorImpl<>(mapSupplier, accumulator, mapMerger(mergeFunction), CH_ID);
}
{% endcodeblock %}

那么怎么解决呢？
既然时merge方法造成的，那就不调merge方法。
我们用自己定义的`accumulator`,用Stream的另一个collect方法
{% codeblock lang:java %}
<R> R collect(Supplier<R> supplier,
              BiConsumer<R, ? super T> accumulator,
              BiConsumer<R, R> combiner);
{% endcodeblock %}
这个方法上面的注释写了一段这个, 前两个参数干什么用的就很清楚了，第三个参数时并行计算用来组合结果的，所以用HashMap的putAll就好了
{% codeblock lang:java %}
R result = supplier.get();
for (T element : this stream)
    accumulator.accept(result, element);
return result;
{% endcodeblock %}
所以解决问题的代码大概就是这样的
{% codeblock lang:java %}
params.stream().collect(LinkedHashMap::new, (m, v) -> m.put(v.getParam(), v.getParamValue()), LinkedHashMap::putAll);
{% endcodeblock %}

当然，升级JDK也能解决这个问题。

