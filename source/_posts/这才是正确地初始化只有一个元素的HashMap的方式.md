---
title: 这才是正确地初始化只有一个元素的HashMap的方式
date: 2019-09-04 17:30:26
categories:
 - Java
tags:
 - Java
---

有时候我们需要一个只放一个元素的Map。 可能一开始是这样的

{% codeblock lang:java %}
var map = new HashMap<String, Object>(1);
{% endcodeblock %}

上面这种写法相当于

{% codeblock lang:java %}
var map = new HashMap<String, Object>(1, 0.75f);
{% endcodeblock %}

这种写法正确吗？

<!-- more -->

我们知道`threshold = capacity * loadFactor`

让我们看看`HashMap中的resize方法`的这部分代码

{% codeblock lang:java %}
if (newThr == 0) {
    float ft = (float)newCap * loadFactor;
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?(int)ft : Integer.MAX_VALUE);
}
threshold = newThr;
{% endcodeblock %}
所以上面这种写法，在putVal开始的时候会调用resize()，导致threshold ==0（1*0.75强转int），然后在putVal的结束的时候
{% codeblock lang:java %}
if (++size > threshold)
    resize();
{% endcodeblock %}
1 > 0 ，所以会再次调用resize()，这样Map的这个table长度就变成2了，但是Map此时是只有一个元素的。
但是，用这种方式就没问题了
{% codeblock lang:java %}
var map = new HashMap<String, Object>(1, 1.f);
{% endcodeblock %}
当然，有时候可以用`Collections.singletonMap`(一个不可变的Map，只包含一个Key和一个Value)
