---
title: Spring-framework源码分析（四）IOC容器初始化
abbrlink: f820fb69
categories:
  - Spring-framework源码分析
date: 2020-07-27 10:45:24
tags:
  - 源码分析
  - Spring
  - Spring-framework
cover: 'https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720195606.jpg'
coverWidth: 1200
coverHeight: 750
---

`Spring IOC` 容器 在初始化时主要做以下三件事：

- `BeanDefinition` 的Resource定位
- `BeanDefinition` 的载入和解析
- `BeanDefinition` 在容器中的注册

<!--more-->

## 准备工作

- 在搭建好的调试环境新建一个模块
- 然后写一个 配置类 `AppConfig`

```java
@ComponentScan("com.github.code13")
public class AppConfig {
}
```

- 新建一个 类 B,并加上 `@Component` 注解

```java
@Component
public class B {

	public B(){
		System.out.println("B创建了");
	}

}
```
- 新建 IOC 容器 

```java
final AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
```

本片文章均围绕着上面这行代码

## AnnotationConfigApplicationContext体系

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/21/20200721115537.png)

`AnnotationConfigApplicationContext` 继承自 `GenericApplicationContext`
`GenericApplicationContext` 继承自 `AbstractApplicationContext`
`AbstractApplicationContext` 继承自 `DefaultResourceLoader`

