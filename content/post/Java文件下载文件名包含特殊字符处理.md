---
title: Java文件下载文件名包含特殊字符处理
date: 2019-04-02 19:52:15
categories: ["Java"]
tags: ["Java", "文件下载"]
toc: false
---

```java
response.setCharacterEncoding("UTF-8");
            response.setContentType("multipart/form-data");
            response.setHeader("Content-Disposition",
                    "attachment;fileName*=UTF-8''" + UriUtils.encode(fileName, "UTF-8"));
```

<!--more-->

重点是`fileName*=UTF-8''` 和`UriUtils.encode`，UriUtils使用的是spring包的**org.springframework.web.util.UriUtils**

