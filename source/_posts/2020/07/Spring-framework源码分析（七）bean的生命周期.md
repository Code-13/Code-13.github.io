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

``` java
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

``` java
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

``` java
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

``` java
@Nullable
protected Object getCachedObjectForFactoryBean(String beanName) {
  return this.factoryBeanObjectCache.get(beanName);
}
```

FactoryBeanRegistrySupport#getObjectFromFactoryBean

``` java
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

``` java
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

``` java
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

``` java
@Override
protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
  return applyBeanPostProcessorsAfterInitialization(object, beanName);
}
```

``` java
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

``` java
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

``` java
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

``` java
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

``` java
protected void clearMergedBeanDefinition(String beanName) {
  RootBeanDefinition bd = this.mergedBeanDefinitions.get(beanName);
  if (bd != null) {
    bd.stale = true;//需要合并
  }
}
```

## doGetBean--4

下面就是 `Bean` 的创建

``` java
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

``` java
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

``` java
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

``` java
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

``` java
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

``` java
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

``` java
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

### `createBeanInstance` 创建 `bean` 实例

- 获取类型
- 获取修饰符，只有 public 才允许创建
- 获取自定义的实例提供器，如果有的话直接获取返回
- 如果有工厂方法名字，使用工厂方法创建
- 推断构造函数
  - 找出合适的构造函数进行自动装配
- 以上都不行，使用默认的无参构造函数进行实例化

``` java
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

#### obtainFromSupplier 实例提供器

这是又一个拓展点

``` java
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

#### instantiateUsingFactoryMethod 工厂方法实例化

``` java
protected BeanWrapper instantiateUsingFactoryMethod(
    String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {

  return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
}
```

ConstructorResolver

保存 BeanFactory ，然后利用 BeanFactory 来做一些事情

``` java
public ConstructorResolver(AbstractAutowireCapableBeanFactory beanFactory) {
  this.beanFactory = beanFactory;
  this.logger = beanFactory.getLogger();
}
```

instantiateUsingFactoryMethod

- 根据工厂方法的是不是唯一来进行创建,如果唯一，就直接创建，不唯一的话还要进行方法获取，然后选出工厂方法，再处理

instantiateUsingFactoryMethod--1

- 创建一个实例持有器,后从 RootBeanDefinition中 获取 factoryBeanName，也就是工厂方法，比如说 bean 注解的方法就是
- 如果存在判断下是不是要创建自身，会报错，创建自身等于无限循环创建了
- 如果不是的话就获取factoryBean即工厂实例，获取工厂类型，设置非静态的。如果获取不到工厂实例，说明是静态方法

``` java
//实例持有器
BeanWrapperImpl bw = new BeanWrapperImpl();
this.beanFactory.initBeanWrapper(bw);

//工厂实例
Object factoryBean;
//工厂类型
Class<?> factoryClass;
//是否是静态的,存在factoryBeanName表示是实例工厂，非静态方法，否则是静态的
boolean isStatic;

//factoryBeanName，也就是工厂方法所属的类的简单名字，比如配置类，这里的factoryBean，不是FactoryBean接口
String factoryBeanName = mbd.getFactoryBeanName();

if (factoryBeanName != null) {
  if (factoryBeanName.equals(beanName)) {
    throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
        "factory-bean reference points back to the same bean definition");
  }
  //获得通过工厂获取 factoryBean
  factoryBean = this.beanFactory.getBean(factoryBeanName);
  //已经创建了
  if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
    throw new ImplicitlyAppearedSingletonException();
  }
  factoryClass = factoryBean.getClass();
  isStatic = false;
}
else {
  //静态的方法，无factoryBean实例
  // It's a static factory method on the bean class.
  if (!mbd.hasBeanClass()) {
    throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
        "bean definition declares neither a bean class nor a factory-bean reference");
  }
  factoryBean = null;
  factoryClass = mbd.getBeanClass();
  isStatic = true;
}
```

instantiateUsingFactoryMethod--2

准备好参数，尝试获取已经解析的数据，第一次都是空的。

``` java
//准备使用的工厂方法
Method factoryMethodToUse = null;
//准备使用的参数包装器
ArgumentsHolder argsHolderToUse = null;
//准备使用的参数
Object[] argsToUse = null;

if (explicitArgs != null) {
  //如果有显示参数就使用这些参数
  argsToUse = explicitArgs;
}
else {
  //解析出的参数
  Object[] argsToResolve = null;
  synchronized (mbd.constructorArgumentLock) {
    factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
    //如果有工厂方法，且构造函数已经解析了
    if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
      // Found a cached factory method...
      argsToUse = mbd.resolvedConstructorArguments;
      if (argsToUse == null) {
        argsToResolve = mbd.preparedConstructorArguments;
      }
    }
  }
  if (argsToResolve != null) {
    argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve, true);
  }
}
```

instantiateUsingFactoryMethod--3

- 如果没解析过，就获取factoryClass的用户定义类型，因为此时factoryClass可能是CGLIB动态代理类型，所以要获取用父类的类型。
- 如果工厂方法是唯一的，就是没重载的，就获取解析的工厂方法，如果不为空，就添加到一个不可变列表里，如果为空的话，就要去找出factoryClass的以及父类的所有的方法，进一步找出方法修饰符一致且名字跟工厂方法名字相同的且是bean注解的方法，并放入列表里。

``` java
if (factoryMethodToUse == null || argsToUse == null) {
  // Need to determine the factory method...
  // Try all methods with this name to see if they match the given arguments.

  //获取用户定义的类
  factoryClass = ClassUtils.getUserClass(factoryClass);

  //方法集合
  List<Method> candidates = null;

  //如果工厂方法唯一，没重载的
  if (mbd.isFactoryMethodUnique) {
    if (factoryMethodToUse == null) {
      //获取解析的工厂方法
      factoryMethodToUse = mbd.getResolvedFactoryMethod();
    }
    if (factoryMethodToUse != null) {
      //存在的话，返回仅包含factoryMethodToUse的不可变列表
      candidates = Collections.singletonList(factoryMethodToUse);
    }
  }
  //如果没找到工厂方法，可能有重载
  if (candidates == null) {
    candidates = new ArrayList<>();
    //获取factoryClass以及父类的所有方法作为候选的方法
    Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
    //过滤出修饰符一样，工厂方法名一样且是bean注解的方法
    for (Method candidate : rawCandidates) {
      if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
        //如果isStatic修饰符一样且名字跟工厂方法名一样就添加
        candidates.add(candidate);
      }
    }
  }
