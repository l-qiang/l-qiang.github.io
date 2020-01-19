---
title: Hexo搭建个人博客
date: 2020-01-17 16:56:05
categories:
 - Blog
tags: 
 - Hexo
 - Github
 - Blog
---

这不完全是一篇搭建个人博客的教程，因为[Hexo](https://hexo.io/zh-cn/docs/)的文档其实写得非常清楚了。但是，我们按照教程来不一定很顺利。所以，这主要还是一个小白的经验之谈。

<!-- more -->


## Hexo

### 我为什么选择Hexo呢？

1. 网上很多人推荐使用Hexo
2. 我本来也打算用Jekyll，但是由于网络原因Ruby的开发环境的安装包我下载不下来。

什么？还有其他的比Hexo好用，为什么不用？别问，问就是Hexo是最好用的。

### 安装Hexo

#### 准备

- [Git](https://git-scm.com/) 
- [Node.js](https://nodejs.org/en/) (Node.js 版本需不低于 8.10，建议使用 Node.js 10.0 及以上版本)

这两我都是用得很少，一般都是现学现用。安装也不难，这就不多说了。

##### Git配置

由于最终我们的是将网站部署到{% label @GitHub Pages %}，所以这里需要配置Git，将Git和GitHub关联上，以便之后将文件push到GitHub。

1. 鼠标右击，选择{% label @Git Bash Here %}打开Git命令行。

2. 配置{% label @user.name %}和{% label @user.email %}

   ```
   git config --global user.name '你的GitHub用户名'
   git config --global user.email '你的GitHub注册邮箱'
   ```

3. 生成SSH密钥

   ```
   ssh-keygen -t rsa -C '你的GitHub注册邮箱'
   ```

   然后直接连续回车就OK，默认不需要设置密码

4. 复制公钥

   打开.ssh文件夹中的id_rsa.pub文件(用文本编辑器打开就好了)，复制里面的内容。（我的在这个目录下：C:\Users\Admin\\.ssh）

5. 到[GitHub设置SSH keys](https://github.com/settings/ssh/new)

   Title的内容随意，Key里面粘贴{% label @第4步 %}中复制的内容，然后点击{% button #, Add SSH Key %}

6. 检查

   打开Git Bash，输入

   ```
   $ ssh git@github.com
   ```

   显示{% label @You've successfully authenticated %}这样的话，就是成功了。

#### 安装

```
$ npm install -g hexo-cli
```

打开一个新的命令行窗口输入

```
$ Hexo
```

能显示出一些帮助信息就说明装成功了

### 建站

#### 初始化网站

找一个你想放网站文件的地方，然后执行下列命令

```
$ hexo init <folder>
$ cd <folder>
$ npm install
```

<a name='folder'>\<folder\></a>换成你想要的文件夹名字。完成后，能看到指定文件夹的目录如下：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

到目前为止，网站就建好了。但是我们要在本地看效果还需要将网站部署到本地服务器。

#### 服务器

1. 安装hexo-server

   ```
   $ npm install hexo-server --save
   ```

2. 启动

   ```
   $ hexo server
   ```

   {% note warning %}

   这里一定要切换到网站文件夹下启动。

   {% endnote %}

   最终看到输出{% label @Hexo is running at http://localhost:4000 %}就是启动好了。

3. 打开浏览器输入{% label @Hexo is running at http://localhost:4000 %}就能看到Hexo默认主题{% label @landscape %}的页面

   {% asset_img hexo_landscape.png landscape主题图片 %}

   

## GitHub Pages

我们现在已经在本地部署过搭建的网站了。为了让其他人也能看到，我们要个人网站部署到GitHub Pages。

这个教程在[Hexo](https://hexo.io/zh-cn/docs/github-pages)也是有的，有{% label @两种 %}方式。我比较推荐第一种，就是使用[Travis CI](https://travis-ci.com/) 。

{% note info %}

**为什么选第一种?**

主要还是我按第二种，GitHub上只存了Hexo生成的页面，但是我想源文件也想存下来，当然我们也可以将源文件也上传到GitHub，达到跟第一种类似的效果。但是，持续集成服务，免费的，它不香吗？

{% endnote %}



如果你按官方教程后最终发现这个：

> User pages must be built from the `master` branch.

那么下面的内容就有用了。因为[Hexo](https://hexo.io/zh-cn/docs/github-pages)的教程是将Hexo的文件放{% label @master %}，然后生成的html放{% label @gh-pages %}的，以前可能是没问题，但是现在GitHub规定你必须将页面放到{% label @master %}。



1. GitHub新建一个 repository。命名为{% label @<你的 GitHub 用户名>.github.io %}。选择{% label @Public %}，勾上{% label @Initialize this repository with a README %}。

2. 然后选择一个地方新建一个空的文件夹。进入文件夹，右键鼠标，选择{% label @Git Bash Here %}打开Git命令行。将刚才建的repository clone下来：

   ```
   $ git clone git@github.com:<你的GitHub用户名>/<你的GitHub用户名>.github.io.git
   ```

   然后新建一个分支用来存放Hexo生成前的文件。（我的分支名就blog）

   ```
   $ git branch <分支名字>
   ```

   然后切换到分支

   ```
   $ git checkout <分支名字>
   ```

3. 将之前建站的文件夹<a href='#folder'>\<folder\></a>下的所有文件及文件夹复制到第2步种新建的文件夹下面。

4. 将第3步种的文件提交到分支，填写注释

   ```
   $ git add .
   $ git commit -m '<注释>'
   ```

5. 将文件push到GitHub

   ```
   $ git remote add origin git@github.com:<你的GitHub用户名>/<你的GitHub用户名>.github.io.git
   $ git push -u origin master
   ```

   上传成功后就能在GitHub上之前建的repository的分支看到了。

6. 将 [Travis CI](https://github.com/marketplace/travis-ci) 添加到你的 GitHub 账户中。

7. 前往 GitHub 的 [Applications settings](https://github.com/settings/installations)，配置 Travis CI 权限，使其能够访问你的 repository。

8. 你应该会被重定向到 Travis CI 的页面。如果没有，请 [手动前往](https://travis-ci.com/)。

9. 在浏览器新建一个标签页，前往 GitHub [新建 Personal Access Token](https://github.com/settings/tokens)，只勾选 `repo` 的权限并生成一个新的 Token。Token 生成后请复制并保存好。

10. 回到 Travis CI，前往你的 repository 的设置页面，在 **Environment Variables** 下新建一个环境变量，**Name** 为 `GH_TOKEN`，**Value** 为刚才你在 GitHub 生成的 Token。确保 **DISPLAY VALUE IN BUILD LOG** 保持 **不被勾选** 避免你的 Token 泄漏。点击 **Add** 保存。

11. 在你的 Hexo 站点文件夹(现在这个站点文件夹为应该为{% label @<你的 GitHub 用户名>.github.io %})中新建一个 `.travis.yml` 文件：

    ```
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
      target_branch: master
      on:
        branch: blog
      local-dir: public
    ```

    {% note warning %}

    这里的内容和Hexo官网的有点区别。这里指定了build的为<分支名字>（{% label @我的是blog %}），目标是master

    {% endnote %}

12. 将 `.travis.yml` 推送（推送方法和之前的文件推送到GitHub类型）到 repository 分支中。Travis CI 应该会自动开始运行，并将生成的文件推送到同一 repository 下的 `master` 分支下。

13. 推送之后等一会,[Travis CI](https://travis-ci.com/) 上就能看到build信息。等[Travis CI](https://travis-ci.com/) build完之后。就能在 `https://<你的 GitHub 用户名>.github.io` 查看你的站点了。可能需要等一会儿。发布成功的话，你应该可以在repository的setting的{% label @GitHub Pages %} 下看到{% label success@Your site is published at https://<你的 GitHub 用户名>.github.io %}。

## Hexo主题

到此，我们已经将网站发布到GitHub Pages了。但是默认的`landscape`，真的不符合我的审美。我们可以取[Hexo主题](https://hexo.io/themes/)网站挑选，也可以去GitHub上搜索`hexo-theme`。我就是去GitHub上挑选的star比较高的[Next主题](hexo-theme)，所以我就以`Next`主题为例。

### 安装

{% tabs Hexo 主题 %}

<!-- tab 安装-> -->

切换到站点文件夹，按我上的的教程此时应该是

```
$ cd <你的 GitHub 用户名>.github.io
$ git clone https://github.com/theme-next/hexo-theme-next themes/next
```

<!-- endtab -->

<!-- tab Hexo 配置 -->

{% codeblock _config.yml %}
theme: next
{% endcodeblock %}

<!-- endtab -->

{% endtabs %}

### 配置

一开始Next主题的很多配置都是关闭的，所以我们可以通过查看[文档](https://theme-next.org/docs/),然后修改主题的`_config.yml`来开启和配置一些功能。下面只讲我遇到过问题的。

#### Post Wordcount

这是一个统计文字的功能，但是我安装步骤弄好之后。页面显示的字数和阅读时间都没正确显示出来。我看GitHub上关于没有正确显示字数的issue都是因为Next升级到`6.x`之后，将统计的插件从[`hexo-wordcount`](https://github.com/willin/hexo-wordcount)换到了[`hexo-symbols-count-time`](https://github.com/theme-next/hexo-symbols-count-time)。而我的都是最新的，最终我在Hexo上找到了解决办法。

> 在某些情况（尤其是更换主题后），如果发现您对站点的更改无论如何也不生效，您可能需要运行该命令。

这个命令就是

```
$ hexo clean
```

我运行完这个之后就正确显示了。

#### Comment Systems

评论系统我选的Gitalk。嗯，我比较喜欢GitHub那一套。

具体过程，按照过程来就行。

需要注意的就是这个配置

{% codeblock next/_config.yml %}
gitalk:  

​	repo: # Repository name to store issues
{% endcodeblock %}

这里填的是 Repository的名字，不是SSH URL。这个填错了页面上会显示`Error`。

另外，评论系统你得发布到GitHub才能正常使用。



{% note info %}

**推送主题修改到GitHub**

因为我们是将Next主题直接clone到我们自己的Repository下。所以，我们推送的文件中，有个.git的文件夹。嵌套仓库可能会出问题。注意命令行的提示信息，会有类似让你执行下面这条命令的提示

```
$ git rm --cached themes/next
```

这条真的有用，执行完就可以提交了。如果不行，你就多试几次，亲测有效。

{% endnote %}



### 使用

#### 标签和分类

这个我们在Next的[文档](https://theme-next.org/docs/theme-settings/custom-pages)中就能弄好了。

#### 发布文章

在站点文件夹下执行如下命令

```
$ hexo new post <title>
```

这条指令其实就是在source/_posts文件夹下建了名为\<title\>.md的文件。

用一个markdown的编辑器打开这个文件就可以开始写文章了。

文章的头部一段内容可以包含标签，分类。本篇文章就是这就这样的

```
---
title: Hexo搭建个人博客
date: 2020-01-17 16:56:05
categories:
 - Blog
tags: 
 - Hexo
 - Github
 - Blog
 ---
```



默认的情况下，文章是全部展开的，我们可以在文章的任何地方添加

```
<!-- more -->
```

控制哪些内容需要点击{% button #,  阅读全文>> %}查看。

还有一些标签的使用可以查看Next[文档](https://theme-next.org/docs/tag-plugins/)和Hexo[文档](https://hexo.io/zh-cn/docs/tag-plugins)。







