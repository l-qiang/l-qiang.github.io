---
title: Hexo添加RSS和站点地图
date: 2020-01-20 13:08:57
categories: ["Blog"]
tags: ["Blog", "博客", "Hexo", "RSS", "sitemap"]
---
<!--more-->

## RSS

给自己的博客添加RSS的话，其他人就可以订阅你的博客了。

`安装`

```shell
npm install hexo-generator-feed --save
```

`配置`

```y
feed:
  type: atom
  path: atom.xml
  limit: 20
  hub:
  content:
  content_limit: 
  content_limit_delim: ' '
  order_by: -date
  icon: 
  autodiscovery: true
  template:
```

详细见[hexo-generator-feed](https://github.com/hexojs/hexo-generator-feed)

## 站点地图[^1]

> 站点地图是一个网站所有链接的容器。很多网站的连接层次比较深，爬虫很难抓取到，站点地图可以方便爬虫抓取网站页面，通过抓取网站页面，清晰了解网站的架构，网站地图一般存放在根目录下并命名sitemap，为爬虫指路，增加网站重要内容页面的收录。站点地图就是根据网站的结构、框架、内容，生成的导航网页文件。站点地图对于提高用户体验有好处，它们为网站访问者指明方向，并帮助迷失的访问者找到他们想看的页面。

`安装`

```shell
npm install hexo-generator-sitemap --save
```

`配置`

```yaml
sitemap:
    path: sitemap.xml
    template:
    rel: false
```

- **path** - Sitemap path. (Default: sitemap.xml)
- **template** - Custom template path. This file will be used to generate sitemap.xml (See [default template](https://github.com/hexojs/hexo-generator-sitemap/blob/master/sitemap.xml))
- **rel** - Add [`rel-sitemap`](http://microformats.org/wiki/rel-sitemap) to the site's header. (Default: `false`)

[^1]:[百度百科](https://baike.baidu.com/item/站点地图/9991876) 