---
title: Spring-framework源码分析（四）BeanDefinition
abbrlink: 1555ca6d
categories:
  - Spring-framework源码分析
date: 2020-07-27 8:45:24
tags:
  - 源码分析
  - Spring
  - Spring-framework
cover: 'https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/20/20200720195606.jpg'
coverWidth: 1200
coverHeight: 750
---

> BeanDefinition 是 Spring-framework 的基石

<!--more-->

## 什么是BeanDefinition

`BeanDefinition` 就是 `spring bean` 的建模对象，就是把一个bean实例化出来的模型对象

## 为什么需要BeanDefinition

把一个 `bean` 实例化出来有 `Class` 就行了，为什么还需要 `BeanDefinition` 呢 ？

因为 `Class` 无法完成 `bean` 的抽象，比如

- bean 的作用域
- bean 的注入模型
- bean 是否是懒加载
- 等等

上述的这些信息 `Class` 是无法抽象出来的，这就是 `BeanDefinition` 的作用

相关接口的介绍

## BeanMetadataElement源数据接口

首先我们要能获得源数据，也就是这个bean是来自哪个对象的，我们就可以获得一些相关的信息。

```java
public interface BeanMetadataElement {

	/**
	 * 首先我们要能获得源数据，也就是这个bean是来自哪个对象的，我们就可以获得一些相关的信息。
	 */

	/**
	 * Return the configuration source {@code Object} for this metadata element
	 * (may be {@code null}).
	 */
	@Nullable
	default Object getSource() {
		return null;
	}

}
```

### BeanMetadataAttribute 元数据属性

```java
public class BeanMetadataAttribute implements BeanMetadataElement {

	/**
	 * 属性名字
	 */
	private final String name;

	/**
	 * 属性值
	 */
	@Nullable
	private final Object value;

	/**
	 * bean的来源
	 */
	@Nullable
	private Object source;


	/**
	 * Create a new AttributeValue instance.
	 * @param name the name of the attribute (never {@code null})
	 * @param value the value of the attribute (possibly before type conversion)
	 */
	public BeanMetadataAttribute(String name, @Nullable Object value) {
		//将名字和值封装起来
		Assert.notNull(name, "Name must not be null");
		this.name = name;
		this.value = value;
	}


	/**
	 * Return the name of the attribute.
	 */
	public String getName() {
		return this.name;
	}

	/**
	 * Return the value of the attribute.
	 */
	@Nullable
	public Object getValue() {
		return this.value;
	}

	/**
	 * Set the configuration source {@code Object} for this metadata element.
	 * <p>The exact type of the object will depend on the configuration mechanism used.
	 */
	public void setSource(@Nullable Object source) {
		this.source = source;
	}

	@Override
	@Nullable
	public Object getSource() {
		return this.source;
	}


	@Override
	public boolean equals(@Nullable Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof BeanMetadataAttribute)) {
			return false;
		}
		BeanMetadataAttribute otherMa = (BeanMetadataAttribute) other;
		return (this.name.equals(otherMa.name) &&
				ObjectUtils.nullSafeEquals(this.value, otherMa.value) &&
				ObjectUtils.nullSafeEquals(this.source, otherMa.source));
	}

	@Override
	public int hashCode() {
		return this.name.hashCode() * 29 + ObjectUtils.nullSafeHashCode(this.value);
	}

	@Override
	public String toString() {
		return "metadata attribute '" + this.name + "'";
	}

}
```

## AttributeAccessor属性访问器

这个就是定义了属性的操作接口，增删改查，获取所有的。

```java
public interface AttributeAccessor {

	/**
	 * Set the attribute defined by {@code name} to the supplied {@code value}.
	 * If {@code value} is {@code null}, the attribute is {@link #removeAttribute removed}.
	 * <p>In general, users should take care to prevent overlaps with other
	 * metadata attributes by using fully-qualified names, perhaps using
	 * class or package names as prefix.
	 * @param name the unique attribute key
	 * @param value the attribute value to be attached
	 */
	void setAttribute(String name, @Nullable Object value);

	/**
	 * Get the value of the attribute identified by {@code name}.
	 * Return {@code null} if the attribute doesn't exist.
	 * @param name the unique attribute key
	 * @return the current value of the attribute, if any
	 */
	@Nullable
	Object getAttribute(String name);

	/**
	 * Remove the attribute identified by {@code name} and return its value.
	 * Return {@code null} if no attribute under {@code name} is found.
	 * @param name the unique attribute key
	 * @return the last value of the attribute, if any
	 */
	@Nullable
	Object removeAttribute(String name);

	/**
	 * Return {@code true} if the attribute identified by {@code name} exists.
	 * Otherwise return {@code false}.
	 * @param name the unique attribute key
	 */
	boolean hasAttribute(String name);

	/**
	 * Return the names of all attributes.
	 */
	String[] attributeNames();

}
```