```

ClassUtils.getUserClass 找出用户定义的类型

``` java
public static Class<?> getUserClass(Class<?> clazz) {
  //如果包含CGLIB的名字符号，尝试获取父类
  if (clazz.getName().contains(CGLIB_CLASS_SEPARATOR)) {
    Class<?> superclass = clazz.getSuperclass();
    if (superclass != null && superclass != Object.class) {
      return superclass;
    }
  }
  return clazz;
}
```

getCandidateMethods 获取所有候选方法包括父类的

``` java
private Method[] getCandidateMethods(Class<?> factoryClass, RootBeanDefinition mbd) {
  if (System.getSecurityManager() != null) {
    return AccessController.doPrivileged((PrivilegedAction<Method[]>) () ->
        (mbd.isNonPublicAccessAllowed() ?
          ReflectionUtils.getAllDeclaredMethods(factoryClass) : factoryClass.getMethods()));
  }
  else {
    //如果允许非Public的方法，就获取所有申明的方法，包括父类的，否则就是Public的方法，包括父类的，默认是允许的
    return (mbd.isNonPublicAccessAllowed() ?
        ReflectionUtils.getAllDeclaredMethods(factoryClass) : factoryClass.getMethods());
  }
}
```

ReflectionUtils.getAllDeclaredMethods 用反射获取方法

如果允许非Public的方法，就获取所有申明的方法，包括父类的，否则就是Public的方法，包括父类的，默认是允许的

``` java
public static Method[] getAllDeclaredMethods(Class<?> leafClass) {
  final List<Method> methods = new ArrayList<>(32);
  doWithMethods(leafClass, methods::add);
  return methods.toArray(EMPTY_METHOD_ARRAY);
}

...//省略其余代码

