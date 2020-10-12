---
title: Ioc-Spring-核心接口
tags:
  - Java
  - Spring
  - Ioc
categories: Spring
index_img: img/spring-logo-480.png
abbrlink: 29662
date: 2019-11-09 09:28:07
---

# Ioc-Spring核心接口

## 内容大纲

> 1. 本章主要讲述Spring Ioc中几个比较重要的接口
>    1. BeanFactory的含义及其使用
>    2. BeanFactoryPostProcessor的作用以及使用
>    3. BeanPostProcessor的作用以及使用
>    4. BeanDefinitionReader的作用以及使用
>    5. FactoryBean的作用以及使用

## 简介

我们一般配置Bean的方式有两种：1、使用Xml进行Bean的配置，2、使用@Component，@Bean等注解进行Bean的配置。

当然Spring为了处理这些不同的配置定义了一个接口（`BeanDefinitionReader`）来统一将这些配置加载并生成BeanDefinition。

当BeanDefinition被加载后，我们出于某些因素需要修改BeanDefinition，Spring向我们提供了`BeanFactoryPostProcessor`回调接口来修改BeanDefinition（在BeanDefinition被加载进BeanFactory时进行回调）。

接着实例化具体的Bean并放入到BeanFactory中，实例化后我们可能需要对实例进行一些包装，比如 AOP。Spring向我们提供了BeanPostProcessor回调接口来对Bean实例进行包装（在Bean实例被创建后进行回调）。

此时Ioc容器已经加载完毕了，但是上面的步骤都是在Bean实例化这个步骤前后进行操作，那我们怎么在操作实例化这个步骤呢？

Spring为提供了FactoryBean这个接口，让开发人员自己实现创建Bean实例的步骤。



## 坐标

Gradle 坐标

```groovy
testCompile 'junit:junit:4.13'

compile 'org.aspectj:aspectjweaver:1.9.5'

compile 'org.springframework:spring-beans:5.2.2.RELEASE'
compile 'org.springframework:spring-aop:5.2.2.RELEASE'
compile 'org.springframework:spring-context:5.2.2.RELEASE'
```



Maven坐标

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>5.2.2.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.2.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>5.2.2.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.5</version>
</dependency>
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.5</version>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.13</version>
    <scope>test</scope>
</dependency>
```



## 核心接口

Spring 框架中

`BeanDefinitionReader`：读取XML或注解配置的Bean并生成BeanDefinition

`BeanFactoryPostProcesser`：BeanFactory加载BeanDefinition后增强的扩展接口，可以`新增/修改BeanDefinition`。

`BeanPostProcesser`：BeanFactory实例化Bean后增强的扩展接口，可以包装Bean，例如AOP。

`BeanFactory`：Spring Ioc的顶级接口，主要负责创建，实例化，管理Bean实例。

`FactoryBean`：Spring提供给开发人员自定义实例化Bean的接口。



### BeanFactory

#### 概述

BeanFactory是访问Spring Bean容器的根接口。该接口由包含多个Bean定义的对象实现，每个定义均由String名称唯一标识。

它负责实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。



#### 源代码

```java
public interface BeanFactory {

  // 用于区分FactoryBean实例，下面讲解FactoryBean接口时会讲到
  String FACTORY_BEAN_PREFIX = "&";

  // 根据Bean名称查找并获取Bean
  Object getBean(String name) throws BeansException;

  // 根据Bean名称和指定类型查找并获取Bean
  <T> T getBean(String name, Class<T> requiredType) throws BeansException;

  // 返回名称查找并返回实例
	// 允许在bean定义指定明确的构造器参数/工厂方法的参数，重写指定的默认参数（如果有的话）
  Object getBean(String name, Object... args) throws BeansException;

  // 根据类型返回指定实例
  <T> T getBean(Class<T> requiredType) throws BeansException;

  // 同上
  <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

  // 返回一个提供者指定的Bean，允许懒按需检索的实例，包括可用性和唯一选择。
  <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

  // 同上
  <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

  // 是否存在Bean
  boolean containsBean(String name);

  // 是否单例Bean
  boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

 	// 是否多例Bean
  boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

  // 检查具有给定名称的Bean是否与指定的类型匹配。
  boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	// 同上
  boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
  
