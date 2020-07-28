---
title: Spring-framework源码分析（五）refresh方法
abbrlink: 53ad1a77
categories:
  - Spring-framework源码分析
date: 2020-07-28 16:53:00
tags:
  - 源码分析
  - Spring
  - Spring-framework
cover: 'https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720195606.jpg'
coverWidth: 1200
coverHeight: 750
---

refresh方法是Spring容器最核心的方法
- 概括了Spring初始化bean的整个生命周期
- 生命周期的回调
- AOP
- Spring解决循环依赖的三级缓存

<!--more-->

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
    // Prepare this context for refreshing.

    //这里是做刷新bean工厂前的一系列赋值操作,主要是为前面创建的Spring工厂很多的属性都是空的，这个方式是为他做一些列的初始化值的操作！
    prepareRefresh();

    // Tell the subclass to refresh the internal bean factory.
    //告诉子类刷新内部bean工厂 检测bean工厂是否存在 判断当前的bean工厂是否只刷新过一次，多次报错，返回当前使用的bean工厂,当该步骤为xml时 会新建一个新的工厂并返回
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // Prepare the bean factory for use in this context.
    //这里是初始化Spring的bean容器，向beanFactory内部注册一些自己本身内置的Bean后置处理器也就是通常说的BeanPostProcessor，这个方法其实也是再初始化工厂！
    prepareBeanFactory(beanFactory);

    try {
      // Allows post-processing of the bean factory in context subclasses.
      //允许在上下文子类中对bean工厂进行后处理,作用是在BeanFactory准备工作完成后做一些定制化的处理! 但是注意，你点进去是空方法，空方法以为着什么？意味着Spring的开发者希望调用者自定义扩展时使用！
      postProcessBeanFactory(beanFactory);

      // Invoke factory processors registered as beans in the context.
      //其实相信看名字，大部分读者都能够猜出，他的目的是扫描非配置类的bd注册到工厂里面，扫描完成之后，开始执行所有的BeanFactoryPostProcessors，这里出现了第一个扩展点，自定义实现BeanFactoryPostProcessors的时候，他的回调时机是在Spring读取了全部的BeanDefinition之后调用的
      invokeBeanFactoryPostProcessors(beanFactory);

      // Register bean processors that intercept bean creation.
      //这里是注册bean的后置处理器 也就是  beanPostProcessor 的实现 还有自己内置的处理器  注意这里并没有调用该处理器，只是将胡处理器注册进来bean工厂! 不知道大家使用过beanPostProcessor接口这个扩展点吗？他就是再这个方法里面被注册到Spring工厂里面的，当然注意一下，他只是注册进去了，并没有执行！记住并没有执行！
      registerBeanPostProcessors(beanFactory);

      // Initialize message source for this context.
      //国际化操作也就是 i18n的资源初始化
      initMessageSource();

      // Initialize event multicaster for this context.
      //Spring为我们提供了对于事件编程的封装，一般来说事件分为三个角色，事件广播器(发布事件)，事件监听器(监听事件)，事件源（具体的事件信息）三个角色,这个方法的目的就是初始化事件的广播器！
      initApplicationEventMulticaster();

      // Initialize other special beans in specific context subclasses.
      //这里又是一个扩展点，内部的方法为空，Spring并没有实现它，供调用者实现！
      onRefresh();

      // Check for listener beans and register them.
      //注册Spring的事件监听器,上面已经说过了，这里是初始化并且注册事件监听器
      registerListeners();

      // Instantiate all remaining (non-lazy-init) singletons.
      //这个方法是一个重点，他是为了实例化所有剩余的（非延迟初始化）单例。我们所说的bean的实例化，注入，解决循环依赖，回调beanPostProcessor等操作都是再这里实现的！
      finishBeanFactoryInitialization(beanFactory);

      // Last step: publish corresponding event.
      //最后一步：发布相应的事件。Spring内置的事件
      finishRefresh();
    }

    catch (BeansException ex) {
      if (logger.isWarnEnabled()) {
        logger.warn("Exception encountered during context initialization - " +
            "cancelling refresh attempt: " + ex);
      }

      // Destroy already created singletons to avoid dangling resources.
      destroyBeans();

      // Reset 'active' flag.
      cancelRefresh(ex);

      // Propagate exception to caller.
      throw ex;
    }

    finally {
      // Reset common introspection caches in Spring's core, since we
      // might not ever need metadata for singleton beans anymore...
      resetCommonCaches();
    }
  }
}
```

图解

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/27/20200727174111.png)


## prepareRefresh()

```java
protected void prepareRefresh() {
  // Switch to active.
  // 将容器激活

  // 设置启动时间
  this.startupDate = System.currentTimeMillis();
  // 设置关闭状态
  this.closed.set(false);
  // 设置激活状态
  this.active.set(true);

  if (logger.isDebugEnabled()) {
    if (logger.isTraceEnabled()) {
      logger.trace("Refreshing " + this);
    }
    else {
      logger.debug("Refreshing " + getDisplayName());
    }
  }

  // Initialize any placeholder property sources in the context environment.
  // 子类自定义个性化的属性设置方法
  initPropertySources();

  // Validate that all properties marked as required are resolvable:
  // see ConfigurablePropertyResolver#setRequiredProperties

  // 校验属性的合法
  getEnvironment().validateRequiredProperties();

  // 保存早期的事件
  // Store pre-refresh ApplicationListeners...
  if (this.earlyApplicationListeners == null) {
    this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
  }
  else {
    // Reset local application listeners to pre-refresh state.
    this.applicationListeners.clear();
    this.applicationListeners.addAll(this.earlyApplicationListeners);
  }

  // Allow for the collection of early ApplicationEvents,
  // to be published once the multicaster is available...
  this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

## obtainFreshBeanFactory()

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
  // 刷新 BeanFactory
  refreshBeanFactory();
  return getBeanFactory();
}
```

GenericApplicationContext#refreshBeanFactory()

```java
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
  if (!this.refreshed.compareAndSet(false, true)) {
    throw new IllegalStateException(
        "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
  }
  // 为 Bean工厂设置一个序列化Id
  this.beanFactory.setSerializationId(getId());
}
```

## prepareBeanFactory(beanFactory)

```java
/**
  * Configure the factory's standard context characteristics,
  * such as the context's ClassLoader and post-processors.
  * @param beanFactory the BeanFactory to configure
  */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  // Tell the internal bean factory to use the context's class loader etc.

  // 设置 类加载器
  beanFactory.setBeanClassLoader(getClassLoader());
  if (!shouldIgnoreSpel) {
    // spel 表达式的解析器
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
  }
  beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

  // Configure the bean factory with context callbacks.
  // 添加 ApplicationContextAwareProcessor ，即 BeanPostProcessor
  beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

  // 设置忽略自动专配的接口，这些接口的实现类不能通过接口来自动注入
  beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
  beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
  beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
  beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
  beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
  beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

  // BeanFactory interface not registered as resolvable type in a plain factory.
  // MessageSource registered (and found for autowiring) as a bean.

  // 注册可以解析的自动装配，即可以自动注入
  beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
  beanFactory.registerResolvableDependency(ResourceLoader.class, this);
  beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
  beanFactory.registerResolvableDependency(ApplicationContext.class, this);

  // Register early post-processor for detecting inner beans as ApplicationListeners.
  // 添加 BeanPostProcessor ApplicationListenerDetector
  beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

  // 添加编译时的Aspectj依赖
  // Detect a LoadTimeWeaver and prepare for weaving, if found.
  if (!IN_NATIVE_IMAGE && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    // Set a temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
  }

  // Register default environment beans.

  // 给 BeanFactory 中注入一些组件
  if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
    // 注册 environment：： ConfigurableEnvironment
    beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
  }
  if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
    // 注册 systemProperties ：： Map
    beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
  }
  if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
    // 注册 systemEnvironment：：Map
    beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
  }
}
```

## postProcessBeanFactory(beanFactory)

AbstractApplicationContext#postProcessBeanFactory(beanFactory)是一个空方法，子类可以做一些特定的拓展

## invokeBeanFactoryPostProcessors(beanFactory)

```java
/**
  * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
  * respecting explicit order if given.
  * <p>Must be called before singleton instantiation.
  */
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  // 拿到所有的 BeanFactoryPostProcessors ，并调用 PostProcessor 注册委托类执行
  PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

  // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
  // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
  if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
  }
}
```

PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory, List<BeanFactoryPostProcessor>)

```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

  // Invoke BeanDefinitionRegistryPostProcessors first, if any.
  // 用来保存已经执行过的后置处理器，避免重复执行
  Set<String> processedBeans = new HashSet<>();

  // 先判断 beanFactory 是不是 BeanDefinitionRegistry
  if (beanFactory instanceof BeanDefinitionRegistry) {
    BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;

    // 这个 List 是用来保存是 BeanFactoryPostProcessor 但不是 BeanDefinitionRegistryPostProcessor 的对象
    List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
    // 这个 List 是用来保存是 BeanDefinitionRegistryPostProcessor 但不是 BeanFactoryPostProcessor 的后置处理器对象
    List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

    // 容器初始化调用时，beanFactoryPostProcessors 应该是空的
    // 遍历所有的 beanFactoryPostProcessors
    for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
      if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {

        // 如果是 BeanDefinitionRegistryPostProcessor 就先执行 BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
        // 并把它加入到 registryProcessors 保存起来
        // 先执行是为了 自定义 实现的 BeanDefinitionRegistryPostProcessor 会被最先执行
        BeanDefinitionRegistryPostProcessor registryProcessor =
            (BeanDefinitionRegistryPostProcessor) postProcessor;
        registryProcessor.postProcessBeanDefinitionRegistry(registry);
        registryProcessors.add(registryProcessor);
      }
      else {
        //如果不是 BeanDefinitionRegistryPostProcessor 把就代表是 BeanFactoryPostProcessor 先保存起来
        regularPostProcessors.add(postProcessor);
      }
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    // Separate between BeanDefinitionRegistryPostProcessors that implement
    // PriorityOrdered, Ordered, and the rest.
    List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

    // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
    // 首先 会先执行 实现了 PriorityOrdered 优先级排序接口的 BeanDefinitionRegistryPostProcessors

    // 先从 容器中获取 所有的 BeanDefinitionRegistryPostProcessor 的 BeanDefinition 对象
    String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);

    for (String ppName : postProcessorNames) {
      // 如果是 PriorityOrdered
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        // 供容器中获取Bean，没有就会创建，然后加到 currentRegistryProcessors 中
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }

    // 先对 currentRegistryProcessors 进行排序
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    // 添加到 BeanDefinitionRegistryPostProcessor 集合中
    registryProcessors.addAll(currentRegistryProcessors);
    // 然后执行
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    //重要!!!!! 在这个过程中有一个非常重要的类 ConfigurationClassPostProcessor, 他实现了 BeanDefinitionRegistryPostProcessor 和 PriorityOrdered ，会被最先执行，他完成了全部的包扫描然后注册进入 BeanDefinition 中
    currentRegistryProcessors.clear();

    // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
    // 然后执行实现了 Order 顺序接口的 BeanDefinitionRegistryPostProcessor, 步骤和上面 PriorityOrdered 的一摸一样
    postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    currentRegistryProcessors.clear();

    // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
    // 再然后执行没有实现排序接口的剩下的所有  BeanDefinitionRegistryPostProcessor, 步骤与上述一致
    boolean reiterate = true;
    while (reiterate) {
      reiterate = false;
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
        if (!processedBeans.contains(ppName)) {
          currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
          processedBeans.add(ppName);
          reiterate = true;
        }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
      currentRegistryProcessors.clear();
    }

    // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
    // 执行 BeanDefinitionRegistryPostProcessor 接口中属于 BeanFactoryPostProcessor 的方法，因为 BeanDefinitionRegistryPostProcessor 继承了 BeanFactoryPostProcessor
    invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
    // 执行自定义的 BeanFactoryPostProcessor
    invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
  }

  else {
    // Invoke factory processors registered with the context instance.
    // 如果说 BeanFactory 没有实现 BeanDefinitionRegistry，那么它并不具有注册 BeanDefinition 的能力，所以只需要执行 BeanFactoryPostProcessor 即可
    invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
  }

  // Do not initialize FactoryBeans here: We need to leave all regular beans
  // uninitialized to let the bean factory post-processors apply to them!

  // BeanDefinitionRegistryPostProcessor 执行完成之后，再去执行剩下的 BeanFactoryPostProcessor，执行步骤和上面的一摸一样

  String[] postProcessorNames =
      beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

  // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
  // Ordered, and the rest.
  List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
  List<String> orderedPostProcessorNames = new ArrayList<>();
  List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  for (String ppName : postProcessorNames) {
    if (processedBeans.contains(ppName)) {
      // skip - already processed in first phase above
      // 因为 BeanDefinitionRegistryPostProcessor 是 BeanFactoryPostProcessor 的子接口
      // 所以当从容器中获取 BeanFactoryPostProcessor 的 BeanDefinition 对象时一定会有实现 BeanDefinitionRegistryPostProcessor 的对象
      //但是 BeanDefinitionRegistryPostProcessor 再最开始已经执行过了，所以需要直接跳过
    }
    else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
      priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
      orderedPostProcessorNames.add(ppName);
    }
    else {
      nonOrderedPostProcessorNames.add(ppName);
    }
  }

  // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
  sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

  // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
  List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
  for (String postProcessorName : orderedPostProcessorNames) {
    orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  sortPostProcessors(orderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

  // Finally, invoke all other BeanFactoryPostProcessors.
  List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
  for (String postProcessorName : nonOrderedPostProcessorNames) {
    nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

  // Clear cached merged bean definitions since the post-processors might have
  // modified the original metadata, e.g. replacing placeholders in values...
  beanFactory.clearMetadataCache();
}
```

```java
/**
  * Invoke the given BeanDefinitionRegistryPostProcessor beans.
  */
private static void invokeBeanDefinitionRegistryPostProcessors(
    Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

  for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
    postProcessor.postProcessBeanDefinitionRegistry(registry);
  }
}

/**
  * Invoke the given BeanFactoryPostProcessor beans.
  */
private static void invokeBeanFactoryPostProcessors(
    Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

  for (BeanFactoryPostProcessor postProcessor : postProcessors) {
    postProcessor.postProcessBeanFactory(beanFactory);
  }
}
```

在 `BeanDefinitionRegistryPostProcessor` 的实现类中，有一个类 `ConfigurationClassPostProcessor` 特别重要，他做了

- 解析 `@Configuration`
- 解析 `@ComponentScan`
- 解析 `@Import`
- ...等等重要的事情

以后会重点解析这个类

## registerBeanPostProcessors(beanFactory)

再一次强调一下，这里只是只是将处理器注册进来bean工厂，但是并未执行

```java
/**
  * Instantiate and register all BeanPostProcessor beans,
  * respecting explicit order if given.
  * <p>Must be called before any instantiation of application beans.
  */
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  // 调用后置处理器委托类进行注册 BeanPostProcessor
  PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

PostProcessorRegistrationDelegate#registerBeanPostProcessors(ConfigurableListableBeanFactory, AbstractApplicationContext)

```java
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

  // 获取所有的 BeanPostProcessor 的 beanDefinition 对象
  // 先注册 PriorityOrdered 优先级接口，再注册 Ordered 排序接口的，最后注册没有实现任何优先级接口的，最后注册 MergedBeanDefinitionPostProcessor
  // 注册就是 调用 beanFactory.addBeanPostProcessor(postProcessor)

  String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

  // Register BeanPostProcessorChecker that logs an info message when
  // a bean is created during BeanPostProcessor instantiation, i.e. when
  // a bean is not eligible for getting processed by all BeanPostProcessors.
  int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
  beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

  // Separate between BeanPostProcessors that implement PriorityOrdered,
  // Ordered, and the rest.
  List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
  List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
  List<String> orderedPostProcessorNames = new ArrayList<>();
  List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      priorityOrderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
        internalPostProcessors.add(pp);
      }
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
      orderedPostProcessorNames.add(ppName);
    }
    else {
      nonOrderedPostProcessorNames.add(ppName);
    }
  }

  // First, register the BeanPostProcessors that implement PriorityOrdered.
  sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

  // Next, register the BeanPostProcessors that implement Ordered.
  List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
  for (String ppName : orderedPostProcessorNames) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    orderedPostProcessors.add(pp);
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      internalPostProcessors.add(pp);
    }
  }
  sortPostProcessors(orderedPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, orderedPostProcessors);

  // Now, register all regular BeanPostProcessors.
  List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
  for (String ppName : nonOrderedPostProcessorNames) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    nonOrderedPostProcessors.add(pp);
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      internalPostProcessors.add(pp);
    }
  }
  registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

  // Finally, re-register all internal BeanPostProcessors.
  sortPostProcessors(internalPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, internalPostProcessors);

  // Re-register post-processor for detecting inner beans as ApplicationListeners,
  // moving it to the end of the processor chain (for picking up proxies etc).

  //最后注册一个 ApplicationListenerDetector 来在 Bean 创建完成后检查是否是 ApplicationListener ,如果是，就向 applicationContext 中添加一个 监听器
  beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

```java
/**
  * Register the given BeanPostProcessor beans.
  */
private static void registerBeanPostProcessors(
    ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {

  if (beanFactory instanceof AbstractBeanFactory) {
    // Bulk addition is more efficient against our CopyOnWriteArrayList there
    ((AbstractBeanFactory) beanFactory).addBeanPostProcessors(postProcessors);
  }
  else {
    for (BeanPostProcessor postProcessor : postProcessors) {
      beanFactory.addBeanPostProcessor(postProcessor);
    }
  }
}
```

## initMessageSource()

```java
/**
  * Initialize the MessageSource.
  * Use parent's if none defined in this context.
  */
protected void initMessageSource() {
  ConfigurableListableBeanFactory beanFactory = getBeanFactory();
  if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
    // 如果有 ，赋值给 MessageSource 属性
    this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
    // Make MessageSource aware of parent MessageSource.
    if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
      HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
      if (hms.getParentMessageSource() == null) {
        // Only set parent context as parent MessageSource if no parent MessageSource
        // registered already.
        hms.setParentMessageSource(getInternalParentMessageSource());
      }
    }
    if (logger.isTraceEnabled()) {
      logger.trace("Using MessageSource [" + this.messageSource + "]");
    }
  }
  else {
    // 没有创建一个 MessageSource, 并注册入容器中
    // Use empty MessageSource to be able to accept getMessage calls.
    DelegatingMessageSource dms = new DelegatingMessageSource();
    dms.setParentMessageSource(getInternalParentMessageSource());
    this.messageSource = dms;
    beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
    if (logger.isTraceEnabled()) {
      logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
    }
  }
}
```

## initApplicationEventMulticaster()

```java
/**
  * Initialize the ApplicationEventMulticaster.
  * Uses SimpleApplicationEventMulticaster if none defined in the context.
  * @see org.springframework.context.event.SimpleApplicationEventMulticaster
  */