## AnnotationConfigApplicationContext 初始化

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
  // 调用默认无参构造器
  this();
  //注册配置类
  register(componentClasses);
  // 容器初始化的全过程
  refresh();
}
```

这三个方法其实都很重要

先看无参构造器：

```java
public AnnotationConfigApplicationContext() {
  //AnnotatedBeanDefinitionReader是一个读取注解的Bean读取器，这里将this传了进去。
  //会注册配置类的后置处理器
  this.reader = new AnnotatedBeanDefinitionReader(this);
  //扫描注解真正的核心再此，会注册一些默认的过滤器
  this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

### 父类加载与初始化

由于 `AnnotationConfigApplicationContext` 继承自 `GenericApplicationContext` ， 所以也会调用 `GenericApplicationContext` 的无参构造函数

```java
public GenericApplicationContext() {
  // 最终的Bean工厂
  this.beanFactory = new DefaultListableBeanFactory();
}
```

当然，在创建 `DefaultListableBeanFactory` 的过程中也有很多东西，这里就先掠过

由于 `GenericApplicationContext` 继承自 `AbstractApplicationContext` ， 所以也会调用 `AbstractApplicationContext` 的无参构造函数
`AbstractApplicationContext` 有很多的静态属性，还有一个静态代码块，不是很重要先跳过

```java
/** ResourcePatternResolver used by this context. */
private ResourcePatternResolver resourcePatternResolver;

...//省略其他代码

public AbstractApplicationContext() {
  this.resourcePatternResolver = getResourcePatternResolver();
}

...//省略其他代码

protected ResourcePatternResolver getResourcePatternResolver() {
  return new PathMatchingResourcePatternResolver(this);
}

```

`AbstractApplicationContext` 继承自 `DefaultResourceLoader` ，并且内部还有一个 `ResourcePatternResolver` 属性，具备了加载任意 `Resource` 的能力

`Resource` 体系 和 `ResourceLoader` 体系见 [Spring-framework源码分析（三）统一资源加载](https://code-13.github.io/blogs/5eef8624.html)

### 无参构造器

了解了 `AnnotationConfigApplicationContext` 的父类初始化做了什么之后，再回来看 `AnnotationConfigApplicationContext` 的无参构造器

```java
public AnnotationConfigApplicationContext() {
  //AnnotatedBeanDefinitionReader是一个读取注解的Bean读取器，这里将this传了进去。
  //会注册配置类的后置处理器
  this.reader = new AnnotatedBeanDefinitionReader(this);
  //扫描注解真正的核心再此，会注册一些默认的过滤器
  this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

在这里初始化了 `AnnotatedBeanDefinitionReader` 以及 `ClassPathBeanDefinitionScanner`，这两个类的作用都是为了注册BeanDefinition

#### AnnotatedBeanDefinitionReader

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
  // 创建 StandardEnvironment
  this(registry, getOrCreateEnvironment(registry));
}
```

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
  Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
  Assert.notNull(environment, "Environment must not be null");
  this.registry = registry;

  // ConditionEvaluator 用于对 @Conditional 及其派生 注解进行处理
  this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);

  // 注册注解配置的处理器
  AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

`AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);` 这段代码预先注册了注解处理的处理器，如 `ConfigurationClassPostProcessor`(这个类相当重要) 等

`AnnotationConfigUtils#registerAnnotationConfigProcessors(BeanDefinitionRegistry)`

```java
public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
  registerAnnotationConfigProcessors(registry, null);
}
```

`AnnotationConfigUtils#registerAnnotationConfigProcessors(BeanDefinitionRegistry,Object)`

```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
    BeanDefinitionRegistry registry, @Nullable Object source) {

  // 拿一下 Bean 工厂
  DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
  if (beanFactory != null) {
    // 为BeanFactory 配置依赖比较器
    if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
      beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
    }
    // 为 BeanFactory 设置自动注入条件解析器
    if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
      beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
    }
  }

  // 定义返回的结果集合
  Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

  // 下面都是为 BeanFactory 提前注册一些很重要的各种各样的 后置处理器，核心方法都是 registerPostProcessor

  if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {

    // ConfigurationClassPostProcessor 用来扫描并注册所有的 BeanDefinition 非常重要

    RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
  if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
  if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition();
    try {
      def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
          AnnotationConfigUtils.class.getClassLoader()));
    }
    catch (ClassNotFoundException ex) {
      throw new IllegalStateException(
          "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
    }
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
  }

  if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
  }

  return beanDefs;
}
```

```java
private static BeanDefinitionHolder registerPostProcessor(
			BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

  definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
  // 把 beanDefinition 注册进 bean工厂
  registry.registerBeanDefinition(beanName, definition);
  return new BeanDefinitionHolder(definition, beanName);
}
```

最终会调用 `DefaultListableBeanFactory#registerBeanDefinition(String,BeanDefinition)`

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

  Assert.hasText(beanName, "Bean name must not be empty");
  Assert.notNull(beanDefinition, "BeanDefinition must not be null");

  if (beanDefinition instanceof AbstractBeanDefinition) {
    try {
      // 验证
      ((AbstractBeanDefinition) beanDefinition).validate();
    }
    catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
          "Validation of bean definition failed", ex);
    }
  }

  // BeanDefinition 最终会以 name 为 key BeanDefinition 为 value 存储在 beanDefinitionMap 中

  BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
  if (existingDefinition != null) {
    if (!isAllowBeanDefinitionOverriding()) {
      throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
    }
    else if (existingDefinition.getRole() < beanDefinition.getRole()) {
      // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
      if (logger.isInfoEnabled()) {
        logger.info("Overriding user-defined bean definition for bean '" + beanName +
            "' with a framework-generated bean definition: replacing [" +
            existingDefinition + "] with [" + beanDefinition + "]");
      }
    }
    else if (!beanDefinition.equals(existingDefinition)) {
      if (logger.isDebugEnabled()) {
        logger.debug("Overriding bean definition for bean '" + beanName +
            "' with a different definition: replacing [" + existingDefinition +
            "] with [" + beanDefinition + "]");
      }
    }
    else {
      if (logger.isTraceEnabled()) {
        logger.trace("Overriding bean definition for bean '" + beanName +
            "' with an equivalent definition: replacing [" + existingDefinition +
            "] with [" + beanDefinition + "]");
      }
    }
    this.beanDefinitionMap.put(beanName, beanDefinition);
  }
  else {
    if (hasBeanCreationStarted()) {
      // Cannot modify startup-time collection elements anymore (for stable iteration)
      synchronized (this.beanDefinitionMap) {
        this.beanDefinitionMap.put(beanName, beanDefinition);
        List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
        updatedDefinitions.addAll(this.beanDefinitionNames);
        updatedDefinitions.add(beanName);
        this.beanDefinitionNames = updatedDefinitions;
        removeManualSingletonName(beanName);
      }
    }
    else {
      // Still in startup registration phase
      this.beanDefinitionMap.put(beanName, beanDefinition);
      this.beanDefinitionNames.add(beanName);
      removeManualSingletonName(beanName);
    }
    this.frozenBeanDefinitionNames = null;
  }

  if (existingDefinition != null || containsSingleton(beanName)) {
    resetBeanDefinition(beanName);
  }
  else if (isConfigurationFrozen()) {
    clearByTypeCache();
  }
}
```

其实，所有的 `BeanDefinition` 都会被注册进 `beanDefinitionMap` 集合中，区别就是先后顺序
```java
/** Map of bean definition objects, keyed by bean name. */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```

AnnotatedBeanDefinitionReader初始化时会做以下工作

- 初始化一个 `ConditionEvaluator` 便于于对 `@Conditional` 及其派生注解进行后续处理
- 提前注册一些比较重要的后置处理器，以便于spring的后续解析，如 `ConfigurationClassPostProcessor`

#### ClassPathBeanDefinitionScanner

`ClassPathBeanDefinitionScanner` 的所有构造函数都会调用这个构造函数

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {

  Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
  this.registry = registry;

  if (useDefaultFilters) {
    // 注解的类过滤在这里 Component 等。此方法是父类 ClassPathScanningCandidateComponentProvider 提供的
    registerDefaultFilters();
  }
  setEnvironment(environment);
  setResourceLoader(resourceLoader);
}
```

