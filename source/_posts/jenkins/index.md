---
title: Jenkins Pipeline
categories: CI
index_img: 'https://img.icons8.com/color/480/000000/jenkins.png'
tags:
  - CI
  - Jenkins
  - Pipeline
abbrlink: 27713
date: 2020-10-09 09:28:07
---

该文章主要Jenkins Pipeline的使用

<!-- more -->

## Jenkins-Pipeline



### Pipeline Block

> 所有有效的声明式流水线必须包含在一个 `pipeline `块中
>
> ```groovy
> pipeline {
>   
> }
> ```
>
> 遵循Groovy语法。
>
> 特点：
>
> 1. 流水线顶层必须是一个 `pipeline{}`。
> 2. 没有分号作为语句分隔符，每条语句都必须在自己的行上。
> 3. 块只能由`节段，指令，步骤或赋值语句`组成。 *属性引用语句被视为无参方法调用。例如：input被视为input()。



### Agent Block

##### 概述

> `agent` 部分指定了整个流水线或特定的部分, 将会在Jenkins环境中执行的位置，这取决于 `agent` 区域的位置。
>
> 该部分必须在 `pipeline` 块的顶层被定义, 但是 stage 级别的使用是可选的。

##### 参数

> `any`：在任何可用的代理上执行流水线或阶段。例如: `agent any`
>
> `label`：在提供了标签的 Jenkins 环境中可用的代理上执行流水线或阶段。 例如: `agent { label 'node-1' }`
>
> `node`： `agent { node { label 'labelName' } }` 和 `agent { label 'labelName' }` 一样, 但是 `node` 允许额外的选项 (比如 `customWorkspace` )。



### Post Block

> `post` 部分定义一个或多个[steps](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps) ，这些阶段根据流水线或阶段的完成情况而运行(取决于流水线中 `post` 部分的位置)。
>
> `post`支持`always`, `changed`, `failure`, `success`, `unstable`, 和 `aborted`块中任意一个。
>
> 这些条件块允许在 `post` 部分的步骤的执行取决于流水线或阶段的完成状态。

#### 块对应的条件

> `always`:无论流水线或阶段的完成状态如何，都允许在 `post` 部分运行该步骤。
>
> `changed`:当前流水线或阶段的完成状态与它之前的运行不同时，才能执行。
>
> `failure`:当前流水线或阶段的完成状态为"failure"，才能执行,。
>
> `success`:当前流水线或阶段的完成状态为"success"，才能执行。
>
> `unstable`:当前流水线或阶段的完成状态为"unstable"，才能执行, 通常由于测试失败,代码违规等造成。通常web UI是黄色。
>
> `aborted`:当前流水线或阶段的完成状态为"aborted"，才能执行, 通常由于流水线被手动的aborted。通常web UI是灰色。



### Stages 块

