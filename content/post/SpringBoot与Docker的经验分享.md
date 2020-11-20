---
title: SpringBoot与Docker的经验分享
date: 2020-06-28 14:19:36
categories: ["Spring Boot", "Docker"]
tags: ["Spring Boot", "Docker"]
---

关于怎么将Spring Boot应用构建Docker镜像的教程，我们可以在Spring Boot的官网找到。

[Spring Boot with Docker](https://spring.io/guides/gs/spring-boot-docker/)

[Spring Boot Docker](https://spring.io/guides/topicals/spring-boot-docker)

从上面两个教程中，我主要了解的就是怎么写好`Dockerfile`，以及怎么构建镜像然后运行。

<!--more-->

## Dockerfile

对于一个Spring Boot应用，`Dockerfile`中要做最基本的就是这些。

1. 指定一个父镜像。
2. 将我们的jar包`COPY`到镜像中
3. `ENTRYPOINT`执行我们的应用程序

这种情况需要我们先将jar打好。但是这也太麻烦了，我想一步到位。

在教程中我们也可以找到。

```dockerfile
FROM openjdk:8-jdk-alpine as build
WORKDIR /workspace/app

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src

RUN ./mvnw install -DskipTests
RUN mkdir -p target/dependency && (cd target/dependency; jar -xf ../*.jar)

FROM openjdk:8-jdk-alpine
VOLUME /tmp
ARG DEPENDENCY=/workspace/app/target/dependency
COPY --from=build ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY --from=build ${DEPENDENCY}/META-INF /app/META-INF
COPY --from=build ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","hello.Application"]
```

这就是多阶段构建，首先使用`maven wrapper`将代码打成jar包，然后解压。再在第二阶段中`COPY`第一阶段的结果。关于为什么要解压，在Spring Boot的官方教程里已经说了，大概意思就是更快吧。

对于我来说，我只需要将`hello.Application`改成自己的**包名+主类名**，然后再加点自己的程序依赖的东西进去就可以了。

我的应用目前的`Dockerfile`如下，没有多大区别，就是多加了两个文件。

```dockerfile
FROM openjdk:8-jdk-alpine as build
WORKDIR /workspace/app

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src

RUN ./mvnw install -DskipTests
RUN mkdir -p target/dependency && (cd target/dependency; jar -xf ../*.jar)

FROM openjdk:8-jdk-alpine
ARG DEPENDENCY=/workspace/app/target/dependency
COPY --from=build ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY --from=build ${DEPENDENCY}/META-INF /app/META-INF
COPY --from=build ${DEPENDENCY}/BOOT-INF/classes /app
COPY kafka_client_jaas.conf /app/kafka/kafka_client_jaas.conf
COPY kafka.client.truststore.jks /app/kafka/kafka.client.truststore.jks
ENTRYPOINT ["java","-cp","app:app/lib/*","com.xxx.xxx.xxxApplication"]
```

## 构建

构建一般来讲都比较顺利，从来没有使用过`maven wrapper`的我就在想，既然`maven wrapper`能让没有maven的直接运行打出jar包，那么要使用私有的maven仓库该怎么指定。

这个问题可以通过`pom.xml`来解决。

```xml
<project>
<repositories>
    <repository>
        <id>xxx</id>
        <name>xxx</name>
        <url>xxx</url>
    </repository>
</repositories>
<pluginRepositories>
       <pluginRepository>
            <id>xxx</id>
            <name>xxx</name>
            <url>xxx</url>
       </pluginRepository>
</pluginRepositories>
</project>
```

## 运行

我们的应用程序的配置文件可能区分测试和生产。我们平时启动一般都是指定`spring.profiles.active`。那么使用docker运行的时候怎么指定呢。

```shell
docker run -e "SPRING_PROFILES_ACTIVE=local" -p 10088:10088 -t 镜像名
```

## 其他

### MySQL

我们的应用程序的数据库是外挂MySQL的，所以镜像中没有MySQL。所以我们的数据库链接`spring.datasource.url`不能使用localhost，而要使用ip地址。另外注意，MySQL要开启权限才能使用ip地址连接上。

### 容器内部

如果我们想看容器內部结构（比如，对于新手的我想看我COPY的文件是不是放在了指定位置），我们需要先运行（毕竟镜像运行起来了才有容器嘛），然后执行下面的命令进入容器内部。

```shell
docker exec -it 容器ID sh
```

容器ID可以通过`docker ps`查看。

### 镜像在哪

使用`Windows Desktop`的我就想，自己构建的镜像在哪，能不能自己拷贝出来。这个就别费这个劲了，镜像在虚拟机的数据里。想弄出来用Docker的命令导出来就好了。