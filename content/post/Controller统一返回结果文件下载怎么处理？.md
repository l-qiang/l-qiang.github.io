---
title: "Controller统一返回结果文件下载怎么处理？"
date: 2021-01-14T21:46:33+08:00
lastmod: 2021-01-14T21:46:33+08:00
categories: ["Spring MVC", "Spring Boot"]
tags: ["文件下载", "ControllerAdvice"]
keywords: ["文件下载", "Controller", "ControllerAdvice", "统一返回结果"]
toc: false
draft: false
---

我们经常会使用一个**自定义**的Controller返回结果`Response`，例如：

```java
public class Response<T> {

    private static final int SUCCESS = 0;
    private static final int ERROR = 1;

    private String message;
    private int code;
    private T data;

    public Response(String message, int code, T data) {
        this.message = message;
        this.code = code;
        this.data = data;
    }

    public static Response<Void> error(String msg) {
        return new Response<>(msg, ERROR, null);
    }

    public static <T> Response<T> success(T data) {
        return new Response<>("success", SUCCESS, data);
    }

    @JsonProperty
    public T data() {
        return this.data;
    }

    @JsonProperty
    public int code() {
        return this.code;
    }

    @JsonProperty
    public String message() {
        return this.message;
    }

    @JsonIgnore
    public boolean isFileData() {
        return data() instanceof FileData;
    }
}
```

但是，关于文件下载，我们一般就是在Controller修改**HttpServletResponse**或者使用**ResponseEntity**。如果，我不想在Controller中写这些，我想跟其他方法一样直接返回`Response`，那该怎么做呢？

<!--more-->

要做到这一点，我们就得在Controller返回之后，HTTP响应之前做一些操作。然后我就发现了`ResponseBodyAdvice`接口。

```java
@ControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)
public class ControllerResponseAdvice implements ResponseBodyAdvice<Response<?>> {

    @ResponseBody
    @ExceptionHandler(Exception.class)
    public Response<Void> handler(Exception ex) {
        return Response.error(ex.getMessage());
    }

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Response<?> beforeBodyWrite(Response<?> body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        if (body.isFileData()) {
            FileData data = (FileData) body.data();
            ServletServerHttpResponse res = ((ServletServerHttpResponse)response);
            res.getHeaders().setContentType(MediaType.MULTIPART_FORM_DATA);
            res.getHeaders().set("Content-Disposition", "attachment;fileName*=UTF-8''" + data.filename());

            try(OutputStream outputStream = res.getBody()) {
                outputStream.write(data.filedata());
            } catch (Exception e) {
                return Response.error("下载失败");
            }
        }
        return body;
    }
}
```

```java
public class FileData {
    private final String filename;
    private final byte[] filedata;

    public FileData(String filename, byte[] filedata) {
        this.filename = filename;
        this.filedata = filedata;
    }

    public String filename() {
        return this.filename;
    }

    public byte[] filedata() {
        return this.filedata;
    }
}
```

这里，我们在`beforeBodyWrite`使用`FileData`来判断是否是**下载文件**。

{{% admonition warning %}}

**统一异常处理**，会在`beforeBodyWrite`方法之前执行。

{{% /admonition %}}

然后在Controller这样使用就行了。

```java
@RestController
@RequestMapping("/demo")
public class DemoController {

    @RequestMapping("/download")
    public Response<FileData> download() throws Exception {
 		return Response.success(fileData)
    }
}
```

{{% admonition warning %}}

这里注意要使用`@RestController`，或者使用`@ResponseBody`。

{{% /admonition %}}

一起都源于我的突发奇想。