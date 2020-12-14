---
title: "Jenkins部署Maven项目"
date: 2020-12-10T13:22:22+08:00
lastmod: 2020-12-10T13:22:22+08:00
categories: ["Jenkins", "Maven", "Spring Boot"]
tags: ["Jenkins", "Maven", "Spring Boot"]
keywords: ["Jenkins", "Maven", "Spring Boot", "多模块", "Publish Over SSH", "Exec command", "502"]
toc: false
draft: false
---

这是关于我使用Jenkins部署Maven项目的经验之谈，包括了部署流程、Maven多模块的一些试验，以及使用`Publish Over SSH`的问题。

<!--more-->

简述我这里使用Jenkins部署的流程。

1. 从SVN将项目Checkout下来。
2. 使用Maven打包项目，生产jar包
3. 将部署的shell脚本传输到目标服务器。
4. 将**第2步**中的jar包传输到目标服务器并使用**第3步**中的脚本启动。

---

首先，新建一个任务。

![图片](/image/Jenkins部署Maven项目/1.png)

{{% admonition warning %}}

这里，我选择**自由风格的软件项目**。虽然我的项目是多模块的，但是，我的这些项目并没有直接的关联。`父pom.xml`只是帮助一些依赖包的管理，它们都是独立部署的，有各自的shell脚本。但是这有一个前提条件，就是`父pom.xml`已经再本地或远程仓库。

Jenkins会使用**任务名称**，在`/home/<用户>/.jenkins/workspace`下生成文件夹，svn检出的代码会放在这个文件夹下。

{{% /admonition %}}



**源码管理**，填写源码的svn地址和用户凭证(需要**Subversion**插件)

![图片](/image/Jenkins部署Maven项目/2.png)

使用Maven**构建**

```
clean install -Dmaven.test.skip=true
```

![图片](/image/Jenkins部署Maven项目/3.png)

**构建后操作**，将项目根目录的shell脚本和jar包传到目标服务器(需要**Publish Over SSH**插件)

![图片](/image/Jenkins部署Maven项目/4.png)

![图片](/image/Jenkins部署Maven项目/5.png)

这里有两个`Transfer Set`，一个传shell脚本，并修改脚本的权限。另一个传jar包，并使用shell脚本启动jar包。

{{% admonition warning "注意" %}}

```shell
#! /bin/bash
chmod 777 /app/bootapp/shell/deploy-gateway.sh
```

这里，如果不加`#! /bin/bash`, 添加这个**Exec command**后会出现保存不了的问题。但是去掉`.sh`却可以保存。

而且，如果`#! /bin/bash`中没有**空格**也会保存不上。

{{% /admonition %}}

附一个shell示例。

```shell
DATE=$(date +%Y%m%d)
DIR=/app/bootapp
PROJECT_NAME=gateway
JARFILE=$PROJECT_NAME-0.0.1-SNAPSHOT.jar
# 创建备份文件夹
if [ ! -d $DIR/backup ];then
  mkdir -p $DIR/backup
fi
cd $DIR || exit
# 停服务
ps -ef | grep $JARFILE | grep -v grep | awk '{print $2}' | xargs kill -9
# 备份旧的jar包
mv $JARFILE backup/$JARFILE$DATE
# 新jar包拷贝到当前目录
mv -f $DIR/jenkins_output/$JARFILE .

if java -jar $JARFILE > logs/out_$PROJECT_NAME.log &
then
  sleep 30
  tail -n 50 logs/out_$PROJECT_NAME.log
fi
cd backup/ || exit
ls -lt|awk 'NR>5{print $NF}'|xargs rm -rf
```

{{% admonition warning "注意" %}}

shell脚本换行符得是`\n`，这个可以在编辑的时候使用IDEA，然后在右下角修改。

![图片](/image/Jenkins部署Maven项目/6.png)

{{% /admonition %}}

