---
title: 使用@sun.misc.Contended注解后竟然...
date: 2020-10-20 16:44:32
categories:
 - Java
tags:
 - Java
 - 伪共享
 - 缓存行
---

一段代码使用@sun.misc.Contended注解后竟然...

<!-- more -->

竟然变快了。

先放代码。

{% codeblock lang:java %}
import java.util.concurrent.CountDownLatch;

import org.openjdk.jol.info.ClassLayout;

public class ContendedTest {
	
	private static final long N = 100000000L;
	
	private static class T {
//		private long x1,x2,x3,x4,x5,x6,x7;
		@sun.misc.Contended
		private volatile long x = 0L;
//		private long x8,x9,x10,x11,x12,x13,x14;
	}
	

	private static T[] ARR = new T[2];
	static {
		ARR[0] = new T();
		ARR[1] = new T();
	}
	
	public static void main(String[] args) throws InterruptedException {
		CountDownLatch cdl = new CountDownLatch(2);
		long start = System.currentTimeMillis();
		new Thread(() -> {
			for(int i = 0; i < N; i++) {
				ARR[0].x = 1L;
			}
			cdl.countDown();
		}).start();
		new Thread(() ->{
			for(int i = 0; i < N; i++) {
				ARR[1].x = 1L;
			}
			cdl.countDown();
		}).start();
		
		cdl.await();
		System.out.println(System.currentTimeMillis() - start);
		System.out.println(ClassLayout.parseInstance(new T()).toPrintable());
	}

}
{% endcodeblock %}

加与不加@Contended注解耗时相差`1s`（使用注解需要加上jvm参数**-XX:-RestrictContended**）。

为什么耗时会相差这么多呢？

这就涉及到两个概念**缓存行**和**伪共享**。

根据**局部性原理**我们知道CPU会取一块连续区域的数据。而缓存呢又以缓存行为单位，所以当两个CPU使用了同一个缓存行的数据就会导致数据不一致性问题，需要耗时来达到缓存一致性。

比如上面例子的**ARR[0]**和**ARR[1]**，在同一个缓存行的时候被两个CPU修改就需要耗时来达到缓存一致。

由于通常情况下，一个缓存行为**64字节**，而一个long变量为**8字节**，所以我们可以在x变量的前后加上7个long变量来使**ARR[0]**和**ARR[1]**的x不在同一缓存行，这样就不会有缓存不一致的问题。高性能队列**Disruptor**里就有这种写法。

到了Java8，有了@sun.misc.Contended注解，就可以不用变量填充来解决伪共享的问题了。

当然，想要时间得要空间来换，使用JOL可以看到不同情况下对象**T**的大小。

{% tabs jol %}

<!-- tab 无处理 -->

```
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           43 c0 00 20 (01000011 11000000 00000000 00100000) (536920131)
     12     4        (alignment/padding gap)                  
     16     8   long T.x                                       0
Instance size: 24 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

<!-- endtab -->

<!-- tab 变量填充 -->

```
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           43 c0 00 20 (01000011 11000000 00000000 00100000) (536920131)
     12     4        (alignment/padding gap)                  
     16     8   long T.x1                                      0
     24     8   long T.x2                                      0
     32     8   long T.x3                                      0
     40     8   long T.x4                                      0
     48     8   long T.x5                                      0
     56     8   long T.x6                                      0
     64     8   long T.x7                                      0
     72     8   long T.x                                       0
     80     8   long T.x8                                      0
     88     8   long T.x9                                      0
     96     8   long T.x10                                     0
    104     8   long T.x11                                     0
    112     8   long T.x12                                     0
    120     8   long T.x13                                     0
    128     8   long T.x14                                     0
Instance size: 136 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

<!-- endtab -->

<!-- tab 注解 -->

```
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           43 c0 00 20 (01000011 11000000 00000000 00100000) (536920131)
     12   132        (alignment/padding gap)                  
    144     8   long T.x                                       0
    152     0        (loss due to the next object alignment)
Instance size: 280 bytes
Space losses: 132 bytes internal + 0 bytes external = 132 bytes total
```

<!-- endtab -->

{% endtabs %}

