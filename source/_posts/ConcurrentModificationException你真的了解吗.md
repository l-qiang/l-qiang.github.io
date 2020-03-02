---
title: ConcurrentModificationException你真的了解吗？
date: 2019-10-11 12:41:15
categories:
 - Java
tags:
 - Java
---

我想很多人第一次遇到这个异常都是因为对`List`使用`foreach`遍历，然后删除元素的导致的。然后我们会在网上查到使用`Iterator`或`for倒序遍历`来解决这个问题。那么`ConcurrentModificationException`是怎么出现的？为什么要使用Iterator或for倒序遍历来解决呢？

<!-- more -->

首先，`ConcurrentModificationException`是`Iterator`的`remove`方法抛出来的，而不是`List`的`remove`方法，而`foreach`的实现原理其实就是`Iterator`。既然`foreach`是通过`Iterator`实现的，那为什么使用`Iterator`可以，而使用`foreach`却不行呢？

别急，让我们先看看`Iterator`的`remove`和`List`的`remove`的区别。

{% tabs remove %}

<!-- tab Iterator -->

{% codeblock ArrayList$Itr lang:java %}
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();
    
    try {
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
{% endcodeblock %}

<!-- endtab -->

<!-- tab List -->

{% codeblock ArrayList lang:java %}
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);
    
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
    elementData[--size] = null; // clear to let GC do its work
    
    return oldValue;
}
{% endcodeblock %}

<!-- endtab -->

{% endtabs %}

可以看到，`Iterator`的的`remove`方法除了调用`List`的`remove`方法之后，还有这样一个操作

```
expectedModCount = modCount;
```

使用`foreach`的时候会调用`checkForComodification`方法

```
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

而我们使用foreach的时候一般都会习惯性的使用List的`remove`，导致了`modCount` 和 `expectedModCount`不相等。



**为什么普通的for循环不会出现`ConcurrentModificationException`异常呢？**

因为普通的for循环没有用到Iterator的方法，也就不存在`modCount` 和 `expectedModCount`是否相等的问题。



但是，普通for循环也要注意要倒序。

{% tabs for %}

<!-- tab 倒序 -->

{% codeblock lang:java %}

public static void main(String[] args) {
    List<Integer> lst = new ArrayList<Integer>(Arrays.asList(1,2,3,4,5,6,7,8,9,10));
        for(int i = lst.size() - 1 ; i >= 0 ; i--) {
            System.out.println(lst.get(i));
            lst.remove(i);
        }
}

{% endcodeblock %}

上面的代码可以正常运行。并且输出和预料的一样是10,9,8,7,6,5,4,3,2,1


<!-- endtab -->

<!-- tab 正序 -->

{% codeblock lang:java %}

public static void main(String[] args) {
    List<Integer> lst = new ArrayList<Integer>(Arrays.asList(1,2,3,4,5,6,7,8,9,10));
        for(int i = 0 ; i < lst.size() ; i++) {
            System.out.println(lst.get(i));
            lst.remove(i);
        }
}

{% endcodeblock %}

这就有问题了，输出1,3,5,7,9。原因就是删除1之后，2挪到了数组下标为0的位置，但是i已经变成1了，而1的位置已经不是2是3了，所以2就错过了。4,6,8,10也是同样的道理。

<!-- endtab -->

{% endtabs %}