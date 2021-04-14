---
title: "博客自定义域名-为什么CNAME内容消失了"
date: 2021-04-14T15:53:01+08:00
lastmod: 2021-04-14T15:53:01+08:00
categories: ["Blog", "Hugo"]
tags: ["Blog", "Hugo"]
keywords: ["Wercker", "CNAME", "Hugo", "Domain"]
draft: true
---

现象是这样的：在static目录下新建了一个`CNAME`文件，并填写了域名`blog.l-qiang.xyz`。但是，使用Git将更改push到Github之后，发现gh-pages分支下的`CNAME`的内容为**空**，`blog.l-qiang.xyz`竟然消失了！！！

<!--more-->

关于博客相关搭建的教程，见：

1. [Hexo搭建个人博客]({{<ref "Hexo搭建个人博客">}})
2. [Hexo添加RSS和站点地图]({{<ref "Hexo添加RSS和站点地图">}})
3. [Hexo迁移到Hugo]({{<ref "Hexo迁移到Hugo">}})
4. [Wercker部署Hugo博客拉取Docker镜像出错问题解决]({{<ref "Wercker部署Hugo博客拉取Docker镜像出错问题解决">}})



## 起因

之前博客一直使用的[l-qiang.github.io](https://l-qiang.github.io)，前段时间在[NameSilo](https://www.namesilo.com/)整了个域名`l-qiang.xyz`，之前没整是因为阿里云太贵了。

然后，Github Pages设置域名的时候直接在界面上设置的。(ps：设置完其实就是在gh-pages分支的根目录添加了一个CNAME文件，内容就是你设置的域名)

今天早上访问博客的时候发现404了，然后上Github上一看，域名设置被清空了。

出现这个现象的原因是：`gh-pages`分支的内容都是通过`Wercker`生成的，所以，每次更新博客，CNAME就被干掉了。

但是，我不能每次更新博客都要重新设置域名吧，所以，我需要`CNAME`文件也被Github托管起来。在Hugo的文档中，可以看到相关说明。

> If you’d like to use a custom domain for your GitHub Pages site, create a file `static/CNAME`. Your custom domain name should be the only contents inside `CNAME`. Since it’s inside `static`, the published site will contain the CNAME file at the root of the published site, which is a requirement of GitHub Pages.

所以