public static void doWithMethods(Class<?> clazz, MethodCallback mc) {
  doWithMethods(clazz, mc, null);
}
```

doWithMethods 递归获取

递归获取所有的方法以及父类的，获取后执行mc.doWith(method)方法，其实就是上面的methods::add把方法添加到列表里，最后转换成数组返回。

``` java
public static void doWithMethods(Class<?> clazz, MethodCallback mc, @Nullable MethodFilter mf) {
  // Keep backing up the inheritance hierarchy.
  Method[] methods = getDeclaredMethods(clazz, false);
  for (Method method : methods) {
    if (mf != null && !mf.matches(method)) {
      continue;
    }
    try {
      mc.doWith(method);
    }
    catch (IllegalAccessException ex) {
      throw new IllegalStateException("Not allowed to access method '" + method.getName() + "': " + ex);
    }
  }
  if (clazz.getSuperclass() != null && (mf != USER_DECLARED_METHODS || clazz.getSuperclass() != Object.class)) {
    doWithMethods(clazz.getSuperclass(), mc, mf);
  }
  else if (clazz.isInterface()) {
    for (Class<?> superIfc : clazz.getInterfaces()) {
      doWithMethods(superIfc, mc, mf);
    }
  }
}
```

instantiateUsingFactoryMethod--4

只有一个无参工厂方法
如果只有一个构造函数的话，看是否有参数，没有的话直接就设置bean定义的属性，然后实例化设置进包装对象。

``` java
//如果只获取到一个方法，且传入的参数为空，且没有设置构造方法参数值
if (candidates.size() == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
  //获取出来
  Method uniqueCandidate = candidates.get(0);
  //没参数的话
  if (uniqueCandidate.getParameterCount() == 0) {
    //设置工厂方法
    mbd.factoryMethodToIntrospect = uniqueCandidate;
    synchronized (mbd.constructorArgumentLock) {
      //设置解析出来的方法
      mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
      //参数也已经解析了
      mbd.constructorArgumentsResolved = true;
      //方法参数为空
      mbd.resolvedConstructorArguments = EMPTY_ARGS;
    }
    //创建实例并设置到持有器里
    bw.setBeanInstance(instantiate(beanName, mbd, factoryBean, uniqueCandidate, EMPTY_ARGS));
    return bw;
  }
}
```

instantiate 工厂方法实例化

``` java
private Object instantiate(String beanName, RootBeanDefinition mbd,
			@Nullable Object factoryBean, Method factoryMethod, Object[] args) {

  try {
    if (System.getSecurityManager() != null) {
      return AccessController.doPrivileged((PrivilegedAction<Object>) () ->
          this.beanFactory.getInstantiationStrategy().instantiate(
              mbd, beanName, this.beanFactory, factoryBean, factoryMethod, args),
          this.beanFactory.getAccessControlContext());
    }
    else {
      //获取实例化策略进行实例化
      return this.beanFactory.getInstantiationStrategy().instantiate(
          mbd, beanName, this.beanFactory, factoryBean, factoryMethod, args);
    }
  }
  catch (Throwable ex) {
    throw new BeanCreationException(mbd.getResourceDescription(), beanName,
        "Bean instantiation via factory method failed", ex);
  }
}
```

SimpleInstantiationStrategy#instantiate

其实默认的实例化策略是CglibSubclassingInstantiationStrategy，就是可以用CGLIB做动态代理，但是仅限于方法注入的形式，所以这里是无参工厂方法还是调用父类SimpleInstantiationStrategy的实现。其实就是调用工厂实例的工厂方法，传入参数，只是参数是个空数组EMPTY_ARGS，返回对象。

``` java
@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
    @Nullable Object factoryBean, final Method factoryMethod, Object... args) {

  try {
    if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
        ReflectionUtils.makeAccessible(factoryMethod);
        return null;
      });
    }
    else {
      //设置 factoryMethod 可访问
      ReflectionUtils.makeAccessible(factoryMethod);
    }

    //获取前面存在的线程本地的FactoryMethod
    Method priorInvokedFactoryMethod = currentlyInvokedFactoryMethod.get();

    try {
      //设置新的
      currentlyInvokedFactoryMethod.set(factoryMethod);
      //调用工厂方法
      Object result = factoryMethod.invoke(factoryBean, args);
      if (result == null) {
        result = new NullBean();
      }
      return result;
    }
    finally {
      if (priorInvokedFactoryMethod != null) {
        //如果线程本地存在，就设置回老的
        currentlyInvokedFactoryMethod.set(priorInvokedFactoryMethod);
      }
      else {
        //否则就删除，等于没设置
        currentlyInvokedFactoryMethod.remove();
      }
    }
  }
  catch (IllegalArgumentException ex) {
    throw new BeanInstantiationException(factoryMethod,
        "Illegal arguments to factory method '" + factoryMethod.getName() + "'; " +
        "args: " + StringUtils.arrayToCommaDelimitedString(args), ex);
  }
  catch (IllegalAccessException ex) {
    throw new BeanInstantiationException(factoryMethod,
        "Cannot access factory method '" + factoryMethod.getName() + "'; is it public?", ex);
  }
  catch (InvocationTargetException ex) {
    String msg = "Factory method '" + factoryMethod.getName() + "' threw exception";
    if (bd.getFactoryBeanName() != null && owner instanceof ConfigurableBeanFactory &&
        ((ConfigurableBeanFactory) owner).isCurrentlyInCreation(bd.getFactoryBeanName())) {
      msg = "Circular reference involving containing bean '" + bd.getFactoryBeanName() + "' - consider " +
          "declaring the factory method as static for independence from its containing instance. " + msg;
    }
    throw new BeanInstantiationException(factoryMethod, msg, ex.getTargetException());
  }
}
```

多个有参工厂方法

怎么处理有多个参数的工厂方法呢，到底哪一个才是应该调用呢。

方法排序

``` java
if (candidates.size() > 1) {  // explicitly skip immutable singletonList
  //排序，根据public优先，参数多的优先
  candidates.sort(AutowireUtils.EXECUTABLE_COMPARATOR);
}
```

AutowireUtils.EXECUTABLE_COMPARATOR

``` java
//先比较修饰符，再比较参数个数 从Public到非Public，参数从多到少
public static final Comparator<Executable> EXECUTABLE_COMPARATOR = (e1, e2) -> {
  int result = Boolean.compare(Modifier.isPublic(e2.getModifiers()), Modifier.isPublic(e1.getModifiers()));
  return result != 0 ? result : Integer.compare(e2.getParameterCount(), e1.getParameterCount());
};
```

准备工作

因为接下去解析的工厂方法是有参数的，所以要进行一些处理，比如获取构造器参数值(如果有传的话)，然后获取自动装配模式，工厂方法一般默认是AUTOWIRE_CONSTRUCTOR，还会定义一个minTypeDiffWeight 差异值，这个就是用在如果有多个工厂方法的时候，看工厂方法的参数和具体装配的bean的类型的差异，取最小的。还定义了有个ambiguousFactoryMethods ，用来存放差异值一样的方法，说明是一样的类型，无法判断要用哪个工厂方法实例化。比如protected UserDao userDao(TestBean2 t2)和public UserDao userDao(TestBean t1)两个构造方法类型差异是一样的，所以不知道要用哪个，就会报异常啦。还有minNrOfArgs也很关键，用来做优化的，如果有最小参数个数，那么一些参数个数少于最小个数的就不需要去判断了，直接跳过，否则就算获取到了也不匹配，反而浪费资源了。如果minNrOfArgs为0，那就说明没有参数个数限制，那后面就需要一个个去判断哪个差异最小，最符合了。

``` java
//构造器参数值
ConstructorArgumentValues resolvedValues = null;
boolean autowiring = (mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
//最小的类型差距
int minTypeDiffWeight = Integer.MAX_VALUE;
//模棱两可的工厂方法集合
Set<Method> ambiguousFactoryMethods = null;

//最小参数个数
int minNrOfArgs;
if (explicitArgs != null) {
  //如果存在显示参数，就是显示参数的个数
  minNrOfArgs = explicitArgs.length;
}
else {
  // We don't have arguments passed in programmatically, so we need to resolve the
  // arguments specified in the constructor arguments held in the bean definition.
  //如果存在构造器参数值，就解析出最小参数个数
  if (mbd.hasConstructorArgumentValues()) {
    ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
    resolvedValues = new ConstructorArgumentValues();
    minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
  }
  else {
    //没有就为 0
    minNrOfArgs = 0;
  }
}
```

自动装配

这段最核心的，自动装配就在里面，首先前面已经过滤出candidates工厂方法了，但是我们这不知道要选哪个去实例化，所以得有个选取规则，首先参数个数不能小于最少的，如果有显示的参数存在，那个数不一致的也不要，然后就获取参数名字探测器去进行每一个工厂方法的参数名字探测，然后创建一个参数持有器，这里面就会涉及自定装配，依赖的参数会实例化。最后根据参数持有器的参数和工厂方法的参数类型作比较，保存最小的差异值的那个，把模棱两可的集合设置空。如果有相同差异值的就放入一个集合里，如果集合有数据，说明不知道用哪个工厂方法来实例化，会报异常

``` java
//遍历每个后选的方法，查看可以获取实例的匹配度
for (Method candidate : candidates) {
  int parameterCount = candidate.getParameterCount();

  if (parameterCount >= minNrOfArgs) {
    ArgumentsHolder argsHolder;

    Class<?>[] paramTypes = candidate.getParameterTypes();
    //显示参数存在，如果长度不对就继续下一个，否则就创建参数持有其持有
    if (explicitArgs != null) {
      // Explicit arguments given -> arguments length must match exactly.
      if (paramTypes.length != explicitArgs.length) {
        continue;
      }
      argsHolder = new ArgumentsHolder(explicitArgs);
    }
    else {
      // Resolved constructor arguments: type conversion and/or autowiring necessary.
      try {
        String[] paramNames = null;
        //获取参数名字探测器
        ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
        if (pnd != null) {
          //存在的话进行探测
          paramNames = pnd.getParameterNames(candidate);
        }
        argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw,
            paramTypes, paramNames, candidate, autowiring, candidates.size() == 1);
      }
      catch (UnsatisfiedDependencyException ex) {
        if (logger.isTraceEnabled()) {
          logger.trace("Ignoring factory method [" + candidate + "] of bean '" + beanName + "': " + ex);
        }
        // Swallow and try next overloaded factory method.
        if (causes == null) {
          causes = new LinkedList<>();
        }
        causes.add(ex);
        continue;
      }
    }

    //根据参数类型匹配，获取类型的差异值
    int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
        argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
    // Choose this factory method if it represents the closest match.
    //保存最小的，说明参数类型相近
    if (typeDiffWeight < minTypeDiffWeight) {
      factoryMethodToUse = candidate;
      argsHolderToUse = argsHolder;
      argsToUse = argsHolder.arguments;
      minTypeDiffWeight = typeDiffWeight;
      ambiguousFactoryMethods = null;//没有模棱两可的方法
    }

    //如果出现类型差异相同，参数个数也相同的，而且需要严格判断，参数长度也一样，常数类型也一样，就可能会无法判定要实例化哪个，就会报异常

    // Find out about ambiguity: In case of the same type difference weight
    // for methods with the same number of parameters, collect such candidates
    // and eventually raise an ambiguity exception.
    // However, only perform that check in non-lenient constructor resolution mode,
    // and explicitly ignore overridden methods (with the same parameter signature).
    else if (factoryMethodToUse != null && typeDiffWeight == minTypeDiffWeight &&
        !mbd.isLenientConstructorResolution() &&
        paramTypes.length == factoryMethodToUse.getParameterCount() &&
        !Arrays.equals(paramTypes, factoryMethodToUse.getParameterTypes())) {
      if (ambiguousFactoryMethods == null) {
        ambiguousFactoryMethods = new LinkedHashSet<>();
        ambiguousFactoryMethods.add(factoryMethodToUse);
      }
      ambiguousFactoryMethods.add(candidate);
    }
  }
}
```

####  determineConstructorsFromBeanPostProcessors 

- 前面是工厂方法的实例化
- 现在可以讲一般的构造方法实例化了，首先会进行处理器处理，找出合适的构造方法：

``` java
@Nullable
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
    throws BeansException {

  if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
    for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
      Constructor<?>[] ctors = bp.determineCandidateConstructors(beanClass, beanName);
      if (ctors != null) {
        return ctors;
      }
    }
  }
  return null;
}
```

SmartInstantiationAwareBeanPostProcessor 接口

AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors

方法很复杂，拷一份过来

``` java
@Override
@Nullable
public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
    throws BeanCreationException {

  // 查找lookup
  if (!this.lookupMethodsChecked.contains(beanName)) {
    if (AnnotationUtils.isCandidateClass(beanClass, Lookup.class)) {
      try {
        Class<?> targetClass = beanClass;
        do {//遍历当前类以及所有父类，找出Lookup注解的方法进行处理
          ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            Lookup lookup = method.getAnnotation(Lookup.class);
            if (lookup != null) {
              Assert.state(this.beanFactory != null, "No BeanFactory available");
              LookupOverride override = new LookupOverride(method, lookup.value());
              try {
                RootBeanDefinition mbd = (RootBeanDefinition)
                    this.beanFactory.getMergedBeanDefinition(beanName);
                mbd.getMethodOverrides().addOverride(override);
              }
              catch (NoSuchBeanDefinitionException ex) {
                throw new BeanCreationException(beanName,
                    "Cannot apply @Lookup to beans without corresponding bean definition");
              }
            }
          });
          targetClass = targetClass.getSuperclass();
        }
        while (targetClass != null && targetClass != Object.class);//遍历父类，直到Object

      }
      catch (IllegalStateException ex) {
        throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
      }
    }
    this.lookupMethodsChecked.add(beanName);//已经检查过
  }

  //先检查一遍缓存
  Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
  if (candidateConstructors == null) {//没找到再同步
    // Fully synchronized resolution now...
    synchronized (this.candidateConstructorsCache) {
      candidateConstructors = this.candidateConstructorsCache.get(beanClass);//再检测一遍，双重检测
      if (candidateConstructors == null) {
        Constructor<?>[] rawCandidates;
        try {
          rawCandidates = beanClass.getDeclaredConstructors();//获取所有构造方法
        }
        catch (Throwable ex) {
          throw new BeanCreationException(beanName,
              "Resolution of declared constructors on bean Class [" + beanClass.getName() +
              "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
        }
        List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
        Constructor<?> requiredConstructor = null;//需要的构造方法
        Constructor<?> defaultConstructor = null;//默认构造方法
        Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);//优先的构造方法，Kotlin的类才有，一般都是空
        int nonSyntheticConstructors = 0;
        for (Constructor<?> candidate : rawCandidates) {
          if (!candidate.isSynthetic()) {//非合成的方法
            nonSyntheticConstructors++;
          }
          else if (primaryConstructor != null) {
            continue;
          }
          MergedAnnotation<?> ann = findAutowiredAnnotation(candidate);//寻找Autowired和value的注解
          if (ann == null) {
            Class<?> userClass = ClassUtils.getUserClass(beanClass);
            if (userClass != beanClass) {//如果是有代理的，找到被代理类
              try {
                Constructor<?> superCtor =//获取构造方法
                    userClass.getDeclaredConstructor(candidate.getParameterTypes());
                ann = findAutowiredAnnotation(superCtor);//继续寻找Autowired和value的注解
              }
              catch (NoSuchMethodException ex) {
                // Simply proceed, no equivalent superclass constructor found...
              }
            }
          }
          if (ann != null) {
            if (requiredConstructor != null) {//有两个Autowired注解，冲突了
              throw new BeanCreationException(beanName,
                  "Invalid autowire-marked constructor: " + candidate +
                  ". Found constructor with 'required' Autowired annotation already: " +
                  requiredConstructor);
            }
            boolean required = determineRequiredStatus(ann);//是否需要，默认是true
            if (required) {
              if (!candidates.isEmpty()) {//如果已经有required=false了，又来了一个required=true的方法就报异常了，这样两个可能就不知道用哪个了
                throw new BeanCreationException(beanName,
                    "Invalid autowire-marked constructors: " + candidates +
                    ". Found constructor with 'required' Autowired annotation: " +
                    candidate);
              }
              requiredConstructor = candidate;
            }
            candidates.add(candidate);//加入集合
          }
          else if (candidate.getParameterCount() == 0) {//是无参构造方法
            defaultConstructor = candidate;
          }
        }
        if (!candidates.isEmpty()) {//如果候选不为空，基本就是有Autowired注解的，转换成数组直接返回
          // Add default constructor to list of optional constructors, as fallback.
          if (requiredConstructor == null) {
            if (defaultConstructor != null) {//添加默认构造函数
              candidates.add(defaultConstructor);
            }
            else if (candidates.size() == 1 && logger.isInfoEnabled()) {
              logger.info("Inconsistent constructor declaration on bean with name '" + beanName +
                  "': single autowire-marked constructor flagged as optional - " +
                  "this constructor is effectively required since there is no " +
                  "default constructor to fall back to: " + candidates.get(0));
            }
          }
          candidateConstructors = candidates.toArray(new Constructor<?>[0]);
        }
        else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
          candidateConstructors = new Constructor<?>[] {rawCandidates[0]};//只有一个函数且有参数
        }
        else if (nonSyntheticConstructors == 2 && primaryConstructor != null &&
            defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
          candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};//有两个非合成方法，有优先方法和默认方法，且不相同
        }
        else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {//只有一个优先的
          candidateConstructors = new Constructor<?>[] {primaryConstructor};
        }
        else {//大于2个没注解的构造方法就不知道要用什么了，所以就返回null
          candidateConstructors = new Constructor<?>[0];
        }
        this.candidateConstructorsCache.put(beanClass, candidateConstructors);
      }
    }
  }
  return (candidateConstructors.length > 0 ? candidateConstructors : null);
}
```

总结如下：

- 如果有多个Autowired，required有true，不管有没有默认构造方法，会报异常。
- 如果只有一个Autowired，required是false，没有默认构造方法，会报警告。
- 如果没有Autowired注解，定义了2个及以上有参数的构造方法，没有无参构造方法，就会报错。
- 其他情况都可以，但是以有Autowired的构造方法优先，然后才是默认构造方法。

如果构造方法存在且是有参数的，那就会调用autowireConstructor进行自动装配，如果不存在基本上无参构造方法了

#### autowireConstructor 有参构造方法自动装配

``` java
protected BeanWrapper autowireConstructor(
    String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {
    
  //ConstructorResolver
  return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}
```

ConstructorResolver#autowireConstructor

这个和前面讲过的工厂方法实例化instantiateUsingFactoryMethod很像，但是有几个地方不一样

- 工厂方法会遍历完所有的方法，然后找出参数类型差异最小的方法，而自动装配构造方法找到一个满足条件的就停止了。
- 参数类型差异算法不一样，工厂方法是用严格的方法getAssignabilityWeight，而自动装配是宽松的方法getTypeDifferenceWeight。

#### instantiateBean 无参构造方法实例化

其实里面就是获取实例化的策略，然后进行instantiate实例化

前面有说过策略是CglibSubclassingInstantiationStrategy的,但是只会在注入的时候起作用

一般的还是用父类SimpleInstantiationStrategy的instantiate。

``` java
protected BeanWrapper instantiateBean(String beanName, RootBeanDefinition mbd) {
  try {
    Object beanInstance;
    if (System.getSecurityManager() != null) {
      beanInstance = AccessController.doPrivileged(
          (PrivilegedAction<Object>) () -> getInstantiationStrategy().instantiate(mbd, beanName, this),
          getAccessControlContext());
    }
    else {
      beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, this);
    }
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
        mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
  }
}
```

BeanUtils#instantiateClass

调用构造方法反射创建对象

``` java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
  Assert.notNull(ctor, "Constructor must not be null");
  try {
    ReflectionUtils.makeAccessible(ctor);
    if (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass())) {
      return KotlinDelegate.instantiateClass(ctor, args);
    }
    else {
      Class<?>[] parameterTypes = ctor.getParameterTypes();
      Assert.isTrue(args.length <= parameterTypes.length, "Can't specify more arguments than constructor parameters");
      Object[] argsWithDefaultValues = new Object[args.length];
      for (int i = 0 ; i < args.length; i++) {
        if (args[i] == null) {
          Class<?> parameterType = parameterTypes[i];
          argsWithDefaultValues[i] = (parameterType.isPrimitive() ? DEFAULT_TYPE_VALUES.get(parameterType) : null);
        }
        else {
          argsWithDefaultValues[i] = args[i];
        }
      }
      return ctor.newInstance(argsWithDefaultValues);
    }
  }
  catch (InstantiationException ex) {
    throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
  }
  catch (IllegalAccessException ex) {
    throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
  }
  catch (IllegalArgumentException ex) {
    throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
  }
  catch (InvocationTargetException ex) {
    throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
  }
}
```

CglibSubclassingInstantiationStrategy#instantiateWithMethodInjection

``` java
@Override
protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
  return instantiateWithMethodInjection(bd, beanName, owner, null);
}

