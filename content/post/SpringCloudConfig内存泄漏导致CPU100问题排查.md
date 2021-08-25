---
title: "Spring Cloud Config内存泄漏导致CPU100%问题排查"
date: 2021-06-09T12:37:05+08:00
lastmod: 2021-06-09T12:37:05+08:00
categories: ["Spring Cloud","JVM"]
tags: ["Spring Cloud Config","JVM"]
keywords: ["Spring Cloud Config" ,"JVM", "MAT", "内存泄漏", "CPU100%", "DeleteOnExitHook"]
---

前段时间，作为工具人的我被要求排查某台服务器`CPU 100%`。当时听了内心狂喜，终于让我遇到CPU 100%问题了。但是，当时服务器着急重启，没有足够的时间给我排查问题，但是我还是在重启之前把堆Dump文件保存了下来。

这两天忙里偷闲就想着分析一下原因。

<!--more-->

## 问题

### CPU 100%

虽然之前没遇到过CPU 100%的问题吧，但是，这个问题该怎么查心里还是有数的。

1. 使用`top`命令找到吃CPU的**Java**进程pid。 

   查到pid为：**31387**，CPU 接近400%，几乎4个核全吃满了。

2. 查询耗CPU的线程id。

   ```bash
   top -Hp 31387
   ```

   这里，查到了4个线程，每个线程都吃满了一个核接近100%。

   选取一个线程id：**31390**。

3. 线程id转16进制。

   ```bash
   printf "%x\n" 31390
   ```

   得到结果：**7a9e**

4. 输出线程堆栈信息。

   ```bash
   jstack 31387 | grep 7a9e -A 60|less
   ```

   结果忘截图了。不过结果很明显，4个线程都是**垃圾收集线程**。

{{% admonition warning 注意 %}}

执行jstack的用户需要与启动Java进程的用户一致。可通过以下命令查看启动Java进程的用户。

```shell
ps -ef | grep java
```

{{% /admonition %}}

### 垃圾收集

所以，究竟是为什么导致垃圾收集吃满了CPU呢？

```bash
jstat -gcutil  31387 1000 100
```

这个我截图了。

![这是一张图片](/image/SpringCloudConfig内存泄漏导致CPU100问题排查/1.png)

`Eden`和`Old`全满了，这很明显是**内存泄漏**了。

但是，内存泄漏的原因是啥呢？

```bash
jmap -dump:format=b,file=dump.txt 31387
```

我将堆dump文件保存了下来。

## 排查

### 大dump文件分析

当时Dump文件保存下来之后，一看大小。好家伙，`7G`多。然后我就面临一个问题，没法将这种大文件从服务器上拿下来。所以当时我就立马进行了打包压缩。

```bash
tar -xcvf dump.tar.gz dump.txt
```

打包之后，就只有`1G`多一点，然后我就从服务器上将压缩包拿到本地然后解压。

开始了漫长的分析过程。

#### jvisualvm

我知道JDK的bin目录下有个叫`jvisualvm`的程序是可以分析堆dump文件的。

直接点击菜单栏的`文件->装入`将**dump.txt**加载进去。

{{% admonition warning 注意 %}}

这里不要使用`VM 核心 dump`载入，不然会报错：**无效的核心dump文件**

{{% /admonition %}}

但是，由于我们的dump文件太大了，所以会报OOM。所以需要修改`jvisualvm`的配置。

配置文件在`%JAVA_HOME%\lib\visualvm\etc\visualvm.conf`。

我将配置修改为`-J-Xms2048m -J-Xmx2048m`，修改之后就可以将dump文件加载进去，但是，很`慢`。基本上无法进行分析工作。

#### jhat

```bash
jhat -J-mx8g dump.txt
```

我用小内存试过，都是OOM，所以我索性直接使用8G配置。

但是，还是不行，依然OOM。似乎它真的是需要8G内存，但是我电脑一共才8G内存。

#### MAT

因为用上面两个工具都太慢了，所以，这次我直接使用`Linux版的MAT`，直接在服务器上分析，借助服务器的强大性能，加快分析的速度。

