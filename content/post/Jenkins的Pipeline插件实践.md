---
title: "Jenkins的Pipeline插件实践"
date: 2020-12-15T21:12:12+08:00
lastmod: 2020-12-15T21:12:12+08:00
categories: ["Jenkins"]
tags: ["Jenkins", "Pipeline", "Maven"]
keywords: ["Jenkins", "Pipeline", "Maven", "多模块", "单独部署"]
draft: false
---

在Jenkins的[官方文档](https://www.jenkins.io/doc/book/)中，可以看到到处都是**Pipeline**的身影，而且文档多数内容都是讲Pipeline。由此可见，Pipeline非常重要，我愿称之为Jenkins的精髓。最重要的是，Pipeline还很好用。接下来，以一个`多模块的Maven项目`为例，记录一下我使用Pipeline的一些经验。

<!--more-->

## 准备

- Pipeline
- Blue Ocean

插件管理里面直接搜就行了。`Blue Ocean`不是必须的，但是强烈推荐。

## 项目结构

```
cloud
|-auth
|-common           #公共包，被其他模块引用
|-gateway
|-service
|-----system
|-----pom.xml
|-service-api
|-----system-api   #被system引用
|-----pom.xml
|-pom.xml
|-deploy.sh
|-Jenkinsfile
```

这个项目结构，我相信已经符合大多数人的需要。所以，下面我给出的`Jenkinsfile`会有一定的参考价值。

## 流水线任务

### 新建

![图片](/image/Jenkins的Pipeline插件实践/1.png)

安装了Pipeline插件就可以新建流水线任务了。

### 流水线脚本来源

![图片](/image/Jenkins的Pipeline插件实践/2.png)

Pipeline任务的重点当然就是流水线脚本了。几乎所有的工作都在脚本里面了。

可以看出，有**两种**方式。

- 直接在Jenkins上写
- 从`SCM`检出

这里我用第二种，从**SVN**上检出`Jenkinsfile`。

### Jenkinsfile

接下来就是重中之重了。

```groovy
pipeline {
    agent any
    parameters {
        choice(name: 'MODULE', choices: ['all', 'auth', 'gateway', 'system'], description: '请选择部署模块!')
        choice(name: 'PUBLISH_SERVER', choices: ['127.0.0.1'], description: '目标服务器')
        choice(name: 'ACTIVE', choices:['prod', 'test', 'dev'], description: 'spring.profiles.active')
    }
    environment {
        // SVN仓库地址
        REPO = 'https://xxx/cloud'
        // Jenkins凭据ID
        REPO_CREDENTIALSID = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
        // Publish Over SSH Remote Directory
        REMOTE_DIR = '/app/cloud'
    }
    tools {
        maven 'apache-maven-3.6.3'
    }
    stages {
        // 检出代码
        stage('Checkout') {
            steps {
                checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[cancelProcessOnExternalsFail: true, credentialsId: "${env.REPO_CREDENTIALSID}", depthOption: 'infinity', ignoreExternalsOption: true, local: '.', remote: "${env.REPO}"]], quietOperation: true, workspaceUpdater: [$class: 'UpdateUpdater']])
            }
        }
        // 将deploy.sh传送到目标服务器
        stage('Transfer Shell Script') {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: "${params.PUBLISH_SERVER}", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "chmod 777 ${REMOTE_DIR}/shell/deploy.sh", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: 'shell', remoteDirectorySDF: false, removePrefix: '', sourceFiles: 'deploy.sh')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: true)])
            }
        }
        // 使用Maven打包
        stage('Maven Package') {
            parallel {
                // 打包所有项目
                stage('Package All') {
                    when {
                        equals expected: "all", actual: "${params.MODULE}"
                    }
                    steps {
                        sh "mvn clean package -Dmaven.test.skip=true"
                    }
                }
                // 打包指定的项目
                stage('Package Specific Module') {
                    when {
                        not {
                            equals expected: "all", actual: "${params.MODULE}"
                        }
                    }
                    steps {
                        sh "mvn clean package -pl com.github.l-qiang:${params.MODULE} -am -amd -Dmaven.test.skip=true"
                    }
                }
            }
        }
        // 部署: 传送jar包, 然后执行deploy.sh
        stage('Deploy') {
            parallel {
                stage('auth') {
                    when {
                        anyOf {
                            equals expected: "auth", actual: "${params.MODULE}"
                            equals expected: "all", actual: "${params.MODULE}"
                        }
                    }
                    steps {
                        sshPublisher(publishers: [sshPublisherDesc(configName: "${params.PUBLISH_SERVER}", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "${REMOTE_DIR}/shell/deploy.sh restart auth ${params.ACTIVE}", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: "auth/target", sourceFiles: "auth/target/*.jar")], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: true)])
                    }
                }
                stage('gateway') {
                    when {
                        anyOf {
                            equals expected: "gateway", actual: "${params.MODULE}"
                            equals expected: "all", actual: "${params.MODULE}"
                        }
                    }
                    steps {
                        sshPublisher(publishers: [sshPublisherDesc(configName: "${params.PUBLISH_SERVER}", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "${REMOTE_DIR}/shell/deploy.sh restart gateway ${params.ACTIVE}", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: "gateway/target", sourceFiles: "gateway/target/*.jar")], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: true)])
                    }
                }
                stage('system') {
                    when {
                        anyOf {
                            equals expected: "system", actual: "${params.MODULE}"
                            equals expected: "all", actual: "${params.MODULE}"
                        }
                    }
                    steps {
                        sshPublisher(publishers: [sshPublisherDesc(configName: "${params.PUBLISH_SERVER}", transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "${REMOTE_DIR}/shell/deploy.sh restart system ${params.ACTIVE}", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: "service/system/target", sourceFiles: "service/system/target/*.jar")], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: true)])
                    }
                }
            }
        }
    }

    options {
        // 丢弃旧的构建, 保持构建的最大个数
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }
}
```

以上就是完整的`Jenkinsfile`了。Pipeline脚本分为`声明式`和`脚本式`, 官方推荐使用`声明式`，所以这里使用的`声明式`。 [Pipeline语法](https://www.jenkins.io/doc/book/pipeline/syntax/)就不多说了，接下来主要讲上面这个`Jenkinsfile`。

由外层往内层，先看整体。

```groovy
pipeline {
    agent any
    parameters {...}
    environment {...}
    tools {...}
    stages {...}
    
    options{...}
}
```

- `agent`

  这里跟Jenkins的节点有关，我这只有一个**master**节点，所以这里使用`any`就行。

- `parameters`

  这里定义一些参数，及参数的默认值，这些参数是可能会经常改变的值，可与用户交互。这些参数在**Pipeline**`运行`的时候，可以`修改`。通过这些参数，我们可以控制Pipleline的`执行流程`。

  在`Blue Ocean`界面中。点`运行`就会弹出下列窗口。

  ![图片](/image/Jenkins的Pipeline插件实践/4.png)

{{% admonition warning %}}

​	**第一次**运行的时候，不会弹框，再运行一次即可。不知道之后的版本的Jenkins会不会修复这个问题。

{{% /admonition %}}

- `environment`

  这里定义一些环境变量，这些参数很少需要改变。

{{% admonition info %}}

​	注意`参数`和`环境变量`都要在`双引号`里面使用，单引号会将内容原样输出。

{{% /admonition %}}

- `tools`

  ```groovy
  maven 'apache-maven-3.6.3'
  ```

  配置Maven。这里的Maven必须是`全局工具配置`里面的。这不是必须的，这里这么做就可以直接执行`mvn`命令而不需要在Jenkins服务器有任何配置。

- `stages`

  这里的步骤大致跟[Jenkins部署Maven项目]({{<ref "Jenkins部署Maven项目">}})差不多。

  1. **Checkout**

     从SVN检出代码。这段代码看似复杂其实很简单。使用`片段生成器`就可以生成。

     ![图片](/image/Jenkins的Pipeline插件实践/5.png)

     ![图片](/image/Jenkins的Pipeline插件实践/6.png)

  2. **Transfer Shell Script**

     将部署脚本传送到目标服务器。相对于之前的[Jenkins部署Maven项目]({{<ref "Jenkins部署Maven项目">}})，这里有所改动。所有项目模块只使用一份[脚本](#部署脚本)。仍然使用`片段生成器`来生成**Publish Over SSH**插件的代码。

{{% admonition info %}}

默认情况下，**Publish Over SSH**不输出脚本日志，需要勾选`Verbose output in console`。

![图片](/image/Jenkins的Pipeline插件实践/7.png)

{{% /admonition %}}

  3. **Maven Package**

     ```groovy
     stage('Maven Package') {
         parallel {
             stage('Package All'){...}
             stage('Package Specific Module'){...}
         }
     }
     ```

     这里就是通过`when`和`parameters`来控制打包`一个模块`还是`所有模块`。这里`parallel`的作用只是在运行的时候在`Blue Ocean`里面看起来更像是一个条件分支，而不是用来并行的。

{{% admonition warning "注意"%}}

```shell
sh "mvn clean package -pl com.github.l-qiang:${params.MODULE} -am -amd -Dmaven.test.skip=true"
```

上述命令如果并行执行，可能会出现问题，因为他们之间有共同的依赖。但是，不会必定出现。

另外，使用`groupId:artifactId`的方式比使用`路径`更加灵活。

{{% /admonition %}}

  4. **Deploy**

     这一步同样使用`when`和`parameters`来控制部署`一个模块`还是`所有模块`。选择所有模块可以并行执行。

- `options`

  一些额外的配置，这里就配置了丢弃旧的构建。更多配置项见[options](https://www.jenkins.io/doc/book/pipeline/syntax/#optionshttps://www.jenkins.io/doc/book/pipeline/syntax/#options)




运行Pipeline之后的结果如下：

![图片](/image/Jenkins的Pipeline插件实践/3.png)

---

## 部署脚本

`deploy.sh`

```shell
ACTION=$1
MODULE_NAME=$2
ACTIVE=$3
JAR_NAME=$MODULE_NAME-0.0.1-SNAPSHOT.jar
cd /app/cloud
#使用说明，用来提示输入参数
usage() {
echo "Usage: sh deploy.sh [start|stop|restart|status] module spring.profiles.active"
exit 1
}

#检查程序是否在运行
is_exist(){
pid=`ps -ef|grep $JAR_NAME|grep -v grep|awk '{print $2}' `
#如果不存在返回1，存在返回0
if [ -z "${pid}" ]; then
return 1
else
return 0
fi
}

#启动方法
start(){
is_exist
if [ $? -eq "0" ]; then
echo "${JAR_NAME} is already running. pid=${pid} ."
else
nohup java -jar $JAR_NAME --spring.profiles.active=$ACTIVE > logs/out_$MODULE_NAME.log 2>&1 &
fi
}

#停止方法
stop(){
is_exist
if [ $? -eq "0" ]; then
kill -9 $pid
else
echo "${JAR_NAME} is not running"
fi
}

#输出运行状态
status(){
is_exist
if [ $? -eq "0" ]; then
echo "${JAR_NAME} is running. Pid is ${pid}"
else
echo "${JAR_NAME} is NOT running."
fi
}

#重启
restart(){
stop
start
}

#根据输入参数，选择执行对应方法，不输入则执行使用说明
case "$ACTION" in
"start")
start
;;
"stop")
stop
;;
"status")
status
;;
"restart")
restart
;;
*)
usage
;;
esac
```