	// 确定与给定名称的bean的类型。 更具体地讲，确定对象的该类型getBean将返回给定名称
  Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	// 同上
  Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;
 
  // 返回指定bean名字实例的所有别名，如果有的话.
  String[] getAliases(String name);
}
```

我们常用的方法就是重载的`getBean()`。



#### Bean生命周期

##### 初始生命周期

1. **BeanNameAware**：将Bean在容器中的名称

2. **BeanClassLoaderAware**：将加载Bean的类加载器提供给Bean的回调接口
3. **BeanFactoryAware**：将管理当前Bean的BeanFactory返回给Bean的回调接口
4. **EnvironmentAware**：返回当前Spring的环境变量的回调接口
5. **EmbeddedValueResolverAware**：
6. **ResourceLoaderAware**： 资源加载器的回调接口
7. **ApplicationEventPublisherAware**：应用事件发送器获取的回调接口
8. **MessageSourceAware**：MessageSource获取的回调接口
9. **ApplicationContextAware**：Application Context获取的回调接口
10. **ServletContextAware**：Servlet Context获取的
11. **BeanPostProcessor.postProcessBeforeInitialization** ：在Bean初始化前执行
12. **InitializingBean**：由BeanFactory设置完所有属性后执行
13. **自定义的初始化方法定义**：`@Bean(initMethod = "init",destroyMethod = "destroy")`
14. **BeanPostProcessor.postProcessAfterInitialization**：在Bean初始化后执行

##### 销毁生命周期

1. **DestructionAwareBeanPostProcessor.postProcessBeforeDestruction**
2. **DisposableBean.destroy**
3. **自定义的销毁方法**：`@Bean(initMethod = "init",destroyMethod = "destroy")`



##### 总结

初始化流程：

- 调用目标对象构造器创建对象
- 通过一系列Aware接口注入一些常用的属性
- 执行BeanPostProcessor.postProcessBeforeInitialization方法
- 执行InitializingBean.afterPropertiesSet方法
- 执行Bean自定义的初始化方法
- 执行BeanPostProcessor.postProcessAfterInitialization方法

销毁流程：

- 执行DestructionAwareBeanPostProcessors.postProcessBeforeDestruction方法
- 执行DisposableBean.destroy方法
- 执行Bean自定义的销毁方法



### BeanFactoryPostBeanPostProcessor

#### 概述

`BeanFactoryPostBeanPostProcessor`是一个回调接口，可以在BeanDefinition被加载到BeanFactory后如果想再次对BeanDefinition进行修改就可以实现此接口。Spring会在加载完BeanDefinition后回调此接口。



注意：再此接口实现中不能与Bean实例进行交互。



#### 源代码

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {

/**
  * 在标准初始化后修改ApplicationContext内的BeanFactory，所有的BeanDefinition都被加载，但没有被创建。
  * 可以覆盖或添加属性，甚至可以用于初始化bean
  */
  void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```



#### 作用

BeanFactoryPostProcessor主要用于当BeanDefinition被加载到BeanFactory中后，对BeanDefinition进行修改。



#### 示例代码

```java
@Configuration
@ComponentScan(basePackages = "io.better.spring.ioc")
public class IocConfiguration {

    @Bean
    public IocBean iocBean() {
        return new IocBean();
    }
}
```

```java
public class IocBean {

  private String name;

  public IocBean() {
    System.out.println("IocBean init");
  }
	// 忽略get/set方法
  public void init() {
    System.out.println("IocBean custom init");
  }
  public void destroy() {
    System.out.println("IocBean custom destroy");
  }
}
```

在IocConfiguration配置中我们并没有指定IocBean的初始化和销毁方法，以及懒加载

```java
@Component
public class IocBeanFactoryProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("BeanFactoryPostProcessor .....");

        BeanDefinition iocBeanDefinition = beanFactory.getBeanDefinition("iocBean");

        iocBeanDefinition.setInitMethodName("init");
        iocBeanDefinition.setDestroyMethodName("destroy");
        iocBeanDefinition.setLazyInit(true);
    }
}
```

在BeanFactoryPostProcessor实现中我们为IocBean指定了初始化和销毁的方法，以及懒加载。