ClassPathScanningCandidateComponentProvider#registerDefaultFilters()

```java
protected void registerDefaultFilters() {
  // 默认的过滤会留下带 @Component 注解的Bean。
  this.includeFilters.add(new AnnotationTypeFilter(Component.class));

  // 支持 JSR-250
  ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
  try {
    this.includeFilters.add(new AnnotationTypeFilter(
        ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
    logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
  }
  catch (ClassNotFoundException ex) {
    // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
  }

  // 支持 JSR-330
  try {
    this.includeFilters.add(new AnnotationTypeFilter(
        ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
    logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
  }
  catch (ClassNotFoundException ex) {
    // JSR-330 API not available - simply skip.
  }
}
```

ClassPathBeanDefinitionScanner 初始化时会做以下工作

- 添加 `@Component` ，`@ManagedBean`，`@Named` 包含过滤器，spring在后续的扫描中只注册被这些注解标注的类
- `@Controller` `@Service` `@Repository` 注解上面都标注了 `@Component` 注解，所以他们也可以被注册

至此，AnnotationConfigApplicationContext 的默认无参构造器执行完成

## 注册主配置类

为什么配置类是通过register(componentClasses)手动put到map当中呢？为什么不是扫描出来的呢？（一般类都是通过扫描出来的），其实也很简单，因为他无法扫描自己，一般类是spring通过解析配置类上的 `@ComponentScan` 注解然后被扫描到，所以无法扫描自己

```java
register(componentClasses);
```

```java
@Override
public void register(Class<?>... componentClasses) {
  Assert.notEmpty(componentClasses, "At least one component class must be specified");

  //使用初始化好的 AnnotatedBeanDefinitionReader 注册配置类
  this.reader.register(componentClasses);
}
```

AnnotatedBeanDefinitionReader#register

```java
public void register(Class<?>... componentClasses) {
  // 遍历所有的配置类，都将他们注册进去
  for (Class<?> componentClass : componentClasses) {
    registerBean(componentClass);
  }
}
```

AnnotatedBeanDefinitionReader#registerBean

```java
public void registerBean(Class<?> beanClass, @Nullable String name) {
  doRegisterBean(beanClass, name, null, null, null);
}
```

最后都会调用到全参的 `doRegisterBean` 方法

```java
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers) {

  // 创建一个 BeanDefinition
  AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);

  // 调用条件解析器判断当前类是否应该跳过，即对 @Conditional 注解的处理
  if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
    return;
  }

  // 为 BeanDefinition 设置一些属性

  abd.setInstanceSupplier(supplier);

  // Scope 作用域信息，如Singleton
  ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
  abd.setScope(scopeMetadata.getScopeName());

  // 生成 bean 的 name
  String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

  // 为 BeanDefinition 设置一些其他的属性如 Lazy，DependsOn， Primary
  AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);

  if (qualifiers != null) {
    for (Class<? extends Annotation> qualifier : qualifiers) {
      if (Primary.class == qualifier) {
        abd.setPrimary(true);
      }
      else if (Lazy.class == qualifier) {
        abd.setLazyInit(true);
      }
      else {
        abd.addQualifier(new AutowireCandidateQualifier(qualifier));
      }
    }
  }

  if (customizers != null) {
    for (BeanDefinitionCustomizer customizer : customizers) {
      customizer.customize(abd);
    }
  }

  BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);

  // 设置 类的代理类型，即是否使用代理，如果使用代理是代理接口还是类
  definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);

  // 注册 BeanDefinition
  BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

BeanDefinitionReaderUtils#registerBeanDefinition(BeanDefinitionHolder,BeanDefinitionRegistry)

```java
public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

  // Register bean definition under primary name.
  String beanName = definitionHolder.getBeanName();
  registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

  // Register aliases for bean name, if any.
  String[] aliases = definitionHolder.getAliases();
  if (aliases != null) {
    for (String alias : aliases) {
      registry.registerAlias(beanName, alias);
    }
  }
}
```

最终会调用 `DefaultListableBeanFactory#registerBeanDefinition(String,BeanDefinition)`，和上面注册一些提前的后置处理器一样

注册主配置类，就是将主配置类注册成 `BeanDefinition`

## refresh()方法

<font color="red">
refresh方法是Spring容器最核心的方法,概括了Spring初始化bean的整个生命周期，包括大家大概熟悉的一些生命周期的回调，以及AOP或者很多大神都分析过的Spring解决循环依赖的三级缓存
</font>

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

下章会详解refresh()方法