### AttributeAccessorSupport 属性访问抽象实现类

这个是对AttributeAccessor接口的抽象实现，定义了一个map来存放名字和属性的映射关系

```java
public abstract class AttributeAccessorSupport implements AttributeAccessor, Serializable {

	/**
	 * 名字与属性相对应
	 */
	/** Map with String keys and Object values. */
	private final Map<String, Object> attributes = new LinkedHashMap<>();

	//设为 null 值就是删除
	@Override
	public void setAttribute(String name, @Nullable Object value) {
		Assert.notNull(name, "Name must not be null");
		if (value != null) {
			this.attributes.put(name, value);
		}
		else {
			removeAttribute(name);
		}
	}

	@Override
	@Nullable
	public Object getAttribute(String name) {
		Assert.notNull(name, "Name must not be null");
		return this.attributes.get(name);
	}

	@Override
	@Nullable
	public Object removeAttribute(String name) {
		Assert.notNull(name, "Name must not be null");
		return this.attributes.remove(name);
	}

	@Override
	public boolean hasAttribute(String name) {
		Assert.notNull(name, "Name must not be null");
		return this.attributes.containsKey(name);
	}

	@Override
	public String[] attributeNames() {
		return StringUtils.toStringArray(this.attributes.keySet());
	}


	/**
	 * Copy the attributes from the supplied AttributeAccessor to this accessor.
	 * @param source the AttributeAccessor to copy from
	 */
	protected void copyAttributesFrom(AttributeAccessor source) {
		Assert.notNull(source, "Source must not be null");
		String[] attributeNames = source.attributeNames();
		for (String attributeName : attributeNames) {
			setAttribute(attributeName, source.getAttribute(attributeName));
		}
	}


	@Override
	public boolean equals(@Nullable Object other) {
		return (this == other || (other instanceof AttributeAccessorSupport &&
				this.attributes.equals(((AttributeAccessorSupport) other).attributes)));
	}

	@Override
	public int hashCode() {
		return this.attributes.hashCode();
	}

}
```

## `BeanMetadataAttributeAccessor` 元数据属性访问器

`BeanMetadataAttributeAccessor` 继承 `AttributeAccessorSupport` 实现 `BeanMetadataElement` ，既能获取源数据，也能提供属性访问：

```java
public class BeanMetadataAttributeAccessor extends AttributeAccessorSupport implements BeanMetadataElement {

	@Nullable
	private Object source;


	/**
	 * Set the configuration source {@code Object} for this metadata element.
	 * <p>The exact type of the object will depend on the configuration mechanism used.
	 */
	public void setSource(@Nullable Object source) {
		this.source = source;
	}

	@Override
	@Nullable
	public Object getSource() {
		return this.source;
	}


	/**
	 * Add the given BeanMetadataAttribute to this accessor's set of attributes.
	 * @param attribute the BeanMetadataAttribute object to register
	 */
	public void addMetadataAttribute(BeanMetadataAttribute attribute) {
		super.setAttribute(attribute.getName(), attribute);
	}

	/**
	 * Look up the given BeanMetadataAttribute in this accessor's set of attributes.
	 * @param name the name of the attribute
	 * @return the corresponding BeanMetadataAttribute object,
	 * or {@code null} if no such attribute defined
	 */
	@Nullable
	public BeanMetadataAttribute getMetadataAttribute(String name) {
		return (BeanMetadataAttribute) super.getAttribute(name);
	}

	@Override
	public void setAttribute(String name, @Nullable Object value) {
		super.setAttribute(name, new BeanMetadataAttribute(name, value));
	}

	@Override
	@Nullable
	public Object getAttribute(String name) {
		BeanMetadataAttribute attribute = (BeanMetadataAttribute) super.getAttribute(name);
		return (attribute != null ? attribute.getValue() : null);
	}

	@Override
	@Nullable
	public Object removeAttribute(String name) {
		BeanMetadataAttribute attribute = (BeanMetadataAttribute) super.removeAttribute(name);
		return (attribute != null ? attribute.getValue() : null);
	}

}
```

