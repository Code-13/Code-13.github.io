---
title: Spring-framework源码分析（二）架构总览与说明
abbrlink: b4e736ce
categories:
  - Spring-framework源码分析
date: 2020-07-21 10:32:03
tags:
  - 源码分析
  - Spring
  - Spring-framework
cover: 'https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720195606.jpg'
coverWidth: 1200
coverHeight: 750
---

## Spring-Framework整体架构

<!--more-->

### Spring 4.x 的架构图 (5.x的没有找到)

![4.x架构图](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721103735.png)

### Spring 5.x 版本的划分，以 `5.2.7.RELEASE` 为例

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721104619.png)

整理成表格如下：

| Core         | IoC Container, Events, Resources, i18n, Validation, Data Binding, Type Conversion, SpEL, AOP. |
| ------------ | ------------------------------------------------------------ |
| Testing      | Mock Objects, TestContext Framework, Spring MVC Test, WebTestClient. |
| Data Access  | Transactions, DAO Support, JDBC, O/R Mapping, XML Marshalling. |
| Web Servlet  | Spring MVC, WebSocket, SockJS, STOMP Messaging.              |
| Web Reactive | Spring WebFlux, WebClient, WebSocket.                        |
| Integration  | Remoting, JMS, JCA, JMX, Email, Tasks, Scheduling, Caching.  |
| Languages    | Kotlin, Groovy, Dynamic Languages.                           |


<font color=red>**无论怎么划分，core 模块中的 `IoC Container` 是重中之重，本系列的分析也以IOC容器为中心**</font>

## IOC容器组成说明

### Spring IOC 核心模块说明

spring 框架 IoC 容器的核心由3个模块组成：`beans（org.springframework.beans）`、`context（org.springframework.context）` 和 `core（org.springframework.core）`。其中，beans 模块和 context 模块是 IoC 容器的基础，再加上 core 这个工具操作类模块，构成了 spring 框架的基础架构。


IoC 容器本质上是管理 bean 及其相关依赖（和依赖关系）的 bean 工厂，用官方注释来说就是：bean container。其中，bean 模块中的 BeanFactory 接口提供了 IoC 容器最基础的功能接口，而 context 模块中的 ApplicationContext 则继承了 BeanFactory ，同时还增加了更多的企业级特性。官方文档和实际应用开发中一般都是以 ApplicationContext 作为 IoC 容器来进行操作。

## Spring IOC 核心类体系

### `ClassPathXmlApplicationContext`

从网上找了可一张 `ClassPathXmlApplicationContext` 的体系结构图;摘自: http://singleant.iteye.com/blog/1177358

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721115220.jpg)

该图为 `ClassPathXmlApplicationContext` 的类继承体系结构，虽然只有一部分，但是它基本上包含了 IOC 体系中大部分的核心类和接口。

### `AnnotationConfigApplicationContext`

下图是我截取自 `Idea` 中的继承结构图： 快捷键(`Ctrl + Alt + U`) 

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721115537.png)

### IoC 容器的可能涉及到的一些核心类体系：

- Resource 体系 和 ResourceLoader 体系：资源的抽象，以及资源的加载，常见的就是 bean 的 xml 配置文件作为资源输入来初始化 ApplicationContext。
- BeanDefinition 体系：bean 定义的封装类形式
- BeanDefinitionReader 体系：用于解析配置元数据，并封装成 BeanDefinition
- BeanFactory 体系：最基础的 bean 工厂定义
- ApplicationContext 体系：BeanFactory 的扩展实现，提供企业级应用特性。
- 扩展的 Aware、PostProcessor 等，其他待补充体系。。。

## Resource 体系 和 ResourceLoader 体系
文档地址

    https://docs.spring.io/spring-framework/docs/5.2.3.RELEASE/spring-framework-reference/core.html#resources

### Resource体系

Resource，对资源的抽象，它的每一个实现类都代表了一种资源的访问策略，如ClasspathResource 、 URLResource ，FileSystemResource 等。

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721121153.jpg)

### ResourceLoad体系

有了资源，就应该有资源加载，Spring 利用 `ResourceLoader` 来进行统一资源加载

`org.springframework.core.io.ResourceLoader`：在接口文档注释中被称为 ：`Strategy interface for loading resources (e.. class path or file system resources)`.策略模式来实现如何加载对应的资源 `Resource` 。

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721121216.png)

## [BeanDefinition](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-definition) 体系 和 BeanDefinitionReader体系

