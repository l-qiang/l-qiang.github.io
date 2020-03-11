---
title: 解决Linux下使用Pdfbox将PDF转图片乱码问题
date: 2020-03-11 18:11:31
categories:
 - Linux
tags:
 - Linux
 - pdfbox
 - STSong Light
---

今天测试跟我说我们的PDF转换图片的时候会出现中文乱码，然而我在自己的电脑上验证的时候发现并不会出现乱码的问题，我的第一反应就是这不是代码的问题，应该是服务器的环境问题。究竟是不是跟我猜想的一样呢？

<!-- more -->

## Invalid or corrupt jarfile

我将需要验证的代码打了个jar包放到服务器上运行。但是却出现了`Invalid or corrupt jarfile`问题。

这个jar包我在本地是试过的。完全没有问题呀。

我查了查网上说的都是`没有指定Main方法`或者是`上传jar包的时候文件损坏了`。但是这跟我的情况都不符合。

那怎么办呢？

首先，这个问题是因为找不到Main方法，那么我强制指定Main方法运行。

{% codeblock lang:shell %}
java -cp pdf.jar packagename.classname
{% endcodeblock %}

一执行就出错了，`java.lang.UnsupportedClassVersionError`这个错误就好办了。`jdk版本的问题`。

## STSong Light

jdk的问题解决后jar包可以执行了，而且执行的结果确实和测试说的一样，中文显示不出来。而且在输出日志中看到了`警告`信息，缺少`STSong Light`字体。

既然缺少字体，那给我们的系统装上字体不就好了。

问题在于我们该上哪去找`STSong Light`字体。

首先，`STSong Light`字体其实就是华文宋体。我们在Windows系统`C:\Windows\Fonts`可以看到很多字体。我们只要宋体就行了。在我的系统上宋体的文件是`simsun.ttc`。

字体找到了，那么接下来就是给Linux装上这个字体。

## 安装字体

在`/usr/share/fonts/`下新建一个文件夹，名字自己定。

{% codeblock lang:shell %}
$ cd /usr/share/fonts/
$ mkdir win
$ cd win/
{% endcodeblock %}

然后将`simsun.ttc`上传到新建的文件夹里。然后执行

{% codeblock lang:shell %}
$ fc-cache -fv
$ fc-list
{% endcodeblock %}

`fc-list`执行后我们可以看到，字体里面有我们新增的宋体。

{% note success %}
如果没有`/usr/share/fonts/`文件夹。那么就是没有安装`fontconfig`。
{% codeblock lang:shell %}
$ yum -y install fontconfig
{% endcodeblock %}
使用上述命名安装就OK了。
{% endnote %}






