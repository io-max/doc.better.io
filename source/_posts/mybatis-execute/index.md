---
title: Mybatis-执行
tags:
  - Java
  - Orm
  - Mybatis
categories: Java
index_img: img/mybatis-logo-480.png
abbrlink: 15687
date: 2019-12-09 09:28:07
---


# Mybatis-语句执行

## 本章讲述的内容

本章主要讲解Mybatis在启动后是如何执行Mapper文件中对应SQL语句的。带着下面几个问题我们正式进入Mybatis语句执行分析。

> 1、Mybatis是如何实现一级缓存的？
>
> 2、Mybatis是如何执行SQL的？有几种执行器？
>
> 3、Mybatis是如何处理查询结果的？



想知道如何使用Mybatis请参考[Mybatis官方文档](https://mybatis.org/mybatis-3/zh/)。

## 相关配置

### 配置XML

```xml
<mapper namespace="io.better.mybatis.mapper.UserMapper">
    <resultMap id="BaseResultMap" type="io.better.mybatis.model.User">
        <!--@mbg.generated-->
        <!--@Table tbl_user-->
        <id column="id" jdbcType="BIGINT" property="id"/>
        <result column="username" jdbcType="VARCHAR" property="username"/>
        <result column="phone" jdbcType="VARCHAR" property="phone"/>
        <result column="address" jdbcType="VARCHAR" property="address"/>
    </resultMap>
    <sql id="Name_Column">
            username
    </sql>
    <sql id="Base_Column_List">
        <!--@mbg.generated-->
        id,
        <include refid="Name_Column"/>,
        phone,
        address
    </sql>
    <select id="selectByPrimaryKey" parameterType="java.lang.Long" resultMap="BaseResultMap">
        <!--@mbg.generated-->
        select
        <include refid="Base_Column_List"/>
        from tbl_user
        where id = #{id,jdbcType=BIGINT}
    </select>
</mapper>
```

### 全局配置XML

```xml
<configuration>
    <properties resource="jdbc.properties"/>
    <settings>
        <setting name="logImpl" value="SLF4J"/>
        <setting name="useGeneratedKeys" value="true"/>
        <setting name="defaultExecutorType" value="BATCH"/>
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
        <!--<package name="io.better.mybatis.mapper"/>-->
        <mapper resource="mybatis/mappers/UserMapper.xml"/>
    </mappers>
</configuration>
```



## 入口

### 测试类

```java
/**
 * @author better create in 2020/4/19 10:30 上午
 */
public class MybatisTest {

    SqlSessionFactory sqlSessionFactory;

    @Before
    public void before() throws IOException {
        String resource = "mybatis/mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    }

    @Test
    public void testList() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        User user = mapper.selectByPrimaryKey(1L);
        System.out.println(user);
    }
}
```

从代码可以看出一共执行了三个操作：

1、调用`SqlSessionFactory`创建`SqlSession`对象。

2、调用SqlSession获取Mapper代理对象。

3、调用Mapper代理对象执行目标方法获取结果。

## 获取SqlSession

### 调用`openSession()`创建SqlSession

```java
public SqlSession openSession() {
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}

private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    // 获取环境配置
    final Environment environment = configuration.getEnvironment();
    // 步骤①
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    // 步骤②
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    // 步骤③
    final Executor executor = configuration.newExecutor(tx, execType);
    // 步骤④
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    // 忽略部分代码
  }
}
```

上诉代码中我们需要重点关注一个点：1、`configuration.getDefaultExecutorType()：`它用于指定Executor的类型，默认即`ExecutorType.SIMPLE`。

在`openSessionFromDataSource`方法中，Mybatis做了下面几个操作：

- 步骤①：创建TransactionFactory，类型为：`JdbcTransactionFactory`。
  - 对应Mybatis配置中 <transactionManager type="JDBC"/>标签。
- 步骤②：创建Transaction事务对象，并对`autoCommit，IsolationLevel` 属性赋值。
- 步骤③：根据`execType`创建执行类型的`Executor`执行器。
- 步骤④：创建`DefaultSqlSession`对象。



#### 创建`Transaction`

源代码如下：

```java
public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
  return new JdbcTransaction(ds, level, autoCommit);
}
```

代码中直接调用`JdbcTransaction`构造器进行Transaction创建。那么JdbcTransaction将就能干什么事情呢？让我们来看看JdbcTransaction的类结构图。

![image-20200428162608413](https://raw.githubusercontent.com/io-better-chen/better.io.images/master/images/image-20200428162608413.png)

从类图可以看出JdbcTransaction主要是负责`提交、回滚、关闭Transaction`，`打开、获取Connection`等操作。



#### 创建`Executor`

`Configuration#newExecutor(Transaction, ExecutorType)`

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  // 判断 executorType 
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    // 批量执行器
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    // 可重用执行器
    executor = new ReuseExecutor(this, transaction);
  } else {
    // 默认执行器
    executor = new SimpleExecutor(this, transaction);
  }
  if (cacheEnabled) {
    // 缓存执行器，类似于一个装饰器
    executor = new CachingExecutor(executor);
  }
  // 将执行器传递给每个插件拦截器
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

代码中可以看出，执行器的类型一共有三种：`批量->BatchExecutor，可重用->ReuseExecutor，简单->SimpleExecutor`。

而`CachingExecutor`是一个装饰类，包装了具体的Executor，主要用于做一些缓存操作。

默认为`SimpleExecutor`执行器，我们这里使用是`BatchExecutor`执行器。可以通过下面的配置修改：

```xml
<setting name="defaultExecutorType" value="BATCH"/>
```

最后将创建好的执行器交给`interceptorChain`，此处代码是为了扩展。

因为代码最终会走向`Plugin.wrap`方法中，并为其创建 `Plugin`这个代理对象（`如果有拦截器处理则创建，没有则无`）。

![image-20200429161316723](https://raw.githubusercontent.com/io-better-chen/better.io.images/master/images/20200429161400.png)



#### 创建`DefaultSqlSession`

源代码：

```java
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;

  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;

  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }
}
```

DefaultSqlSession的构造比较简单，仅仅是对类中部分属性进行赋值。

### 总结

在获取SqlSession阶段主要是初始化了`Transaction，Executor，SqlSession`三个对象。

## 获取指定Mapper

### 调用`getMapper()`获取Mapper

```java
// DefaultSqlSession.getMapper
public <T> T getMapper(Class<T> type) {
  return configuration.getMapper(type, this);
}
// Configuration.getMapper
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}
```

代码最终调用到了Configuration对象中。

因为Mybatis在启动加载阶段将所有的Mapper接口都注册到了`Configuration.mapperRegistry`对象中。



### 创建MapperProxy

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  // 获取到MapperProxy工厂
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  try {
    // 创建MapperProxy对象
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
  }
}
// MapperProxyFactory
public T newInstance(SqlSession sqlSession) {
  // 创建MapperProxy对象
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  // 调用JDK的Proxy.newProxyInstance() 创建实例
  return newInstance(mapperProxy);
}
```

从代码可以看出Mybatis在获取每一个Mapper接口时都为其创建了一个MapperProxy代理对象。最终调用JDK创建代理。

### 总结

**`在获取Mapper接口阶段主要是操作是：为指定Mapper接口创建代理对象MapperProxy`**。



## 执行SQL

### 调用`MapperProxy.invoke`

当调用SqlSession返回Mapper接口中的方法时，会调用到MapperProxy对象的`invoke`方法。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  if (Object.class.equals(method.getDeclaringClass())) {
    return method.invoke(this, args);
  } else {
    // cachedInvoker会生成MapperMethodInvoker对象，并将其缓存
    // 调用invoke执行方法
    return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
  }
}

// cachedInvoker方法
private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
  // 被调用过的方法会被缓存起来，避免重复创建
  return methodCache.computeIfAbsent(method, m -> {
    if (m.isDefault()) {
      if (privateLookupInMethod == null) {
        return new DefaultMethodInvoker(getMethodHandleJava8(method));
      } else {
        return new DefaultMethodInvoker(getMethodHandleJava9(method));
      }
    } else {
      // 默认第一次都会创建PlainMethodInvoker。
      return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
    }
  });
}
```

Mybatis会先调用`cachedInvoker`方法，判断当前执行的方法是否已经创建过`MethodInvoker`：

1、如果没有则创建`MapperMethod`封装当前执行的方法：

​	1、如果执行的方法不是Default方法，则创建`PlainMethodInvoker` 并赋值。

​    2、如果执行的是Default方法，则创建`DefaultMethodInvoker`并赋值。

2、如果已经创建，则直接从`methodCache`中获取。



### 调用`MapperMethodInvoker.invoke`

源代码如下：

```java
public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
  // 调用上面cachedInvoker方法创建的MapperMethod对象，执行SQL
  return mapperMethod.execute(sqlSession, args);
}
```

调用上一步创建的`MapperMethod`对象执行目标Mapper方法。



### 调用`MapperMethod.execute`

源代码如下：

```java
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  // 判断sql指令类型：insert，select，update，delete
  switch (command.getType()) {
    Object param = method.convertArgsToSqlCommandParam(args);
    // 插入指令
    case INSERT: {
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    // 更新指令
    case UPDATE: {
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    // 删除指令
    case DELETE: {
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    // 查询指令
    case SELECT:
      // 
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
				// List结果集
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        // Map结果集
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        // Cursor结果集
        result = executeForCursor(sqlSession, args);
      } else {
        // 单个结果集
        result = sqlSession.selectOne(command.getName(), param);		// 忽略部分代码
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }// 忽略部分代码
  return result;
}
```

我们可以看出`MapperMethod`类是执行Mapper方法的核心类。

在这个方法中我们分别看到了Mybatis对新增，更新，删除，查询所执行的操作。

通过判断`command`指令类型来断定SQL语句的类型。改对象(`command`)是在MapperMethod构造器中被初始化。

`新增，更新，删除，查询`四个操作最终都调用了`sqlSession`中对应的方法。

而`executeForMany，executeForMap，executeForCursor`三个方法最终分别调用了`SqlSession`的`selectList，selectMap，selectCursor`方法。



**`可以看出Mybatis的SQL执行最终是在SqlSession中完成的`**。



### SqlSession



#### 获取单个结果-`selectList`

```java
public <T> T selectOne(String statement, Object parameter) {
	// 调用同类的selectList方法
  List<T> list = this.selectList(statement, parameter);
  if (list.size() == 1) {			// 返回第一个
    return list.get(0);
  }
  else if (list.size() > 1) { // 抛出异常
    throw new TooManyResultsException();
  } else {
    return null;
  }
}

public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    MappedStatement ms = configuration.getMappedStatement(statement);
    // 调用执行器执行查询
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } finally {
  }
}
```

在`selectOne`方法中调用了`selectList`方法。并对结果集进行了判断，超过一个抛出异常。

在`selectList`方法中`Mybatis`获取到了解析`Mapper`文件时生成的`MappedStatement`对象。并将其传递给了`Executor`。



**而上面的`executeForMany，executeForMap，executeForCursor`三个方法查询时最终都调用的是`selectList`方法**。



#### 执行查询-`Executor`

`CachingExecutor.query`

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameterObject);
  // 调用包装的执行器创建缓存Key
  CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
  // 继续调用同类Query方法
  return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