## BeanDefinition

BeanDefinition 接口继承 AttributeAccessor 和 BeanMetadataElement

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	/**
	 * Scope identifier for the standard singleton scope: {@value}.
	 * <p>Note that extended bean factories might support further scopes.
	 * @see #setScope
	 * @see ConfigurableBeanFactory#SCOPE_SINGLETON
	 */
	/**
	 * 单例
	 */
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

	/**
	 * Scope identifier for the standard prototype scope: {@value}.
	 * <p>Note that extended bean factories might support further scopes.
	 * @see #setScope
	 * @see ConfigurableBeanFactory#SCOPE_PROTOTYPE
	 */
	/**
	 * 原型
	 */
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


	/**
	 * Role hint indicating that a {@code BeanDefinition} is a major part
	 * of the application. Typically corresponds to a user-defined bean.
	 */
	/**
	 * 角色
	 */
	int ROLE_APPLICATION = 0;

	/**
	 * Role hint indicating that a {@code BeanDefinition} is a supporting
	 * part of some larger configuration, typically an outer
	 * {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
	 * {@code SUPPORT} beans are considered important enough to be aware
	 * of when looking more closely at a particular
	 * {@link org.springframework.beans.factory.parsing.ComponentDefinition},
	 * but not when looking at the overall configuration of an application.
	 */
	int ROLE_SUPPORT = 1;

	/**
	 * Role hint indicating that a {@code BeanDefinition} is providing an
	 * entirely background role and has no relevance to the end-user. This hint is
	 * used when registering beans that are completely part of the internal workings
	 * of a {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
	 */
	int ROLE_INFRASTRUCTURE = 2;


	// Modifiable attributes

	/**
	 * Set the name of the parent definition of this bean definition, if any.
	 */
	void setParentName(@Nullable String parentName);

	/**
	 * Return the name of the parent definition of this bean definition, if any.
	 */
	@Nullable
	String getParentName();

	/**
	 * Specify the bean class name of this bean definition.
	 * <p>The class name can be modified during bean factory post-processing,
	 * typically replacing the original class name with a parsed variant of it.
	 * @see #setParentName
	 * @see #setFactoryBeanName
	 * @see #setFactoryMethodName
	 */
	/**
	 * 设置bean类名字,这个可能会在后置处理器中修改，比如用了 GCLIB 代理配置类的时候就会改
	 */
	void setBeanClassName(@Nullable String beanClassName);

	/**
	 * Return the current bean class name of this bean definition.
	 * <p>Note that this does not have to be the actual class name used at runtime, in
	 * case of a child definition overriding/inheriting the class name from its parent.
	 * Also, this may just be the class that a factory method is called on, or it may
	 * even be empty in case of a factory bean reference that a method is called on.
	 * Hence, do <i>not</i> consider this to be the definitive bean type at runtime but
	 * rather only use it for parsing purposes at the individual bean definition level.
	 * @see #getParentName()
	 * @see #getFactoryBeanName()
	 * @see #getFactoryMethodName()
	 */
	@Nullable
	String getBeanClassName();

	/**
	 * Override the target scope of this bean, specifying a new scope name.
	 * @see #SCOPE_SINGLETON
	 * @see #SCOPE_PROTOTYPE
	 */
	/**
	 * 设置范围，是单例，还是原型
	 */
	void setScope(@Nullable String scope);

	/**
	 * Return the name of the current target scope for this bean,
	 * or {@code null} if not known yet.
	 */
	@Nullable
	String getScope();

	/**
	 * Set whether this bean should be lazily initialized.
	 * <p>If {@code false}, the bean will get instantiated on startup by bean
	 * factories that perform eager initialization of singletons.
	 */
	/**
	 * 设置是否是懒加载
	 */
	void setLazyInit(boolean lazyInit);

	/**
	 * Return whether this bean should be lazily initialized, i.e. not
	 * eagerly instantiated on startup. Only applicable to a singleton bean.
	 */
	boolean isLazyInit();

	/**
	 * Set the names of the beans that this bean depends on being initialized.
	 * The bean factory will guarantee that these beans get initialized first.
	 */
	/**
	 * 设置需要先加载的依赖的类名字数组
	 */
	void setDependsOn(@Nullable String... dependsOn);

	/**
	 * Return the bean names that this bean depends on.
	 */
	@Nullable
	String[] getDependsOn();

	/**
	 * Set whether this bean is a candidate for getting autowired into some other bean.
	 * <p>Note that this flag is designed to only affect type-based autowiring.
	 * It does not affect explicit references by name, which will get resolved even
	 * if the specified bean is not marked as an autowire candidate. As a consequence,
	 * autowiring by name will nevertheless inject a bean if the name matches.
	 */
	/**
	 * 设置是否适合给其他类做自动装配
	 */
	void setAutowireCandidate(boolean autowireCandidate);

	/**
	 * Return whether this bean is a candidate for getting autowired into some other bean.
	 */
	boolean isAutowireCandidate();

	/**
	 * Set whether this bean is a primary autowire candidate.
	 * <p>If this value is {@code true} for exactly one bean among multiple
	 * matching candidates, it will serve as a tie-breaker.
	 */
	void setPrimary(boolean primary);

	/**
	 * Return whether this bean is a primary autowire candidate.
	 */
	/**
	 * 设置是否优先自动装配
	 */
	boolean isPrimary();

	/**
	 * Specify the factory bean to use, if any.
	 * This the name of the bean to call the specified factory method on.
	 * @see #setFactoryMethodName
	 */
	/**
	 * 设置FactoryBean的名字
	 */
	void setFactoryBeanName(@Nullable String factoryBeanName);

	/**
	 * Return the factory bean name, if any.
	 */
	@Nullable
	String getFactoryBeanName();

	/**
	 * Specify a factory method, if any. This method will be invoked with
	 * constructor arguments, or with no arguments if none are specified.
	 * The method will be invoked on the specified factory bean, if any,
	 * or otherwise as a static method on the local bean class.
	 * @see #setFactoryBeanName
	 * @see #setBeanClassName
	 */
	void setFactoryMethodName(@Nullable String factoryMethodName);

	/**
	 * Return a factory method, if any.
	 */
	@Nullable
	String getFactoryMethodName();

	/**
	 * Return the constructor argument values for this bean.
	 * <p>The returned instance can be modified during bean factory post-processing.
	 * @return the ConstructorArgumentValues object (never {@code null})
	 */
	/**
	 * 获取构造函数的参数
	 */
	ConstructorArgumentValues getConstructorArgumentValues();

	/**
	 * Return if there are constructor argument values defined for this bean.
	 * @since 5.0.2
	 */
	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}

	/**
	 * Return the property values to be applied to a new instance of the bean.
	 * <p>The returned instance can be modified during bean factory post-processing.
	 * @return the MutablePropertyValues object (never {@code null})
	 */
	MutablePropertyValues getPropertyValues();

	/**
	 * Return if there are property values defined for this bean.
	 * @since 5.0.2
	 */
	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}

	/**
	 * Set the name of the initializer method.
	 * @since 5.1
	 */
	/**
	 * 设置初始化方法 对应@PostConstruct
	 */
	void setInitMethodName(@Nullable String initMethodName);

	/**
	 * Return the name of the initializer method.
	 * @since 5.1
	 */
	@Nullable
	String getInitMethodName();

	/**
	 * Set the name of the destroy method.
	 * @since 5.1
	 */
	/**
	 * 设置销毁时候的方法 对应@PreDestroy
	 */
	void setDestroyMethodName(@Nullable String destroyMethodName);

	/**
	 * Return the name of the destroy method.
	 * @since 5.1
	 */
	@Nullable
	String getDestroyMethodName();

	/**
	 * Set the role hint for this {@code BeanDefinition}. The role hint
	 * provides the frameworks as well as tools an indication of
	 * the role and importance of a particular {@code BeanDefinition}.
	 * @since 5.1
	 * @see #ROLE_APPLICATION
	 * @see #ROLE_SUPPORT
	 * @see #ROLE_INFRASTRUCTURE
	 */
	void setRole(int role);

	/**
	 * Get the role hint for this {@code BeanDefinition}. The role hint
	 * provides the frameworks as well as tools an indication of
	 * the role and importance of a particular {@code BeanDefinition}.
	 * @see #ROLE_APPLICATION
	 * @see #ROLE_SUPPORT
	 * @see #ROLE_INFRASTRUCTURE
	 */
	int getRole();

	/**
	 * Set a human-readable description of this bean definition.
	 * @since 5.1
	 */
	void setDescription(@Nullable String description);

	/**
	 * Return a human-readable description of this bean definition.
	 */
	@Nullable
	String getDescription();


	// Read-only attributes

	/**
	 * Return a resolvable type for this bean definition,
	 * based on the bean class or other specific metadata.
	 * <p>This is typically fully resolved on a runtime-merged bean definition
	 * but not necessarily on a configuration-time definition instance.
	 * @return the resolvable type (potentially {@link ResolvableType#NONE})
	 * @since 5.2
	 * @see ConfigurableBeanFactory#getMergedBeanDefinition
	 */
	ResolvableType getResolvableType();

	/**
	 * Return whether this a <b>Singleton</b>, with a single, shared instance
	 * returned on all calls.
	 * @see #SCOPE_SINGLETON
	 */
	boolean isSingleton();

	/**
	 * Return whether this a <b>Prototype</b>, with an independent instance
	 * returned for each call.
	 * @since 3.0
	 * @see #SCOPE_PROTOTYPE
	 */
	boolean isPrototype();

	/**
	 * Return whether this bean is "abstract", that is, not meant to be instantiated.
	 */
	boolean isAbstract();

	/**
	 * Return a description of the resource that this bean definition
	 * came from (for the purpose of showing context in case of errors).
	 */
	@Nullable
	String getResourceDescription();

	/**
	 * Return the originating BeanDefinition, or {@code null} if none.
	 * <p>Allows for retrieving the decorated bean definition, if any.
	 * <p>Note that this method returns the immediate originator. Iterate through the
	 * originator chain to find the original BeanDefinition as defined by the user.
	 */
	@Nullable
	BeanDefinition getOriginatingBeanDefinition();

}