@Override
protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
    @Nullable Constructor<?> ctor, Object... args) {

  // Must generate CGLIB subclass...
  return new CglibSubclassCreator(bd, owner).instantiate(ctor, args);
}
```

CglibSubclassCreator#instantiate

- 创建一个GCLIB的增强器子类
- 如果传入构造方法是空的话就用BeanUtils进行GCLIB子类的实例化,否则就用增强子类的构造方法直接实例化。
- 然后设置方法拦截器，这样调用的时候就会进入拦截器里，就可以去容器里寻找，有就直接返回，没有就创建。

``` java
public Object instantiate(@Nullable Constructor<?> ctor, Object... args) {
  Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
  Object instance;
  if (ctor == null) {
    instance = BeanUtils.instantiateClass(subclass);
  }
  else {
    try {
      Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
      instance = enhancedSubclassConstructor.newInstance(args);
    }
    catch (Exception ex) {
      throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
          "Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
    }
  }
  // SPR-10785: set callbacks directly on the instance instead of in the
  // enhanced class (via the Enhancer) in order to avoid memory leaks.
  Factory factory = (Factory) instance;
  factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
      new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
      new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
  return instance;
}
```

至此，spring bean 实例化就已经完成了，但是此时的Bean只实例化出来，什么都没有做，是一个白板

### applyMergedBeanDefinitionPostProcessors

处理器修改合并bean定义

前面实例化完成了，之后要进行处理器的处理，也就是 `MergedBeanDefinitionPostProcessor` 处理器的处理：

``` java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
  for (MergedBeanDefinitionPostProcessor processor : getBeanPostProcessorCache().mergedDefinition) {
    processor.postProcessMergedBeanDefinition(mbd, beanType, beanName);
  }
}
```

MergedBeanDefinitionPostProcessor 的实现有很多，但是绝大多数为spring定义的

- CommonAnnotationBeanPostProcessor
  - 实现待补充
- AutowiredAnnotationBeanPostProcessor
  - 实现待补充
  - AutowiredAnnotationBeanPostProcessor#applyMergedBeanDefinitionPostProcessors主要做的事就是用处理器把属性和方法上的自动装配的信息记录下来，放到bean定义里，后面进行填充的时候可以用到，其中就包括CommonAnnotationBeanPostProcessor的生命周期方法和webService，ejb，Resource注解信息以及AutowiredAnnotationBeanPostProcessor的Autowired和Value注解信息。

### populateBean 属性填充

#### InstantiationAwareBeanPostProcessor

``` java
//在填充属性前，还会给InstantiationAwareBeanPostProcessor的处理器处理，给他们机会让他们去修改bean实例，这里也是扩展点

// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
// state of the bean before properties are set. This can be used, for example,
// to support styles of field injection.
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
  for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
    if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
      // 如果返回 false 就停止属性填充
      return;
    }
  }
}
```

#### 获取自动装配模式

``` java
PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

//获取自动装配模式,默认是no

int resolvedAutowireMode = mbd.getResolvedAutowireMode();
if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
  MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
  // Add property values based on autowire by name if applicable.
  if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
    autowireByName(beanName, mbd, bw, newPvs);
  }
  // Add property values based on autowire by type if applicable.
  if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
    autowireByType(beanName, mbd, bw, newPvs);
  }
  pvs = newPvs;
}
```

#### InstantiationAwareBeanPostProcessor#postProcessProperties

这里会进行InstantiationAwareBeanPostProcessor处理器的postProcessProperties处理；拓展点

``` java
boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

PropertyDescriptor[] filteredPds = null;
if (hasInstAwareBpps) {
  if (pvs == null) {
    // 创建PropertyValues
    pvs = mbd.getPropertyValues();
  }
  // 继续后置处理器处理
  for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
    PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
    if (pvsToUse == null) {
      if (filteredPds == null) {
        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }
      pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
      if (pvsToUse == null) {
        return;
      }
    }
    pvs = pvsToUse;
  }
}
if (needsDepCheck) {
  if (filteredPds == null) {
    filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
  }
  checkDependencies(beanName, mbd, filteredPds, pvs);
}

