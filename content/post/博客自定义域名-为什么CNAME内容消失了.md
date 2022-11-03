---
title: "博客自定义域名-为什么CNAME内容消失了"
date: 2021-04-14T15:53:01+08:00
lastmod: 2021-04-14T15:53:01+08:00
categories: ["Blog", "Hugo"]
tags: ["Blog", "Hugo"]
keywords: ["Wercker", "CNAME", "Hugo", "Domain"]
draft: false
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

所以，我在static下创建了`CNAME`文件且在里面写上了`blog.l-qiang.xyz`。然后问题就出现了。

## 定位问题

`CNAME`文件提交到Github上后，在`main`分支的`CNAME`文件中是有内容的，所以问题就出在`Wercker`部署这一步。那么我就重点看`wercker.yml`。

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
        version: "0.82.0"
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

首先，我们在本地使用hugo进行`build`肯定是没有问题的。所以问题出在`deploy`这一部分，这部分就两个步骤。

1. install-packages

   这是安装软件包。

2. lukevivier/gh-pages@0.2.1

   这是wercker的Step store里面的步骤。所以出问题的原因，极有可能是这一步。

将`lukevivier/gh-pages@0.2.1`的源码下载下来。

源码很简单，总共包含4个文件：

- README.md
- run.sh
- step.yml
- wercker-step.yml

从Wercker的文档[Creating Steps](https://devcenter.wercker.com/development/steps/creating-steps/)中可以了解到，我重点需要关注的就是`run.sh`和`step.yml`

`run.sh`内容如下：

```bash
#!/bin/sh

# confirm environment variables
if [ ! -n "$WERCKER_GH_PAGES_TOKEN" ]
then
  fail "missing option \"token\", aborting"
fi

# use repo option or guess from git info
if [ -n "$WERCKER_GH_PAGES_REPO" ]
then
  repo="$WERCKER_GH_PAGES_REPO"
elif [ 'github.com' == "$WERCKER_GIT_DOMAIN" ]
then
  repo="$WERCKER_GIT_OWNER/$WERCKER_GIT_REPOSITORY"
else
  fail "missing option \"repo\", aborting"
fi

info "using github repo \"$repo\""

# remote path
remote="https://$WERCKER_GH_PAGES_TOKEN@github.com/$repo.git"

# if directory provided, cd to it
if [ -d "$WERCKER_GH_PAGES_BASEDIR" ]
then
  cd $WERCKER_GH_PAGES_BASEDIR
fi

# remove existing commit history
rm -rf .git

# generate cname file
if [ -n $WERCKER_GH_PAGES_DOMAIN ]
then
  echo $WERCKER_GH_PAGES_DOMAIN > CNAME
fi


# setup branch
branch="gh-pages"
if [[ "$WERCKER_GH_PAGES_REPO" =~ $WERCKER_GIT_OWNER\.github\.(io|com)$ ]]; then
	branch="master"
fi


# init repository
git init

git config user.email "pleasemailus@wercker.com"
git config user.name "werckerbot"

git add .
git commit -m "deploy from $WERCKER_STARTED_BY"
result="$(git push -f $remote master:$branch)"

if [[ $? -ne 0 ]]
then
  warning "$result"
  fail "failed pushing to github pages"
else
  success "pushed to github pages"
fi
```

从上面的脚本中可以看到，这一段：

```bash
# generate cname file
if [ -n $WERCKER_GH_PAGES_DOMAIN ]
then
  echo $WERCKER_GH_PAGES_DOMAIN > CNAME
fi
```

如果`$WERCKER_GH_PAGES_DOMAIN`的字符串长度不为0，那么将会把`$WERCKER_GH_PAGES_DOMAIN`写到`CNAME`。

那么，`$WERCKER_GH_PAGES_DOMAIN`这个变量哪来的呢？先看`step.yml`。

`step.yml`内容如下：

```yaml
name: gh-pages
version: 0.2.1
summary: deploy to github pages
category: imported
tags:
- github
- pages
- deploy
properties: []
```

从官方文档对properties的解释了解到。
> The `properties` field contains metadata describing the parameters that are available for the step. This is a map; the key is the name of the step, and the value is a object with the following properties:
>
> - `name` - the name of the property
> - `type` - the type of the data of the parameter. Currently supported: `string`.
> - `required` - boolean indicating if the parameter is required or not (currently not enforced).
> - `default` - value that gets used, if no parameter was provided through the `wercker.yml`

上面的`properties`为空，所以参数都是从`wercker.yml`提供的。

```yaml
- lukevivier/gh-pages@0.2.1:
        token: $GIT_TOKEN
        basedir: public
```

我这里提供了两个参数，根据官方文档会生成两个环境变量，`WERCKER_GH_PAGES_BASEDIR`和`WERCKER_GH_PAGES_TOKEN`。这对应了上面`run.sh`中的`WERCKER_GH_PAGES_`开头的变量。但是上面是没有配置`domain`。所以`WERCKER_GH_PAGES_DOMAIN`变量是不存在的。

那么，`if [ -n $WERCKER_GH_PAGES_DOMAIN ]`应该不成立，不生成`CNAME`才对呀。

咋一看，确实。

但是，如果`$WERCKER_GH_PAGES_DOMAIN`不存在。那么`if [ -n $WERCKER_GH_PAGES_DOMAIN ]`会变成`if [ -n ]`，这会造成这个判断始终为**true**;

所以，如果在`wercker.yml`中的`lukevivier/gh-pages@0.2.1`没有配置`domain`，那么`CNAME`始终是空的，且覆盖了build之后的我们在static下的`CNAME`文件。

## 解决办法

其中一种解决办法其实从上面已经知道了。

1. `lukevivier/gh-pages@0.2.1`添加`domain`为自定义域名。

2. `static`下添加`CNAME`，但是不要使用`lukevivier/gh-pages@0.2.1`。

   可以换一个steps，`Step store`有很多gh-pages步骤。

3. 修改`lukevivier/gh-pages@0.2.1`的代码，生成自己的步骤。

   只要修改`if [ -n $WERCKER_GH_PAGES_DOMAIN ]`为`if [ -n "$WERCKER_GH_PAGES_DOMAIN" ]`即可。这就是告诉我们为什么要在变量外加双引号的原因。

{{% admonition warning "注意" %}}

2022.11.03: 此时Wercker已经停服了，所以此文所述问题已经不复存在

{{% /admonition %}}