```

## AbstractBeanDefinition

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/30/20200730104359.png)

其实就是继承了上面的bean元数据访问器并且实现了bean定义接口，这样的话就可以让bean的元数据和bean的定义接口联系起来了，也就是数据和操作结合了

```java
	//this.beanClasss是对象，可以是String也可以是Class
	@Override
	@Nullable
	public String getBeanClassName() {
		Object beanClassObject = this.beanClass;
		if (beanClassObject instanceof Class) {
			return ((Class<?>) beanClassObject).getName();
		}
		else {
			return (String) beanClassObject;
		}
	}
	
	//可以传Class
	public void setBeanClass(@Nullable Class<?> beanClass) {
		this.beanClass = beanClass;
	}
	//判断beanClass是否是个Class对象
	public boolean hasBeanClass() {
		return (this.beanClass instanceof Class);
	}

	//定义了一个克隆的抽象方法
	public abstract AbstractBeanDefinition cloneBeanDefinition();
```

## GenericBeanDefinition

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/30/20200730110308.png)

继承 AbstractBeanDefinition 主要是实现了克隆的方法

```java
@Override
public AbstractBeanDefinition cloneBeanDefinition() {
  return new GenericBeanDefinition(this);
}
```

## AnnotatedBeanDefinition

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/30/20200730110500.png)

加了两个新方法，其实就是注解的元数据和工厂方法的元数据，这些数据在进行解析处理的时候需要用到。

```java
public interface AnnotatedBeanDefinition extends BeanDefinition {