if (pvs != null) {
  applyPropertyValues(beanName, mbd, bw, pvs);
}
```

ImportAwareBeanPostProcessor#postProcessProperties

这个就是前面如果配置类有Configuration注解，会进行动态代理，会实现EnhancedConfiguration接口，里面有个setBeanFactory接口方法，会传入beanFactory。

``` java
@Override
public PropertyValues postProcessProperties(@Nullable PropertyValues pvs, Object bean, String beanName) {
  // Inject the BeanFactory before AutowiredAnnotationBeanPostProcessor's
  // postProcessProperties method attempts to autowire other configuration beans.
  if (bean instanceof EnhancedConfiguration) {
    ((EnhancedConfiguration) bean).setBeanFactory(this.beanFactory);
  }
  return pvs;
}
```

`CommonAnnotationBeanPostProcessor` 的 `postProcessProperties`

处理 @Resource 注解

这里就处理我们前面 `applyMergedBeanDefinitionPostProcessors` 处理注入注解元数据。

``` java
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
  InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass(), pvs);
  try {
    metadata.inject(bean, beanName, pvs);
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
  }
  return pvs;
}
```

``` java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
  Collection<InjectedElement> checkedElements = this.checkedElements;
  Collection<InjectedElement> elementsToIterate =
      (checkedElements != null ? checkedElements : this.injectedElements);
  if (!elementsToIterate.isEmpty()) {
    for (InjectedElement element : elementsToIterate) {
      if (logger.isTraceEnabled()) {
        logger.trace("Processing injected element of bean '" + beanName + "': " + element);
      }
      element.inject(target, beanName, pvs);
    }
  }
}
```

``` java
protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
				throws Throwable {

  //属性注入
  if (this.isField) {
    Field field = (Field) this.member;
    ReflectionUtils.makeAccessible(field);
    field.set(target, getResourceToInject(target, requestingBeanName));
  }
  else {
    if (checkPropertySkipping(pvs)) {
      return;
    }
    //方法注入
    try {
      Method method = (Method) this.member;
      ReflectionUtils.makeAccessible(method);
      method.invoke(target, getResourceToInject(target, requestingBeanName));
    }
    catch (InvocationTargetException ex) {
      throw ex.getTargetException();
    }
  }
}
```

ResourceElement#getResourceToInject

``` java
@Override
protected Object getResourceToInject(Object target, @Nullable String requestingBeanName) {
  return (this.lazyLookup ? buildLazyResourceProxy(this, requestingBeanName) :
      getResource(this, requestingBeanName));
}
```

``` java
protected Object getResource(LookupElement element, @Nullable String requestingBeanName)
			throws NoSuchBeanDefinitionException {

  if (StringUtils.hasLength(element.mappedName)) {
    return this.jndiFactory.getBean(element.mappedName, element.lookupType);
  }
  if (this.alwaysUseJndiLookup) {
    return this.jndiFactory.getBean(element.name, element.lookupType);
  }
  if (this.resourceFactory == null) {
    throw new NoSuchBeanDefinitionException(element.lookupType,
        "No resource factory configured - specify the 'resourceFactory' property");
  }
  return autowireResource(this.resourceFactory, element, requestingBeanName);
}
```

会调到 CommonAnnotationBeanPostProcessor#autowireResource

这里主要就是根据依赖描述进行工厂的解析，最后都是调用getBean(String name, Class<T> requiredType)方法，最后获取之后再注册到依赖映射里。其实这个有个resolveBeanByName，看起来好像是名字优先，其实也会获取类型的。

``` java
protected Object autowireResource(BeanFactory factory, LookupElement element, @Nullable String requestingBeanName)
    throws NoSuchBeanDefinitionException {

  //自动装配的对象
  Object resource;

  //自动装配的名字
  Set<String> autowiredBeanNames;

  //依赖的属性名
  String name = element.name;

  if (factory instanceof AutowireCapableBeanFactory) {
    AutowireCapableBeanFactory beanFactory = (AutowireCapableBeanFactory) factory;
    //创建依赖描述
    DependencyDescriptor descriptor = element.getDependencyDescriptor();
    //不包含依赖对象，或者依赖对象的bean定义，只能通过解析类型去处理
    if (this.fallbackToDefaultTypeMatch && element.isDefaultName && !factory.containsBean(name)) {
      autowiredBeanNames = new LinkedHashSet<>();
      resource = beanFactory.resolveDependency(descriptor, requestingBeanName, autowiredBeanNames, null);
      if (resource == null) {
        throw new NoSuchBeanDefinitionException(element.getLookupType(), "No resolvable resource object");
      }
    }
    else {
      //获取依赖对象，里面就是getBean
      resource = beanFactory.resolveBeanByName(name, descriptor);
      //装配好的bean名字
      autowiredBeanNames = Collections.singleton(name);
    }
  }
  else {
    resource = factory.getBean(name, element.lookupType);
    autowiredBeanNames = Collections.singleton(name);
  }

  if (factory instanceof ConfigurableBeanFactory) {
    ConfigurableBeanFactory beanFactory = (ConfigurableBeanFactory) factory;
    for (String autowiredBeanName : autowiredBeanNames) {
      if (requestingBeanName != null && beanFactory.containsBean(autowiredBeanName)) {
        //注册依赖关系
        beanFactory.registerDependentBean(autowiredBeanName, requestingBeanName);
      }
    }
  }

  return resource;
}
```

AutowiredAnnotationBeanPostProcessor#postProcessProperties

处理 @Autowire 注解

和前面CommonAnnotationBeanPostProcessor的很像，都是这个模板。

``` java
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
  InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
  try {
    metadata.inject(bean, beanName, pvs);
  }
  catch (BeanCreationException ex) {
    throw ex;
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
  }
  return pvs;
}
```

但是最后是AutowiredFieldElement类型的方法，前面是ResourceElement的，他们都是InjectedElement的子类，ResourceElement把处理属性和方法放一起判断处理，自动装配的AutowiredFieldElement和AutowiredMethodElement是分开处理的。

AutowiredFieldElement#inject

``` java
@Override
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
  Field field = (Field) this.member;
  Object value;
  if (this.cached) {
    value = resolvedCachedArgument(beanName, this.cachedFieldValue);
  }
  else {
    DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
    desc.setContainingClass(bean.getClass());
    Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
    Assert.state(beanFactory != null, "No BeanFactory available");
    TypeConverter typeConverter = beanFactory.getTypeConverter();
    try {
      value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
    }
    catch (BeansException ex) {
      throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
    }
    synchronized (this) {
      if (!this.cached) {
        if (value != null || this.required) {
          this.cachedFieldValue = desc;
          registerDependentBeans(beanName, autowiredBeanNames);
          if (autowiredBeanNames.size() == 1) {
            String autowiredBeanName = autowiredBeanNames.iterator().next();
            if (beanFactory.containsBean(autowiredBeanName) &&
                beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
              this.cachedFieldValue = new ShortcutDependencyDescriptor(
                  desc, autowiredBeanName, field.getType());
            }
          }
        }
        else {
          this.cachedFieldValue = null;
        }
        this.cached = true;
      }
    }
  }
  if (value != null) {
    ReflectionUtils.makeAccessible(field);
    field.set(bean, value);
  }
}
```

AutowiredAnnotationBeanPostProcessor#resolvedCachedArgument

``` java
@Nullable
private Object resolvedCachedArgument(@Nullable String beanName, @Nullable Object cachedArgument) {
  if (cachedArgument instanceof DependencyDescriptor) {
    DependencyDescriptor descriptor = (DependencyDescriptor) cachedArgument;
    Assert.state(this.beanFactory != null, "No BeanFactory available");
    return this.beanFactory.resolveDependency(descriptor, beanName, null, null);
  }
  else {
    return cachedArgument;
  }
}
```

关于属性主注入：

其实不管哪种注入的注解，原理都是一样的，都要根据名字和类型去容器里找，然后建立依赖关系

DefaultListableBeanFactory#resolveDependency

``` java
@Override
@Nullable
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
    @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

  descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
  if (Optional.class == descriptor.getDependencyType()) {
    return createOptionalDependency(descriptor, requestingBeanName);
  }
  else if (ObjectFactory.class == descriptor.getDependencyType() ||
      ObjectProvider.class == descriptor.getDependencyType()) {
    return new DependencyObjectProvider(descriptor, requestingBeanName);
  }
  else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
    return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
  }
  else {
    Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
        descriptor, requestingBeanName);
    if (result == null) {
      result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
    }
    return result;
  }
}
```

DefaultListableBeanFactory#doResolveDependency

``` java
@Nullable
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
    @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

  InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
  try {
    Object shortcut = descriptor.resolveShortcut(this);
    if (shortcut != null) {
      return shortcut;
    }

    Class<?> type = descriptor.getDependencyType();
    Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
    if (value != null) {
      if (value instanceof String) {
        String strVal = resolveEmbeddedValue((String) value);
        BeanDefinition bd = (beanName != null && containsBean(beanName) ?
            getMergedBeanDefinition(beanName) : null);
        value = evaluateBeanDefinitionString(strVal, bd);
      }
      TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
      try {
        return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
      }
      catch (UnsupportedOperationException ex) {
        // A custom TypeConverter which does not support TypeDescriptor resolution...
        return (descriptor.getField() != null ?
            converter.convertIfNecessary(value, type, descriptor.getField()) :
            converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
      }
    }

    Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
    if (multipleBeans != null) {
      return multipleBeans;
    }

    Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
    if (matchingBeans.isEmpty()) {
      if (isRequired(descriptor)) {
        raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
      }
      return null;
    }

    String autowiredBeanName;
    Object instanceCandidate;

    if (matchingBeans.size() > 1) {
      autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
      if (autowiredBeanName == null) {
        if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
          return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
        }
        else {
          // In case of an optional Collection/Map, silently ignore a non-unique case:
          // possibly it was meant to be an empty collection of multiple regular beans
          // (before 4.3 in particular when we didn't even look for collection beans).
          return null;
        }
      }
      instanceCandidate = matchingBeans.get(autowiredBeanName);
    }
    else {
      // We have exactly one match.
      Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
      autowiredBeanName = entry.getKey();
      instanceCandidate = entry.getValue();
    }

    if (autowiredBeanNames != null) {
      autowiredBeanNames.add(autowiredBeanName);
    }
    if (instanceCandidate instanceof Class) {
      instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
    }
    Object result = instanceCandidate;
    if (result instanceof NullBean) {
      if (isRequired(descriptor)) {
        raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
      }
      result = null;
    }
    if (!ClassUtils.isAssignableValue(type, result)) {
      throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
    }
    return result;
  }
  finally {
    ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
  }
}
```

@AutoWire 自动注入，现根据type，后根据 name

### initializeBean 初始化

经过前面两步，实例已经初始化，属性也已经注入进去，此时就应该执行初始化方法

``` java
// 初始化
exposedObject = initializeBean(beanName, exposedObject, mbd);
```

``` java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
  if (System.getSecurityManager() != null) {
    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
      invokeAwareMethods(beanName, bean);
      return null;
    }, getAccessControlContext());
  }
  else {
    // 执行Aware接口处理器回调
    invokeAwareMethods(beanName, bean);
  }

  Object wrappedBean = bean;
  if (mbd == null || !mbd.isSynthetic()) {
    // 初始化前
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  }

  try {
    // 初始化
    invokeInitMethods(beanName, wrappedBean, mbd);
  }
  catch (Throwable ex) {
    throw new BeanCreationException(
        (mbd != null ? mbd.getResourceDescription() : null),
        beanName, "Invocation of init method failed", ex);
  }
  if (mbd == null || !mbd.isSynthetic()) {
    // 初始化后
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  }

  return wrappedBean;
}
```

#### invokeAwareMethods

处理BeanNameAware，BeanClassLoaderAware，BeanFactoryAware接口方法，这里BeanFactoryAware的接口要注意，如果前面配置类有CGLIB动态代理的话，这里这个方法就重复了，也就是setBeanFactory被重复调用了两次，前面一次是在EnhancedConfiguration接口的方法。

``` java
private void invokeAwareMethods(String beanName, Object bean) {
  // 此处只回调部分 aware 接口
  if (bean instanceof Aware) {
    if (bean instanceof BeanNameAware) {
      ((BeanNameAware) bean).setBeanName(beanName);
    }
    if (bean instanceof BeanClassLoaderAware) {
      ClassLoader bcl = getBeanClassLoader();
      if (bcl != null) {
        ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
      }
    }
    if (bean instanceof BeanFactoryAware) {
      ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
    }
  }
}
```

#### applyBeanPostProcessorsBeforeInitialization

所有BeanPostProcessor 处理，如果有返回null了就直接返回，否则就把处理后的对象替换原来的，返回最新对象。

``` java
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
    throws BeansException {

  Object result = existingBean;
  for (BeanPostProcessor processor : getBeanPostProcessors()) {
    Object current = processor.postProcessBeforeInitialization(result, beanName);
    if (current == null) {
      return result;
    }
    result = current;
  }
  return result;
}
```

这里是一个拓展点，这里说几个常用的 spring 的实现

ApplicationContextAwareProcessor#postProcessBeforeInitialization

如果发现bean有实现那一堆接口任意一个的话，就会执行invokeAwareInterfaces方法。

``` java
@Override
@Nullable
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
      bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
      bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
    return bean;
  }

  AccessControlContext acc = null;

  if (System.getSecurityManager() != null) {
    acc = this.applicationContext.getBeanFactory().getAccessControlContext();
  }

  if (acc != null) {
    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
      invokeAwareInterfaces(bean);
      return null;
    }, acc);
  }
  else {
    invokeAwareInterfaces(bean);
  }

  return bean;
}
```

ApplicationContextAwareProcessor#invokeAwareInterfaces

其实就是设置一些值，方便外部获取，做一些扩展。

``` java
private void invokeAwareInterfaces(Object bean) {
  if (bean instanceof EnvironmentAware) {
    ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
  }
  if (bean instanceof EmbeddedValueResolverAware) {
    ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
  }
  if (bean instanceof ResourceLoaderAware) {
    ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
  }
  if (bean instanceof ApplicationEventPublisherAware) {
    ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
  }
  if (bean instanceof MessageSourceAware) {
    ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
  }
  if (bean instanceof ApplicationContextAware) {
    ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
  }
}
```

ConfigurationClassPostProcessor内部类ImportAwareBeanPostProcessor#postProcessBeforeInitialization

如果是ImportAware类型的，就会设置bean的注解信息。

``` java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
  if (bean instanceof ImportAware) {
    ImportRegistry ir = this.beanFactory.getBean(IMPORT_REGISTRY_BEAN_NAME, ImportRegistry.class);
    AnnotationMetadata importingClass = ir.getImportingClassFor(ClassUtils.getUserClass(bean).getName());
    if (importingClass != null) {
      ((ImportAware) bean).setImportMetadata(importingClass);
    }
  }
  return bean;
}
```

InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization

生命周期元数据，这个时候要进行调用了。

``` java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
  try {
    metadata.invokeInitMethods(bean, beanName);
  }
  catch (InvocationTargetException ex) {
    throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "Failed to invoke init method", ex);
  }
  return bean;
}
```

``` java
public void invokeInitMethods(Object target, String beanName) throws Throwable {
  Collection<LifecycleElement> checkedInitMethods = this.checkedInitMethods;
  Collection<LifecycleElement> initMethodsToIterate =
      (checkedInitMethods != null ? checkedInitMethods : this.initMethods);
  if (!initMethodsToIterate.isEmpty()) {
    for (LifecycleElement element : initMethodsToIterate) {
      if (logger.isTraceEnabled()) {
        logger.trace("Invoking init method on bean '" + beanName + "': " + element.getMethod());
      }
      element.invoke(target);
    }
  }
}
```

``` java
public void invoke(Object target) throws Throwable {
  ReflectionUtils.makeAccessible(this.method);
  this.method.invoke(target, (Object[]) null);
}
```

#### invokeInitMethods 执行初始化方法

- 如果实现了 InitializingBean 接口的话，就可能要调用 afterPropertiesSet 方法，这是一个回调方法
- 调用自定义的初始化方法 invokeCustomInitMethod

``` java
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
    throws Throwable {

  // 实现 InitializingBean ，并调用 afterPropertiesSet 方法
  boolean isInitializingBean = (bean instanceof InitializingBean);
  if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
    if (logger.isTraceEnabled()) {
      logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
    }
    if (System.getSecurityManager() != null) {
      try {
        AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
          ((InitializingBean) bean).afterPropertiesSet();
          return null;
        }, getAccessControlContext());
      }
      catch (PrivilegedActionException pae) {
        throw pae.getException();
      }
    }
    else {
      ((InitializingBean) bean).afterPropertiesSet();
    }
  }

  // 执行自定义的初始化方法
  if (mbd != null && bean.getClass() != NullBean.class) {
    String initMethodName = mbd.getInitMethodName();
    if (StringUtils.hasLength(initMethodName) &&
        !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
        !mbd.isExternallyManagedInitMethod(initMethodName)) {
      invokeCustomInitMethod(beanName, bean, mbd);
    }
  }
}
```

invokeCustomInitMethod

获取bean的自定义初始化方法，如果自身或者父类是接口类型的话，就反射出接口方法来，最后调用。

``` java
protected void invokeCustomInitMethod(String beanName, Object bean, RootBeanDefinition mbd)
    throws Throwable {

  String initMethodName = mbd.getInitMethodName();
  Assert.state(initMethodName != null, "No init method set");
  Method initMethod = (mbd.isNonPublicAccessAllowed() ?
      BeanUtils.findMethod(bean.getClass(), initMethodName) :
      ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));

  if (initMethod == null) {
    if (mbd.isEnforceInitMethod()) {
      throw new BeanDefinitionValidationException("Could not find an init method named '" +
          initMethodName + "' on bean with name '" + beanName + "'");
    }
    else {
      if (logger.isTraceEnabled()) {
        logger.trace("No default init method named '" + initMethodName +
            "' found on bean with name '" + beanName + "'");
      }
      // Ignore non-existent default lifecycle methods.
      return;
    }
  }

  if (logger.isTraceEnabled()) {
    logger.trace("Invoking init method  '" + initMethodName + "' on bean with name '" + beanName + "'");
  }
  Method methodToInvoke = ClassUtils.getInterfaceMethodIfPossible(initMethod);

  if (System.getSecurityManager() != null) {
    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
      ReflectionUtils.makeAccessible(methodToInvoke);
      return null;
    });
    try {
      AccessController.doPrivileged((PrivilegedExceptionAction<Object>)
          () -> methodToInvoke.invoke(bean), getAccessControlContext());
    }
    catch (PrivilegedActionException pae) {
      InvocationTargetException ex = (InvocationTargetException) pae.getException();
      throw ex.getTargetException();
    }
  }
  else {
    try {
      ReflectionUtils.makeAccessible(methodToInvoke);
      methodToInvoke.invoke(bean);
    }
    catch (InvocationTargetException ex) {
      throw ex.getTargetException();
    }
  }
}
```

ClassUtils#getInterfaceMethodIfPossible

``` java
public static Method getInterfaceMethodIfPossible(Method method) {
  if (!Modifier.isPublic(method.getModifiers()) || method.getDeclaringClass().isInterface()) {
    return method;
  }
  return interfaceMethodCache.computeIfAbsent(method, key -> {
    Class<?> current = key.getDeclaringClass();
    while (current != null && current != Object.class) {
      Class<?>[] ifcs = current.getInterfaces();
      for (Class<?> ifc : ifcs) {
        try {
          return ifc.getMethod(key.getName(), key.getParameterTypes());
        }
        catch (NoSuchMethodException ex) {
          // ignore
        }
      }
      current = current.getSuperclass();
    }
    return key;
  });
}
```

#### applyBeanPostProcessorsAfterInitialization

调用初始化后处理器，Aop就是在此处处理的

``` java
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

