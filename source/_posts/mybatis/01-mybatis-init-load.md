---
title: Mybaits 启动加载
tags:
  - Java
  - Orm
  - Mybatis
categories: Java
index_img: img/mybatis-logo-480.png
abbrlink: 38506
date: 2019-10-09 09:28:07
---
# Mybatis-启动加载

## 本章节讲述的内容

> 本章节主要分析Mybatis在启动时做了那些操作。带着以下疑问去阅读本章内容。本章内容涉及了大量的mybatis源码。
>
> 1、Mybaits是如何解析配置文件（mybatis-config.xml）？
>
> 2、Mybaits是如何解析Mapper文件（mapper.xml）？
>
> 3、Mybatis是如何设计

想知道如何使用Mybatis请参考[Mybatis官方文档](https://mybatis.org/mybatis-3/zh/)。

## 相关配置依赖

本文使用的Mybatis版本：`3.5.4`，不依赖任何Spring的环境。

### 坐标依赖

Maven坐标：

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.4</version>
</dependency>
```



Gradle坐标：

```groovy
compile 'org.mybatis:mybatis:3.5.4'
```

### XML配置

```xml
<configuration>
    <settings>
        <setting name="logImpl" value="SLF4J"/>
        <setting name="useGeneratedKeys" value="true"/>
        <setting name="logPrefix" value="source-code-analysis"/>
    </settings>

    <typeAliases>
        <package name="io.better.mybatis.model"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/source-code-analysis"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
    
    <mappers>
        <mapper resource="mybatis/mappers/UserMapper.xml"/>
    </mappers>
</configuration>
```



**Mybatis的启动时主要分为两个阶段：`配置文件加载解析阶段，Mapper文件加载解析阶段`。**



## 启动

```java
public class MybatisTest {

    SqlSessionFactory sqlSessionFactory;

    @Before
    public void before() throws IOException {
        String resource = "mybatis/mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    }

    @Test
    public void testSqlSessionFactoryInit() {
        System.out.println(sqlSessionFactory);
    }
}
```



## 配置文件加载解析阶段

### 概述

在这个阶段我们会去了解Mybatis是如何解析配置文件的？



### 构建开始

从官网文档我们知道使用Mybatis第一步就是使用`SqlSessionFactoryBuilder`来构建`SqlSessionFactory`。

而`SqlSessionFactoryBuilder`类提供了多个重载的`build`方法：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164710.png" alt="image-20200430102857721" style="zoom:50%;" />

这里我们使用的是`build(InputStream)`方法。进入该方法的源代码：

### 执行-`SqlSessionFactoryBuilder.build()` 

源代码如下：

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    // 创建XMLConfigBuilder，该对象用于解析Mybatis配置文件
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    // build会用解析配置文件的结果来创建SqlSessionFactory
    return build(parser.parse());
  } catch (Exception e) {
  } finally {}
}
```

其中 `build`方法比较简单，就是使用`XMLConfigBuilder.parse`生成的`Configuration`对象创建`SqlSessionFactory`。

```java
public SqlSessionFactory build(Configuration config) {
  return new DefaultSqlSessionFactory(config);
}
```

而`XMLConfigBuilder.parse`方法主要描述了Mybatis解析配置文件并生成Configuration对象的过程。

成员变量：

![image-20200430103839043](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164704.png)

- `parsed：`标识了配置是否已经解析。
- `parser：`用于解析XML配置文件
- `environment：`Mybatis的环境配置
- `localReflectorFactory：`反射工厂

构造器：

![image-20200430104004899](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164659.png)

`org.apache.ibatis.builder.xml.XMLConfigBuilder#parse`

```java
public Configuration parse() {
  if (parsed) {		// 已经解析过直接抛出异常
    throw new BuilderException("Each XMLConfigBuilder can only be used once.");
  }
  parsed = true;		// 将parsed置为true，避免重复解析。
	// 调用parser获取配置文件中的configuration节点
  // 调用parseConfiguration方法继续解析
  parseConfiguration(parser.evalNode("/configuration"));
  return configuration;
}
```



### 执行-`XMLConfigBuilder.parseConfiguration()`

源代码如下：

```java
private void parseConfiguration(XNode root) {
  try {
    // 解析properties标签配置
    propertiesElement(root.evalNode("properties"));
    // 解析settings标签配置
    Properties settings = settingsAsProperties(root.evalNode("settings"));
		// 使用setting配置日志实现类
    loadCustomLogImpl(settings);
    // 解析typeAliases标签配置
    typeAliasesElement(root.evalNode("typeAliases"));
    // 解析plugins标签配置
    pluginElement(root.evalNode("plugins"));
    // 使用settings配置对configuration对象中的属性进行赋值
    settingsElement(settings);
    // 解析environments标签配置
    environmentsElement(root.evalNode("environments"));
    // 解析mappers标签配置, 开启Mapper文件解析阶段
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```

在`parseConfiguration`方法中，Mybatis对配置中的不同标签分别进行了处理，这里只列举常用的标签配置，例如：`properties，settings，typeAliases，plugins，environments，mappers`等标签进行了处理。



```java
private void parseConfiguration(XNode root) {
  try {
    // 解析properties标签配置
    propertiesElement(root.evalNode("properties"));
    // 解析settings标签配置
    Properties settings = settingsAsProperties(root.evalNode("settings"));
		// 使用setting配置日志实现类
    loadCustomLogImpl(settings);
    // 解析typeAliases标签配置
    typeAliasesElement(root.evalNode("typeAliases"));
    // 解析plugins标签配置
    pluginElement(root.evalNode("plugins"));
    // 使用settings配置对configuration对象中的属性进行赋值
    settingsElement(settings);
    // 解析environments标签配置
    environmentsElement(root.evalNode("environments"));
    // 解析mappers标签配置, 开启Mapper文件解析阶段
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```



#### 解析`Properties`标签

从上面代码可以看出`propertiesElement`方法是解析Properties标签的核心方法，源代码如下：

```java
private void propertiesElement(XNode context) throws Exception {
  if (context != null) {
    // 获取properties标签中resource和url配置
    String resource = context.getStringAttribute("resource");
    String url = context.getStringAttribute("url");
    if (resource != null) {
      // 使用resource路径加载properties
      defaults.putAll(Resources.getResourceAsProperties(resource));
    } else if (url != null) {
      // 使用url加载properties
      defaults.putAll(Resources.getUrlAsProperties(url));
    }
    // 获取到configuration中已有的属性
    Properties vars = configuration.getVariables();
    if (vars != null) {
      // 合并
      defaults.putAll(vars);
    } 
    parser.setVariables(defaults);
    // 重新赋值
    configuration.setVariables(defaults);
  }
}
```

可以看出解析出来的`Properties`属性，最终被赋值到了`Configuration`中的`variables`对象中。

```java
// Configuration的成员变量
protected Properties variables = new Properties();
```



#### 解析`Settings`标签

使用类似的方式查看`settingsAsProperties`方法中处理`Setting`标签的逻辑。

```java
private Properties settingsAsProperties(XNode context) {
  if (context == null) {
    return new Properties();
  }
  // 获取所有setting子节点，并将其转成properties
  Properties props = context.getChildrenAsProperties();
	// 通过反射获取Configuration类的元数据信息
  MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
  // 判断配置的属性在Configuration是否存在
  for (Object key : props.keySet()) {
    // 判断Configuration对象是否存在此配置的Setter方法。
    if (!metaConfig.hasSetter(String.valueOf(key))) {
      throw new BuilderException("此配置无效：" + key);
    }
  }
  return props;
}
```

`MetaClass.forClass`方法作用是使用`localReflectorFactory`创建一个`Reflector`实例，该实例中包含了Configuration类所有的`构造器，方法，属性`等。

![](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164650.png)



#### 使用`Settings`标签

上面我们知道了`Settings`配置时如何解析的，但却没有使用到`Settings`配置。

`loadCustomVfs(settings)：`忽略

 `loadCustomLogImpl(settings)：`设置Mybatis的日志实现。

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164645.png" alt="image-20200430111703591"  />

`settingsElement(settings)：`设置Mybatis所有全局配置，Settings没有则使用默认配置。

![image-20200430111538374](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164633.png)



#### 解析`typeAliasesElement`标签

源代码如下:

```java
private void typeAliasesElement(XNode parent) {
  // 获取到typeAliases标签下所有的子标签
  for (XNode child : parent.getChildren()) {
    // 如果标签以package开头，说明配置的是整个包的别名
    if ("package".equals(child.getName())) {
      String typeAliasPackage = child.getStringAttribute("name");
      // registerAliases方法会扫描整个包下的类，并生成其对应的Class对象
      // 最后添加到configuration.typeAliasRegistry属性对象中
      configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
    } 
    // 否则就是以typeAlias开头的标签，配置的是单个别名
    else {
      // 获取到别名
      String alias = child.getStringAttribute("alias");
      // 获取别名对应的class
      String type = child.getStringAttribute("type");
      try {
        // 根据type解析出其真实的class对象
        Class<?> clazz = Resources.classForName(type);
        if (alias == null) {
          // 别名为空，默认使用类名做为别名
          typeAliasRegistry.registerAlias(clazz);
        } else {
          typeAliasRegistry.registerAlias(alias, clazz);
        }
      } catch (ClassNotFoundException e) {
      }
    }
  }
}
```

从代码可以看出配置`typeAliases`有两种方式：

1、使用package标签配置整个包下的别名。

2、使用typeAlias标签配置单个别名。



获取到`别名对应的Class对象`后，最终调用了`typeAliasRegistry.registerAlias`方法注册了别名。而`typeAliasRegistry`对象引用之Configuration对象。

![image-20200430112247768](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164624.png)



让我们进入`typeAliasRegistry.registerAlias`方法查看具体注册逻辑

```java
public void registerAlias(String alias, Class<?> value) {
  if (alias == null) {
    throw new TypeException("The parameter alias cannot be null");
  }
  // issue #748
  String key = alias.toLowerCase(Locale.ENGLISH);
  // 判断alias是否已经存在，且对应的class不一致，抛出异常。
  if (typeAliases.containsKey(key) && typeAliases.get(key) != null && !typeAliases.get(key).equals(value)) {
    throw new TypeException("");
  }
  // 放入typeAliases中
  typeAliases.put(key, value);
}
```

让我们看一下该类的结构图：

![image-20200430112955479](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164616.png)

从类结构图能看出`TypeAliasRegistry`类中重载了很多`registerAliases`方法，同时`typeAliases`保存了所有注册的别名信息。并且在初始化时默认添加了很多基础类型对应的别名。



结论：Mybatis将解析的别名配置，最终放入了`Configuration`中的`TypeAliasRegistry`对象中。



#### 解析`plugins`标签

源代码如下：

```java
private void pluginElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 获取配置的interceptor对应的class信息
      String interceptor = child.getStringAttribute("interceptor");
      // 获取plugin标签下的属性
      Properties properties = child.getChildrenAsProperties();
      // resolveClass方法解析具体interceptor的class
      Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).getDeclaredConstructor().newInstance();
      // 将属性赋值给Interceptor
      interceptorInstance.setProperties(properties);
      // 添加到Configuration中
      configuration.addInterceptor(interceptorInstance);
    }
  }
}
```

可以看出解析Interceptor最终通过反射初始化被添加到了Configuration中 。

![image-20200430141031684](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164611.png)



#### 解析`environmentsElement`标签

参考Xml：

```xml
<environments default="development">
  <environment id="development">
    <transactionManager type="JDBC"/>
    <dataSource type="POOLED">
      <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
      <property name="url" value="jdbc:mysql://localhost:3306/source-code-analysis"/>
      <property name="username" value="root"/>
      <property name="password" value="123456"/>
    </dataSource>
  </environment>
</environments>
```



源代码：

```java
private void environmentsElement(XNode context) throws Exception {
  if (context != null) {
    if (environment == null) {
      // 获取默认的environment，即上面的development
      environment = context.getStringAttribute("default");
    }
    // 获取到所有的environment子标签
    for (XNode child : context.getChildren()) {
      String id = child.getStringAttribute("id");
      // environment是否和默认配置的一致
      if (isSpecifiedEnvironment(id)) {
        // 创建事务管理器，即上面的JDBC事务管理器
        TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
        // 创建数据源工厂
        DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
        // 并获取到数据源
        DataSource dataSource = dsFactory.getDataSource();
        // 构建Environment
        Environment.Builder environmentBuilder = new Environment.Builder(id)
          .transactionFactory(txFactory)
          .dataSource(dataSource);
        // 赋值给configuration对象
        configuration.setEnvironment(environmentBuilder.build());
      }
    }
  }
}
```

从代码可以看出，Mybatis获取环境变量后创建了两个核心对象：1、TransactionFactory。2、DataSource，其对应的方法是

`transactionManagerElement()`， `dataSourceElement()`两个方法。底层实现都是通过反射进行实例化。

![image-20200430142709538](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164602.png)

我们以`PooledDataSourceFactory`子类为例。

![image-20200430142750298](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164552.png)



至此Mybatis的第一阶段配置文件的解析到此结束，这里还有Mapper文件的解析（下面讲解）。



### 总结

- 解析出来的`Properties`配置，最终被赋值到了`Configuration`中的`variables`对象中。
- 解析出来的`typeAliases`配置，最终放入了`Configuration`中的`typeAliasRegistry`对象中。
- 解析出来的`Plugins`配置，最终放入了Configuration中的`interceptorChain`对象中。
- 解析出来的`environments`配置，用于`创建了数据源和事务工厂`。最终构建`Environment`对象赋值给Configuration中。



从上面处理流程可以看出Configuration非常的重要，几乎所有的配置最终都会汇聚到Configuration对象中。



#### 执行流程

```txt
> org.apache.ibatis.session.SqlSessionFactoryBuilder#build(java.io.InputStream) -> Mybatis构建入口 
  > org.apache.ibatis.builder.xml.XMLConfigBuilder#parseConfiguration -> 解析configuration标签配置
   > org.apache.ibatis.builder.xml.XMLConfigBuilder#propertiesElement -> 解析properties标签配置
   > org.apache.ibatis.builder.xml.XMLConfigBuilder#settingsAsProperties -> 解析settings标签配置
   > org.apache.ibatis.builder.xml.XMLConfigBuilder#typeAliasesElement -> 解析typeAliases标签配置
   > org.apache.ibatis.builder.xml.XMLConfigBuilder#pluginElement -> 解析plugins标签配置
   > org.apache.ibatis.builder.xml.XMLConfigBuilder#environmentsElement -> 解析environments标签配置
   > org.apache.ibatis.builder.xml.XMLConfigBuilder#mapperElement -> 解析mappers标签配置
```



## Mapper文件加载解析阶段

### 概述 

在这个阶段我们将深入了解Mybatis是如何解析Mapper文件。如何绑定Mapper接口和文件的关系。



### 疑问

在进入分析之前我们先思考几个问题：

1、Mybatis是如何`Select，Insert，Delete，Update`标签的？

2、Mybatis是如何处理`Include`标签的？

3、Mybatis是如何处理动态SQL标签（`set，foreach，if`）？

4、Mybatis是如何处理`resultMap`标签的？



### 入口

在上个阶段的最后一行方法：

![image-20200430144550431](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164546.png)

红圈内的方法就是解析Mapper文件流程的入口方法。

![image-20200430151542891](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164537.png)

从图中可以看出解析Mapper文件有两种方式：

1. 使用Package标签扫描整个包下的Mapper接完成注册。

2. 使用Mapper标签配置单个Mapper。
   1. Mapper标签支持三种方式 ：`resource（路径），url（网络），class（class类）`来加载Mapper。



### 使用`Package`解析

进入 `configuration.addMappers(mapperPackage)`方法，源代码如下：

```java
public void addMappers(String packageName) {
  mapperRegistry.addMappers(packageName);
}
```

从`mapperRegistry`对象的命名可以看出，该对象主要是负责注册Mapper的。

进入`addMappers`方法：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164530.png" alt="image-20200430163542757"  />

代码会获取到包下所有Mapper对应的Class信息，并进行遍历调用 `addMapper`方法。

我们可以看出Mapper解析是在MapperRegistry类中完成的，那么MapperRegistry是如何存储注册的Mapper的，来看看其类结构图：

![image-20200430163959992](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164524.png)

`knownMappers`充当了一个重要的角色，它存储了注册过的Mapper。而`MapperProxyFactory`对象则用于创建MapperProxy对象，此对象会对Mapper进行代理。



接着进入`addMapper`方法查看添加细节。

#### 执行`MapperRegistry.addMapper`添加`Mapper`

```java
public <T> void addMapper(Class<T> type) {
  if (type.isInterface()) {		// 接口才处理
    if (hasMapper(type)) {
      throw new BindingException("此Mapper已经存在");
    }
    boolean loadCompleted = false;
    try {
      knownMappers.put(type, new MapperProxyFactory<>(type));
      // 创建MapperAnnotationBuilder构建起
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      // 构建Mapper
      parser.parse();
      loadCompleted = true;
    } finally {}
  }
}
```

该方法主要做了以下几个操作：

- 将Mapper对应的Class对象放入knownMappers对象中。
- 创建MapperAnnotationBuilder构建器构建Mapper。



#### 执行`MapperAnnotationBuilder.parse`解析`Mapper`

源代码如下：

```java
public void parse() {
  String resource = type.toString();
  if (!configuration.isResourceLoaded(resource)) {		// 判断当前资源是否加载过
    // 步骤①
    loadXmlResource();
    configuration.addLoadedResource(resource);	// 添加到Configuration对象中，标识已经加载过
    assistant.setCurrentNamespace(type.getName());
    parseCache();
    parseCacheRef();
    for (Method method : type.getMethods()) {
      try {
        if (!method.isBridge()) {
          // 步骤②
          parseStatement(method);
        }
      } catch (IncompleteElementException e) {}
    }
  }
  parsePendingMethods();
}
```

代码中我们看到了最为重要的两个方法：`loadXmlResource()：`用于加载Mapper对应的Xml文件，`parseStatement()`：解析Mapper方法中的注解信息。

解析过的`resource`最终会添加到Configuration中的`loadedResources`（Set类型）对象中。



##### `loadXmlResource`

从上面可以得知该方法主要用于解析Mapper对应的xml文件。源代码如下：

```java
private void loadXmlResource() {
  if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
    // 将全类名中的.替换成/,以方便下面Resources获取输入流
    // io.better.mybatis.mapper.UserMapper -> io/better/mybatis/mapper/UserMapper.xml
    String xmlResource = type.getName().replace('.', '/') + ".xml";
    // 获取到Mapper对应的Xml文件输入流
    InputStream inputStream = type.getResourceAsStream("/" + xmlResource);
    if (inputStream == null) {
        inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
    }
    if (inputStream != null) {
      // 步骤①
      XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
      // 步骤②
      xmlParser.parse();
    }
  }
}
```

从代码中可以看出，Mybatis通过替换`Mapper全路径`获取到`xml对应resource中的目录层级`，并获取到InputStream流。

举例：`io.better.mybatis.mapper.UserMapper -> resource/io/better/mybatis/mapper/UserMapper.xml`。

**所以使用原生Mybatis需要注意xml放置的位置**。

紧接着Mybatis创建了`XMLMapperBuilder`对象，调用其parse方法进一步的解析xml文件。

关于上面步骤①和步骤②的流程下面`Resource解析方式`会复用到，如果需要了解请直接跳转至Resource解析方法。



##### `parseStatement`

新版Mybatis支持注解执行SQL语句，通过`@SELECT，@INSERT，@UPDATE，@DELETE`注解来执行SQL语句。此方法就是解析这些注解并生成`MappedStatement`对象。

源代码：

```java
void parseStatement(Method method) {
  // 获取参数来类型
  Class<?> parameterTypeClass = getParameterType(method);
  LanguageDriver languageDriver = getLanguageDriver(method);
  // 获取到注解中编写的SQL语句
  SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
  if (sqlSource != null) {
    Options options = method.getAnnotation(Options.class);
    // 生成statementId
    final String mappedStatementId = type.getName() + "." + method.getName();
    Integer fetchSize = null;
    Integer timeout = null;
    // 默认为PreparedStatement类型
    StatementType statementType = StatementType.PREPARED;
    ResultSetType resultSetType = configuration.getDefaultResultSetType();
    //  获取到SQL指令类型，新增? 更新? 查询? 删除?
    SqlCommandType sqlCommandType = getSqlCommandType(method);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = !isSelect;
    boolean useCache = isSelect;
		// 获取主键生成器
    KeyGenerator keyGenerator;
    String keyProperty = null;
    String keyColumn = null;
    if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
      // first check for SelectKey annotation - that overrides everything else
      SelectKey selectKey = method.getAnnotation(SelectKey.class);
      if (selectKey != null) {
        keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
        keyProperty = selectKey.keyProperty();
      } else if (options == null) {
        keyGenerator = configuration.isUseGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
      } else {
        keyGenerator = options.useGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        keyProperty = options.keyProperty();
        keyColumn = options.keyColumn();
      }
    } else {
      keyGenerator = NoKeyGenerator.INSTANCE;
    }
		
    if (options != null) {
      if (FlushCachePolicy.TRUE.equals(options.flushCache())) {
        flushCache = true;
      } else if (FlushCachePolicy.FALSE.equals(options.flushCache())) {
        flushCache = false;
      }
      useCache = options.useCache();
      fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null; //issue #348
      timeout = options.timeout() > -1 ? options.timeout() : null;
      statementType = options.statementType();
      if (options.resultSetType() != ResultSetType.DEFAULT) {
        resultSetType = options.resultSetType();
      }
    }

    String resultMapId = null;
    // 获取ResultMap对象
    ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
    if (resultMapAnnotation != null) {
      resultMapId = String.join(",", resultMapAnnotation.value());
    } else if (isSelect) {
      resultMapId = parseResultMap(method);
    }

    // 忽略参数
    // 构建MappedStatement
    assistant.addMappedStatement();
  }
}
```

方法内部创建的对象和Mapper标签几乎一致，并且这种方式代码阅读行较差，简单的SQL语句还行，但是复杂的不行。此处不做细致讲解。（后期可能补上）。



### 使用`Mapper`标签解析

入口：

![image-20200430183533866](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164159.png)

可以看出一共有三种方式，分别是`resource，url，class` 。前两种最终都创建了XMLMapperBuilder对象，而最后一种和Package解析方式类似。



我们重点关注`XMLMapperBuilder`这个对象。到这里与Package解析中的`loadXmlResource`方法处理一致。



#### 执行`XMLMapperBuilder.parse`

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501112922.png" alt="image-20200501112922353" style="zoom:50%;" />

重点关注两个方法，分别是：`configurationElement：`用于解析XML中各个标签，`bindMapperForNamespace：`将Mapper接口绑定到Configuration上。



#### 执行`XMLMapperBuilder.configurationElement`

![image-20200501113147828](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501113148.png)

红圈中我们能够看到Mybatis对XML中的各个标签都进行了处理。

- ```
  parameterMapElement -> 处理parameterMap标签。
  ```

- ```
  resultMapElements -> 处理resultMap标签
  ```

- ```
  sqlElement -> 处理sql标签
  ```

- ```
  buildStatementFromContext -> 处理Select，Insert，Delete，Update标签
  ```

接下来让我们逐个分析这些方法是如何处理标签的。



#### 执行`bindMapperForNamespace`方法

该方法主要作用是将解析后的XML和Mapper接口绑定到Configuration对象中。

```java
private void bindMapperForNamespace() {
  String namespace = builderAssistant.getCurrentNamespace();
  if (namespace != null) {
    Class<?> boundType = Resources.classForName(namespace);
    if (boundType != null) {
      if (!configuration.hasMapper(boundType)) {
        // 标识当前XML已经解析过
        configuration.addLoadedResource("namespace:" + namespace);
        // 添加到Configuration中
        configuration.addMapper(boundType);
      }
    }
  }
}
```



#### 解析`ResultMap`标签

```java
private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings, Class<?> enclosingType) {
  String type = resultMapNode.getStringAttribute("type"); // 这里删除了一些代码
  Class<?> typeClass = resolveClass(type);		// 解析type获取对应的Class对象
  if (typeClass == null) {
    typeClass = inheritEnclosingType(resultMapNode, enclosingType);
  }
  Discriminator discriminator = null;
  List<ResultMapping> resultMappings = new ArrayList<>(additionalResultMappings);
  List<XNode> resultChildren = resultMapNode.getChildren();
  for (XNode resultChild : resultChildren) {
    if ("constructor".equals(resultChild.getName())) {
      processConstructorElement(resultChild, typeClass, resultMappings);  // 步骤④：处理constructor标签
    } else if ("discriminator".equals(resultChild.getName())) {
      // 步骤③: 处理discriminator标签
      discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
    } else {  // 处理其他标签，例如：id，result标签
      List<ResultFlag> flags = new ArrayList<>();
      if ("id".equals(resultChild.getName())) {
        flags.add(ResultFlag.ID);
      }
      // 步骤①：调用buildResultMappingFromContext构建ResultMapping
      resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
    }
  }
  // 获取到标签中的id，extends，autoMapping属性
  String id = resultMapNode.getStringAttribute("id",
          resultMapNode.getValueBasedIdentifier());
  String extend = resultMapNode.getStringAttribute("extends");
  Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
  // 初始化ResultMapResolver对象，用于解析生成ResultMap对象
  ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
  try {
    // 步骤②： 执行解析
    return resultMapResolver.resolve();
  } catch (IncompleteElementException  e) {
  }
}
```

图中步骤③和步骤④最终都会 调用步骤①的方法，所以我们重点关注步骤①和步骤② 这两个方法。



##### 生成`ResultMapping`

进入`buildResultMappingFromContext`方法，源代码如下：

```java
private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) {
  String property;
  if (flags.contains(ResultFlag.CONSTRUCTOR)) {
    property = context.getStringAttribute("name");
  } else {
    property = context.getStringAttribute("property");
  }
  String column = context.getStringAttribute("column");
  String javaType = context.getStringAttribute("javaType");
  String jdbcType = context.getStringAttribute("jdbcType");
  String nestedSelect = context.getStringAttribute("select");
  String nestedResultMap = context.getStringAttribute("resultMap", () ->
    processNestedResultMappings(context, Collections.emptyList(), resultType));
  String notNullColumn = context.getStringAttribute("notNullColumn");
  String columnPrefix = context.getStringAttribute("columnPrefix");
  String typeHandler = context.getStringAttribute("typeHandler");
  String resultSet = context.getStringAttribute("resultSet");
  String foreignColumn = context.getStringAttribute("foreignColumn");
  boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
  Class<?> javaTypeClass = resolveClass(javaType);
  Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
  JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
  // 最终调用buildResultMapping生成了ResultMapping对象
  return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
}
```

可以看出这个方法逻辑比较简单，将子节点中配置的属性获取出来，最终组装了ResultMapping对象。



##### 生成`ResultMap`



`org.apache.ibatis.builder.ResultMapResolver#resolve`

```java
public ResultMap resolve() {
  // 调用工具类的
  return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
}
```

`addResultMap`

```java
public ResultMap addResultMap(String id,Class<?> type,String extend,Discriminator discriminator,
    List<ResultMapping> resultMappings,Boolean autoMapping) {
  
  id = applyCurrentNamespace(id, false);
  extend = applyCurrentNamespace(extend, true);
	// 如果当前resultMap存在父resultMap
  if (extend != null) {
    // configuration是否已经加载过父resultMap
    if (!configuration.hasResultMap(extend)) {
      throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
    }
   	// 根据父resultMap的Id获取出父resultMap对象
    ResultMap resultMap = configuration.getResultMap(extend);
    // 去除父resultMap的resultMapping集合
    List<ResultMapping> extendedResultMappings = new ArrayList<>(resultMap.getResultMappings());
    // 去重
    extendedResultMappings.removeAll(resultMappings);
    boolean declaresConstructor = false;
    // 遍历当前resultMap的resultMapping，判断其是否包含Constructor
    for (ResultMapping resultMapping : resultMappings) {
      if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
        declaresConstructor = true;
        break;
      }
    }
    // 如果当前resultMap声明了Constructor标签，则删除父resultMap中声明的Constructor标签
    if (declaresConstructor) {
      extendedResultMappings.removeIf(
        resultMapping -> resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR));
    }
    // 将父resultMap的resultMapping添加到当前resultMap的resultMapping集合中
    resultMappings.addAll(extendedResultMappings);
  }
  // 构建ResultMap对象
  ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
      .discriminator(discriminator).build();
  // 将构建好的ResultMap对象添加到Configuration中
  configuration.addResultMap(resultMap);
  return resultMap;
}
```

因为resultMap标签可以继承其他的resultMap标签，而这个方法主要就是为了处理resultMap继承的问题。可以得出结论：**XML中的`resultMap标签`最终会被解析成`ResultMap对象`并复制给了Configuration**。



#### 解析`SQL`标签

resultMap标签解析完，继续看`sql标签`的解析过程：

方法入口：

```java
sqlElement(context.evalNodes("/mapper/sql"));
```

获取到XML中所有的sql标签节点对象。

```java
private void sqlElement(List<XNode> list, String requiredDatabaseId) {
  for (XNode context : list) {
    String databaseId = context.getStringAttribute("databaseId");
    // 获取到sql标签的ID
    String id = context.getStringAttribute("id");
    // 和namespace进行拼接
    id = builderAssistant.applyCurrentNamespace(id, false);
    // 判断是否已经存在次sql标签
    if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
      // 不存在，添加到sqlFragments中
      // sqlFragments存在于configuration中
      sqlFragments.put(id, context);
    }
  }
}
```

该方法功能很简单，就是将`sql标签的id和对应的XNode节点对象`放入到`sqlFragments`中。

`sqlFragments`对象在后面解析`select|insert|update|delete`标签中的include标签时会使用到。



#### 解析`Crud`标签

方法入口 ：

```java
buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
```

获取的所有的CRUD标签。

```java
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
  // 获取到所有的select，insert，delete，update标签节点
  for (XNode context : list) {
    // 为每一个crud标签创建XMLStatementBuilder对象，用于构建每个标签所对应的MapperStatement对象
    final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
    // 解析并生成MapperStatement对象
    statementParser.parseStatementNode();
  } // 忽略了异常处理
}
```

该方法对获取到的`insert，select，update，delete`节点进行了遍历，并实例化了`XMLStatementBuilder`构建器来构建`MapperdStatement`。

`XMLStatementBuilder`会将每个`crud标签`构建成一个`MapperdStatement对象`。它两是一对一的关系，后面调用方法执行SQL时就会获取到方法对应的MapperdStatement对象。



进入`parseStatementNode`方法：

重点关注标记的步骤

```java
public void parseStatementNode() {
  String id = context.getStringAttribute("id");
  String databaseId = context.getStringAttribute("databaseId");

  if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
    return;
  }
  String nodeName = context.getNode().getNodeName();
  SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
  boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
  boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
  boolean useCache = context.getBooleanAttribute("useCache", isSelect);
  boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

  // 步骤①
  XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
  includeParser.applyIncludes(context.getNode());

  String parameterType = context.getStringAttribute("parameterType");
  Class<?> parameterTypeClass = resolveClass(parameterType);

  String lang = context.getStringAttribute("lang");
  LanguageDriver langDriver = getLanguageDriver(lang);

  // 步骤②
  processSelectKeyNodes(id, parameterTypeClass, langDriver);

  // 
  KeyGenerator keyGenerator;
  String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
  keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
  if (configuration.hasKeyGenerator(keyStatementId)) {
    keyGenerator = configuration.getKeyGenerator(keyStatementId);
  } else {
    keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
        configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
        ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
  }
	// 步骤③
  SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
  StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
  Integer fetchSize = context.getIntAttribute("fetchSize");
  Integer timeout = context.getIntAttribute("timeout");
  String parameterMap = context.getStringAttribute("parameterMap");
  String resultType = context.getStringAttribute("resultType");
  Class<?> resultTypeClass = resolveClass(resultType);
  String resultMap = context.getStringAttribute("resultMap");
  String resultSetType = context.getStringAttribute("resultSetType");
  ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
  if (resultSetTypeEnum == null) {
    resultSetTypeEnum = configuration.getDefaultResultSetType();
  }
  String keyProperty = context.getStringAttribute("keyProperty");
  String keyColumn = context.getStringAttribute("keyColumn");
  String resultSets = context.getStringAttribute("resultSets");
  // 步骤④
  builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
      fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
      resultSetTypeEnum, flushCache, useCache, resultOrdered,
      keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}
```

这段代码很长，无需全部阅读，只需查看标记的核心步骤即可。

- 步骤①：处理curd标签中的的Include标签。
- 步骤②：处理insert标签中的selectKey标签。
- 步骤③：处理curd标签中的动态标签 。
- 步骤④：将使用最终的crud标签生成MappedStatement对象。



##### 处理`Include`标签

在分析`XMLIncludeTransformer.applyIncludes`方法前，我们先来看几个问题，以便我们更好的理解Mybatis对 `<include/>`的处理逻辑：

- Mybatis是如何拆分含有`<include/>`标签的SQL？拆分各部分SQL关系如何维护？
- Mybatis是如何处理嵌套`<include/>`标签？

带着这两个问题，我们进入`applyIncludes`方法。

```java
private void applyIncludes(Node source, final Properties variablesContext, boolean included) {
	//  步骤②
  if (source.getNodeName().equals("include")) {
    // 步骤⑤ 从sqlFragments中获取include所引用的id，即sql标签中编写的SQL语句。
    Node toInclude = findSqlFragment(getStringAttribute(source, "refid"), variablesContext);
    Properties toIncludeContext = getVariablesContext(source, variablesContext);
    // 步骤⑥ 递归，处理toInclude中嵌套的include标签
    applyIncludes(toInclude, toIncludeContext, true);
    // 如果（toInclude节点）和 source（引入toInclude节点）不在同一个document下
    if (toInclude.getOwnerDocument() != source.getOwnerDocument()) {
      // 将（toInclude节点）导入到 source 节点中去
      toInclude = source.getOwnerDocument().importNode(toInclude, true);
    }
    // 步骤⑦ 将 source 节点替换成（toInclude节点）
    source.getParentNode().replaceChild(toInclude, source);
    // 为了方便阅读对源代码进行了修改
    Node toIncludeParentNode = toInclude.getParentNode()；
    // （toInclude节点）含有子节点，可能是嵌套的include标签
    while (toInclude.hasChildNodes()) {
      // 将（toInclude节点）下的子节点添加到父节点的集合中，并在（toInclude节点）前面
      toIncludeParentNode.insertBefore(toInclude.getFirstChild(), toInclude);
    }
    // 步骤⑧ 从父节点中删除（toInclude节点）
    toIncludeParentNode.removeChild(toInclude);
  } 
  // 步骤①
  // source对象为DeferredElementImpl类型，即include，select，insert，update，delete等标签
  else if (source.getNodeType() == Node.ELEMENT_NODE) {

    // 步骤③
    NodeList children = source.getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
      // 步骤④
      applyIncludes(children.item(i), variablesContext, included);
    }
  } 
  // source对象为DeferredTextImpl类型，
  else if (included && (source.getNodeType() == Node.TEXT_NODE || source.getNodeType() == Node.CDATA_SECTION_NODE) && !variablesContext.isEmpty()) {
    // 步骤⑨ 最终替换对象里的data属性
    source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
  }
}


```

这段代码大量使用了递归调用，来处理嵌套标签。

执行流程：

- 判断节点类型是否为元素节点
  - 是 `--> 步骤①`
    - 获取到子节点，根据`include`标签进行拆分。`--> 步骤③`
    - 遍历子节点，递归处理子节点。 `--> 步骤④`
  - 否 `--> 步骤②`  
    - 从`sqlFragments`中获取到引入SQL节点。 `-->步骤⑤`
    - 递归处理引入的SQL节点，防止引入的SQL中存在嵌套的Include节点。 `--> 步骤⑥`
    - 逐层将`include标签`替换成`include中实际的SQL`。 `-->步骤⑦` 
    - 最终删除`include标签`。 `--> 步骤⑧`
- 直接设置节点值 `-->步骤⑨`



讲述了大致流程，这里会有一个小疑问，Mybatis到底是如何获取子节点的？是如何拆分的？为了更好的理解这个问题， 引入一张`select标签`引入`<include/>`标签的XML解析Debug图：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164132.png" alt="include解析" style="zoom:50%;" />

从上图画红圈的地方可以，整个`select`语句被拆分成了三个部分，每个对象中都有：

- `select` 一行产生了一个`DeferredTextImpl`类型的 对象，简称对象A。
- `include`一行产生了一个`DeferredElementImpl`类型的对象，简称对象B。
- `form` 和` where`两行产生了一个`DeferredTextImpl`类型的对象，简称对象C。

每个对象中都含有以下三个属性：

- `data`标识了当前对象所对应的`sql`语句内容。

- `previousSibling`标识了当前对象的前一个对象。

- `nextSibling`标识来当前对象的下一个对象 。

三个对象前后之前的关系：

```tex
对象A.previousSibling -> 对象C
对象B.previousSibling -> 对象A
对象C.previousSibling -> 对象B
```

**这里我们可以看出Mybatis使用了类似  `双向链表` 的结构来管理标签被拆分后形成的对象。**



**这里有一个疑问，为什么`select`被拆分成了一行，而`from`和`where`为什么没被拆分成两行，而是拆分成了一行 ？**

为了验证这个问题，将`<include/>`替换掉，生成的对象属性如下图。未引入Include的XML解析Debug图：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501164004.png" style="zoom:50%;" />

**这里可以得出的结论就是：Mybatis会以`<include/>`标签为分割线进行SQL语句的切分。**



步骤②处理SelectKey标签代码不做分析，感兴趣的可以自行查看源代码。



##### 处理`dynamic`标签

在进入`langDriver.createSqlSource`方法前我们先来思考两个问题？

- Mybatis是如何区分动态SQL和静态SQL？
- Mybatis是如何处理包含动态标签的SQL？

带着这两个问题我们继续分析：

```java
public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
  // 创建XMLScriptBuilder构建起
  XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
  // 构建script节点
  return builder.parseScriptNode();
}
```

我们可以看出Mybatis创建了`XMLScriptBuilder`构建器来构建`动态标签`。我们先来看看XMLScriptBuilder类结构：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501154915.png" alt="image-20200501154915783" style="zoom:50%;" />

从类结构图可以看出有8个处理动态标签的处理类，使用`isDynamic()`判断是否为动态标签 ，使用`nodeHandlerMap`来存储动态标签处理器。

让我进入`parseScriptNode`方法一探究竟：

```java
public SqlSource parseScriptNode() {
  // 解析动态标签
  MixedSqlNode rootSqlNode = parseDynamicTags(context);
  SqlSource sqlSource;
  if (isDynamic) {
    // 动态SQL源
    sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
  } else {
    // 静态SQL源
    sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
  }
  return sqlSource;
}
```

从代码中可以看出Mybatis为动态SQL创建了`DynamicSqlSource`子类，为静态SQL创建了`RawSqlSource`子类。到这里我们的第一个问题就比较清晰了，**Mybatis使用了不同的SqlSource子类来区分动态SQL和静态SQL**。

进入`parseDynamicTags`方法，查看解析动态标签的具体方法：

```java
protected MixedSqlNode parseDynamicTags(XNode node) {
  List<SqlNode> contents = new ArrayList<>();
  // 步骤①
  NodeList children = node.getNode().getChildNodes();
  for (int i = 0; i < children.getLength(); i++) {
    XNode child = node.newXNode(children.item(i));
    // 步骤②
    if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
      // 步骤③
      String data = child.getStringBody("");
      TextSqlNode textSqlNode = new TextSqlNode(data);
      if (textSqlNode.isDynamic()) {
        contents.add(textSqlNode);
        isDynamic = true;
      } else {
        // 步骤③
        contents.add(new StaticTextSqlNode(data));
      }
    } 
    // 步骤④
    else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { 8
      String nodeName = child.getNode().getNodeName();
      // 步骤⑤
      NodeHandler handler = nodeHandlerMap.get(nodeName);
      if (handler == null) {
        throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
      }
      // 步骤⑤
      handler.handleNode(child, contents);
      isDynamic = true;
    }
  }
  return new MixedSqlNode(contents);
}
```

配合着源代码梳理一下执行流程，以便更好的理解Mybatis处理逻辑：

- 获取到`crud标签`下所有的`子节点`，并进行遍历 --> 步骤①
- 判断子节点的类型是否是元素节点（即动态标签节点）
  - 是 --> `动态标签` `--> 步骤④`
    - 根据动态标签名称从nodeHandlerMap获取出对应的处理器  `--> 步骤⑤`
    - 调用处理器的handleNode方法处理  `--> 步骤⑤`
  - 否 --> `txt文本类型的普通SQL语句`
    - 获取到节点中的`Body`即SQL语句，创建`StaticTextSqlNode`对象包装 `--> 步骤③`

从上面的类图我们已经知道`各自动态标签都对应各自的Handle`，那Handle是如何添加到`nodeHandlerMap`中的呢？

答案就在`XMLScriptBuilder`的构造方法中 。

![image-20200501161032311](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501161032.png)



接着我们来看看各个Handle对各自标签的处理过程，我们以`TrimHandler`为准，进入`handleNode`方法一探究竟：

```java
public void handleNode(XNode nodeToHandle, List<SqlNode> targetContents) {
  // 递归处理当前动态标签里包含的其他动态标签，即嵌套动态标签。
  MixedSqlNode mixedSqlNode = parseDynamicTags(nodeToHandle);
  String prefix = nodeToHandle.getStringAttribute("prefix");
  String prefixOverrides = nodeToHandle.getStringAttribute("prefixOverrides");
  String suffix = nodeToHandle.getStringAttribute("suffix");
  String suffixOverrides = nodeToHandle.getStringAttribute("suffixOverrides");
  // 构建TrimSqlNode对象
  TrimSqlNode trim = new TrimSqlNode(configuration, mixedSqlNode, prefix, prefixOverrides, suffix, suffixOverrides);
 	// 添加到集合中
  targetContents.add(trim);
}
```

上诉代码将`<trim/>`签配置的属性都读取出来，并最终生成了一个`TrimSqlNode`对象。**这里我们可以猜测：是不是每一个标签都会对应拥有一个`SqlNode`呢？**

是的你没猜错，Mybatis为每种动态标签都生成了对应的SqlNode。下面为SqlNode的继承图 ：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200501163950.png" alt="image-20200424174851998" style="zoom:50%;" />



**到这里，我们上述的第二个问题就非常清晰了：**

**Mybatis使用不同的`NodeHandler`来处理不同的动态标签，使用不同的`SqlNode`来接收不同动态标签的配置。**



到此SQL解析完成了吗？其实并没有，Mybatis对静态SQL做了进一步的操作，替换`#{}`为`?`。我们知道Mybatis会为静态SQL创建`RawSqlSource`对象。而这个替换操作就是在这里面发生的。

```java
public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
  SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
  Class<?> clazz = parameterType == null ? Object.class : parameterType;
  // 解析SqlSource
  sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<>());
}
```



```java
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
  ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
  // 将 #{} 替换成 ?
  GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
  String sql = parser.parse(originalSql);
  // 最后创建StaticSqlSource的sqlSource
  return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
}
```

至此步骤 ③执行完成，动态SQL和静态SQL解析完成。



#### 构建`MappedStatement`

一切准备工作都已完成，到了最后步骤④，在这个步骤中会将前面几个步骤产生的结果组装到一个叫MappedStatement对象中。

最终将这个对象添加到Configuration中，可自行查看。

```java
builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
    fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
    resultSetTypeEnum, flushCache, useCache, resultOrdered,
    keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
```



此致Mybatis整个启动加载的过程分析完毕。



### 总结

在这个阶段我们清楚的认知了Mybatis对Mapper解析的过程，使用了`XMLStatementBuilder`来解析Mapper中的标签，`XMLScriptBuilder`来解析Mapper中的动态标签。

使用`ResultMapResolver`来解析`ResultMap`标签，并将其每一个子节点映射成了`ResultMapping`对象。

使用不同的`NodeHandler`来处理不同的动态标签，为不同动态标签创建不同的`SqlNode`对象。