	/**
	 * Obtain the annotation metadata (as well as basic class metadata)
	 * for this bean definition's bean class.
	 * @return the annotation metadata object (never {@code null})
	 */
	AnnotationMetadata getMetadata();

	/**
	 * Obtain metadata for this bean definition's factory method, if any.
	 * @return the factory method metadata, or {@code null} if none
	 * @since 4.1.1
	 */
	@Nullable
	MethodMetadata getFactoryMethodMetadata();

}
```

## AnnotatedGenericBeanDefinition

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/30/20200730110800.png)

AnnotatedGenericBeanDefinition将通用bean定义 `GenericBeanDefinition` 和注解bean定义 `AnnotatedBeanDefinition` 拼起来了

```java
public class AnnotatedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {

	/**
	 * 注解元数据
	 */
	private final AnnotationMetadata metadata;

	/**
	 * 工厂方法元数据
	 */
	@Nullable
	private MethodMetadata factoryMethodMetadata;

	/**
	 * 通过 beanClass 来获取元数据
	 */
	/**
	 * Create a new AnnotatedGenericBeanDefinition for the given bean class.
	 * @param beanClass the loaded bean class
	 */
	public AnnotatedGenericBeanDefinition(Class<?> beanClass) {
		setBeanClass(beanClass);
		this.metadata = AnnotationMetadata.introspect(beanClass);
	}

