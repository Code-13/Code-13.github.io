---
title: Spring-framework源码分析（七）bean的生命周期
abbrlink: cf22b3ec
categories:
  - Spring-framework源码分析
date: 2020-07-29 11:54:09
tags:
  - 源码分析
  - Spring
  - Spring-framework
cover: 'https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720195606.jpg'
coverWidth: 1200
coverHeight: 750
---

实例化
填充属性
初始化
销毁

<!--more-->

# getBean

getBean(String name)

# doGetBean()

- name 就是bean名字
- requiredType 表示需要的类型，如果有类型，创建后会进行类型转换
- args 表示参数，也就是构造方法的参数
- typeCheckOnly 表示只是做检查，并不是真的要用，这个会影响一些逻辑

## doGetBean--1

```java
// 转换 Bean 的名称
String beanName = transformedBeanName(name);
Object bean;

// Eagerly check singleton cache for manually registered singletons.
// 从已经实例化好的缓存中获取单例bean，因为我们此前可能已经将其实例化了
Object sharedInstance = getSingleton(beanName);
// 如果缓存中可以拿到，说明已经实例化过了
if (sharedInstance != null && args == null) {
  if (logger.isTraceEnabled()) {
    if (isSingletonCurrentlyInCreation(beanName)) {
      logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
          "' that is not fully initialized yet - a consequence of a circular reference");
    }
    else {
      logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
    }
  }
  bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
```

`AbstractAutowireCapableBeanFactory`#`getObjectForBeanInstance`

```java
@Override
protected Object getObjectForBeanInstance(
    Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

  //如果有正在创建的bean要建立依赖关系

  String currentlyCreatedBean = this.currentlyCreatedBean.get();
  if (currentlyCreatedBean != null) {
    registerDependentBean(beanName, currentlyCreatedBean);
  }

  return super.getObjectForBeanInstance(beanInstance, name, beanName, mbd);
}
```

`AbstractBeanFactory`#`getObjectForBeanInstance`

```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

  // Don't let calling code try to dereference the factory if the bean isn't a factory.
  //是否是FactoryBean名字的前缀
  if (BeanFactoryUtils.isFactoryDereference(name)) {
    if (beanInstance instanceof NullBean) {
      return beanInstance;
    }
    if (!(beanInstance instanceof FactoryBean)) {
      //不是FactoryBean的话名字有&会报异常
      throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
    }
    if (mbd != null) {
      mbd.isFactoryBean = true;
    }
    return beanInstance;
  }

  // Now we have the bean instance, which may be a normal bean or a FactoryBean.
  // If it's a FactoryBean, we use it to create a bean instance, unless the
  // caller actually wants a reference to the factory.
  if (!(beanInstance instanceof FactoryBean)) {
    //不是FactoryBean就直接返回
    return beanInstance;
  }

  Object object = null;
  if (mbd != null) {
    mbd.isFactoryBean = true;
  }
  else {
    //从FactoryBean的缓存中获取
    object = getCachedObjectForFactoryBean(beanName);
  }
  if (object == null) {
    // Return bean instance from factory.
    FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
    // Caches object obtained from FactoryBean if it is a singleton.
    if (mbd == null && containsBeanDefinition(beanName)) {
      mbd = getMergedLocalBeanDefinition(beanName);
    }
    boolean synthetic = (mbd != null && mbd.isSynthetic());
    object = getObjectFromFactoryBean(factory, beanName, !synthetic);
  }
  return object;
}
```

`getCachedObjectForFactoryBean`

```java
@Nullable
protected Object getCachedObjectForFactoryBean(String beanName) {
  return this.factoryBeanObjectCache.get(beanName);
}
```

