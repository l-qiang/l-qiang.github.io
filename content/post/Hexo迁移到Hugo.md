---
title: "Hexo迁移到Hugo"
date: 2020-11-23T16:35:28+08:00
lastmod: 2020-12-04T16:04:28+08:00
categories: ["Blog", "Hexo", "Hugo"]
tags: ["Blog", "Hexo", "Hugo", "博客迁移", "wercker"]
draft: false
---

之前一直一直使用Hexo，后来发现Hugo比Hexo看起来就好用，于是就从Hexo迁到Hugo。

Hexo建博客及本文的一些操作可参考[Hexo搭建个人博客]({{<ref "Hexo搭建个人博客">}})。

<!--more-->

## 准备

{{% admonition info %}}

我的github账号为`l-qiang`，直接代替`<你的GitHub用户名>`。

例如：`<你的GitHub用户名>.github.io`以`l-qiang.github.io`代替

{{% /admonition %}}

1. 新建一个目录将Github上的[l-qiang.github.io](https://github.com/l-qiang/l-qiang.github.io)clone下来。

   ```shell
   git clone git@github.com:l-qiang/l-qiang.github.io.git
   ```

2. 新建空的`main`分支（用于存放hugo new site出来的内容）。

   ```shell
   cd l-qiang.github.io
   git checkout --orphan main
   git reset --hard
   git commit --allow-empty -m "Initializing main branch"
   git push --upstream main
   ```

3. 将`main`分支设置为Github(`l-qiang.github.io`)仓库默认分支。

   `https://github.com/l-qiang/l-qiang.github.io/settings/branches`

4. 用**第2步**中同样的方式新建`gh-pages`分支（用于存放hugo build后public文件夹的内容）。

{{% admonition question "新建相关" %}}

如果你之前没有建过`l-qiang.github.io`仓库，那就新建一个，忽略上述**2-3步**。

{{% /admonition  %}}

{{% admonition info "迁移相关"%}}

我还建了一个`blog-pages`分支，放之前`master`分支的内容，并把`master`分支删了，同时把原先`blog`分支的

.travis.yml改成

```yaml
sudo: false
language: node_js
node_js:
  - 10 # use nodejs v10 LTS
cache: npm
branches:
  only:
    - blog # build blog branch only
script:
  - hexo generate # generate static files
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  target_branch: blog-pages # master 改为了 blog-pages
  on:
    branch: blog
  local-dir: public
```

把github pages设置为`blog-pages`分支。这样让我们的博客在迁移之前，还以Hexo生成的页面展示

{{% /admonition %}}

## Hugo建站

1. [下载Hugo](https://github.com/gohugoio/hugo/releases)

   建议配置下环境变量，这样你就可以在啥目录下都可以执行`hugo`

2. 随便找个目录执行下列命令。建议不要和上面建的目录搞混了。

   ```shell
   hugo new site quickstart
   ```

   可以就是直接用`quickstart`。
   
3. 将`quickstart`目录的内容直接复制到`l-qiang.github.io`目录（git clone下来的目录）

4. 添加主题，我用的even主题

   ```shell
   git submodule add https://github.com/olOwOlo/hugo-theme-even.git themes/even
   ```

5. 将代码push到Github`l-qiang.github.io`仓库的main分支。

{{% admonition info %}}

之前用Hexo的时候，是把主题的代码也提交到了Github，这里通过submodule的方式可以让Github的仓库的主题直接链接到`https://github.com/olOwOlo/hugo-theme-even.git`。

push后的结果可以参考我的[l-qiang.github.io](https://github.com/l-qiang/l-qiang.github.io)的`main`分支

{{% /admonition %}}



{{% admonition warning "注意" %}}

2022.11.03: 此时wercker已经停服，后面的步骤可省略。构建部署参考：

[Hugo之Wercker停服解决方案-Github Actions]({{<ref "Hugo之Wercker停服解决方案-Github Actions">}})

{{% /admonition %}}

## wercker

用wercker的好处就是，我们写完东西只要将更改push到`main`分支就行，wercker可以帮我们把`main`分支的build之后推到`gh-pages`分支。

wercker难点就是hugo的官方文档都是八百年前的教程了。

1. wercker.yml

   ```yaml
   box: golang:latest
   
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

   这个可以直接拿去用。除了`theme`修改为自己的主题其他的都不用动。这里的一些内容我稍后根据自己的理解解释。

2. 将wercker.yml推送到`main`分支后。

3. 在wercker根据步骤创建application。

4. 设置`environment`

   ![这是一张图片](/image/Hexo迁移到Hugo/1.png)

   `GIT_TOKEN`由[这里-点我点我](https://github.com/settings/tokens)生成。

5. 新建的application已经由`build`了，需要新增一个pipeline`deploy`

   ![这是一张图片](/image/Hexo迁移到Hugo/2.png)

6. 然后在`workflow`的`build`后面添加`deploy`

   ![这是一张图片](/image/Hexo迁移到Hugo/3.png)

{{% admonition info "wercker.yml" %}}

```yaml
- arjen/hugo-build@2.14.0:
        version: "0.79.0"
        theme: even
        flags: --environment=production
```

```yaml
- lukevivier/gh-pages@0.2.1:
        token: $GIT_TOKEN
        basedir: public
```

这俩都是[Steps Store](https://app.wercker.com/steps)的。搜`hugo-build`和`gh-pages`可以找到，也有使用文档。`@2.14.0`指版本号。

旧版本可能不好使，可以在[Steps Store](https://app.wercker.com/steps)找新的。

```yaml
- script:
        name: initialize and update git submodules
        code: |
            git submodule init
            git submodule update --remote --recursive
```

这部分就是处理我们的主题的，因为我们的主题没有在自己的仓库中，而是链接到`https://github.com/olOwOlo/hugo-theme-even.git`，若是主题代码直接提到自己仓库，那这步可以省略

{{% /admonition %}}