	/**
	 * 从注解元数据获取 beanClass
	 */
	/**
	 * Create a new AnnotatedGenericBeanDefinition for the given annotation metadata,
	 * allowing for ASM-based processing and avoidance of early loading of the bean class.
	 * Note that this constructor is functionally equivalent to
	 * {@link org.springframework.context.annotation.ScannedGenericBeanDefinition
	 * ScannedGenericBeanDefinition}, however the semantics of the latter indicate that a
	 * bean was discovered specifically via component-scanning as opposed to other means.
	 * @param metadata the annotation metadata for the bean class in question
	 * @since 3.1.1
	 */
	public AnnotatedGenericBeanDefinition(AnnotationMetadata metadata) {
		Assert.notNull(metadata, "AnnotationMetadata must not be null");
		if (metadata instanceof StandardAnnotationMetadata) {
			setBeanClass(((StandardAnnotationMetadata) metadata).getIntrospectedClass());
		}
		else {
			setBeanClassName(metadata.getClassName());
		}
		this.metadata = metadata;
	}

	/**
	 * 从注解元数据与方法元数据创建
	 */
	/**
	 * Create a new AnnotatedGenericBeanDefinition for the given annotation metadata,
	 * based on an annotated class and a factory method on that class.
	 * @param metadata the annotation metadata for the bean class in question
	 * @param factoryMethodMetadata metadata for the selected factory method
	 * @since 4.1.1
	 */
	public AnnotatedGenericBeanDefinition(AnnotationMetadata metadata, MethodMetadata factoryMethodMetadata) {
		this(metadata);
		Assert.notNull(factoryMethodMetadata, "MethodMetadata must not be null");
		setFactoryMethodName(factoryMethodMetadata.getMethodName());
		this.factoryMethodMetadata = factoryMethodMetadata;
	}


	@Override
	public final AnnotationMetadata getMetadata() {
		return this.metadata;
	}

	@Override
	@Nullable
	public final MethodMetadata getFactoryMethodMetadata() {
		return this.factoryMethodMetadata;
	}

}
```

## RootBeanDefinition 

![](https://cdn.jsdelivr.net/gh/code-13/cloudimage/images/2020/07/30/20200730112853.png)

`RootBeanDefinition` 的主要作用是将其他类型的 BeanDefinition 在运行时合并起来，spring基本内部的定义都是用这个

实现太长，各别解释

```java
/**
 * 在读/写和创建实例的方法有关的字段、缓存字段时使用此锁。
 */
final Object constructorArgumentLock = new Object();
/**
 * 在读/写后处理器相关缓存字段时使用此锁
 */
final Object postProcessingLock = new Object();

// 默认允许启用缓存
boolean allowCaching = true;

// 你的 FactoryMethod 有没有专门设置值【捷径方法】
boolean isFactoryMethodUnique = false;

@Nullable
// bean 中包含的范型 ？？？？？？【待定】
volatile ResolvableType targetType;

/**
 * 缓存之前确定好的 Bean 实例类型
 */
@Nullable
volatile Class<?> resolvedTargetType;

/**
 * 缓存之前确定好的创建实例的方法返回的类型
 */
@Nullable
volatile ResolvableType factoryMethodReturnType;
/**
 * 缓存之前确定好的创建实例的方法【可能是构造方法、可能是工厂方法】
 */
@Nullable
Executable resolvedConstructorOrFactoryMethod;
/**
 * 我们有没有缓存创建实例的入参【捷径方法】
 */
