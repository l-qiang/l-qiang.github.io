---
title: "Jenkins安装与使用"
date: 2020-12-02T13:49:36+08:00
categories: ["Jenkins"]
tags: ["Jenkins", "Linux", "OpenJDK"]
keywords: ["Jenkins", "Linux", "OpenJDK"]
draft: false
---

<!--more-->

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

   这里我直接使用rz。（没有的这个命令的话需要安装`lrzsz`）

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

- 中文

  由于不能访问外网，这里选择了离线安装。

  通过**Manage Jenkins**->**Manage Plugins**->**advanced**下的**Upload Plugin**上传`.hpi`即可安装

  我们可以在[https://plugins.jenkins.io/](https://plugins.jenkins.io/)下载插件。

  搜索`Localization`

  ![这是一张图片](/image/Jenkins安装与使用/2.png)

  选择`Localization: Chinese (Simplified)`，然后下载`.hpi`文件。

  ![这是一张图片](/image/Jenkins安装与使用/4.png)

  用下载的文件一上传就发现报错了。因为这个插件需要依赖[Localization Support ≥ 1.1](https://plugins.jenkins.io/localization-support/)，所以需要先将依赖的插件安装完。

- Maven Integration

  俄罗斯套娃似的依赖，所以，请务必在线安装。
  
  在线安装实在是好用太多，在已安装插件里能将插件降级，比如上面的本地化插件就是因为版本太高导致效果不太好。
  
  ![这是一张图片](/image/Jenkins安装与使用/1.png)
  
  可以看到，会自动安装依赖的插件。
  
  ![这是一张图片](/image/Jenkins安装与使用/3.png)
  
  虽然，这里显示一些插件安装失败了，但是[重启Jenkins](http://localhost:8080/restart)之后，`Maven Integration`已经安装好了
  
- Publish Over SSH

  添加这个插件可以在系统配置里添加发布的服务器。

- Subversion

  添加完这个插件可以，新建的任务里就有`Subversion`选项了。

{{% admonition info%}}

`降低版本`有时候是必须的，比如这里我就因为其他插件版本太高导致无法成功安装`Maven Integration`

{{% /admonition %}}

## 安装Maven

我们这里选择直接使用Jenkins的`全局工具配置`来安装Maven，选择**自动安装**，然后在第一次使用Maven的时候就会把Maven安装上。

安装目录在`/home/用户目录/.jenkins/tools/hudson.tasks.Maven_MavenInstallation/maven-3.6.3`

## 部署

流程：

1. 使用`Subversion`将SVN上的Maven项目Checkout下来。Checkout的代码在`/home/用户目录/.jenkins/workspace`。
2. 使用`Maven`打包项目
3. 使用`Publish Over SSH`，将第2步中生成的jar包上传到目标服务器后，使用shell脚本启动应用程序。