### [BeanDefinition](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-definition)

org.springframework.beans.factory.config.BeanDefinition：bean 对象及其相关元数据在 IoC 容器中的表示形式，简单理解就是 bean 对象的具体描述信息的封装类

根据官方文档描述，BeanDefinition 包含以下这些元信息：

- 所定义的 bean 的实现类的全限定名
- 一些 bean 行为配置元素，例如 作用域、回调函数（初始化、解构函数等）
- 相关的 bean 依赖
- 其他的一些新建 bean 时需要的初始设置，例如连接池的 size 限制，可以当成是非 bean 依赖的其他依赖，尤其是一些可以直接配置的 例如 String、基本类型的内部属性。

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721142937.png)

### BeanDefinitionReader 体系

org.springframework.beans.factory.support.BeanDefinitionReader：主要是对加载的 Resource 进行解析，并转换成对应的 BeanDefinition 结构。

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721143043.png)

## BeanFactory 与 ApplicationContext

### BeanFactory体系

`org.springframework.beans.factory.BeanFactory`： IoC 容器的根接口类，定义获取 bean 及 bean 的相关属性的基础方法。BeanFactory 的3个直接子类如下：

- `ListableBeanFactory` ：根据各种条件获取 bean 的配置清单 。
- `AutowireCapableBeanFactory` ：提供创建 bean 、自动注入、初始化以及应用 bean 的后处理器 。
- `HierarchicalBeanFactory` ：增加了对 parentFactory 的支持。

最终的默认实现类 `org.springframework.beans.factory.support.DefaultListableBeanFactory`，内部维护了一个 BeanDefinition Map，并根据 BeanDefinition 进行 bean 的创建和维护管理等。

```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721143414.png)

### ApplicationContext体系

真正有使用意义的 IoC 容器类是 org.springframework.context.ApplicationContext ，包括像 ClassPathXmlApplicationContext、FileSystemXmlApplicationContext 等实现类，像前面提到的，ApplicationContext 继承 BeanFactory 的特性，同时还进行了大量的扩展，提供了很多企业级应用特性，例如对国际化、事件机制、资源加载、AOP 集成、各类特定的应用层上下文，支持 J2EE 等。

- 继承了 `org.springframework.context.MessageSource` 接口，提供国际化支持
- 继承了 `org.springframework.context.ApplicationEventPublisher` 接口，提供强大的事件机制
- 继承了 `org.springframework.core.env.EnvironmentCapable` 接口，提供 `Environment` 相关操作，包括 `profiles and properties`，其中 `properties` 包含了 properties 文件、JVM 变量、系统变量等。具体 docs 参考 https://docs.spring.io/spring-framework/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-environment
- 扩展了 `org.springframework.core.io.ResourceLoader`，用于各种 `Resource` 的加载策略
- web 支持
- AOP 支持
- 。。。

在 `AnnotationConfigApplicationContext` 体系中， `AbstractApplicationContext` 的子类中 `GenericApplicationContext` 中包含了 `DefaultListableBeanFactory` 属性；在 `GenericApplicationContext` 初始化时会创建一个 `DefaultListableBeanFactory` 对象，这便是最终的 `BeanFactory`

```java
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {

	private final DefaultListableBeanFactory beanFactory;

	@Nullable
	private ResourceLoader resourceLoader;

	private boolean customClassLoader = false;

	private final AtomicBoolean refreshed = new AtomicBoolean();


	/**
	 * Create a new GenericApplicationContext.
	 * @see #registerBeanDefinition
	 * @see #refresh
	 */
	public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory(); //最终的BeanFactory
	}

....其余代码
```

网上找来的一张图：

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721145301.png)

这张图虽然没有 `AnnotationConfigApplicationContext` ，但是原理是相同的

## 总结

- 上面的五个体系( `BeanFactory` 与 `ApplicationContext` 可以看作相同的体系)是 `Spring IOC` 最核心的部分
- 后面的源码解析都以这五个部分为主
- SpringBoot等都是在此基础之上进行扩展

## 参考

- [Spring-framework 5.2.7.RELEASE 官方文档](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/index.html)
- [Spring5源码分析(003)——IoC容器总览和说明](https://www.cnblogs.com/wpbxin/p/12791042.html)
- [【死磕 Spring】—– IOC 之深入理解 Spring IoC](http://cmsblogs.com/?p=2652)
- [Spring IOC（一）概览](https://www.cnblogs.com/dennyzhangdd/p/7652075.html)