##### 测试代码

```java
@Test
public void testIocBeanFactoryPostProcessor() {
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(IocConfiguration.class);

    IocBean bean = (IocBean) applicationContext.getBean("iocBean");

    System.out.println(bean);
    applicationContext.close();
}
```

##### 测试结果

```txt
BeanFactoryPostProcessor .....
IocBean Constructor
IocBean custom init
io.better.spring.ioc.IocBean@9225652
IocBean custom destroy
```

从测试结果可以看出，我们在BeanFactoryPostProcessor中成功的操作了已经被BeanFactory加载的BeanDefinition并对其进行了修改。



**到这里你可能会想既然能修改BeanDefinition，那能不能注册一个BeanDefinition呢？**

要想往BeanFactory中注册一个BeanDefinition需要实现接口`BeanDefinitionRegistryPostProcessor`。

该接口是BeanFactoryPostProcessor的子接口。



#### `BeanDefinitionRegistryPostProcessor`



##### 源代码

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
  // 
  void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```

##### 示例代码

```java
@Component
public class IocBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    System.out.println("BeanDefinitionRegistryPostProcessor");
    
    // 创建IocRegistryTestBean的BeanDefinition
    BeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(IocRegistryTestBean.class)
      .addPropertyValue("name", "registry post processor inject")
      .getBeanDefinition();
		// 注册进BeanFactory
    registry.registerBeanDefinition("iocRegistryTestBean", beanDefinition);
  }

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {


    BeanDefinition iocRegistryTestBean = beanFactory.getBeanDefinition("iocRegistryTestBean");

    iocRegistryTestBean.setInitMethodName("init");
    iocRegistryTestBean.setDestroyMethodName("destroy");
    iocRegistryTestBean.setLazyInit(true);
  }
}
```

```java
@Configuration
@ComponentScan(basePackages = "io.better.spring.ioc")
public class IocConfiguration {
}
```



##### 测试代码

```java
@Test
public void testIocBeanFactoryPostProcessor() {
  AnnotationConfigApplicationContext applicationContext = 
    																new AnnotationConfigApplicationContext(IocConfiguration.class);

  IocRegistryTestBean iocRegistryTestBean = 
    														(IocRegistryTestBean) applicationContext.getBean("iocRegistryTestBean");

  System.out.println(iocRegistryTestBean);
  applicationContext.close();
}
```

##### 测试结果

```txt
BeanDefinitionRegistryPostProcessor
BeanFactoryPostProcessor .....
IocRegistryTestBean custom init
IocRegistryTestBean{name='registry post processor inject'}
IocRegistryTestBean custom destroy
```

从执行结果看出在`BeanDefinitionRegistryPostProcessor`想BeanFactory中注册了一个IocRegistryTestBean实例，并指定了其初始化和销毁的方法以及懒加载。



`BeanDefinitionRegistryPostProcessor`比`BeanFactoryPostProcessor`先执行。



#### 其他

ConfigurationClassPostProcessor作用



### BeanDefinitionReader

#### 概述

BeanDefinitionReader是一个接口，主要是读取XML或注解标识的类并将其转成BeanDefinition，提供了使用`Resource`和`String location`参数指定加载方法。

此接口是个规范接口，具体的BeanDefinitionReader无需实现此接口。BeanDefinition读取的简单接口。



#### 源代码

```java
public interface BeanDefinitionReader {

  /**
    * 返回通过BeanDefinitionRegistry接口暴露的BeanFactory去注册BeanDefinition
    */
  BeanDefinitionRegistry getRegistry();

  @Nullable
  ResourceLoader getResourceLoader();

  // 返回用于Bean类的类加载器。
  // null建议不要急于加载Bean类，而只是用类名注册Bean定义，并在以后解析（或永不解析）相应的类。
  @Nullable
  ClassLoader getBeanClassLoader();

  // 返回BeanNameGenerator用于匿名Bean（未指定显式Bean名称）
  BeanNameGenerator getBeanNameGenerator();
	
  // 从指定的Resource中加载BeanDefinition
  int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;

  // 从指定的Resource数组中加载BeanDefinition
  int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;

  // 同上
  int loadBeanDefinitions(String location) throws BeanDefinitionStoreException;
	