> 包含一系列一个或多个 [stage](https://jenkins.io/zh/doc/book/pipeline/syntax/#stage) 指令, `stages` 部分是流水线描述的大部分"工作" 的位置。

```groovy
pipeline {
    agent any
    stages { 
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

### Steps块

> `steps` 部分在给定的 `stage` 指令中执行的定义了一系列的一个或多个[steps](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps)



## 指令

#### environment

> `environment` 指令制定一个 键-值对序列，该序列将被定义为所有步骤的环境变量，或者是特定于阶段的步骤， 这取决于 `environment` 指令在流水线内的位置。
>
> 该指令支持一个特殊的助手方法 `credentials()` ，该方法可用于在Jenkins环境中通过标识符访问预定义的凭证。

#### options

> `options` 指令允许从流水线内部配置特定于流水线的选项。

##### 可用选项

- buildDiscarder

  为最近的流水线运行的特定数量保存组件和控制台输出。例如: `options { buildDiscarder(logRotator(numToKeepStr: '1')) }`

- disableConcurrentBuilds

  不允许同时执行流水线。 可被用来防止同时访问共享资源等。 例如: `options { disableConcurrentBuilds() }`

- overrideIndexTriggers

  允许覆盖分支索引触发器的默认处理。 如果分支索引触发器在多分支或组织标签中禁用, `options { overrideIndexTriggers(true) }` 将只允许它们用于促工作。否则, `options { overrideIndexTriggers(false) }` 只会禁用改作业的分支索引触发器。

- skipDefaultCheckout

  在`agent` 指令中，跳过从源代码控制中检出代码的默认情况。例如: `options { skipDefaultCheckout() }`

- skipStagesAfterUnstable

  一旦构建状态变得UNSTABLE，跳过该阶段。例如: `options { skipStagesAfterUnstable() }`

- checkoutToSubdirectory

  在工作空间的子目录中自动地执行源代码控制检出。例如: `options { checkoutToSubdirectory('foo') }`

- timeout

  设置流水线运行的超时时间, 在此之后，Jenkins将中止流水线。例如: `options { timeout(time: 1, unit: 'HOURS') }`

- retry

  在失败时, 重新尝试整个流水线的指定次数。 For example: `options { retry(3) }`

- timestamps

  预谋所有由流水线生成的控制台输出，与该流水线发出的时间一致。 例如: `options { timestamps() }`



##### 可选的阶段选项

- skipDefaultCheckout

  在 `agent` 指令中跳过默认的从源代码控制中检出代码。例如: `options { skipDefaultCheckout() }`

- timeout

  设置此阶段的超时时间, 在此之后， Jenkins 会终止该阶段。 例如: `options { timeout(time: 1, unit: 'HOURS') }`

- retry

  在失败时, 重试此阶段指定次数。 例如: `options { retry(3) }`

- timestamps

  预谋此阶段生成的所有控制台输出以及该行发出的时间一致。例如: `options { timestamps() }`



### Tools

> 定义自动安装和放置 `PATH` 的工具的一部分。

```groovy
pipeline {
    agent any
    tools {
        maven 'apache-maven-3.0.1' 
    }
    stages {
        stage('Example') {
            steps {
                sh 'mvn --version'
            }
        }
    }
}
```



```groovy
when {
	branch 'development'
}
```

when 用于判断



```groovy
pipeline {
  agent any

  stages {
    stage('Test') {
      steps {
        /* `make check` 在测试失败后返回非零的退出码；
         * 使用 `true` 允许流水线继续进行
         */
        sh 'make check || true' 
        junit '**/target/*.xml' 
      }
    }
  }
}
```





```groovy
choice(
  description: 'Run flyway database migration using latest master branch from prices in what environment?',
  name: 'environment',
  choices: ['PRE', 'PRO']
)
```



input

```groovy
input {
  message "是否继续?"
  ok "发布 "
  submitter "alice,bob"
  parameters {
    string(name: 'IP', defaultValue: '0.0.0.0', description: '线上节点 1')
  }
}
```



```sh
nohup java -jar -Dspring.profiles.active=prod /data/jars/admin-io-better-cn-api/master/admin-io-better-cn-api-master.jar >/data/nohup.out 2>&1 &
```



```sh
ps -ef |grep java|grep admin-io-better-cn-api-master.jar|grep prod|grep -v grep|awk '{print $2}'
```





## 环境变量

> BUILD_ID
>
> 当前构建的 ID，与 Jenkins 版本 1.597+ 中创建的构建号 BUILD_NUMBER 是完全相同的。
>
> BUILD_NUMBER
>
> 当前构建号，比如 “153”。
>
> BUILD_TAG
>
> 字符串 ``jenkins-${JOB_NAME}-${BUILD_NUMBER}``。可以放到源代码、jar 等文件中便于识别。
>
> BUILD_URL
>
> 可以定位此次构建结果的 URL（比如 http://buildserver/jenkins/job/MyJobName/17/ ）
>
> EXECUTOR_NUMBER
>
> 用于识别执行当前构建的执行者的唯一编号（在同一台机器的所有执行者中）。这个就是你在“构建执行状态”中看到的编号，只不过编号从 0 开始，而不是 1。
>
> JAVA_HOME
>
> 如果你的任务配置了使用特定的一个 JDK，那么这个变量就被设置为此 JDK 的 JAVA_HOME。当设置了此变量时，PATH 也将包括 JAVA_HOME 的 bin 子目录。
>
> JENKINS_URL
>
> Jenkins 服务器的完整 URL，比如 https://example.com:port/jenkins/ （注意：只有在“系统设置”中设置了 Jenkins URL 才可用）。
>
> JOB_NAME
>
> 本次构建的项目名称，如 “foo” 或 “foo/bar”。
>
> NODE_NAME
>
> 运行本次构建的节点名称。对于 master 节点则为 “master”。
>
> WORKSPACE
>
> workspace 的绝对路径。