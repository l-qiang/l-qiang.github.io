---
title: "Linux新增用户及Jenkins使用新用户发布应用"
date: 2020-12-01T16:21:09+08:00
categories: ["Linux", "Jenkins"]
tags: ["Linux", "Jenkins", "SSH"]
keywords: ["Linux", "Jenkins", "SSH", "权限"]
draft: false
---

<!--more-->

### 新增用户

```shell
useradd name
passwd name
```

### SSH权限

```shell
vim /etc/ssh/sshd_config
```

在文件后面添加`AllowUsers name`，然后重启SSH服务。

{{% admonition info %}}

这里我还配置了`PermitRootLogin no`来禁用了root登录，配置之后将不能使用root连SSH，可以使用新建的用户通过su切换到root

{{% /admonition %}}

```
service sshd restart
```

此时，就已经可以通过新建的账号通过SSH连接到服务器

### Jenkins

由于已经禁用了root用户，所以Jenkins这边需要改为新建的用户。

通过`系统管理->系统设置`下的**Publish over SSH**中的*SSH Servers*修改。

修改完之后，发现通过Jenkins发布应用还是会报错

> ERROR: Exception when publishing, exception message [Permission denied]

由于之前的一些Jenkins输出的文件和启动脚本都是由root创建。所以使用新的用户的时候，没有权限。

所以，我这里直接将需要Jenkins操作的文件的所有者改为新用户。

```shell
chown -R <新用户>:<新用户所属组> jenkins操作根目录
```



做完这些，仍然有问题，看日志发现是`kill`进程失败了。

由于之前的进程是root启动的，新用户没有权限kill，所以我们需要使用root用户先kill掉。

