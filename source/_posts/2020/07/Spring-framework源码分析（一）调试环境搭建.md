---
title: 'Spring-framework源码分析（一）调试环境搭建'
abbrlink: 83dc2df5
categories:
  - Spring-framework
date: 2020-07-20 19:21:09
tags:
  - 源码分析
  - Spring
  - Spring-framework
cover: 'https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720195606.jpg'
coverWidth: 1200
coverHeight: 750
---

> 本文主要内容： 搭建spring-framework 编译调试环境源码

<!--more-->

## 准备工作

### 准备一下工具

- `JDK11`(或者JDK8以上即可,本系列采用JDK11)
- [idea](https://www.jetbrains.com/idea/) 开发工具

### 源码准备


- 前往Spring的官方仓库：https://github.com/spring-projects/spring-framework


- 将代码`clone`或者`Download ZIP`均可,这里推荐`clone`的方式，因为可以方便的对比变化：


![仓库](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720200632.png)


- clone 完成之后如图所示：


![文件](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720201545.png)


## Idea导入项目


- 打开 `Idea`，选择 `open or import`：

![open or import](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720202627.png)

- 选择刚才 `clone` 的文件夹：


![文件夹](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720202720.png)


- Spring会自动下载 `gradle`，但可能会很慢，耐心等待一下即可


- 在 `build.gradle` 文件中的 `repositories` 加入[阿里云仓库](https://developer.aliyun.com/mirror/maven)，如图所示

```gradle
repositories {
  mavenCentral()
  maven { url 'https://maven.aliyun.com/repository/public/' }
  maven { url "https://repo.spring.io/libs-spring-framework-build" }
  maven { url "https://repo.spring.io/milestone" } // Reactor
}
```

- 在项目的根目录下，有一个 `import-into-idea` 文件，里面描述了如何导入idea，但其实不按照上面也行


## 新建模块


1. 由于 `Spring` 源码使用 `gradle` 构建源码，所以我们需要新建一个 `gradle` 模块


![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720205554.png)


![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720205605.png)


2. 在 build.gradle 引入依赖

```gradle
dependencies {
  compile(project(":spring-beans"))
  compile(project(":spring-context"))
  compile(project(":spring-core"))
  compile(project(":spring-aop"))
  optional("org.aspectj:aspectjweaver")
  testCompile group: 'junit', name: 'junit', version: '4.12'
}
```