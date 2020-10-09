---
title: Ioc-Spring-Bean定义
tags:
  - Java
  - Spring
  - Ioc
categories: Spring
index_img: 'https://better-io-blog.oss-cn-beijing.aliyuncs.com/20201009092147.png'
abbrlink: 14358
date: 2020-10-09 09:34:43
---


# Ioc-BeanDefinition



## 内容大纲

> 本文主要讲述BeanDefinition的作用，以及BeanDefinition子接口或实现类的使用和介绍。
>
> 不会涉及BeanDefinitionReader（读取生成BeanDefinition）和BeanDefinitionRegistry（注册BeanDefinition）相关的东西，只关注BeanDefinition本身的东西。



## BeanDefinition

### 前言



### 概述

BeanDefinition描述了一个bean实例，该实例具有属性值，构造函数参数值以及具体实现所提供的更多信息。

BeanDefinition主要是用来描述Bean，里面存放Bean元数据：比如`Bean类名、scope、属性、构造函数参数列表、依赖的Bean、是否是单例类、是否是懒加载`等一些列信息。



### 类继承图

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200517180454.png" alt="image-20200513192525817" style="zoom:50%;" />



### 源代码

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
	// 设置Bean实例在Ioc中的名称
  void setBeanClassName(@Nullable String beanClassName);
  String getBeanClassName();
  
	// 设置是否单例
  void setScope(@Nullable String scope);
  String getScope();

  // 设置是否懒加载
  void setLazyInit(boolean lazyInit);
  boolean isLazyInit();

  // 设置当前Bean依赖的Bean
  void setDependsOn(@Nullable String... dependsOn);
  String[] getDependsOn();

  // 当前Bean是否是唯一
  void setPrimary(boolean primary);
  boolean isPrimary();
  
  // 返回此bean的构造函数参数值。
  // ConstructorArgumentValues包装了当前Bean的构造函数参数值
  ConstructorArgumentValues getConstructorArgumentValues();
  
  // 返回要应用到Bean的新实例的属性值。
  // MutablePropertyValues包装了当前Bean的属性值
  MutablePropertyValues getPropertyValues();

  // 设置自定义初始化方法
  void setInitMethodName(@Nullable String initMethodName);
  String getInitMethodName();

  // 设置自定义销毁方法
  void setDestroyMethodName(@Nullable String destroyMethodName);
  String getDestroyMethodName();

  // 是否单例
  boolean isSingleton();
  // 是否多例
  boolean isPrototype();
  // 是否抽象
  boolean isAbstract();

  // 返回原始的BeanDefinition；如果没有，则返回null。 允许获取修饰的bean定义（如果有）。
  // 请注意，此方法返回直接发起者。 遍历发起者链以找到用户定义的原始BeanDefinition。
  BeanDefinition getOriginatingBeanDefinition();
}
```

代码中BeanDefinition继承了`AttributeAccessor`, `BeanMetadataElement`两接口，让我们来看一下这两个接口。

### `BeanMetadataElement`

```java
public interface BeanMetadataElement {
  // 返回此元数据元素的配置源Object（可以为null）。
  default Object getSource() {
    return null;
  }
}
```



### `AttributeAccessor`

```java
// 定义用于将元数据附加到任意对象或从任意对象访问元数据的通用协定的接口。
public interface AttributeAccessor {