### registerDisposableBeanIfNecessary 销毁回调

``` java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
  AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
  // 不是原型并且有销毁接口的
  if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
    if (mbd.isSingleton()) {
      // 单例的情况，注册销毁回调
      // Register a DisposableBean implementation that performs all destruction
      // work for the given bean: DestructionAwareBeanPostProcessors,
      // DisposableBean interface, custom destroy method.
      registerDisposableBean(beanName, new DisposableBeanAdapter(
          bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));
    }
    else {
      //自定义的，注册到 Scope
      // A bean with a custom scope...
      Scope scope = this.scopes.get(mbd.getScope());
      if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
      }
      scope.registerDestructionCallback(beanName, new DisposableBeanAdapter(
          bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));
    }
  }
}
```

AbstractBeanFactory#requiresDestruction

``` java
protected boolean requiresDestruction(Object bean, RootBeanDefinition mbd) {
  return (bean.getClass() != NullBean.class && (DisposableBeanAdapter.hasDestroyMethod(bean, mbd) ||
      (hasDestructionAwareBeanPostProcessors() && DisposableBeanAdapter.hasApplicableProcessors(
          bean, getBeanPostProcessorCache().destructionAware))));
}
```

DisposableBeanAdapter#hasDestroyMethod

实现 DisposableBean 和 AutoCloseable 接口的，或者 INFER_METHOD 等于 destroyMethodName 且有 close，shutdown 方法名的

