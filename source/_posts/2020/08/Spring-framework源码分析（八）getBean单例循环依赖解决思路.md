---
title: Spring-framework源码分析（八）getBean单例循环依赖解决思路
abbrlink: 30e9671f
categories:
  - Spring-framework源码分析
tags:
  - 源码分析
  - Spring
  - Spring-framework
cover: 'https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720195606.jpg'
coverWidth: 1200
coverHeight: 750
---

- 循环依赖如何发生？
- spring如何解决循环依赖

<!--more-->

## 循环依赖

```java
@Component
public class A {
	@Autowired
	private B b;
}

@Component
public class B {
	@Autowired
	private A a;
}
```

A 需要 B，B 需要 A，此时就会有循环依赖，spring自动帮我们解决了此类循环依赖，但是如果是构造器注入 spring 是解决不了的，因为 spring 解决循环依赖的做法是未等Bean创建完成就提前将实例暴露出去，方便其他 Bean 进行引用。

## spring 解决循环依赖

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/08/10/20200810163606.jpeg)

当 spring 完成bean的实例化后，在调用 populateBean 填充属性之前，会先将实例暴露到第三级缓存 singletonFactories 中

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  // 从一级缓存获取，key=beanName value=bean
  Object singletonObject = this.singletonObjects.get(beanName);
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    synchronized (this.singletonObjects) {
      // 从二级缓存获取，key=beanName value=bean
      singletonObject = this.earlySingletonObjects.get(beanName);
      // 是否允许循环引用, allowEarlyReference 默认为true
      if (singletonObject == null && allowEarlyReference) {
        /**
         * 三级缓存获取，key=beanName value=objectFactory，objectFactory中存储getObject()方法用于获取提前曝光的实例
         * 为什么不直接将实例缓存到二级缓存，而要多此一举将实例先封装到objectFactory中？
         *
         * 因为主要关键点在getObject()方法并非直接返回实例，而是对实例又使用
         * SmartInstantiationAwareBeanPostProcessor 的 getEarlyBeanReference 方法对bean进行处理
         * 
         * 也就是说，当spring中存在该后置处理器，所有的单例bean在实例化后都会被进行提前曝光到三级缓存中，
         * 但是并不是所有的bean都存在循环依赖，也就是三级缓存到二级缓存的步骤不一定都会被执行，有可能曝光后直接创建完成，没被提前引用过，
         * 就直接被加入到一级缓存中。因此可以确保只有提前曝光且被引用的bean才会进行该后置处理
         */
        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
         /**
          * 通过getObject()方法获取bean，通过此方法获取到的实例不单单是提前曝光出来的实例，
          * 它还经过了SmartInstantiationAwareBeanPostProcessor的getEarlyBeanReference方法处理过。
          * 这也正是三级缓存存在的意义，可以通过重写该后置处理器对提前曝光的实例，在被提前引用时进行一些操作
          */
          singletonObject = singletonFactory.getObject();
          /**
          * 放入到二级缓存中
          */
          this.earlySingletonObjects.put(beanName, singletonObject);
          /**
          * 从三级缓存中删除
          */
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}
```

```java
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
```

## 三级缓存

- singletonObject
    一级缓存，该缓存key = beanName, value = bean;这里的bean是已经创建完成的，该bean经历过实例化->属性填充->初始化以及各类的后置处理。因此，一旦需要获取bean时，我们第一时间就会寻找一级缓存
- earlySingletonObjects
    二级缓存，该缓存key = beanName, value = bean;这里跟一级缓存的区别在于，该缓存所获取到的bean是提前曝光出来的，是还没创建完成的。也就是说获取到的bean只能确保已经进行了实例化，但是属性填充跟初始化肯定还没有做完，因此该bean还没创建完成，仅仅能作为指针提前曝光，被其他bean所引用
- singletonFactories
    三级缓存，该缓存key = beanName, value = beanFactory;在bean实例化完之后，属性填充以及初始化之前，如果允许提前曝光，spring会将实例化后的bean提前曝光，也就是把该bean转换成beanFactory并加入到三级缓存。在需要引用提前曝光对象时再通过singletonFactory.getObject()获取。

## getEarlyBeanReference

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
  Object exposedObject = bean;
  if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
    for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
      exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
    }
  }
  return exposedObject;
}
```

getEarlyBeanReference 在 spring 内部只有一个实现 AbstractAutoProxyCreator ,此处是 spring AOP对象创建的一个时机

```java
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
  Object cacheKey = getCacheKey(bean.getClass(), beanName);
  this.earlyProxyReferences.put(cacheKey, bean);
  return wrapIfNecessary(bean, beanName, cacheKey);
}
```

getEarlyBeanReference目的就是为了后置处理，给一个在提前曝光时操作bean的机会
