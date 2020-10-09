---
title: Ioc-Spring-BeanDefinition读取
tags:
  - Java
  - Spring
  - Ioc
categories: Spring
index_img: 'https://better-io-blog.oss-cn-beijing.aliyuncs.com/20201009092147.png'
abbrlink: 19914
date: 2020-06-09 09:35:35
---

# Ioc-BeanDefinition的读取与注册

## 本文内容

>  本文主要讲述BeanDefinition的读取与注册，主要涉及到的接口是BeanDefinitionReader和BeanDefinitionRegistry。
>
>  本文会从这两个接口出发，分析其实现类，不同环境的不同实现，及其优缺点。

## BeanDefinitionReader

### 简介

BeanDefinitionReader接口是个一个BeanDefinition的读取规范定义接口，定义了 "资源" 和 "字符串" 位置参数指定加载方法。

当然，具体的BeanDefinitionReader可以为BeanDefinition添加特定于其BeanDefinition格式的其他加载和注册方法。

具体的BeanDefinitionReader不必实现此接口。它仅对希望遵循标准命名约定的bean定义读者提供建议。

### 类继承图

![image-20200516101622162](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200517180030.png)

老版本中经常使用的是`XMLBeanDefinitionReader`，注解驱动版本中经常使用`AnnotatedBeanDefinitionReader`。

### 源代码

```java
public interface BeanDefinitionReader {

  // 返回一个BeanFactory注册BeanDefinition，其实现了BeanDefinitionRegistry接口
  // 封装了与BeanDefinition处理相关的方法。
  BeanDefinitionRegistry getRegistry();

	// 返回资源加载器以用于资源位置
  // 可以检查ResourcePatternResolver接口并进行相应的转换，以针对给定的资源模式加载多个资源。
  // 返回值为null表示此BeanDefinitionReader无法使用绝对资源加载
  ResourceLoader getResourceLoader();

  // 返回用于加载Bean的类加载器
  // null建议不要急于加载Bean类，而只是用类名注册Bean定义，并在以后（或永不解析）相应的类。
  ClassLoader getBeanClassLoader();

  // 返回BeanNameGenerator用于匿名Bean名称（未指定显式Bean名称）。
  BeanNameGenerator getBeanNameGenerator();

  // 使用指定的Resource加载BeanDefinition
  int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;

  // 使用指定的Resource数组加载BeanDefinition
  int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;

  // 使用指定的location加载BeanDefinition
  int loadBeanDefinitions(String location) throws BeanDefinitionStoreException;

  // 使用指定的location数组加载BeanDefinition
  int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;
}
```

BeanDefinitionReader重载了不同参数的`loadBeanDefinitions`方法用于加载BeanDefinition。





### `AbstractBeanDefinitionReader`

#### 概述

AbstractBeanDefinitionReader是实现了BeanDefinitionReader接口抽象基类。提供常见的属性，例如要处理的BeanFactory以及用于加载bean类的类加载器。



#### 源代码

```java
public abstract class AbstractBeanDefinitionReader implements BeanDefinitionReader, EnvironmentCapable {

  private final BeanDefinitionRegistry registry;	  // BeanDefinition注册器
  private ResourceLoader resourceLoader; 		// 资源加载器
  private ClassLoader beanClassLoader;			// 类加载器
  private Environment environment;					// 环境变量
  private BeanNameGenerator beanNameGenerator = DefaultBeanNameGenerator.INSTANCE;  // Bean名称生成器

  protected AbstractBeanDefinitionReader(BeanDefinitionRegistry registry) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    this.registry = registry;

    // 如果实现了资源加载器接口
    if (this.registry instanceof ResourceLoader) {
      this.resourceLoader = (ResourceLoader) this.registry;
    }else {
      // 创建默认资源加载器
      this.resourceLoader = new PathMatchingResourcePatternResolver();
    }
    // 判断拥有环境变量信息
    if (this.registry instanceof EnvironmentCapable) {
      this.environment = ((EnvironmentCapable) this.registry).getEnvironment();
    }
    else {
      // 创建一个标准的环境变量对象
      this.environment = new StandardEnvironment();
    }
  }
	
  public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int count = 0;
    for (Resource resource : resources) {
      // 最终会调用子类实现的loadBeanDefinitions方法
      count += loadBeanDefinitions(resource);
    }
    return count;
  }
}
```