1. 下载Linux版MAT。

   [下载地址](https://www.eclipse.org/mat/downloads.php)

2. 将MAT传到服务器解压后，执行以下命令。

   ```bash
   ./mat/ParseHeapDump.sh dump.txt org.eclipse.mat.api:suspects org.eclipse.mat.api:overview org.eclipse.mat.api:top_components
   ```

3. 然后将结果中的`dump_Leak_Suspects.zip`、`dump_System_Overview.zip`和`dump_Top_Components.zip`下载到本地分析。

{{% admonition info %}}

这里可能会有疑问，`org.eclipse.mat.api:suspects org.eclipse.mat.api:overview org.eclipse.mat.api:top_components`这些参数是啥，我怎么知道要这么填参数。

附上贴心的传送门：[Batch mode](https://help.eclipse.org/2021-03/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Ftasks%2Frunningleaksuspectreport.html)

{{% /admonition %}}

{{% admonition warning 注意 %}}

因为环境的原因，可能需要配置`MemoryAnalyzer.ini`。

比如：我添加了`-vm`配置，将其配置为JDK的bin目录。另外，将内存直接给了8g。

详细配置传送门：[Memory Analyzer Configuration](https://help.eclipse.org/2021-03/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Ftasks%2Frunningleaksuspectreport.html)

{{% /admonition %}}

##### 内存泄漏分析报告

![这是一张图片](/image/SpringCloudConfig内存泄漏导致CPU100问题排查/2.png)

![这是一张图片](/image/SpringCloudConfig内存泄漏导致CPU100问题排查/3.png)

```java
class DeleteOnExitHook {
    private static LinkedHashSet<String> files = new LinkedHashSet<>();
    static {
        // DeleteOnExitHook must be the last shutdown hook to be invoked.
        // Application shutdown hooks may add the first file to the
        // delete on exit list and cause the DeleteOnExitHook to be
        // registered during shutdown in progress. So set the
        // registerShutdownInProgress parameter to true.
        sun.misc.SharedSecrets.getJavaLangAccess()
            .registerShutdownHook(2 /* Shutdown hook invocation order */,
                true /* register even if shutdown in progress */,
                new Runnable() {
                    public void run() {
                       runHooks();
                    }
                }
        );
    }

    private DeleteOnExitHook() {}

    static synchronized void add(String file) {
        if(files == null) {
            // DeleteOnExitHook is running. Too late to add a file
            throw new IllegalStateException("Shutdown in progress");
        }

        files.add(file);
    }

    static void runHooks() {
        LinkedHashSet<String> theFiles;

        synchronized (DeleteOnExitHook.class) {
            theFiles = files;
            files = null;
        }

        ArrayList<String> toBeDeleted = new ArrayList<>(theFiles);

        // reverse the list to maintain previous jdk deletion order.
        // Last in first deleted.
        Collections.reverse(toBeDeleted);
        for (String filename : toBeDeleted) {
            (new File(filename)).delete();
        }
    }
}
```

很明显，就是`DeleteOnExitHook`的静态变量的这个`LinkedHashSet`内存泄漏了。

{{% admonition info %}}

另外，除了在服务器上分析。我又在本地机器上使用`Windows`版的MAT分析了试试。

结果出乎我的意料：**完全没问题**。

MAT,YYDS。

大胆猜测`jvisualvm`不行，`MAT`行！ 的原因是：

`MAT`分析会生成文件落到磁盘，所以没有`jvisualvm`那么吃内存。

{{% /admonition %}}

## 解决办法

其实，知道有内存泄漏的问题的时候，我就知道是因为`Spring Cloud Config`版本太低了。之前升级过，但是，这台服务器的`Spring Cloud Config`一直是弃用的还是跑的老版本，只要升级版本即可。

在Github上对于这个内存泄漏问题也有相关issue。

[MemoryLeak related to the HealthCheck](https://github.com/spring-cloud/spring-cloud-config/issues/668#issuecomment-347925388)

另外，这不是只有`Spring Cloud Config`会出现这种问题，因为究其原因还是`File.deleteOnExit API 存在缺陷。`

详见：[JDK-4872014](https://bugs.openjdk.java.net/browse/JDK-4872014)

{{% admonition info %}}

```java
public void deleteOnExit() {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkDelete(path);
        }
        if (isInvalid()) {
            return;
        }
        DeleteOnExitHook.add(path);
    }
```

`File.deleteOnExit`的作用就是在JVM退出的时候删除文件。

调用`File.deleteOnExit`会将路径添加到`DeleteOnExitHook`中的LinkedHashSet里面，如果不断添加，那么就会内存溢出。

所以`File.deleteOnExit`不能随便作为`File.delete()`的候选方法，比如：我怕`File.delete()`没删掉，然后调用`File.deleteOnExit`。这么做，如果是会创建大量文件的话，那么就很容易把内存撑爆了。

{{% /admonition %}}