  // 同上
  int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;
}
```



#### 实例代码

我们以`XmlBeanDefinitionReader`为例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean name="iocBean" class="io.better.spring.ioc.IocBean" init-method="init" destroy-method="destroy">
        <property name="name" value="test-xml"/>
    </bean>
</beans>
```

```java
public GenericXmlApplicationContext(String... resourceLocations) {
   load(resourceLocations);
   refresh();
}
```

```java
public void load(String... resourceLocations) {
   this.reader.loadBeanDefinitions(resourceLocations);
}
```

```java
private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
```

可以看出`GenericXmlApplicationContext`中持有了`XmlBeanDefinitionReader`的引用，并在初始化时调用了XmlBeanDefinitionReader的loadBeanDefinitions方法加载了BeanDefinition。

#### 测试代码

```java
@Test
public void testIocBeanDefinitionReader() {
    GenericXmlApplicationContext applicationContext = new GenericXmlApplicationContext("Ioc.xml");

    System.out.println(applicationContext.getBean("iocBean"));
}
```



### BeanPostProcessor

#### 概述

BeanPostProcessor为一个接口，用于在BeanFactory实例化Bean后对Bean进行扩展需要实现的接口，Spring会在初始化每个Bean后会回调这个接口。

如果有多个`BeanPostProcessor`实例，我们可以通过设置`order`属性或实现`Ordered`接口来控制执行顺序。

BeanPostProcessor接口由两个回调方法组成，即`postprocessbeforeinitialize()`和`postprocessafterinitialize()`。

我们可以对实例化的Bean进行包装或修改，例如`Aop`（使用的是`AbstractAdvisingBeanPostProcessor`类）就是一个很好的例子。



#### 源代码

```java
public interface BeanPostProcessor {

  // 在InitializingBean.afterPropertiesSet或自定义初始化方法之前应用这个BeanPostProcessor
  // 此时该Bean已经填充了属性值。
  // 这个方法可以对原始实例进行包装，默认实现按原样返回给定的bean。
  default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }

  // 在InitializingBean.afterPropertiesSet或自定义初始化方法之后应用这个BeanPostProcessor
  // 此时该Bean已经填充了属性值。
  // 对于FactoryBean，将为FactoryBean实例和由FactoryBean创建的对象（从Spring 2.0开始）调用此回调。 
  // post-processor可以通过FactoryBean检查的相应bean实例来决定是应用到FactoryBean还是创建的对象，还是两者都应用。
  // 与所有其他BeanPostProcessor回调相反，此回调还将在
  // InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation方法触发短路后被调用。
  default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }
}
```



我们利用BeanPostProcessor来实现一个简单的Aop代理功能，示例代码如下：



#### 示例代码

```java
@Component
public class IocBeanPostProcessor implements BeanPostProcessor {
  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    if (bean instanceof IocBean) {
      IocBean iocBean = (IocBean) bean;
      // 设置Name属性
      iocBean.setName("test-BeanPostProcessor");
      System.out.println("IocBeanPostProcessor Before " + beanName);
    }
    return bean;
  }

  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (bean instanceof IocBean) {
      System.out.println("IocBeanPostProcessor After " + beanName);
      
      // 为IocBean创建代理对象
      IocBean iocBean = (IocBean) bean;

      Enhancer enhancer = new Enhancer();
      enhancer.setSuperclass(IocBean.class);
      enhancer.setCallback(new CglibProxy(iocBean));
      return enhancer.create();
    }
    return bean;
  }
}
// Cglib执行程序
class CglibProxy implements MethodInterceptor {

  private Object proxyTarget;

  public CglibProxy(Object proxyTarget) {
    this.proxyTarget = proxyTarget;
  }

  @Override
  public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {

    System.out.println("BeanPostProcessor Cglib 代理执行前");

    Object invoke = method.invoke(proxyTarget, objects);

    System.out.println("BeanPostProcessor Cglib 代理执行后");
    return invoke;
  }
}
```

代码中在Before中为IocBean的Name属性进行了赋值，在After中为IocBean创建了代理对象。



#### 测试代码