boolean constructorArgumentsResolved = false;
/**
 * 缓存之前确定好的创建实例的入参【可能是构造方法、可能是工厂方法】【这个是完整的一套入参】
 */
@Nullable
Object[] resolvedConstructorArguments;
/**
 * 缓存之前确定好的创建实例的入参【可能是构造方法、可能是工厂方法】【这个是不完整的一套入参，可能少东西】
 */
@Nullable
Object[] preparedConstructorArguments;
/**
 * 标记此 bd 是否被应用过后处理器的钩子【MergedBeanDefinitionPostProcessor】
 */
boolean postProcessed = false;
/**
 * 标记量，表明此 bd 是否要执行创造 bean 前的钩子【用于决定 aop 是否可以织入】
 */
@Nullable
volatile Boolean beforeInstantiationResolved;

/**
 * 
 **/
@Nullable
private BeanDefinitionHolder decoratedDefinition;

/**
 *
 **/
@Nullable
private AnnotatedElement qualifiedElement;

/**
 *
 **/
@Nullable
private Set<Member> externallyManagedConfigMembers;

/**
 *
 **/
@Nullable
private Set<String> externallyManagedInitMethods;

/**
 *
 **/
@Nullable
private Set<String> externallyManagedDestroyMethods;
```

## MergedBeanDefinition

在Spring中,关于bean定义,其Java建模模型是接口BeanDefinition, 其变种有RootBeanDefinition，ChildBeanDefinition，还有GenericBeanDefinition，AnnotatedGenericBeanDefinition,ScannedGenericBeanDefinition等等。这些概念模型抽象了不同的关注点。关于这些概念模型，除了有概念，也有相应的Java建模模型，甚至还有通用的实现部分AbstractBeanDefinition。但事实上，关于BeanDefinition，还有一个概念也很重要，这就是MergedBeanDefinition(中文也许应该翻译成"合并了的bean定义"?),但这个概念并没有相应的Java模型对应。但是它确实存在，并且Spring专门为它提供了一个生命周期回调定义接口MergedBeanDefinitionPostProcessor用于扩展。

### MergedBeanDefinition的生成

AbstractBeanFactory#getMergedLocalBeanDefinition(String beanName)

```java
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
  // Quick check on the concurrent map first, with minimal locking.
  RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
  if (mbd != null && !mbd.stale) {
    return mbd;
  }
  return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}