`CachingExecutor.query`

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
  throws SQLException {
  // 获取缓存
  Cache cache = ms.getCache();
  // 默认第一次为空
  if (cache != null) {
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, boundSql);
			// private final TransactionalCacheManager tcm = new TransactionalCacheManager();
      // 从事务缓存管理器中获取，并判断是否已经存在执行过
      List<E> list = (List<E>) tcm.getObject(cache, key);
      // 如果为空，说明没执行过
      if (list == null) {
        // 调用包装执行器执行
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        // 将执行的结果放入缓存中
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  // 调用包装执行器执行query
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

`BaseExecutor.query`

```java
// 父类BaseExecutor.query
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  try {
    queryStack++;
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      // 
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      // 执行查询,最终执行的结果会放入localCache对象中。
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
  return list;
}
```

可以看出三个方法中都是做了很多缓存操作，提高效率，



#### 核心查询-`doQuery`

##### `BatchExecutor.doQuery (Core)`

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    flushStatements();
    Configuration configuration = ms.getConfiguration();
    // 创建StatementHandler
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameterObject, rowBounds, resultHandler, boundSql);
    // 获取Connection，如果没有则使用DataSource进行创建
    Connection connection = getConnection(ms.getStatementLog());
    // 创建Statement对象，并设置timeOut和fetchSize属性
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 进行参数替换
    handler.parameterize(stmt);
    // 执行SQL查询，获取到结果集，调用ResultSetHandler.handleResultSets进行结果集处理。
    return handler.query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

该方法主要的作用是操作`Statement`对象。`1、创建StatementHandler处理对象。2、获取Connection连接。3、初始化Statement对象。4、进行参数替换。5、执行SQL完成查询`。



#### 处理查询结果

##### `ResultSetHandler.handleResultSets`

源代码如下：

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
  final List<Object> multipleResults = new ArrayList<>();

  int resultSetCount = 0;
  ResultSetWrapper rsw = getFirstResultSet(stmt);
	// 获取当前Mapper的ResultMap
  List<ResultMap> resultMaps = mappedStatement.getResultMaps();
  int resultMapCount = resultMaps.size();
	// 遍历resultMaps
  while (rsw != null && resultMapCount > resultSetCount) {
    // 获取resultMap
    ResultMap resultMap = resultMaps.get(resultSetCount);
    // 处理ResultSet
    handleResultSet(rsw, resultMap, multipleResults, null);
    // 获取下一个ResultSet
    rsw = getNextResultSet(stmt);
    cleanUpAfterHandlingResultSet();
    resultSetCount++;
  }
	// 处理ResultSet
  String[] resultSets = mappedStatement.getResultSets();
  if (resultSets != null) {
    while (rsw != null && resultSetCount < resultSets.length) {
      // 获取ResultMapping
      ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
      if (parentMapping != null) {
        String nestedResultMapId = parentMapping.getNestedResultMapId();
        ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
        handleResultSet(rsw, resultMap, null, parentMapping);
      }
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
  }

  return collapseSingleResultList(multipleResults);
}
```



## 未完待续