`AbstractBeanDefinitionReader`对`BeanDefinitionReader`进行了简单的实现，声明`registry`，`resourceLoader`，`beanClassLoader`，`environment`，`beanNameGenerator`等属性。



### `XmlBeanDefinitionReader`

#### 概述

XmlBeanDefinitionReader用于读取XML，将实际的XML文档读取委托给`BeanDefinitionDocumentReader`接口的实现。

此类加载DOM Document 并将BeanDefinitionDocumentReader应用于该Document。

#### 源代码

```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    // BeanDefinitionDocumentReader读取解析Document并转换成BeanDefinition
    private Class<? extends BeanDefinitionDocumentReader> documentReaderClass =
      DefaultBeanDefinitionDocumentReader.class;
    // 命名空间解析器	
    private NamespaceHandlerResolver namespaceHandlerResolver;
    // 将XML文件加载成Document对象
    private DocumentLoader documentLoader = new DefaultDocumentLoader();
    // 用于解析XML中的DTD文件
    private EntityResolver entityResolver;
    // 绑定当前线程正在处理的XML文件集合 
    private final ThreadLocal<Set<EncodedResource>> resourcesCurrentlyBeingLoaded =
      new NamedThreadLocal<>("XML bean definition resources currently being loaded");

    // 构造器
    public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
      super(registry);
    }

    // 实现BeanDefinitionReader定义的接口
    @Override
    public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
      return loadBeanDefinitions(new EncodedResource(resource));
    }

    // 重载的loadBeanDefinitions方法
    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
      // 获取当前线程正在处理的XML集合
      Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
      if (currentResources == null) { // 为空则初始化
        currentResources = new HashSet<>(4);  // 默认大小为4
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
      }
      if (!currentResources.add(encodedResource)) {   // 将当前正在处理的encodedResource添加到Set集合中
        throw new BeanDefinitionStoreException(				// 添加失败报错
          "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
      }
      // 获取到XMl文件的输入流
      InputStream inputStream = encodedResource.getResource().getInputStream();
      // 一个InputSource代表一个XML实体
      InputSource inputSource = new InputSource(inputStream);
      if (encodedResource.getEncoding() != null) {		// 为InputSource设置编码
        inputSource.setEncoding(encodedResource.getEncoding());
      }
      // 调用doLoadBeanDefinitions方法(真正的执行操作)
      return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
    }
    // 忽略部分代码
  }

  // 根据指定的XML文件加载BeanDefinition
  protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    throws BeanDefinitionStoreException {

    try {
      // 使用documentLoader将XML转换成Document
      Document doc = doLoadDocument(inputSource, resource);
      // 注册BeanDefinition
      int count = registerBeanDefinitions(doc, resource);
      return count;
    }
    catch (BeanDefinitionStoreException ex) {}
    catch (SAXParseException ex) {}
    catch (SAXException ex) {}
    catch (ParserConfigurationException ex) {}
    catch (IOException ex) {}
    catch (Throwable ex) {}
  }

  // 将XML转换成Docment
  protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
    return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
                                            getValidationModeForResource(resource), isNamespaceAware());
  }

  // 注册BeanDefinition
  public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 创建BeanDefinition文档读取器，
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
  }

  // 创建BeanDefinitionDocumentReader对象，用于将Document解析成BeanDefinition
  protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
    return BeanUtils.instantiateClass(this.documentReaderClass);
  }

  // XmlReaderContext是ReaderContext的扩展，提供对XmlBeanDefinitionReader中配置的NamespaceHandlerResolver的访问。
  public XmlReaderContext createReaderContext(Resource resource) {
    return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
                                this.sourceExtractor, this, getNamespaceHandlerResolver());
  }

  // 命名空间解析器
  public NamespaceHandlerResolver getNamespaceHandlerResolver() {
    if (this.namespaceHandlerResolver == null) {
      this.namespaceHandlerResolver = createDefaultNamespaceHandlerResolver();
    }
    return this.namespaceHandlerResolver;
  }
}
```

`XMLBeanDefinitionReader`实际上只做了`Document`文档的解析操作，真正的解析BeanDefinition操作交给了`BeanDefinitionDocumentReader`（接口）实例（默认为`DefaultBeanDefinitionDocumentReader`）来解析。



#### 示例代码

Xml