protected void initApplicationEventMulticaster() {
  // 获取 BeanFactory
  // 判断容器中是否有 applicationEventMulticaster 组件，有就取出，没有就新建一个 SimpleApplicationEventMulticaster 并注册进容器中
  ConfigurableListableBeanFactory beanFactory = getBeanFactory();
  if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
    this.applicationEventMulticaster =
        beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
    if (logger.isTraceEnabled()) {
      logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
    }
  }
  else {
    this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
    beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
    if (logger.isTraceEnabled()) {
      logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
          "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
    }
  }
}
```

## onRefresh()方法

仍然是个空方法，供子类拓展实现

## registerListeners()

```java
/**
  * Add beans that implement ApplicationListener as listeners.
  * Doesn't affect other listeners, which can be added without being beans.
  */
protected void registerListeners() {

  // 先注册 指定的 Listener
  // Register statically specified listeners first.
  for (ApplicationListener<?> listener : getApplicationListeners()) {
    getApplicationEventMulticaster().addApplicationListener(listener);
  }

  // Do not initialize FactoryBeans here: We need to leave all regular beans
  // uninitialized to let post-processors apply to them!

  // 从容器内获取 所有 ApplicationListener 的 beanDefinition ，把他们添加到派发器中
  String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
  for (String listenerBeanName : listenerBeanNames) {
    getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
  }

  // Publish early application events now that we finally have a multicaster...
  // 发布早期事件
  Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
  this.earlyApplicationEvents = null;
  if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
    for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
      getApplicationEventMulticaster().multicastEvent(earlyEvent);
    }
  }
}
```

## finishBeanFactoryInitialization(beanFactory)

这个方法是一个重点，他是为了实例化所有剩余的（非延迟初始化）单例

```java
/**
  * Finish the initialization of this context's bean factory,
  * initializing all remaining singleton beans.
  */
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
  // Initialize conversion service for this context.
  // 实例化 类型转换组件
  if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
      beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
    beanFactory.setConversionService(
        beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
  }

  // Register a default embedded value resolver if no bean post-processor
  // (such as a PropertyPlaceholderConfigurer bean) registered any before:
  // at this point, primarily for resolution in annotation attribute values.
  // 实例化值解析器
  if (!beanFactory.hasEmbeddedValueResolver()) {
    beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
  }

  // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
  String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
  for (String weaverAwareName : weaverAwareNames) {
    getBean(weaverAwareName);
  }

  // Stop using the temporary ClassLoader for type matching.
  beanFactory.setTempClassLoader(null);

  // Allow for caching all bean definition metadata, not expecting further changes.
  beanFactory.freezeConfiguration();

  // Instantiate all remaining (non-lazy-init) singletons.
  // 初始化剩下的所有的单实例Bean
  beanFactory.preInstantiateSingletons();
}
```

会调用 DefaultListableBeanFactory#preInstantiateSingletons 方法

```java
@Override
public void preInstantiateSingletons() throws BeansException {
  if (logger.isTraceEnabled()) {
    logger.trace("Pre-instantiating singletons in " + this);
  }

  // Iterate over a copy to allow for init methods which in turn register new bean definitions.
  // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
  // 将所有的BeanDefinition的name信息加载到一个副本中
  List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

  // 遍历所有的 BeanDefinition 的 name
  // Trigger initialization of all non-lazy singleton beans...
  for (String beanName : beanNames) {
    // 获取 BeanDefinition
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);

    // 如果不是 抽象的 并且是 单例的 并且 不是懒加载的
    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {

      // 当前 bean 是不是 工厂方法
      if (isFactoryBean(beanName)) {
        Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
        if (bean instanceof FactoryBean) {
          FactoryBean<?> factory = (FactoryBean<?>) bean;
          boolean isEagerInit;
          if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
            isEagerInit = AccessController.doPrivileged(
                (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                getAccessControlContext());
          }
          else {
            isEagerInit = (factory instanceof SmartFactoryBean &&
                ((SmartFactoryBean<?>) factory).isEagerInit());
          }
          if (isEagerInit) {
            // 判断 工厂方法 需不需要马上初始化，如果是 getBean
            getBean(beanName);
          }
        }
      }
      else {
        // 不是工厂方法，直接 getBean
        getBean(beanName);
      }
    }
  }

  // 所有Bean都利用getBean创建完成之后
  // 检查Bean是不是 SmartInitializingSingleton 接口，如果是 就执行 afterSingletonsInstantiated 方法
  // Trigger post-initialization callback for all applicable beans...
  // 触发 Bean 初始化 之后 回调
  for (String beanName : beanNames) {
    Object singletonInstance = getSingleton(beanName);
    if (singletonInstance instanceof SmartInitializingSingleton) {
      SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
      if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
          smartSingleton.afterSingletonsInstantiated();
          return null;
        }, getAccessControlContext());
      }
      else {
        smartSingleton.afterSingletonsInstantiated();
      }
    }
  }
}
```

由此可以看出 getBean()方法是创建 Bean 的核心方法，bean的声明周期的各项回调以及BeanPostProcessor的调用均在此处

## finishRefresh()

```java
/**
  * Finish the refresh of this context, invoking the LifecycleProcessor's
  * onRefresh() method and publishing the
  * {@link org.springframework.context.event.ContextRefreshedEvent}.
  */
