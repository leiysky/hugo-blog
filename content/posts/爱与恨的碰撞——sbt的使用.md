---
title: "爱与恨的碰撞——sbt的使用"
date: 2020-02-11T14:07:49+08:00
draft: true
---

# 前言

sbt 是 Scala 官方推荐的 build 工具，也是 Scala 用户中极为推崇的工具。

但是我就奇了怪了，这玩意真的像你们说的那样好用吗？官方文档写的像论文一样晦涩难懂，配置项多如繁星，甚至有各种官方文档没有明确写出来的“隐藏功能”……

但是有一说一，sbt 的自由度确实很高，毕竟直接使用 Scala 作为配置语言，这点我还是很爱的。

自由度高加上垃圾的文档带来的问题就是每次使用都要查半天资料，所以在此进行记录，以节约我宝贵的生命。

# 安装

直接上官网下载即可。

macOS 下可以使用 brew 进行安装：
```shell
$ brew install sbt
```

目前的最新版本为 1.3.8。

# 包管理

sbt 使用 Apache Ivy 作为包管理器，而 Ivy 一般可以使用 Maven 的仓库来拉取依赖。

我们都知道 Maven 的中央仓库下载速度已经慢到一定地步了，不换源基本没法用，因此这部分主要讲一下怎么换源。

国内比较知名的 Maven 源很少，主要就是几大云厂商提供的，比如阿里云。阿里云的 Maven 源速度很快，但缺点是包有残缺。我目前使用的是华为云的源，目前来看还算比较好用，使用的时候优先级略低于阿里云。

sbt 的换源不像其他包管理器那样方便，按照官方文档的说法它必须在多个地方进行配置，设计的十分恶心。

这里提供一种相对比较人性化的方式。

首先我们要修改 sbt 的仓库配置 **~/.sbt/repositories** ，默认情况下应该是没有这个文件的，我们需要创建一个。

之后在文件中填入如下内容：
```conf
[repositories]
    local
    alicloud: https://maven.aliyun.com/repository/public
    huaweicloud-maven: https://repo.huaweicloud.com/repository/maven/
    maven-central: https://repo1.maven.org/maven2/
    huaweicloud-ivy: https://repo.huaweicloud.com/repository/ivy/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
```

local 表示的就是 Maven 和 Ivy 的本地仓库，优先级最高。其他的看着配就行。

一般来说写好这个配置之后就应该 work 了，但 sbt 不行，它反人类的一个地方就在于它还需要修改 sbt-launch.jar 的启动配置。

官方推荐的方式是进入到 sbt-launch.jar 包里面(???)修改一个 **sbt.boot.properties** 文件。

我推荐不这么做，因为实在是太恶心了。实际上虽然官方文档在 Repositories 那部分没有提到(它是在 Library Dependencies 这部分里一笔带过的)，但是我们可以通过传一个启动参数或者环境变量的方式指定使用自定义的仓库配置。

我们可以通过修改 **sbtopts** 的方式来指定启动参数，使用 brew 进行安装的话这个文件的目录应该是 **/usr/local/etc/sbtopts** ，我们在末尾追加上一行：
```shell
-Dsbt.override.build.repos=true
```

之后在 sbt shell 中输入：
```shell
sbt> show overrideBuildResolvers
[info] true
```

如果像上面一样输出 **true** 的话就说明配置成功了。

怎么样，是不是很恶心？

# build.sbt

build.sbt 是 sbt 的项目配置文件，类似于 Maven 的 pom.xml。

build.sbt 的亮点在于它直接使用 Scala 编写配置，但是第一次写的时候你根本不知道要写些什么东西。

看一下官方给的样例：
```scala
ThisBuild / version      := "0.1.0"
ThisBuild / scalaVersion := "2.12.7"
ThisBuild / organization := "com.example"

val scalaTest = "org.scalatest" %% "scalatest" % "3.0.5"
val gigahorse = "com.eed3si9n" %% "gigahorse-okhttp" % "0.3.1"
val playJson  = "com.typesafe.play" %% "play-json" % "2.6.9"

lazy val hello = (project in file("."))
  .aggregate(helloCore)
  .dependsOn(helloCore)
  .enablePlugins(JavaAppPackaging)
  .settings(
    name := "Hello",
    libraryDependencies += scalaTest % Test,
  )

lazy val helloCore = (project in file("core"))
  .settings(
    name := "Hello Core",
    libraryDependencies ++= Seq(gigahorse, playJson),
    libraryDependencies += scalaTest % Test,
  )
```

简单来说一个配置的组成是这样的：

* 一些元信息定义，比如 version
* 一些依赖定义，依赖的描述方式和 Java 差不多
* project 的定义，采用声明式的方式

详细的内容会在后面提到。

# sbt shell

sbt shell 就是 sbt 为我们提供的交互式客户端。一般的 build 工具都是只提供 CLI，通过子命令之类的方式来交互，比如说 npm, go, cargo, Maven etc. 但是我们的 sbt 就很高级，直接提供了一个 console(其实是因为 JVM 启动太慢了，所以如果每次都跑 CLI 命令会体验极差)。

最常用的几种命令：

* **compile**: 编译
* **run**: 编译并运行
* **exit**: 退出
* **test**: 跑测试
* **clean**: 清理

sbt shell 甚至提供了 batch mode，一次跑多条命令，并且不用启动 console（然并卵）。

sbt shell 还有个比较高级的功能，就是提供了一些类似 bash 的功能：

* **!**  历史记录
* **!!**  重新执行上一条
* **!:**  显示所有的之前的命令
* **!:n** 显示最后 n 条命令
* **!n**  执行第 n 条命令
* **!-n** 执行倒数第 n 条命令
* **!string** 执行最近的一条以 string 为前缀的命令
* **!?string** 执行最近一条包含 string 的命令

简直碉堡了。

## To be continued