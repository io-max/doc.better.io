---
title: Ioc-Spring-BeanDefinition注册
tags:
  - Java
  - Spring
  - Ioc
categories: Spring
index_img: img/spring-logo-480.png
abbrlink: 48793
date: 2020-01-09 09:35:51
---

# Ioc-Bean注册

## 本文内容

> 本文主要讲述BeanDefinitionRegistry接口是如何注册BeanDefinition？
>
> 主要讲解DefaultListableBeanFactory，AnnotationConfigApplicationContext，GenericApplicationContext三个上下文是如何对BeanDefinition进行注册的。

## BeanDefinitionRegistry

### 前言

上面讲到了BeanDefinition的读取方式，拿到BeanDefinition后需要将其注册到容器中，Spring是用什么将其注册到容器中的呢？

没错就是上面源代码中频繁提到的BeanDefinitionRegistry对象。



### 简介

BeanDefinitionRegistry是包含BeanDefinition的注册的接口，这是Spring Bean工厂软件包中唯一封装Bean定义注册的接口。

标准BeanFactory接口仅涵盖对完全配置的工厂实例的访问。Spring的`BeanDefinitionReader`希望可以使用此接口的实现。

Spring核心中的已知实现者是`DefaultListableBeanFactory`和`GenericApplicationContext`。



### 类继承图

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200517180016.png" alt="image-20200517161204618" style="zoom:50%;" />

我们看到了`DefaultListableBeanFactory`、`GenericApplicationContext`、`AnnotationConfigApplicationContext`等类。



重点关注上面几个类的实现。

### 源代码

```java
public interface BeanDefinitionRegistry extends AliasRegistry {
  // 注册BeanDefinition
  void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException;

  // 根据指定的Bean名称删除BeanDefinition
  void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

  // 根据指定的Bean名称获取BeanDefinition
  BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

  // 是否包含指定的Bean名称
  boolean containsBeanDefinition(String beanName);

  // 返回注册器中所有的BeanDefinition名称
  String[] getBeanDefinitionNames();

 	// 获取BeanDefinition的数量
  int getBeanDefinitionCount();

  // 指定的Bean名称是否被使用
  boolean isBeanNameInUse(String beanName);
}
```

接口中定义了BeanDefinition增删改查的接口，并继承了AliasRegistry接口。



### `AliasRegistry`

#### 简介

顾名思义AliasRegistry是一个别名注册器，用于注册别名。



#### 源代码

```java
public interface AnnotationConfigRegistry {

  // 注册指定的Class数组
  void register(Class<?>... componentClasses);

  // 注册指定包路径下的Bean
  void scan(String... basePackages);
}
```



### `GenericApplicationContext`

#### 简介

GenericApplicationContext是通用ApplicationContext实现，其内部引用了`DefaultListableBeanFactory`实例，实现BeanDefinitionRegistry接口，以便允许将任何Bean定义读取器应用于该接口。



#### 使用

通用做法是通过BeanDefinitionRegistry接口注册各种Bean定义，然后调用`refresh()`以使用应用程序上下文语义来初始化这些Bean（处理org.springframework.context.ApplicationContextAware，自动检测BeanFactoryPostProcessors等）。



#### 特点

与为每次刷新创建一个新的内部BeanFactory实例的其他ApplicationContext实现相反，此上下文的内部BeanFactory从一开始就可用，以便能够在其上注册BeanDefinition。`refresh()只能被调用一次`。

对于应该以可刷新方式读取特殊bean定义格式的自定义应用程序上下文实现，请考虑从**AbstractRefreshableApplicationContext**基类派生。



#### 源代码