protected void finishRefresh() {
  // Clear context-level resource caches (such as ASM metadata from scanning).
  clearResourceCaches();

  // Initialize lifecycle processor for this context.
  // 初始化生命周期有关的处理器
  initLifecycleProcessor();

  // Propagate refresh to lifecycle processor first.
  // 拿到 上一步定义的生命周期处理器，回调 onRefresh()方法
  getLifecycleProcessor().onRefresh();

  // Publish the final event.
  // 发布容器刷新完成事件
  publishEvent(new ContextRefreshedEvent(this));

  // Participate in LiveBeansView MBean, if active.
  if (!IN_NATIVE_IMAGE) {
    LiveBeansView.registerApplicationContext(this);
  }
}
```

```java
/**
  * Initialize the LifecycleProcessor.
  * Uses DefaultLifecycleProcessor if none defined in the context.
  * @see org.springframework.context.support.DefaultLifecycleProcessor
  */
protected void initLifecycleProcessor() {

  // 从容器中找是否有生命周期组件，有就赋值，没有就新建一个 DefaultLifecycleProcessor ,加入到容器中

  ConfigurableListableBeanFactory beanFactory = getBeanFactory();
  if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
    this.lifecycleProcessor =
        beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
    if (logger.isTraceEnabled()) {
      logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
    }
  }
  else {
    DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
    defaultProcessor.setBeanFactory(beanFactory);
    this.lifecycleProcessor = defaultProcessor;
    beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
    if (logger.isTraceEnabled()) {
      logger.trace("No '" + LIFECYCLE_PROCESSOR_BEAN_NAME + "' bean, using " +
          "[" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
    }
  }
}
```

至此，容器刷新完成