FactoryBeanRegistrySupport#getObjectFromFactoryBean

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
  if (factory.isSingleton() && containsSingleton(beanName)) {
    synchronized (getSingletonMutex()) {
      Object object = this.factoryBeanObjectCache.get(beanName);
      if (object == null) {
        object = doGetObjectFromFactoryBean(factory, beanName);
        // Only post-process and store if not put there already during getObject() call above
        // (e.g. because of circular reference processing triggered by custom getBean calls)
        Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
        if (alreadyThere != null) {
          object = alreadyThere;
        }
        else {
          if (shouldPostProcess) {
            if (isSingletonCurrentlyInCreation(beanName)) {
              // Temporarily return non-post-processed object, not storing it yet..
              return object;
            }
            beforeSingletonCreation(beanName);
            try {
              //进行后置处理器处理
              object = postProcessObjectFromFactoryBean(object, beanName);
            }
            catch (Throwable ex) {
              throw new BeanCreationException(beanName,
                  "Post-processing of FactoryBean's singleton object failed", ex);
            }
            finally {
              afterSingletonCreation(beanName);
            }
          }
          if (containsSingleton(beanName)) {
            //如果包含了FactoryBean，就将创建的对象缓存
            this.factoryBeanObjectCache.put(beanName, object);
          }
        }
      }
      return object;
    }
  }
  else {
    //FactoryBean是原型的话
    Object object = doGetObjectFromFactoryBean(factory, beanName);
    if (shouldPostProcess) {
      try {
        object = postProcessObjectFromFactoryBean(object, beanName);
      }
      catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
      }
    }
    return object;
  }
}
```

`doGetObjectFromFactoryBean`

就是调用 `getObject()` 创建对象，如果返回null的话，就封装成一个 `NullBean`

```java
private Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName) throws BeanCreationException {
  Object object;
  try {
    if (System.getSecurityManager() != null) {
      AccessControlContext acc = getAccessControlContext();
      try {
        object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
      }
      catch (PrivilegedActionException pae) {
        throw pae.getException();
      }
    }
    else {
      object = factory.getObject();
    }
  }
  catch (FactoryBeanNotInitializedException ex) {
    throw new BeanCurrentlyInCreationException(beanName, ex.toString());
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
  }

  // Do not accept a null value for a FactoryBean that's not fully
  // initialized yet: Many FactoryBeans just return null then.
  if (object == null) {
    if (isSingletonCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(
          beanName, "FactoryBean which is currently in creation returned null from getObject");
    }
    object = new NullBean();
  }
  return object;
}
```

`beforeSingletonCreation` 和 `afterSingletonCreation`

这个就是标记下，正在创建这个bean，创建处理完了就清除标记。

```java
protected void beforeSingletonCreation(String beanName) {
  if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);//没有在排除范围里内且添加不成功，可能就是循环引用了
  }
}

protected void afterSingletonCreation(String beanName) {
	if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
		throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
	}
}
```

`postProcessObjectFromFactoryBean`

就是进行 `BeanPostProcessor` 的 `postProcessAfterInitialization` 处理，也就是可以扩展的地方。

```java
@Override
protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
  return applyBeanPostProcessorsAfterInitialization(object, beanName);
}
```

```java
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {

  Object result = existingBean;
  for (BeanPostProcessor processor : getBeanPostProcessors()) {
    Object current = processor.postProcessAfterInitialization(result, beanName);
    if (current == null) {
      return result;
    }
    result = current;
  }
  return result;
}
```

## doGetBean--2

- 判断是不是正在创建的原型类型，原型类型无法解决循环应用问题，会报异常。
- 原型的定义就是每次都是需要新的对象，如果A是原型，引用B，这个时候B也是要被创建的，如果B是原型，也引用A，那A也要被重新创建，然后A里面又要引用B，又重新创建，这样下去就变成创建的死循环了，所以原型循环引用不行
- 如果发现父工厂存在， `bean` 定义不存在，就会从父工厂去找，让父工厂去创建。这种情况比较少

```java
// Fail if we're already creating this bean instance:
// We're assumably within a circular reference.
//原型的循环引用报错
if (isPrototypeCurrentlyInCreation(beanName)) {
  throw new BeanCurrentlyInCreationException(beanName);
}

// Check if bean definition exists in this factory. 如果bean定义不存在，就检查父工厂是否有
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
  // Not found -> check parent.
  String nameToLookup = originalBeanName(name);
  if (parentBeanFactory instanceof AbstractBeanFactory) {
    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
        nameToLookup, requiredType, args, typeCheckOnly);
  }
  else if (args != null) {
    // Delegation to parent with explicit args.
    return (T) parentBeanFactory.getBean(nameToLookup, args);
  }
  else if (requiredType != null) {
    // No args -> delegate to standard getBean method.
    return parentBeanFactory.getBean(nameToLookup, requiredType);
  }
  else {
    return (T) parentBeanFactory.getBean(nameToLookup);
  }
}
```

## doGetBean--3

- 先判断是否仅仅是检查类型，不是就标记为已经创建，并设置需要 bean 定义合并标记
- 处理 `dependsOn` 依赖

```java
if (!typeCheckOnly) {
  // 将Bean标记为已经创建
  markBeanAsCreated(beanName);
}