  // 将名称定义的属性设置为提供的值。 如果value为null，则删除该属性。
  void setAttribute(String name, @Nullable Object value);
	// 获取由名称标识的属性的值。 如果属性不存在，则返回null。
  Object getAttribute(String name);
  // 删除由名称标识的属性的值。 如果属性不存在，则返回null。
  Object removeAttribute(String name);
  // 存在由名称标识的属性的值返回true，否则返回false
  boolean hasAttribute(String name);
	// 返回所有属性的名称。
  String[] attributeNames();
}
```



### `AbstractBeanDefinition`

##### 概述

AbstractBeanDefinition是BeanDefinition最完整的实现，内部提供了大量的的属性字段来封装Bean的元数据信息。

##### 源代码

```java
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
  implements BeanDefinition, Cloneable {
	// 忽略部分常量

	// 属性字段
  // BeanDefinition对应的Class
  private volatile Object beanClass;

  private String scope = SCOPE_DEFAULT;

  private boolean abstractFlag = false;

  private Boolean lazyInit; // 是否懒加载
  private int autowireMode = AUTOWIRE_NO;   // 自动注入模式
  private int dependencyCheck = DEPENDENCY_CHECK_NONE;  // 依赖检查
  private String[] dependsOn;			// 依赖
  private boolean autowireCandidate = true;
  private boolean primary = false;	
  private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();
  private Supplier<?> instanceSupplier;
  private boolean nonPublicAccessAllowed = true;
  private boolean lenientConstructorResolution = true;
  private String factoryBeanName;
  private String factoryMethodName;
  private ConstructorArgumentValues constructorArgumentValues;  // 构造函数参数
  private MutablePropertyValues propertyValues;				// 属性参数
  private MethodOverrides methodOverrides = new MethodOverrides();
  private String initMethodName;
  private String destroyMethodName;
  private boolean enforceInitMethod = true;
  private boolean enforceDestroyMethod = true;
  private boolean synthetic = false;
  private int role = BeanDefinition.ROLE_APPLICATION;
  private String description;
  private Resource resource;
  
  // 忽略属性get/set方法
  
  @Override
	public Object clone() {
		return cloneBeanDefinition();
	}
	// 克隆BeanDefinition
	public abstract AbstractBeanDefinition cloneBeanDefinition();
}
```



#### `GenericBeanDefinition`

##### 概述

GenericBeanDefinition是一站式的用于标准bean定义。 像任何bean定义一样，它允许指定一个类以及可选的构造函数参数值和属性值。通过其`parentName`属性灵活的指定父BeanDefinition。

##### 源代码

```java
public class GenericBeanDefinition extends AbstractBeanDefinition {

  @Nullable
  private String parentName;

  public GenericBeanDefinition() {
    super();
  }

  public GenericBeanDefinition(BeanDefinition original) {
    super(original);
  }
	
  // 实现克隆BeanDefinition
  public AbstractBeanDefinition cloneBeanDefinition() {
    return new GenericBeanDefinition(this);
  }

  @Override
  public boolean equals(@Nullable Object other) {
    if (this == other) {
      return true;
    }
    if (!(other instanceof GenericBeanDefinition)) {
      return false;
    }
    GenericBeanDefinition that = (GenericBeanDefinition) other;
    return (ObjectUtils.nullSafeEquals(this.parentName, that.parentName) && super.equals(other));
  }
}
```

可以看出GenericBeanDefinition继承了AbstractBeanDefinition类，新增了一个`parentName`属性。GenericBeanDefinition因为这个属性可以动态定义父依赖项，而不是将角色“硬编码”为根bean定义。

##### 示例代码

```java
@Test
public void testGenericBeanDefinition() {

  AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();

  GenericBeanDefinition parentBeanDefinition = new GenericBeanDefinition();
  parentBeanDefinition.setBeanClass(IocParentBean.class);
  parentBeanDefinition.setBeanClassName(IocParentBean.class.getName());
  parentBeanDefinition.setInitMethodName("init");

  applicationContext.registerBeanDefinition("iocParentBean", parentBeanDefinition);

  GenericBeanDefinition childBeanDefinition = new GenericBeanDefinition();
  childBeanDefinition.setBeanClass(IocBean.class);
  childBeanDefinition.setParentName(parentBeanDefinition.getBeanClassName());

  applicationContext.registerBeanDefinition("iocBean", childBeanDefinition);

  System.out.println(applicationContext.getBeanDefinition("iocParentBean"));
  System.out.println(applicationContext.getBeanDefinition("iocBean"));
}
```

##### 示例结果

```txt
Generic bean: class [io.better.spring.ioc.IocParentBean]; scope=; abstract=false; lazyInit=null; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=init; destroyMethodName=null
 
Generic bean with parent 'io.better.spring.ioc.IocParentBean': class [io.better.spring.ioc.IocBean]; scope=; abstract=false; lazyInit=null; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null
```



#### `RootBeanDefinition`

##### 概述

一个RooBeanDefinition表示合并的BeanDefinition，该定义在运行时支持Spring BeanFactory中的特定bean。它可能是由多个相互继承的原始bean定义创建的，通常定义为GenericBeanDefinitions。`RootBeanDefinition`本质上是运行时的 "统一" BeanDefinition 视图。

但是，从Spring 2.5开始，以编程方式注册bean定义的首选方法是GenericBeanDefinition类。

##### 源代码

```java
public class RootBeanDefinition extends AbstractBeanDefinition {

