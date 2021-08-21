---
title: 'Spring-framework源码分析（一）调试环境搭建'
abbrlink: 83dc2df5
categories:
  - Spring-framework源码分析
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


- 由于 `Spring` 源码使用 `gradle` 构建源码，所以我们需要新建一个 `gradle` 模块


![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720205554.png)


- 取个模块名，我这里就叫做 `spring-examples`：


![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720205605.png)


- 在 `build.gradle` 引入依赖

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


## 测试

- 写一个配置类

``` java
package com.github.code13.app;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

/**
 * @author Code13
 * @date 2020-07-14 14:27
 */
@ComponentScan("com.github.code13")
//@EnableAspectJAutoProxy 开启Aop注解功能，现在暂时不需要
public class AppConfig {
}
```

- 写一个 `bean`

``` java
package com.github.code13.bean;

import org.springframework.stereotype.Component;

/**
 * @author Code13
 * @date 2020-07-14 14:46
 */
@Component
public class B {

	public B(){
		System.out.println("B创建了");
	}

	public void test1(){
		System.out.println("被增强的方法");
	}

}
```

- 测试类

``` java
package com.github.code13;

import com.github.code13.app.AppConfig;
import com.github.code13.bean.B;
import org.junit.jupiter.api.Test;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

/**
 * AOP测试
 *
 * @author Code13
 * @date 2020-07-14 14:45
 */
class AopTest {

	@Test
	void test1(){
		final AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
		final B bean = ac.getBean(B.class);
		bean.test1();
		System.out.println(bean.getClass().getName());
	}

}
```

- 查看结果


    如果没有什么问题的话，调试环境应该已经搭建成功


![success](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721090744.png)


## 问题

**其实我们在搭建的过程中可能会遇见各种各样的问题**，下面尝试对这些问题给出解决方案

### `Github` 下载缓慢的问题

- [完美解决github访问速度慢](https://zhuanlan.zhihu.com/p/93436925)
- [GitHub国内镜像源](https://www.zhihu.com/question/38192507)
- [利用Gitee不改Host，解决GitHub下载速度慢的方法，亲测有效](https://www.jianshu.com/p/de183d44bd27)

其实关于Github的镜像源有很多，这里还是推荐国内的 [Gitee](https://www.gitee.com)；快是真的快


### `Spring` 源码编译问题

- [spring源码系列（六）——番外篇如何编译spring的源码](https://blog.csdn.net/java_lyvee/article/details/107300648)

- [Spring源码分析环境搭建](https://www.bilibili.com/video/BV1ui4y137K3)