```java
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {
	// BeanFactory，此对象也实现了BeanDefinitionRegistry接口
  private final DefaultListableBeanFactory beanFactory;

  public GenericApplicationContext() {
    this.beanFactory = new DefaultListableBeanFactory();
  }
	
  // 注册BeanDefinition
  public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {

    this.beanFactory.registerBeanDefinition(beanName, beanDefinition);
  }
	// 删除BeanDefinition
  public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
    this.beanFactory.removeBeanDefinition(beanName);
  }
	
  // 获取BeanDefinition
  public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
    return this.beanFactory.getBeanDefinition(beanName);
  }

	// Bean是否被使用
  public boolean isBeanNameInUse(String beanName) {
    return this.beanFactory.isBeanNameInUse(beanName);
  }
	
  // 注册bean别名
  public void registerAlias(String beanName, String alias) {
    this.beanFactory.registerAlias(beanName, alias);
  }

	// 删除bean别名
  public void removeAlias(String alias) {
    this.beanFactory.removeAlias(alias);
  }

  // 是否存在别名
  public boolean isAlias(String beanName) {
    return this.beanFactory.isAlias(beanName);
  }
	
  // 注册Bean
  public <T> void registerBean(@Nullable String beanName, Class<T> beanClass,
                               @Nullable Supplier<T> supplier, BeanDefinitionCustomizer... customizers) {

    ClassDerivedBeanDefinition beanDefinition = new ClassDerivedBeanDefinition(beanClass);
    if (supplier != null) {
      beanDefinition.setInstanceSupplier(supplier);
    }
    for (BeanDefinitionCustomizer customizer : customizers) {
      customizer.customize(beanDefinition);
    }

    String nameToUse = (beanName != null ? beanName : beanClass.getName());
    registerBeanDefinition(nameToUse, beanDefinition);
  }
}
```

`GenericApplicationContext`内部持有了`DefaultListableBeanFactory`引用，并简单的实现了**BeanDefinitionRegistry**的接口，不过最终操作还是交给了`DefaultListableBeanFactory`对象来进行完成。



#### 示例代码

```java
@Test
public void testGenericApplicationContext() {
  GenericApplicationContext applicationContext = new GenericApplicationContext();

  AnnotatedBeanDefinitionReader definitionReader = new AnnotatedBeanDefinitionReader(applicationContext);
  definitionReader.registerBean(IocBean.class);

  applicationContext.refresh();

  IocBean iocBean = applicationContext.getBean(IocBean.class);
  System.out.println(iocBean);
}
```



### `DefaultListableBeanFactory`

#### 简介

DefaultListableBeanFactory是Spring的ConfigurableListableBeanFactory和BeanDefinitionRegistry接口的默认实现，基于BeanDefinition元数据的成熟bean工厂，可通过后处理器进行扩展。



#### 使用

典型的用法是在访问bean之前先注册所有bean定义（可能是从bean定义文件中读取）。因此，按名称查找Bean是对本地Bean定义表进行的廉价操作，该操作对预先解析的Bean定义元数据对象进行操作。对于`ListableBeanFactory`接口的替代实现，请看一下`StaticListableBeanFactory`，它管理现有的bean实例，而不是根据bean定义创建新的bean实例。



#### 源代码