```xml
<bean name="iocBean" class="io.better.spring.ioc.IocBean" init-method="init" destroy-method="destroy">
    <property name="name" value="test-xml"/>
</bean>
```



```java
@Test
public void testXmlBeanDefinitionReader() {
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();

    BeanDefinitionReader definitionReader = new XmlBeanDefinitionReader(applicationContext);
    definitionReader.loadBeanDefinitions("Ioc.xml");

    applicationContext.refresh();

    IocBean iocBean = (IocBean) applicationContext.getBean("iocBean");
    System.out.println(iocBean);
}
```



```txt
IocBean Constructor
IocBean custom init
IocBean{name='test-xml'}
```





#### BeanDefinitionDocumentReader

##### 简介

SPI，`BeanDefinitionDocumentReader`用于解析带有BeanDefinition的Xml Document。



##### 源代码

```java
public interface BeanDefinitionDocumentReader {

  //注册BeanDefinition，doc：当前Xml对应的Document对象，readerContext：当前BeanDefinitionReader的长下文
  void registerBeanDefinitions(Document doc, XmlReaderContext readerContext)
    throws BeanDefinitionStoreException;
}
```



#### DefaultBeanDefinitionDocumentReader

##### 简介

BeanDefinitionDocumentReader的唯一实现类，封装了解析并组装BeanDefinition的具体操作。



##### 源代码

```java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {

  public static final String BEAN_ELEMENT = BeanDefinitionParserDelegate.BEAN_ELEMENT;

	// 常量，标识一些标签
  public static final String NESTED_BEANS_ELEMENT = "beans";
  public static final String ALIAS_ELEMENT = "alias";
  public static final String NAME_ATTRIBUTE = "name";
  public static final String ALIAS_ATTRIBUTE = "alias";
  public static final String IMPORT_ELEMENT = "import";
  public static final String RESOURCE_ATTRIBUTE = "resource";
  public static final String PROFILE_ATTRIBUTE = "profile";
	
  // 当前BeanDefinitionReader的上下文
  private XmlReaderContext readerContext;
	// 解析Document中的值并赋值给BeanDefinition
  private BeanDefinitionParserDelegate delegate;

  // 核心入口
  @Override
  public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    doRegisterBeanDefinitions(doc.getDocumentElement());
  }

  // 核心方法
  // 
  protected void doRegisterBeanDefinitions(Element root) {

    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);

		// 处理Xml前置操作
    preProcessXml(root);
    // 解析BeanDefinition
    parseBeanDefinitions(root, this.delegate);
    // 处理Xml后置操作
    postProcessXml(root);

    this.delegate = parent;
  }
	
  // 解析BeanDefinition核心方法
  protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
      NodeList nl = root.getChildNodes();
      for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (node instanceof Element) { // 是否是元素标签
          Element ele = (Element) node;
          if (delegate.isDefaultNamespace(ele)) {
            // 处理默认元素
            parseDefaultElement(ele, delegate);
          }
          else {
            delegate.parseCustomElement(ele);
          }
        }
      }
    }
    else {
      delegate.parseCustomElement(root);
    }
  }
	
  // 解析默认元素
  private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    // import标签
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
      importBeanDefinitionResource(ele);
    }
    // alias标签
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
      processAliasRegistration(ele);
    }
    // beans标签
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
      processBeanDefinition(ele, delegate);
    }
    // 内嵌bean标签
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
      // recurse
      doRegisterBeanDefinitions(ele);
    }
  }

  // 解析beans标签，并调用registry进行注册
  protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // BeanDefinitionHolder内部包装了一个BeanDefinition
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
      bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
      try {
				// 注册BeanDefinition
        BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
      }
      catch (BeanDefinitionStoreException ex) {}
      // 发布注册完成事件
      getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
  }
}
```

在`DefaultBeanDefinitionDocumentReader`中处理`Bean标签`的逻辑交给了`BeanDefinitionParserDelegate`对象来进行操作，并通过`XmlReaderContext`获取到`Registry`，最后将解析好的BeanDefinition注册进容器。



伴随着Xml方法的繁琐，笨重，Spring在4.2版本后提供了注解配置的方式来替换Xml配置的方法，那么注解是怎样读取 BeanDefinition的呢？

### `AnnotatedBeanDefinitionReader`

#### 概述

