---
title: Tomcat日志截断
date: 2020-01-22 12:30:43
categories: ["Tomcat", "Linux"]
tags: ["Tomcat", "logrotate", "Linux"]
toc: false
---

为了避免生产环境`Tomcat`的`catalina.out`越来越大，我们需要轮替`catalina.out`，那么我们该怎么做呢？

关于这个问题，我在`Tomcat`的`FAQ`[^1]中找到了答案。

<!--more-->

{{% admonition question "How do I rotate catalina.out?" %}}

CATALINA_BASE/logs/catalina.out does not rotate. But it should not be an issue because nothing should be printing to standard output since you are using a logging package, right?

If you really must rotate catalina.out, here are some techniques you can use:

1. If you are using jsvc 1.0.4 or later (from [Apache Commons Daemon](https://commons.apache.org/proper/commons-daemon/) project) to launch Tomcat, you can send SIGUSR1 signal to jsvc to get it to re-open its log files ([Jira Ticket](https://issues.apache.org/jira/browse/DAEMON-95)). You can couple this with 'logrotate' or your favorite log-rotation utility (including good-old 'mv') to re-name catalina.out at intervals and then get jsvc to re-open the original (catalina.out) file and continue writing to it.
2. Use 'logrotate' with the 'copytruncate' option. This allows you to externally rotate catalina.out without changing anything within Tomcat.
3. Modify bin/catalina.sh (or bin/catalina.bat) to pipe output from the JVM into a piped-logger such as [cronolog](https://linux.die.net/man/1/cronolog) or Apache httpd's [rotatelogs](https://httpd.apache.org/docs/2.4/logs.html#piped) (note that the previous reference is for Apache httpd documentation and *is not applicable to Tomcat* – it merely illustrates the concept).
   See also the patch in [Bug 53930, "Allow capture of catalina stdout/stderr to a command instead of just a file"](https://bz.apache.org/bugzilla/show_bug.cgi?id=53930).

{{% /admonition %}}

大概意思是`catalina.out`的轮替不应该是一个问题，因为你用了日志包的话没啥东西会打印到标准输出。但是你真的要轮替`catalina.out`有三种方式。

1. 使用jsvc 1.0.4 +
2. 使用logrotate
3. 使用cronolog或者rotatelogs

我选择了第二种，因为第二种最简单，最常用。而且不需要改Tomcat的任何东西。最重要的是，我们的`Linux`服务器已经有logrotate（一般都有的）。



那么，我们该怎么做呢？很简单，只要做一件事就行了。

1. 切换到`/etc/logrotate.d`

   ```shell
   cd /etc/logrotate.d
   ```

2. 新建文件

   ```
   touch tomcat
   ```

3. 编辑新建的文件

   ```
   vim tomcat
   ```

   然后将下面的内容写到文件里就好了
   
   ```
   /app/apache-tomcat-8.0.43/logs/catalina.out
   {
       copytruncate
       daily
       dateext
       rotate 7
       compress
       missingok
       size 10M
   }
   ```
   
   - `/app/apache-tomcat-8.0.43/logs/catalina.out`是`你的catalina.out的路径`
   - `copytruncate` 就是关键了，复制截断
   - `daily`表示每天轮替
   -  `dateext`表示使用日期作为后缀
   - `rotate 7`表示轮替最多保留之前的数据几次，超出的将被删除或邮件接收
   -  `compress`表示轮替下来的日志会被压缩
   - `missingok`表示如果日志丢失，不报错继续滚动下一个日志
   - `size 10M`表示日志文件超过10M才轮替
   
   [更多配置项>>>](https://linux.die.net/man/8/logrotate)

做完上面这些我们就等着定时执行就好了，我们也可以手动强制执行轮替来试试。

```shell
/usr/sbin/logrotate -vf /etc/logrotate.d/tomcat
```

[^1]:Tomcat FAQ https://cwiki.apache.org/confluence/display/TOMCAT/Logging#Logging-Q10 