``` java
public static boolean hasDestroyMethod(Object bean, RootBeanDefinition beanDefinition) {
  if (bean instanceof DisposableBean || bean instanceof AutoCloseable) {
    return true;
  }
  String destroyMethodName = beanDefinition.getDestroyMethodName();
  if (AbstractBeanDefinition.INFER_METHOD.equals(destroyMethodName)) {
    return (ClassUtils.hasMethod(bean.getClass(), CLOSE_METHOD_NAME) ||
        ClassUtils.hasMethod(bean.getClass(), SHUTDOWN_METHOD_NAME));
  }
  return StringUtils.hasLength(destroyMethodName);
}
```

DisposableBeanAdapter#hasApplicableProcessors

如果有处理器来处理的话。

``` java
public static boolean hasApplicableProcessors(Object bean, List<DestructionAwareBeanPostProcessor> postProcessors) {
  if (!CollectionUtils.isEmpty(postProcessors)) {
    for (DestructionAwareBeanPostProcessor processor : postProcessors) {
      if (processor.requiresDestruction(bean)) {
        return true;
      }
    }
  }
  return false;
}
```

DefaultSingletonBeanRegistry#registerDisposableBean

其实就是把bean注册进去，到时候回调就行。

``` java
public void registerDisposableBean(String beanName, DisposableBean bean) {
  synchronized (this.disposableBeans) {
    this.disposableBeans.put(beanName, bean);
  }
}
```

至此，整个 getBean的流程都已经走完了。

- 循环依赖问题
- 后置处理器拓展问题
- AOP的实现原理问题