AnnotatedBeanDefinitionReader是个方便的适配器，用于以编程方式注册Bean类。这是`ClassPathBeanDefinitionScanner`的替代方法，它应用注解的相同解析，但仅适用于显式注册的类。

**该类不仅仅读取BeanDefinition，同时会注册BeanDefinition。**



#### 源代码

```java
public class AnnotatedBeanDefinitionReader {
	// BeanDefinition注册器
  private final BeanDefinitionRegistry registry;
	// Bean名称生成器
  private BeanNameGenerator beanNameGenerator = AnnotationBeanNameGenerator.INSTANCE;
  private ScopeMetadataResolver scopeMetadataResolver = new AnnotationScopeMetadataResolver();
  // 	
  private ConditionEvaluator conditionEvaluator;

  // 构造函数
  public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
    this(registry, getOrCreateEnvironment(registry));
  }
	
  // 使用指定的Class数组注册Bean
  public void register(Class<?>... componentClasses) {
    for (Class<?> componentClass : componentClasses) {
      registerBean(componentClass);
    }
  }

	// 使用执行的Class注册Bean
  public void registerBean(Class<?> beanClass) {
    doRegisterBean(beanClass, null, null, null, null);
  }

 	// 使用指定的Class和名称注册Bean
  public void registerBean(Class<?> beanClass, @Nullable String name) {
    doRegisterBean(beanClass, name, null, null, null);
  }
  
	// 使用指定的Class和Annotation Class数组注册Bean
  public void registerBean(Class<?> beanClass, Class<? extends Annotation>... qualifiers) {
    doRegisterBean(beanClass, null, qualifiers, null, null);
  }
  
  // 忽略部分重载的registerBean方法

  // 真正执行注册BeanDefinition的地方
  private <T> void doRegisterBean(Class<T> beanClass, String name,
                                  Class<? extends Annotation>[] qualifiers, Supplier<T> supplier,
                                  BeanDefinitionCustomizer[] customizers) {
		// 创建BeanDefinition
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
      return;
    }

    abd.setInstanceSupplier(supplier);
    // 解析BeanDefinition的作用域
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    // 生成Bean名称
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
		
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    // 处理Bean上声明的注解信息
    if (qualifiers != null) {
      for (Class<? extends Annotation> qualifier : qualifiers) {
        if (Primary.class == qualifier) {
          abd.setPrimary(true); // 唯一
        }
        else if (Lazy.class == qualifier) {
          abd.setLazyInit(true); // 懒加载
        }
        else {
          abd.addQualifier(new AutowireCandidateQualifier(qualifier));
        }
      }
    }
    // 定制BeanDefinition，与BeanDefinitionBuilder.applyCustomizers 用法一致
    if (customizers != null) {
      for (BeanDefinitionCustomizer customizer : customizers) {
        customizer.customize(abd);
      }
    }
		// 生成BeanDefinitionHolder
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    // 创建作用域代理
    definitionHolder = 
      AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 向容器中注册Bean
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
  }
}
```

可以看出AnnotatedBeanDefinitionReader注册Bean的方法明显比XmlBeanDefinitionReader的处理方式简单了很多，



#### 示例代码

```java
@Test
public void testAnnotatedBeanDefinitionReader() {
  // 创建ApplicationContext
  AnnotationConfigApplicationContext applicationContext = 
    																							new AnnotationConfigApplicationContext(IocBean.class);
	// 获取Bean	
  IocBean iocBean = applicationContext.getBean(IocBean.class);
  System.out.println(iocBean);
}
```



`AnnotationConfigApplicationContext`类中声明了`AnnotatedBeanDefinitionReader`，`ClassPathBeanDefinitionScanner`对象，并在构造函数中做了初始化。



<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200517180020.png" alt="image-20200517154640954" style="zoom:50%;" />



`ClassPathBeanDefinitionScanner`用于扫描`ClassPath`下带`@Component，@Repository，@Service，@Controller`注解的类。



### 总结

至此BeanDefinitionReader的分析完成，我们一共分析了两个BeanDefinitionReader规范实现，分别对应`Xml读取（XmlBeanDefinitionReader）`和`注解读取（AnnotatedBeanDefinitionReader）`。

`XmlBeanDefinitionReader`：内部将BeanDefinition的读取交给了BeanDefinitionDocumentReader来进行操作。

`AnnotatedBeanDefinitionReader`：不仅能读取BeanDefinition，并且还能注册BeanDefinition。
