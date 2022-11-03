---
title: "Wercker部署Hugo博客拉取Docker镜像出错问题解决"
date: 2020-12-04T17:27:46+08:00
lastmod: 2020-12-04T17:27:46+08:00
categories: ["Hugo", "Blog", "Docker"]
tags: ["Hugo", "Blog", "Docker", "Wercker", "Docker Hub"]
keywords: ["pull image", "https://www.docker.com/increase-rate-limit"]
toc: false
draft: false
---

话不多说，直接上错误信息。

> fetch failed to pull image golang: API error (500): {"message":"toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit"}

<!--more-->

小盆友，你是否有很多问号。

没关系[传送门](https://keyes.ie/wercker-docker-pull-rate-limit/)来解答。

如果你通过上面的链接已经解决，那很好。下面不用看了。

---

如果你是通过[Hexo迁移到Hugo]({{<ref "Hexo迁移到Hugo">}})建的博客，想直接复制粘贴，那么，我这直接给出`wercker.yml`

```yaml
box:
  id: golang:latest
  username: $DOCKERHUB_USERNAME
  password: $DOCKERHUB_PASSWORD

build:
  steps:
    - script:
        name: initialize and update git submodules
        code: |
            git submodule init
            git submodule update --remote --recursive
    - arjen/hugo-build@2.14.0:
        version: "0.79.0"
        theme: even
        flags: --environment=production

deploy:
  steps:
    - install-packages:
            packages: git ssh-client
    - lukevivier/gh-pages@0.2.1:
        token: $GIT_TOKEN
        basedir: public
```

另外，还需要配置环境变量，如图。

![图片](/image/Wercker部署Hugo博客拉取Docker镜像出错问题解决/1.png)

其中`DOCKERHUB_PASSWORD`可以是Docker Hub**密码**，也可以是**TOKEN**（从[这里获取](https://hub.docker.com/settings/security)）。



_如果，你使用自己的Docker Hub账号还是会有这种问题，那么就使用[钞能力](https://www.docker.com/pricing)吧。_



{{% admonition warning "注意" %}}

2022.11.03: 此时Wercker已经停服了，所以此文所述问题已经不复存在

{{% /admonition %}}