代码中删除了无关的代码，因为我们关注的是`BeanDefinitionRegistry`相关的操作。

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
  implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {

  private boolean allowBeanDefinitionOverriding = true;

  // 管理所有的BeanDefinition
  private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
  // 管理所有的单例Bean或非单例名称，key=依赖的Class
  private final Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<>(64);
  // 管理所有的单例Bean名称，key=依赖的Class
  private final Map<Class<?>, String[]> singletonBeanNamesByType = new ConcurrentHashMap<>(64);
  // 根据注册的顺序，管理所有的BeanDefinition名称
  private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
  // 管理手动注册的单示例Bean
  private volatile Set<String> manualSingletonNames = new LinkedHashSet<>(16);

	// 构造函数
  public DefaultListableBeanFactory() {
    super();
  }

	// 注册BeanDefinition
  public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {
		
		// 判断次BeanDefinition是否被加载过
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) { 		// 被加载过
      // 是否允许BeanDefinition重写
      if (!isAllowBeanDefinitionOverriding()) {
        throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
      }
      // 新的BeanDefinition会覆盖就的BeanDefinition
      this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    // 未被加载过
    else {	
      if (hasBeanCreationStarted()) {		// 是否已经开始创建Bean
        synchronized (this.beanDefinitionMap) {
          // 将BeanDefinition放入到beanDefinitionMap中
          this.beanDefinitionMap.put(beanName, beanDefinition);
          List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
          updatedDefinitions.addAll(this.beanDefinitionNames);
          updatedDefinitions.add(beanName);
          this.beanDefinitionNames = updatedDefinitions;
          removeManualSingletonName(beanName);
        }
      }
      else {
				// 将BeanDefinition放入到beanDefinitionMap中
        this.beanDefinitionMap.put(beanName, beanDefinition);
        this.beanDefinitionNames.add(beanName);
        // 删除
        removeManualSingletonName(beanName);
      }
      this.frozenBeanDefinitionNames = null;
    }
		// 已经存在此BeanDefinition或者此beanName已经存在
    if (existingDefinition != null || containsSingleton(beanName)) {
      // 重置此BeanDefinition
      resetBeanDefinition(beanName);
    }
  }

	// 删除BeanDefinition
  // ①：删除beanDefinitionNames中name，②：调用resetBeanDefinition从beanDefinitionMap中删除
  public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
		// 从Map中删除BeanDefinition
    BeanDefinition bd = this.beanDefinitionMap.remove(beanName);
    if (bd == null) {
      // 删除失败（不存在此BeanDefinition），抛出异常
      throw new NoSuchBeanDefinitionException(beanName);
    }
		// 是否有Bean已经被创建
    if (hasBeanCreationStarted()) {
      // 加锁
      synchronized (this.beanDefinitionMap) {
        List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames);
       	// 删除
        updatedDefinitions.remove(beanName);
        this.beanDefinitionNames = updatedDefinitions;  // 重新赋值
      }
    }
    else {
			// 直接删除
      this.beanDefinitionNames.remove(beanName);
    }
		// 重置BeanDefinition
    resetBeanDefinition(beanName);
  }

	// 重置BeanDefinition
  protected void resetBeanDefinition(String beanName) {
    // 如果存在则删除指定Bean名称对应的合并的BeanDefinition
    clearMergedBeanDefinition(beanName);

    destroySingleton(beanName);
		// 执行过滤出的MergedBeanDefinitionPostProcessor
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
      if (processor instanceof MergedBeanDefinitionPostProcessor) {
        ((MergedBeanDefinitionPostProcessor) processor).resetBeanDefinition(beanName);
      }
    }
		// 遍历BeanDefinition名称Map
    for (String bdName : this.beanDefinitionNames) {
      if (!beanName.equals(bdName)) {
        BeanDefinition bd = this.beanDefinitionMap.get(bdName);
        if (bd != null && beanName.equals(bd.getParentName())) {
          // 重置BeanDefinition
          resetBeanDefinition(bdName);
        }
      }
    }
  }

	// 获取BeanDefinition
	public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
    // 调用beanDefinitionMap.get()
		BeanDefinition bd = this.beanDefinitionMap.get(beanName);
		if (bd == null) {
      // 没有抛出异常
			throw new NoSuchBeanDefinitionException(beanName);
		}
		return bd;
	}
  
	// 是否存在BeanDefinition
	public boolean containsBeanDefinition(String beanName) {
		Assert.notNull(beanName, "Bean name must not be null");
    // 操作成员变量beanDefinitionMap
		return this.beanDefinitionMap.containsKey(beanName);
	}
}
```

`DefaultListableBeanFactory`声明了多个`beanDefinitionNames（List类型）`，`beanDefinitionMap（Map类型）`等属性，它们分别用于存储`BeanDefinition名称`和`BeanDefinition`。

在对`BeanDefinitionRegistry`接口方法的实现中底层操作的都是这两个`beanDefinitionNames`，`beanDefinitionMap`属性。



#### 示例代码

```java
@Test
public void testDefaultListableBeanFactory() {

    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

    BeanDefinition beanDefinition = new AnnotatedGenericBeanDefinition(IocBean.class);
    beanFactory.registerBeanDefinition(IocBean.class.getName(), beanDefinition);

    System.out.println(Arrays.toString(beanFactory.getBeanDefinitionNames()));
}
```



### AnnotationConfigApplicationContext

#### 简介

`AnnotationConfigApplicationContext`是一个独立的应用程序上下文，接受组件类作为输入-特别是使用@Configuration注释的类，还可以使用javax.inject注释使用普通的@Component类型和符合JSR-330的类。



#### 使用

允许使用`register（Class ...）`方法一对一注册类，以及使用`scan（String ...）方法`进行类路径扫描，这两个方法定义在`AnnotationConfigRegistry`接口中。

如果有多个`@Configuration`类，则在`以后的类中定义的@Bean方法将覆盖在先前的类中定义的方法`。可以利用此属性通过额外的@Configuration类有意覆盖某些bean定义。



#### 源代码



```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
	// 注解BeanDefinition的读取器和注册器
  private final AnnotatedBeanDefinitionReader reader;
	// ClassPath下BeanDefinition扫描器，扫描Component，Service，Controller，Repository等注解标识的类 
  private final ClassPathBeanDefinitionScanner scanner;

  public AnnotationConfigApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
  }
	// 使用指定类的Class数组进行初始化
  public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    register(componentClasses);
    refresh();
  }
  // 使用指定包路径数组进行初始化
  public AnnotationConfigApplicationContext(String... basePackages) {
    this();
    scan(basePackages);
    refresh();
  }


  public void register(Class<?>... componentClasses) {
    Assert.notEmpty(componentClasses, "At least one component class must be specified");
    // 调用AnnotatedBeanDefinitionReader注册Bean
    this.reader.register(componentClasses);
  }

  @Override
  public void scan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    this.scanner.scan(basePackages);
  }

	// 重写父类的registerBean方法，因为reader拥有注册Bean的功能
  public <T> void registerBean(@Nullable String beanName, Class<T> beanClass,
                               @Nullable Supplier<T> supplier, BeanDefinitionCustomizer... customizers) {

    this.reader.registerBean(beanClass, beanName, supplier, customizers);
  }
}
```

AnnotationConfigApplicationContext引用了`ClassPathBeanDefinitionScanner`，`AnnotatedBeanDefinitionReader`属性。

并将最终的Bean注册交予了`AnnotatedBeanDefinitionReader`来处理，而`ClassPathBeanDefinitionScanner`则用于扫描类路径下所有带`Component，Service，Controller，Repository`注解的类。



#### 示例代码

```java
@Test
public void testAnnotationConfigApplicationContext() {
  AnnotationConfigApplicationContext applicationContext = 
    																								new AnnotationConfigApplicationContext(IocBean.class);

  IocBean bean = applicationContext.getBean(IocBean.class);
  System.out.println(bean);
}
```

可以看出AnnotationConfigApplicationContext的使用非常的简单，只需简单传递一个要初始化的Bean的Class即可。



### 总结

`GenericApplicationContext`：对BeanDefinitionRegistry做了通用的实现，内部引用了DefaultListableBeanFactory，将操作转发给了DefaultListableBeanFactory。

`DefaultListableBeanFactory`：该类封装了处理BeanDefinition的逻辑，底层操作了许多Map。

`AnnotationConfigApplicationContext`：该类虽然继承了`GenericApplicationContext`类，但却每调用父类的方法，而是将注册Bean的操作转发给了内部的AnnotatedBeanDefinitionReader对象。

`SimpleBeanDefinitionRegistry`：简单的BeanDefinitionRegistry，未内置工厂，可用作测试BeanDefinition 注册。