try {
  // 获取合并过的BeanDefinition, BeanDefinition.getParentName如果不为null，需要先获取，然后合并BeanDefinition
  RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
  // 检查合并过的BeanDefinition
  checkMergedBeanDefinition(mbd, beanName, args);

  /**
    * 先查看 bean 有没有依赖项，有就先创建依赖项
    */
  // Guarantee initialization of beans that the current bean depends on.
  String[] dependsOn = mbd.getDependsOn();
  if (dependsOn != null) {
    for (String dep : dependsOn) {
      if (isDependent(beanName, dep)) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
      }
      registerDependentBean(dep, beanName);
      try {
        getBean(dep);
      }
      catch (NoSuchBeanDefinitionException ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
      }
    }
  }
```

markBeanAsCreated

```java
protected void markBeanAsCreated(String beanName) {
  if (!this.alreadyCreated.contains(beanName)) {
    synchronized (this.mergedBeanDefinitions) {
      //需要重新获取合并一下bean定义，以免元数据被同时修改
      if (!this.alreadyCreated.contains(beanName)) {
        // Let the bean definition get re-merged now that we're actually creating
        // the bean... just in case some of its metadata changed in the meantime.
        clearMergedBeanDefinition(beanName);
        this.alreadyCreated.add(beanName);
      }
    }
  }
}
```

```java
protected void clearMergedBeanDefinition(String beanName) {
  RootBeanDefinition bd = this.mergedBeanDefinitions.get(beanName);
  if (bd != null) {
    bd.stale = true;//需要合并
  }
}
```

## doGetBean--4

上面就是 `Bean` 的创建

```java
// Create bean instance.
if (mbd.isSingleton()) {
  // 单例创建
  sharedInstance = getSingleton(beanName, () -> {
    try {
      return createBean(beanName, mbd, args);
    }
    catch (BeansException ex) {
      // Explicitly remove instance from singleton cache: It might have been put there
      // eagerly by the creation process, to allow for circular reference resolution.
      // Also remove any beans that received a temporary reference to the bean.
      destroySingleton(beanName);
      throw ex;
    }
  });
  bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

getSingleton(String beanName, ObjectFactory<?> singletonFactory)

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  Assert.notNull(beanName, "Bean name must not be null");
  synchronized (this.singletonObjects) {

    // singletonObjects 就是缓存的已经创建好的单例bean
    // 如果缓存中有，就直接返回；没有再创建

    Object singletonObject = this.singletonObjects.get(beanName);

    if (singletonObject == null) {
      if (this.singletonsCurrentlyInDestruction) {
        // 如果bean已经被销毁，就抛出异常
        throw new BeanCreationNotAllowedException(beanName,
            "Singleton bean creation not allowed while singletons of this factory are in destruction " +
            "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
      }
      if (logger.isDebugEnabled()) {
        logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
      }
      // 创建前做一个正在创建的标记
      beforeSingletonCreation(beanName);
      // 是否创建单例成功，感觉这个用处不大
      boolean newSingleton = false;
      boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
      if (recordSuppressedExceptions) {
        this.suppressedExceptions = new LinkedHashSet<>();
      }
      try {
        //调用ObjectFactory的getObject创建bean对象
        singletonObject = singletonFactory.getObject();
        //只要获取了就是新单例
        newSingleton = true;
      }
      catch (IllegalStateException ex) {
        // Has the singleton object implicitly appeared in the meantime ->
        // if yes, proceed with it since the exception indicates that state.
        singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
          throw ex;
        }
      }
      catch (BeanCreationException ex) {
        if (recordSuppressedExceptions) {
          for (Exception suppressedException : this.suppressedExceptions) {
            ex.addRelatedCause(suppressedException);
          }
        }
        throw ex;
      }
      finally {
        if (recordSuppressedExceptions) {
          this.suppressedExceptions = null;
        }
        afterSingletonCreation(beanName);
      }
      if (newSingleton) {
        //如果是新的单例，就加入到单例集合
        addSingleton(beanName, singletonObject);
      }
    }
    return singletonObject;
  }
}
```

`createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)`

- 解析类型
- 进行方法覆盖
- `InstantiationAwareBeanPostProcessor` 处理

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {

  if (logger.isTraceEnabled()) {
    logger.trace("Creating instance of bean '" + beanName + "'");
  }
  RootBeanDefinition mbdToUse = mbd;

  // Make sure bean class is actually resolved at this point, and
  // clone the bean definition in case of a dynamically resolved Class
  // which cannot be stored in the shared merged bean definition.

  //解析出bean的类型 ； 很复杂
  Class<?> resolvedClass = resolveBeanClass(mbd, beanName);

  //能解析出来，没有BeanClass只有BeanClassName
  if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    //重新创建RootBeanDefinition
    mbdToUse = new RootBeanDefinition(mbd);
    //设置解析的类型
    mbdToUse.setBeanClass(resolvedClass);
  }

  // Prepare method overrides.
  try {
    //准备方法覆盖，处理look-up方法
    mbdToUse.prepareMethodOverrides();
  }
  catch (BeanDefinitionValidationException ex) {
    throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
        beanName, "Validation of method overrides failed", ex);
  }

  try {
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.

    //后置处理器在实例化之前处理，如果返回不为null，直接就返回了
    // 此时的后置处理器是 InstantiationAwareBeanPostProcessor

    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
      return bean;
    }
  }
  catch (Throwable ex) {
    throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
        "BeanPostProcessor before instantiation of bean failed", ex);
  }

  try {
    //真正创建 bean
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isTraceEnabled()) {
      logger.trace("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
  }
  catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
    // A previously detected exception with proper bean creation context already,
    // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
    throw ex;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
        mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
  }
}
```