	// BeanDefinitionHolder存储有Bean的名称、别名、BeanDefinition
  private BeanDefinitionHolder decoratedDefinition;
  // AnnotatedElement表示此VM中当前正在运行的程序的带注释元素。方便使用反射读取注释信息
  private AnnotatedElement qualifiedElement;
  volatile boolean stale; 	// 确定是否需要重新合并定义
  boolean allowCaching = true;
  boolean isFactoryMethodUnique = false;
  volatile ResolvableType targetType;
  volatile Class<?> resolvedTargetType; // 缓存给定BeanDefinition的对应的Class。
  volatile Boolean isFactoryBean; // 如果该bean是工厂bean，则进行缓存。
  volatile ResolvableType factoryMethodReturnType;	// 缓存通用类型的工厂方法的返回类型。
  volatile Method factoryMethodToIntrospect;  // 缓存用于自省的唯一工厂方法候选。
  
  final Object constructorArgumentLock = new Object(); // 以下四个构造函数字段的通用锁。
  Executable resolvedConstructorOrFactoryMethod; // 缓存已解析的构造函数或工厂方法
  boolean constructorArgumentsResolved = false; // 将构造函数参数标记为已解析。
  Object[] resolvedConstructorArguments; // 构造函数解析的参数数组
  Object[] preparedConstructorArguments;  // 缓存部分准备好的构造函数参数。

  final Object postProcessingLock = new Object(); // 以下两个后处理字段的通用锁

  boolean postProcessed = false;  // 指示是否应用MergedBeanDefinitionPostProcessor
  volatile Boolean beforeInstantiationResolved;  // 表示实例化之前的后处理器已启动

  private Set<Member> externallyManagedConfigMembers;
  private Set<String> externallyManagedInitMethods;
  private Set<String> externallyManagedDestroyMethods;
}
```

RootBeanDefiniiton保存了以下信息：

1. 持有的BeanDefinitionHolder定义了id、别名与Bean的对应关系。
2. AnnotatedElement获取Bean的注解信息。
3. 具体的工厂方法（Class类型），包括工厂方法的返回类型，工厂方法的Method对象
4. 缓存了构造函数、构造函数参数。



#### `ChildBeanDefinition`

从其父级继承设置的Bean的BeanDefinition。ChildBeanDefinition对父beanDefinition有固定的依赖性。ChildBeanDefinition将从父对象继承构造函数参数值，属性值和方法替代，并可以选择添加新值。如果指定了init方法，destroy方法和/或静态工厂方法，则它们将覆盖相应的父设置。其余设置将始终从子定义中获取：取决于，自动装配模式，依赖项检查，单例，懒加载。



从Spring 2.5开始，以编程方式注册Bean定义的首选方法是GenericBeanDefinition类。



------

### `AnnotatedBeanDefinition`



AnnotatedBeanDefinition扩展了BeanDefinition，可向外暴露Bean的`AnnotationMetadata`信息，无需加载Bean。

```java
public interface AnnotatedBeanDefinition extends BeanDefinition {
	// 返回Bean的注解元数据信息
  AnnotationMetadata getMetadata();
	
  // 获取此bean定义的factory方法的元数据（如果有）
  MethodMetadata getFactoryMethodMetadata();
}
```



#### `ScannedGenericBeanDefinition`

##### 概述

基于ASM ClassReader的GenericBeanDefinition类的扩展，支持通过AnnotatedBeanDefinition接口公开的注解元数据。此类不会尽早加载Bean类。而是从ASM ClassReader解析的“ .class”文件本身中检索所有相关的元数据。

它在功能上等效于AnnotatedGenericBeanDefinition.AnnotatedGenericBeanDefinition（AnnotationMetadata），但按类型区分已扫描的bean和已通过其他方式注册或检测的bean。

```java
public class ScannedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {

  private final AnnotationMetadata metadata;

  public ScannedGenericBeanDefinition(MetadataReader metadataReader) {
    Assert.notNull(metadataReader, "MetadataReader must not be null");
    // 获取到注解元数据信息
    this.metadata = metadataReader.getAnnotationMetadata();
    setBeanClassName(this.metadata.getClassName());
  }

