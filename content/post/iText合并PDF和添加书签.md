---
title: iText合并PDF和添加书签
date: 2020-01-10 20:37:23
categories: ["Java"]
tags: ["Java", "PDF", "iText"]
toc: false
---

关于这个问题，其实在iText官网和Stack Overflow上面都有答案。之所以还要记录是想有更多人看到简单易懂的解决办法。因为我不想再有人直接CSDN搜一个，不管是否要新引入Jar包也不管代码是否复杂，然后告诉我参考那个写。

[官网例子](https://itextpdf.com/en/resources/examples/itext-5-legacy/merging-documents-bookmarks#39-mergewithoutlines.java)

<!--more-->

下面是一个示例

```java
public static void merge(List<String> srcFiles, String desFile) throws Exception {
        Document document = new Document();
        PdfCopy copy = new PdfCopy(document, new FileOutputStream(desFile));
        document.open();
        int page = 1;
        List<HashMap<String, Object>> outlines = new ArrayList<HashMap<String, Object>>();
        for (String scrFile : srcFiles) {
            PdfReader reader = new PdfReader(scrFile);
            copy.addDocument(reader);
            // add outline element
            HashMap<String, Object> outline = new HashMap<String, Object>();
            outline.put("Title", FilenameUtils.getBaseName(scrFile)); // 书签的名字
            outline.put("Action", "GoTo");
            outline.put("Page", String.format("%d Fit", page));
            outlines.add(outline);
            // update page count
            page += reader.getNumberOfPages();
            reader.close();
        }
        copy.setOutlines(outlines);
        document.close();
}
```