```java
@Test
public void testIocBeanPostProcessor() {
    AnnotationConfigApplicationContext applicationContext = 
      																		new AnnotationConfigApplicationContext(IocConfiguration.class);
    IocBean bean = applicationContext.getBean(IocBean.class);
    bean.say();
}
```

当我们从Application Context获取的Bean实例是在BeanPostProcessor中创建的代理对象。



#### 测试结果

```txt
IocBean Constructor  								 // 初始化bean
IocBeanPostProcessor Before iocBean  // 设置name属性
IocBeanPostProcessor After iocBean	 // 创建Cglib代理
IocBean Constructor									 // cglib代理类调用
// 执行Say方法
BeanPostProcessor Cglib 代理执行前			
IocBean  Say :test-BeanPostProcessor
BeanPostProcessor Cglib 代理执行后
```



### FactoryBean

#### 概述

FactoryBean是一个接口，如果Bean实现此接口，则它将用作对象公开的工厂，而不是直接用作将自身公开的bean实例。

FactoryBeans可以支持单例和原型，并且可以按需延迟创建对象，也可以在启动时急于创建对象。 



注意：实现此接口的Bean不能用作普通Bean。 FactoryBean以Bean样式定义（名称，别名和普通Bean一致），但是为Bean引用公开的对象始终是由`Factory.getObject()`方法创建的对象。



#### 作用

在Spring中我们注册实例化Bean的方式一般都是使用`@Component，@Bean`等注解将Bean注册到容器中，整个过程对于开发人员来说是不可见的（除非阅读源码）。Spring为了更好的扩展提供了`FactoryBean`这个接口让开发人员可以自定义Bean的实例化和注册过程。

实现了`FactoryBean<T>`接口的Bean，Spring会向容器中注册两个Bean，一个是FactoryBean实例本身，一个是`FactoryBean.getObject()`方法返回值所代表的Bean。

根据该Bean的ID从BeanFactory中获取的实际上是`FactoryBean.getObject()`返回的对象，而不是FactoryBean本身，如果要获取FactoryBean对象，请在id前面加一个`&`符号来获取（和BeanFactory中的静态变量`FACTORY_BEAN_PREFIX`对应）。



#### 源代码

```java
public interface FactoryBean<T> {

  String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";
	
  // 调用获取对象，可以是任意类型的对象
  T getObject() throws Exception;

  // 获取对象类型
  Class<?> getObjectType();
	
  // 是否是单例
  default boolean isSingleton() {
    return true;
  }
}
```



#### 示例代码

```java
public class IocBean implements FactoryBean<Object> {

  private String name;

  public IocBean() {
    System.out.println("IocBean init");
    System.out.println(this);
  }

  public void init() {
    System.out.println("IocBean custom init");
  }
  public void destroy() {
    System.out.println("IocBean custom destroy");
  }

  @Override
  public Object getObject() throws Exception {
    System.out.println("FactoryBean Create IocBean Instance");
    return new IocBean();
  }

  @Override
  public Class<?> getObjectType() {
    return IocBean.class;
  }
}

```

##### 测试代码

```java
@Test
public void testIoc() {
  AnnotationConfigApplicationContext applicationContext = 
    																			new AnnotationConfigApplicationContext(IocConfiguration.class);

  IocBean bean = (IocBean) applicationContext.getBean("iocBean");
  IocBean factoryBean = (IocBean) applicationContext.getBean("&iocBean");

  System.out.println(bean);
  System.out.println(factoryBean);
  applicationContext.close();
}
```

##### 执行结果

```txt
// 第一次初始化
IocBean init
io.better.spring.ioc.IocBean@159f197
// BeanFactory.getObject()执行
FactoryBean Create IocBean Instance
// 第二次初始化
IocBean init
io.better.spring.ioc.IocBean@6fe7aac8
// sout输出内容
io.better.spring.ioc.IocBean@6fe7aac8
io.better.spring.ioc.IocBean@159f197
```

##### 结论

从执行结果可以看出IocBean的实例对象是由`FactoryBean.getObject()`方法创建的。

通过打印从`容器获取FactoryBean实例`和`构造器中打印语句`得出：IocBean类被实例化了两次

- 第一次实例化的是FactoryBean实例
- 第二次实例化的是IocBean实例（是通过FactoryBean.getObject方法创建的）