  @Override
  public final AnnotationMetadata getMetadata() {
    return this.metadata;
  }
}
```

`MetadataReader`：用于访问类元数据的简单入口，由`ASM org.springframework.asm.ClassReader`读取。



##### 示例代码

```java
@Component("iocBean1")
@Order(1)
public class IocBean {
```



```java
@Test
public void testScannedGenericBeanDefinition() throws Exception {
  SimpleMetadataReaderFactory simpleMetadataReaderFactory = new SimpleMetadataReaderFactory();
  MetadataReader metadataReader = 
    											simpleMetadataReaderFactory.getMetadataReader("io.better.spring.ioc.IocBean");

  ScannedGenericBeanDefinition beanDefinition = new ScannedGenericBeanDefinition(metadataReader);

  AnnotationMetadata metadata = beanDefinition.getMetadata();
  Set<String> annotationTypes = metadata.getAnnotationTypes();

  System.out.println(annotationTypes);
}
```



##### 示例结果

```txt
[org.springframework.stereotype.Component, org.springframework.core.annotation.Order]
```



#### AnnotatedGenericBeanDefinition

##### 概述

AnnotatedGenericBeanDefinition

这个GenericBeanDefinition变体主要用于测试希望在AnnotatedBeanDefinition上运行的代码，例如Spring组件扫描支持中的策略实现（默认定义类是org.springframework.context.annotation.ScannedGenericBeanDefinition，它也实现了AnnotatedBeanDefinition接口） 。



```java
public class AnnotatedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {
  private final AnnotationMetadata metadata;

  private MethodMetadata factoryMethodMetadata;

  public AnnotatedGenericBeanDefinition(Class<?> beanClass) {
    setBeanClass(beanClass);
    this.metadata = AnnotationMetadata.introspect(beanClass);
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



##### 示例代码

```java
@Test
public void testAnnotatedGenericBeanDefinition() {
  AnnotatedGenericBeanDefinition beanDefinition = new AnnotatedGenericBeanDefinition(IocBean.class);
  
  System.out.println(beanDefinition.getMetadata().getAnnotationTypes());
}
```



##### 示例结果

```txt
[org.springframework.stereotype.Component]
```



#### `ConfigurationClassBeanDefinition`

##### 概述

该类是`ConfigurationClassBeanDefinitionReader`中的私有静态内部类。

而`ConfigurationClassBeanDefinitionReader`作用是将`@Configuration`注解标识的类生成`ConfigurationClass`实例，在通过`ConfigurationClassBeanDefinition`将其转换成BeanDefinition并注册到`BeanDefinitionRegistry`中。

而`ConfigurationClassBeanDefinition`的作用就是将`@Configuration`标识的类生成BeanDefinition（XML等其他配置源无效）。



##### 源代码

```java
private static class ConfigurationClassBeanDefinition extends RootBeanDefinition implements AnnotatedBeanDefinition {

  private final AnnotationMetadata annotationMetadata;

  private final MethodMetadata factoryMethodMetadata;

  public ConfigurationClassBeanDefinition(ConfigurationClass configClass, MethodMetadata beanMethodMetadata) 
  {
    this.annotationMetadata = configClass.getMetadata();
    this.factoryMethodMetadata = beanMethodMetadata;
    setLenientConstructorResolution(false);
  }

  public AnnotationMetadata getMetadata() {
    return this.annotationMetadata;
  }
}
```

在ConfigurationClassBeanDefinition构造函数中我们看到ConfigurationClass类，一个ConfigurationClass代表着一个用户定义的@Configuration类。



## 总结

- **AbstractBeanDefinition**：
  - AbstractBeanDefinition是完善且具体的BeanDefinition类的基类，其中排除了GenericBeanDefinition，RootBeanDefinition和ChildBeanDefinition的常用属性。自动装配常数与AutowireCapableBeanFactory接口中定义的常数匹配。
- **GenericBeanDefinition**：
  - GenericBeanDefinition是一站式的用于标准bean定义。 像任何bean定义一样，它允许指定一个类以及可选的构造函数参数值和属性值。
- **AnnotatedBeanDefinition**：
  - 表示注解类型的BeanDefinition，有两个重要的属性，AnnotationMetadata，MethodMetadata分别表示BeanDefinition的注解元信息和方法元信息。
- **RootBeanDefinition**：
  - 代表一个`Xml，Java Config`来的BeanDefinition
- **ChildBeanDefinition**:
  - 可以让子Bean定义拥有从父Bean定义哪里继承配置的能力
- **AnnotatedGenericBeanDefinition**：
  - 表示`@Configuration`注解注释的Bean
- **ScannedGenericBeanDefinition**：
  - 表示`@Component、@Service、@Controller、@Repository`等注解注释的Bean






