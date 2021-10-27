---
title: "Git多账号配置"
date: 2021-10-24T22:07:41+08:00
lastmod: 2021-10-24T22:07:41+08:00
categories: ["Git"]
tags: ["Git"]
keywords: ["Git", "多账号", "SSH"]
draft: false
---

网上有太多关于Git多账号配置的文章了，给我的感觉就是很乱，没有找到一篇可以收藏的，所以还是决定自己写一篇记录一下。

暂时记录我的使用情况，后续有补充或修改再更新。

<!--more-->

先简要说一下多账户配置的重点步骤：

1. 每个账号生成一对SSH密钥。
2. 配置SSH密钥公钥。
3. 配置SSH密钥私钥。

如果上面的步骤都成功了，那么可以使用类似`ssh -T git@github.com`的命令测试SSH连接。

如果SSH连接成功，将显示类似以下内容：

> Hi l-qiang! You've successfully authenticated, but GitHub does not provide shell access.

### 生成SSH密钥

```shell
ssh-keygen -t rsa -C 'your_email@example.com'
```

{{% admonition warning "注意" %}}

​	![](/image/Git多账号配置/1.png)

这里需要输入保存密钥的路径名称，每个账号应该都不一样，否则会覆盖。

{{% /admonition %}}

### 配置公钥

1. 找到**生成的SSH密钥**的公钥。(例如：C:\Users\admin\.ssh\id_rsa_github.pub，这个位置就是上面保存密钥路径的位置。）
2. 复制**公钥**的内容。
3. 将复制好的公钥内容添加到Github或GitLab。

### 配置私钥

由于默认情况下，使用的是`~/.ssh/id_rsa`这个密钥，生成的密钥修改了路径所以需要配置私钥路径。（这点可以通过`ssh  -Tv git@github.com`确定。`v`选项可以输出debug信息。）

配置私钥有**两种**方式：`ssh-add`命令和`config`文件。**推荐使用`config`文件**。（能否只使用`ssh-add`待验证，按理需要`config`文件一样有密钥和主机的映射关系才对，但是目前我去除config文件是没问题的）。

#### ssh-add

```shell
ssh-add ~/.ssh/id_rsa_github
```

{{% admonition warning "注意" %}}

`~/.ssh/id_rsa_github`是私钥路径名称，这与前面生成的密钥是填写的路径有关。

另外，直接执行`ssh-add`，可能会出现一下错误信息：

> Could not open a connection to your authentication agent.

这个时候，需要执行`ssh-add`之前先执行以下命令：

```
ssh-agent bash
```

{{% /admonition %}}

使用这种方式很麻烦，每次都需要执行`ssh-agent`。所以，有方法自启动`ssh-agent`。

附上Github的教程：[在 Git for Windows 上自动启动 `ssh-agent`](https://docs.github.com/cn/authentication/connecting-to-github-with-ssh/working-with-ssh-key-passphrases)

#### config（推荐）

在`~/.ssh/`目录（也就是存放密钥的目录）下创建**config**文件，并将以下内容修改后写入**config**文件。

```
Host github.com
 HostName github.com
 User git
 IdentityFile ~/.ssh/id_rsa_github
Host gitlab.com
 HostName gitlab.com
 User git
 IdentityFile ~/.ssh/id_rsa_gitlab
```

{{% admonition warning "注意" %}}

**HostName** ：需要ssh连接过去的主机名。

**User**：登录主机的用户名。

**IdentityFile**：私钥路径。

**Host**对应与`ssh git@github.com`中@后面的内容。

否则将会报以下错误：

> git@github.com: Permission denied (publickey).

{{% /admonition %}}

### 其他

下面这些命令我不将其归于Git的多账号配置。

```shell
git config --global user.name 'username'
git config --global user.email 'email'

git config --local user.name 'username'
git config --local user.email 'email'
```

`git config --global`修改的是`~/.gitconfig`文件。(全局)

`git config --local`修改的是对应Repository下的`.git/config`文件，所以必须得有Repository才行。（局部）