`resolveBeforeInstantiation` 实例化前处理

```java
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
  Object bean = null;
  if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    // Make sure bean class is actually resolved at this point.
    // 非合成的，且有 InstantiationAwareBeanPostProcessors
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      Class<?> targetType = determineTargetType(beanName, mbd);
      if (targetType != null) {
        bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
        if (bean != null) {
          bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
      }
    }
    mbd.beforeInstantiationResolved = (bean != null);
  }
  return bean;
}
```

`applyBeanPostProcessorsBeforeInstantiation` 实例化前处理器处理

```java
@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
  for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
    Object result = bp.postProcessBeforeInstantiation(beanClass, beanName);
    if (result != null) {
      return result;
    }
  }
  return null;
}
```

`applyBeanPostProcessorsAfterInitialization` 初始化后处理器处理

```java
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {

  Object result = existingBean;
  for (BeanPostProcessor processor : getBeanPostProcessors()) {
    Object current = processor.postProcessAfterInitialization(result, beanName);
    if (current == null) {
      return result;
    }
    result = current;
  }
  return result;
}
```

InstantiationAwareBeanPostProcessor 拓展点

- 不论前还是后只要返回就不会再创建

## doCreateBean

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {

  // Instantiate the bean.
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    //获取 factoryBean 实例缓存
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    // 没有 factoryBean 就创建
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  // 获取原始 bean
  Object bean = instanceWrapper.getWrappedInstance();
  Class<?> beanType = instanceWrapper.getWrappedClass();

  //不为空的bean
  if (beanType != NullBean.class) {
    mbd.resolvedTargetType = beanType;
  }

  // 处理器修改合并的 BeanDefinition 定义
  // Allow post-processors to modify the merged bean definition.
  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      try {
        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      }
      catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Post-processing of merged bean definition failed", ex);
      }
      mbd.postProcessed = true;
    }
  }

  // 提前暴露一个早期的单例工厂，以便解决循环引用

  // Eagerly cache singletons to be able to resolve circular references
  // even when triggered by lifecycle interfaces like BeanFactoryAware.
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
      logger.trace("Eagerly caching bean '" + beanName +
          "' to allow for resolving potential circular references");
    }
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  // 初始化阶段

  // Initialize the bean instance.
  Object exposedObject = bean;
  try {
    // 属性填充
    populateBean(beanName, mbd, instanceWrapper);
    // 初始化
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }
  catch (Throwable ex) {
    if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
      throw (BeanCreationException) ex;
    }
    else {
      throw new BeanCreationException(
          mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
    }
  }

  if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
      if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
      }
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
          if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
            actualDependentBeans.add(dependentBean);
          }
        }
        if (!actualDependentBeans.isEmpty()) {
          throw new BeanCurrentlyInCreationException(beanName,
              "Bean with name '" + beanName + "' has been injected into other beans [" +
              StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
              "] in its raw version as part of a circular reference, but has eventually been " +
              "wrapped. This means that said other beans do not use the final version of the " +
              "bean. This is often the result of over-eager type matching - consider using " +
              "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
        }
      }
    }
  }

  // Register bean as disposable.
  try {
    //注册可销毁的bean
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
  }
  catch (BeanDefinitionValidationException ex) {
    throw new BeanCreationException(
        mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
  }

  return exposedObject;
}
```

`createBeanInstance` 创建 `bean` 实例

- 获取类型
- 获取修饰符，只有 public 才允许创建
- 获取自定义的实例提供器，如果有的话直接获取返回
- 如果有工厂方法名字，使用工厂方法创建
- 推断构造函数
  - 找出合适的构造函数进行自动装配
- 以上都不行，使用默认的无参构造函数进行实例化

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
  // Make sure bean class is actually resolved at this point.
  // 获取类型
  Class<?> beanClass = resolveBeanClass(mbd, beanName);

  // 检查类 是不是 public 修饰的
  if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
    throw new BeanCreationException(mbd.getResourceDescription(), beanName,
        "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
  }

  // 如果 mbd 有拓展的实例提供器，可以直接从里面获取实例
  Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
  if (instanceSupplier != null) {
    return obtainFromSupplier(instanceSupplier, beanName);
  }

  // 有工厂方法的，从工厂方法获取
  if (mbd.getFactoryMethodName() != null) {
    return instantiateUsingFactoryMethod(beanName, mbd, args);
  }

  // Shortcut when re-creating the same bean...
  //标记下，防止重复创建同一个bean
  boolean resolved = false;
  //是否需要自动装配，构造器有参数的需要
  boolean autowireNecessary = false;
  if (args == null) {
    // 无参
    synchronized (mbd.constructorArgumentLock) {
      if (mbd.resolvedConstructorOrFactoryMethod != null) {
        //有解析的构造器或者工厂方法
        resolved = true;
        autowireNecessary = mbd.constructorArgumentsResolved;
      }
    }
  }
  //有构造参数的或者工厂方法
  if (resolved) {
    //构造器有参数的
    if (autowireNecessary) {
      return autowireConstructor(beanName, mbd, null, null);
    }
    else {
      return instantiateBean(beanName, mbd);
    }
  }

  //从bean后置处理器中为自动装配寻找构造方法
  // Candidate constructors for autowiring?
  Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
  if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
      mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
    return autowireConstructor(beanName, mbd, ctors, args);
  }

  // 找出最合适的默认构造方法
  // Preferred constructors for default construction?
  ctors = mbd.getPreferredConstructors();
  if (ctors != null) {
    return autowireConstructor(beanName, mbd, ctors, null);
  }

  // 最后才用最简单的默认构造方法
  // No special handling: simply use no-arg constructor.
  return instantiateBean(beanName, mbd);
}
```

obtainFromSupplier实例提供器

这是又一个拓展点

```java
protected BeanWrapper obtainFromSupplier(Supplier<?> instanceSupplier, String beanName) {
  Object instance;

  String outerBean = this.currentlyCreatedBean.get();
  this.currentlyCreatedBean.set(beanName);
  try {
    instance = instanceSupplier.get();
  }
  finally {
    if (outerBean != null) {
      // 不为空还是这是老的
      this.currentlyCreatedBean.set(outerBean);
    }
    else {
      // 为空删除
      this.currentlyCreatedBean.remove();
    }
  }

  if (instance == null) {
    instance = new NullBean();
  }
  BeanWrapper bw = new BeanWrapperImpl(instance);
  initBeanWrapper(bw);
  return bw;
}
```