```

```java
protected RootBeanDefinition getMergedBeanDefinition(String beanName, BeanDefinition bd)
    throws BeanDefinitionStoreException {

  return getMergedBeanDefinition(beanName, bd, null);
}
```

```java
protected RootBeanDefinition getMergedBeanDefinition(
    String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
    throws BeanDefinitionStoreException {

  synchronized (this.mergedBeanDefinitions) {

    // 准备一个RootBeanDefinition变量引用，用于记录要构建和最终要返回的BeanDefinition，
    // 这里根据上下文不难猜测 mbd 应该就是 mergedBeanDefinition 的缩写。
    RootBeanDefinition mbd = null;


    RootBeanDefinition previous = null;

    // Check with full lock now in order to enforce the same merged instance.
    if (containingBd == null) {
      mbd = this.mergedBeanDefinitions.get(beanName);
    }

    if (mbd == null || mbd.stale) {
      previous = mbd;
      if (bd.getParentName() == null) {

        /**
          * bd不是一个ChildBeanDefinition的情况,换句话讲，这 bd应该是 :
          * 1. 一个独立的 GenericBeanDefinition 实例，parentName 属性为null
          * 2. 或者是一个 RootBeanDefinition 实例，parentName 属性为null
          * 此时mbd直接使用一个bd的复制品
          */

        // Use copy of given root bean definition.
        if (bd instanceof RootBeanDefinition) {
          mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
        }
        else {
          mbd = new RootBeanDefinition(bd);
        }
      }
      else {

        /**
          * bd是一个ChildBeanDefinition的情况,这种情况下，需要将bd和其parent bean definition 合并到一起，形成最终的 mbd
          * 面是获取bd的 parent bean definition 的过程，最终结果记录到 pbd，
          *
          * 并且可以看到该过程中递归使用了getMergedBeanDefinition(), 为什么呢?因为 bd 的 parent bd 可能也是个ChildBeanDefinition，所以该过程需要递归处理
          */

        // Child bean definition: needs to be merged with parent.
        BeanDefinition pbd;
        try {
          String parentBeanName = transformedBeanName(bd.getParentName());
          if (!beanName.equals(parentBeanName)) {
            pbd = getMergedBeanDefinition(parentBeanName);
          }
          else {
            BeanFactory parent = getParentBeanFactory();
            if (parent instanceof ConfigurableBeanFactory) {
              pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
            }
            else {
              throw new NoSuchBeanDefinitionException(parentBeanName,
                  "Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
                  "': cannot be resolved without a ConfigurableBeanFactory parent");
            }
          }
        }
        catch (NoSuchBeanDefinitionException ex) {
          throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
              "Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
        }

        /**
          * 现在已经获取 bd 的 parent bd 到 pbd，这个 pbd 也是已经合并过的
          * 这里使用pbd创建最终的mbd，然后再使用bd覆盖一次，这样就相当于mbd来自两个BeanDefinition:当前 BeanDefinition 及其合并的("Merged")双亲 BeanDefinition,
          * 然后mbd就是针对当前bd的一个MergedBeanDefinition(合并的BeanDefinition)了。
          */

        // Deep copy with overridden values.
        mbd = new RootBeanDefinition(pbd);
        mbd.overrideFrom(bd);
      }

      // Set default singleton scope, if not configured before.
      if (!StringUtils.hasLength(mbd.getScope())) {
        mbd.setScope(SCOPE_SINGLETON);
      }

      // A bean contained in a non-singleton bean cannot be a singleton itself.
      // Let's correct this on the fly here, since this might be the result of
      // parent-child merging for the outer bean, in which case the original inner bean
      // definition will not have inherited the merged outer bean's singleton status.
      if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
        mbd.setScope(containingBd.getScope());
      }

      // Cache the merged bean definition for the time being
      // (it might still get re-merged later on in order to pick up metadata changes)
      if (containingBd == null && isCacheBeanMetadata()) {
        this.mergedBeanDefinitions.put(beanName, mbd);
      }
    }
    if (previous != null) {
      copyRelevantMergedBeanDefinitionCaches(previous, mbd);
    }
    return mbd;
  }
}
```

从上面的 `MergedBeanDefinition` 的获取过程可以看出，一个 `MergedBeanDefinition` 其实就是一个"合并了的 `BeanDefinition` "，最终以 `RootBeanDefinition` 的类型存在。

### MergedBeanDefinition的应用

下面是类 `AbstractAutowireCapableBeanFactory` 中bean创建方法createBean()的实现，可以看到，针对传入的参数 RootBeanDefinition mbd,也就是上面生成的 `MergedBeanDefinition` ,专门有一个 `applyMergedBeanDefinitionPostProcessors` 调用，这里就是容器中注册的 `MergedBeanDefinitionPostProcessor` 的应用阶段 ：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, 
	final @Nullable Object[] args)	throws BeanCreationException {
        // ...

    // 创建bean POJO 对象
    instanceWrapper = createBeanInstance(beanName, mbd, args);

    // ...        
    
    // 修改 merged bean definition 的 BeanPostProcessor 的执行	
      // ==== 调用 MergedBeanDefinitionPostProcessor ====
    applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName); 
  
      // ...

    // 填充 bean 属性:依赖注入处理，属性设置
    populateBean(beanName, mbd, instanceWrapper);

      // ...
    
    // 初始化 bean : 调用设置的初始化方法，接口定义的初始化方法，
    // 以及相应的 pre-/post-init 生命周期回调函数
    initializeBean(beanName, exposedObject, mbd);       

    // ...

    // 如果当前 bean 实现类有关销毁时的接口或者函数，将它进行相应的登记
    // 供容器关闭时执行相应的回调函数	
    registerDisposableBeanIfNecessary(beanName, bean, mbd);                   
}
```

```java
// 找到容器中注册的所有BeanPostProcessor中每一个MergedBeanDefinitionPostProcessor，
// 将它们应用到指定的RootBeanDefinition mbd上，这里 mbd 其实就是一个 MergedBeanDefinition。
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, 
    Class<?> beanType, String beanName) {
  for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof MergedBeanDefinitionPostProcessor) {
      MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
      bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
    }
  }
}
```
