---
title: "Jenkins安装与使用"
date: 2020-12-02T13:49:36+08:00
categories: ["Jenkins"]
tags: ["Jenkins", "Linux", "OpenJDK"]
keywords: ["Jenkins", "Linux", "OpenJDK"]
draft: true
---

## 安装准备JDK

这里，我安装`java-1.8.0-openjdk-devel`。可以使用下列命令搜索OpenJDK版本。

```shell
yum search openjdk
```

输出如下：

```
java-1.6.0-openjdk.x86_64 : OpenJDK Runtime Environment
java-1.6.0-openjdk-demo.x86_64 : OpenJDK Demos
java-1.6.0-openjdk-devel.x86_64 : OpenJDK Development Environment
java-1.6.0-openjdk-javadoc.x86_64 : OpenJDK API Documentation
java-1.6.0-openjdk-src.x86_64 : OpenJDK Source Bundle
java-1.7.0-openjdk.x86_64 : OpenJDK Runtime Environment
java-1.7.0-openjdk-demo.x86_64 : OpenJDK Demos
java-1.7.0-openjdk-devel.x86_64 : OpenJDK Development Environment
java-1.7.0-openjdk-javadoc.noarch : OpenJDK API Documentation
java-1.7.0-openjdk-src.x86_64 : OpenJDK Source Bundle
java-1.8.0-openjdk.x86_64 : OpenJDK Runtime Environment
java-1.8.0-openjdk-debug.x86_64 : OpenJDK Runtime Environment with full debug on
java-1.8.0-openjdk-demo.x86_64 : OpenJDK Demos
java-1.8.0-openjdk-demo-debug.x86_64 : OpenJDK Demos with full debug on
java-1.8.0-openjdk-devel.x86_64 : OpenJDK Development Environment
java-1.8.0-openjdk-devel-debug.x86_64 : OpenJDK Development Environment with full debug on
java-1.8.0-openjdk-headless.x86_64 : OpenJDK Runtime Environment
java-1.8.0-openjdk-headless-debug.x86_64 : OpenJDK Runtime Environment with full debug on
java-1.8.0-openjdk-javadoc.noarch : OpenJDK API Documentation
java-1.8.0-openjdk-javadoc-debug.noarch : OpenJDK API Documentation for packages with debug on
java-1.8.0-openjdk-src.x86_64 : OpenJDK Source Bundle
java-1.8.0-openjdk-src-debug.x86_64 : OpenJDK Source Bundle for packages with debug on
icedtea-web.x86_64 : Additional Java components for OpenJDK - Java browser plug-in and Web Start implementation
```

这里，我选择`java-1.8.0-openjdk-devel`。这是开发版，包含jps，jinfo等命令，`java-1.8.0-openjdk`则没有这些。

然后使用下列命令安装。

```shell
yum install java-1.8.0-openjdk-devel
```

安装完就可以直接使用。

## 安装Jenkins

在[官方文档](https://www.jenkins.io/doc/book/installing/)，可以看到有很多种安装方式。这里，我选择[WAR files](https://www.jenkins.io/doc/book/installing/war-file/)（因为需要安装Jenkins的机器上不了外网）。

1. [下载war包](https://www.jenkins.io/download/)。

2. 将war包上传到服务器。

   这里我直接使用rz。没有的这个命令的话需要安装`lrzsz`

   ```shell
   rz
   ```

3. 启动Jenkins

   ```
   nohup java -jar jenkins.war &
   ```

## 配置Jenkins

通过http://ip:8080访问Jenkins。按照配置向导完成配置。

{{% admonition warning %}}

由于安装Jenkins的机器不能访问外网，所以会有`Jenkins实例已离线`的提示。这里忽略进行下一步就行。

{{% /admonition %}}

### 插件安装

由于不能访问外网，所以只能离线安装了。

通过**Manage Jenkins**->**Manage Plugins**->**advanced**下的**Upload Plugin**上传`.hpi`即可安装

- 中文

  我们可以在[https://plugins.jenkins.io/](https://plugins.jenkins.io/)下载插件。

  搜索`Localization`

  ![这是一张图片](/image/Jenkins安装与使用/2.png)

  选择`Localization: Chinese (Simplified)`，然后下载`.hpi`文件。

  ![这是一张图片](/image/Jenkins安装与使用/4.png)

  用下载的文件一上传就发现报错了。因为这个插件需要依赖[Localization Support ≥ 1.1](https://plugins.jenkins.io/localization-support/)，所以需要先将依赖的插件安装完。

  安装完，效果如下

  ![这是一张图片](/image/Jenkins安装与使用/1.png)

- Maven Integration

  这个插件离线安装的话，额，俄罗斯套娃似的依赖。所以我放弃了，选择申请机器访问外网。

  