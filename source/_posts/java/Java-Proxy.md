---
title: Proxy-代理
tags: [Java, Proxy]
categories: Proxy
index_img: /img/01/06.png
abbrlink: 22654
date: 2020-05-06 16:34:51
---

该文章主要讲述代模式以及其实现方式，主要是 `静态代理` , `动态代理`, `Cglib代理`.

<!-- more -->

# Proxy

## 概述

[代理模式](https://zh.wikipedia.org/wiki/代理模式)是一种设计模式，提供了对目标对象额外的访问方式，即通过代理对象访问目标对象，这样可以在不修改原目标对象的前提下，提供额外的功能操作，扩展目标对象的功能。

简言之，代理模式就是设置一个中间代理来控制访问原目标对象，以达到增强原对象的功能和简化访问方式。

在`Java`中实现代理主要有三种方式：

- 静态代码
- 动态代理
- Cglib代理

## 静态代理

在静态代理中，需要为每一个被代理对象创建一个代理类，并实现同一个接口。

示例代码：

```java
// 父接口
public interface IConsumer {
  // 消费
  String consumer();
}
// 代理对象
class ConsumerProxy implements IConsumer {
  private Consumer consumer;

  public ConsumerProxy(Consumer consumer) {
    this.consumer = consumer;
  }
  @Override
  public String consumer() {
    System.out.println("代理对象执行前的代码");
    String consumer = this.consumer.consumer();
    System.out.println("代理对象执行后的代码");
    return consumer;
  }
}
// 被代理对象
class Consumer implements IConsumer {
  @Override
  public String consumer() {
    System.out.println("消费方法被调用了");
    return "Success";
  }
}
```

从代码中看出，`ConsumerProxy`代理对象持有了`Consumer`被代理对象的引用，并在`consumer方法`中调用了`被代理对象的consumer方法`。来看看实际测试代码：

测试代码：

```java
@Test
public void testStaticProxy() {
    Consumer consumer = new Consumer();
    IConsumer consumerProxy = new ConsumerProxy(consumer);
    System.out.println(consumerProxy.consumer());
}
```

输出结果：

```txt
代理对象执行前的代码
消费方法被调用了
代理对象执行后的代码
Success
```

**优点：可以最大程度扩展被代理对象的功能。**

**缺点：被代理对象会随着代理对象的增加而增加，代码冗余。如果接口新增方法，代理对象和被代理对象都需要实现。**



基于静态代理的缺点，有没有一种代理能够动态的生成代理对象呢？

## 动态代理

### 简介

动态代理利用了[JDK API](http://tool.oschina.net/uploads/apidocs/jdk-zh/)，动态的在内存中构建代理对象，从而实现对目标对象的代理功能。

动态代理又被称为JDK代理或接口代理。

相比于静态代理， 动态代理的优势在于可以很方便的对代理类的函数进行统一的处理，而不用修改每个代理类的函数。

先来看一个示例，了解一下动态代理的基本使用。

### 示例代码

```java
// ①
public interface IDynamic {
    void dynamicProxy();
}
// ②
class Dynamic implements IDynamic {
    @Override
    public void dynamicProxy() {
        System.out.println("目标方法执行了");
    }
}
// ③
class DynamicProxyInvocation implements InvocationHandler {
    private Object proxyTarget;

    public DynamicProxyInvocation(Object proxyTarget) {
        this.proxyTarget = proxyTarget;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        System.out.println("代理执行前");
        Object invokeResult = method.invoke(proxyTarget, args);
        System.out.println("代理执行后");
        return invokeResult;
    }
}
```

#### 测试类

```java
@Test
public void testDynamicProxy() {
    Dynamic proxyTarget = new Dynamic();
    IDynamic dynamicProxy = (IDynamic) Proxy.newProxyInstance(proxyTarget.getClass().getClassLoader(),
            proxyTarget.getClass().getInterfaces(), new DynamicProxyInvocation(proxyTarget));
    dynamicProxy.dynamicProxy();
}
```

#### 测试结果

```txt
代理执行前
目标方法执行了
代理执行后
```



在上面的代码中，看到了很多未知的接口和类，主要是`Proxy类`，`InvocationHandler接口`。



### `Proxy`

#### 疑问

`Proxy.newProxyInstance()`方法返回的`代理类是如何生成`？为什么Jdk 的动态代理`被代理类必须实现接口`？



#### 简介

Proxy提供用于创建动态代理类和实例的静态方法，它还是由这些方法创建的所有动态代理类的`超类`。



#### 源代码

从上面的示例代码可以看出代理类的实例是由`Proxy.newProxyInstance`返回的，那么我们重点关注`Proxy.newProxyInstance`这个方法。



##### 创建代理实例

```java
protected InvocationHandler h;

protected Proxy(InvocationHandler h) {
  Objects.requireNonNull(h);
  this.h = h;
}

public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h) {
  // 忽略部分代码
  
  // 步骤①
  Class<?> cl = getProxyClass0(loader, intfs);

  if (sm != null) {
    checkNewProxyPermission(Reflection.getCallerClass(), cl);
  }
	// 步骤②
  final Constructor<?> cons = cl.getConstructor(constructorParams);
  final InvocationHandler ih = h;
  if (!Modifier.isPublic(cl.getModifiers())) {
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
      public Void run() {
        cons.setAccessible(true);
        return null;
      }
    });
  }
  return cons.newInstance(new Object[]{h});
  // 忽略部分代码
}
```

在`newProxyInstance`方法中有两个比较核心的步骤，分别如下：

步骤①：获取代理类的`Class`实例。

步骤②：获取代理类带有`InvocationHandler`参数的构造方法。

除了上面两个比较重要的步骤，还需要关注`InvocationHandler h`和`Proxy(InvocationHandler h)`，后面会进行讲解。



##### 获取代理类字节码

`getProxyClass0`最终会调用到`ProxyClassFactory.apply()`方法中，具体操作细节，可自行Debug查看调用链。

```java
private static final class ProxyClassFactory implements BiFunction<ClassLoader, Class<?>[], Class<?>> {

  private static final String proxyClassNamePrefix = "$Proxy";

  @Override
  public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

    // 忽略部分代码

    long num = nextUniqueNumber.getAndIncrement();
    String proxyName = proxyPkg + proxyClassNamePrefix + num;
		// 步骤①
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
      proxyName, interfaces, accessFlags);
    try {
   		// 步骤② 定义Class，native方法
      return defineClass0(loader, proxyName,
                          proxyClassFile, 0, proxyClassFile.length);
    } catch (ClassFormatError e) {
      throw new IllegalArgumentException(e.toString());
    }
}
```

步骤①：调用`ProxyGenerator.generateProxyClass`生成代理类.class的字节数组。

步骤②：调用`defineClass0`生成.class文件，并加载到Jvm中。



#### 手动生成Class

从上面我们得知了Proxy类在最后调用了`ProxyGenerator.generateProxyClass`方法生成了代理类的`.class`字节数组。那么生成的`.class`结构是怎样的呢？让我们来手动触发调用一下。



##### 示例代码

```java
public class ProxyGeneratorTest {

  public static void main(String[] args) throws Exception {
    byte[] dynamicObj = ProxyGenerator.generateProxyClass(
      "ManualGeneratorDynamicClass", new Class[]{IDynamic.class}, 17);

    FileOutputStream out = new FileOutputStream(new File("ManualGeneratorDynamicClass.class"));
    out.write(dynamicObj);
    out.flush();
    out.close();
  }
}
```

##### 代理类`.class`文件

```java
public final class ManualGeneratorDynamicClass extends Proxy implements IDynamic {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;
		
  	// 调用父类Proxy的构造器对父类中InvocationHandler属性进行了赋值
  	// 而这个构造器是在Proxy.newProxyInstance()方法中被调用
    public ManualGeneratorDynamicClass(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void dynamicProxy() throws  {
        try {
          	// 调用了父类的InvocationHandler实例的invoke方法
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
          	// 获取到被代理类实现的接口中的方法
            m3 = Class.forName("io.better.jdk.proxy.dynamicproxy.IDynamic").getMethod("dynamicProxy");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

从代码可以看出`ManualGeneratorDynamicClass`类不仅继承了`Proxy`类（解释了Porxy为什么是所有代理类的超类），还实现了`被代理类`实现的接口（解释了为什么被代理类必须实现接口？）。



### `InvocationHandler`

通过上面对Proxy的了解，我们知道了代理类的方法调用最终会调用到InvocationHandler实例的invoke方法。

#### 简介 

InvocationHandler是代理实例的调用处理程序（InvocationHandler实例）实现的接口。
`每个代理实例都有一个关联的调用处理程序`。 当一个方法是在代理实例调用，方法调用进行编码，并分发给invoke的调用处理程序的方法。

#### 源代码

```java
public interface InvocationHandler {

  // 处理代理实例的方法调用并返回结果。
  // 该方法将在调用处理程序时的方法是在一个代理实例，它与相关的调用来调用。
  // proxy: 类型为Proxy
  // method: 目标执行的方法
  // args: 方法执行所需的参数
  public Object invoke(Object proxy, Method method, Object[] args)
    throws Throwable;
}
```



### 总结

#### 优缺点

优点：

- 运行时动态生成代理类，和被代理类解耦。

缺点：

- 被代理类必须实现接口，否则不能使用动态代理。



#### 代理机制

动态代理类（以下简称为代理类）是一种类，该类实现`创建类时(调用newProxyInstance方法时)`在运行时指定的`接口列表(interfaces参数)`，代理接口是由代理类实现的接口。代理实例是代理类的实例。`每个代理实例都有一个关联的调用处理程序对象，该对象实现接口InvocationHandler`。

通过其代理接口之一对代理实例进行的方法调用将分派给该实例的调用处理程序的invoke方法，并传递该`代理实例（proxy参数）`，一个标识所调用方法的`java.lang.reflect.Method对象（method参数）`以及一个数组包含参数的Object类型`（args参数）`。

## Cglib代理

### 前言

上面我使用了动态代理，知道了动态代理一些优缺点，为了弥补Jdk动态代理的缺点，Cglib诞生了，被代理类无需实现接口也能被代理。

### 简介

`cglib`-字节码生成库是用于生成和转换Java字节码的高级API。AOP，测试，数据访问框架使用它来生成动态代理对象并拦截字段访问。



### 示例代码

```java
package io.better.jdk.proxy.cglibproxy;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;
import java.util.Objects;

/**
 * @author better create in 2020/5/8 5:56 下午
 */
public class CglibBean {

    void proxy() {
        System.out.println("proxy execute ....");
    }
}

class CglibProxyFactory implements MethodInterceptor {

    private Object proxyTarget;

    public CglibProxyFactory setProxyTarget(Object proxyTarget) {
        this.proxyTarget = proxyTarget;
        return this;
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("cglib 执行前");
        Object result = method.invoke(proxyTarget, objects);
        System.out.println("cglib 执行后");
        return result;
    }

    public Object getProxyInstance() {
        if (Objects.isNull(proxyTarget)) {
            throw new IllegalArgumentException("被代理对象不能为空");
        }
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(proxyTarget.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }
}
```

#### 测试代码

```java
@Test
public void testCglib() {
    CglibBean proxyTarget = new CglibBean();
    CglibBean proxyInstance = (CglibBean) new CglibProxyFactory().setProxyTarget(proxyTarget).getProxyInstance();
    proxyInstance.proxy();
}
```

#### 测试结果

```txt
cglib 执行前
proxy execute ....
cglib 执行后
```



### 源代码

从示例代码可以看出`Enhancer`类是创建代理对象的核心，那么Enhancer是如何创建代理类的呢？创建的代理类结构是如何呢？

在代码中一共操作了四部：

1. 创建Enhancer对象。

2. 调用setSuperclass设置父类。

3. 调用setCallback设置回调。

4. 调用create创建代理实例。



#### 创建Enhancer对象

##### 构造器描述

创建一个新的增强器。每个生成的对象都应使用一个新的Enhancer对象，并且不应在线程之间共享。要创建生成的类的其他实例，请使用Factory接口。



##### 继承图

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200509172422.png" alt="image-20200509104246223" style="zoom:50%;" />

##### Enhancer构造器

```java
private static final Source SOURCE = new Source(Enhancer.class.getName());

public Enhancer() {
    super(SOURCE);
}

// 父类AbstractClassGenerator构造器
protected AbstractClassGenerator(Source source) {
  this.source = source;
}
```

代码中将Enhancer的名称封装到了Source实例中，并调用父类AbstractClassGenerator构造器进行赋值 。



#### 设置父类

##### 方法描述

设置生成的类将继承的类。 为了方便起见，如果提供的超类实际上是接口，则将使用适当的参数来调用setInterfaces。 非接口参数不能声明为final，并且必须具有可访问的构造函数。



##### 方法`setSuperclass`

```java
private Class superclass;

public void setSuperclass(Class superclass) {
	// 如果为接口，则获取并调用setInterfaces方法
  if (superclass != null && superclass.isInterface()) {
    setInterfaces(new Class[]{ superclass });
  } else if (superclass != null && superclass.equals(Object.class)) {
    // affects choice of ClassLoader
    this.superclass = null;
  } else {
    // 给superclass字段赋值
    this.superclass = superclass;
  }
}

private Class[] interfaces;

public void setInterfaces(Class[] interfaces) {
  this.interfaces = interfaces;
}
```

方法逻辑比较简单就是给Enhancer实例中的字段进行赋值。



#### 设置回调

##### 方法描述

设置要使用的单个回调。 如果使用createClass则被忽略。



##### 方法`setCallback`

```java
private Callback[] callbacks;

public void setCallback(final Callback callback) {
    setCallbacks(new Callback[]{ callback });
}

public void setCallbacks(Callback[] callbacks) {
  if (callbacks != null && callbacks.length == 0) {
    throw new IllegalArgumentException("Array cannot be empty");
  }
  this.callbacks = callbacks;
}
```

该方法也是给Enhancer实例中的`callbacks`字段进行赋值



#### 创建代理实例

##### 方法描述

如有必要，生成一个新类，并使用指定的回调（如果有）来创建一个新的对象实例。 使用超类的no-arg构造函数。



##### 入口-`Enhancer.create`

```java
public Object create() {
    classOnly = false;
    argumentTypes = null;
    return createHelper();
}
```

`createHelper`

```java
private Object createHelper() {
  preValidate();
  Object key = KEY_FACTORY.newInstance((superclass != null) ? superclass.getName() : null,
                                       ReflectUtils.getNames(interfaces),
                                       filter == ALL_ZERO ? null : new WeakCacheKey<CallbackFilter>(filter),
                                       callbackTypes,
                                       useFactory,
                                       interceptDuringConstruction,
                                       serialVersionUID);
  this.currentKey = key;
  Object result = super.create(key);
  return result;
}
```

`KEY_FACTORY.newInstance`生成的`key`需要特别注意，后面在`生成代理类Class时会用此key与Class一对一绑定`。

继续查看父类的`create`方法。



#### `AbstractClassGenerator.create`

```java
protected Object create(Object key) {
  try {
    ClassLoader loader = getClassLoader();
    Map<ClassLoader, ClassLoaderData> cache = CACHE;
    ClassLoaderData data = cache.get(loader);
    // 忽略部分代码
    this.key = key;
    // 步骤①，创建代理类字节码核心入口
    Object obj = data.get(this, getUseCache());
    if (obj instanceof Class) {
      return firstInstance((Class) obj);
    }
    // 步骤②，根据创建代理类实例
    return nextInstance(obj);
  } catch (RuntimeException e) {
  }
}
```

上诉代码中忽略了部分代码，重点关注`步骤①` 和`步骤②`对应的两个方法。

步骤①：调用`ClassLoaderData.get()`获取代理类Class对象。

步骤②：使用代理类Class对象创建代理实例。



##### 步骤① 

知道了代理对象是通过`ClassLoaderData.get`方法获取的，那么必须了解`ClassLoaderData`的作用及其结构。



###### `ClassLoaderData`

那么`ClassLoaderData`类有什么作用呢？通过Debug来看看ClassLoaderData内部结构。

![image-20200509150401851](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200509172428.png)

类结构图：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200509172432.png" alt="image-20200509150935264" style="zoom:50%;" />

`generatedClasses`：用于存储已经生成的Class对象。

`reservedClassNames`：用于存储已经解析的Class名称（全路径）。

`classLoader`：加载生成Class对象的加载器。

可以看出ClassLoaderData内部管理生成的Class类和加载Class所需的ClassLoader，可以简单`理解为生成Class并存储Class的容器`。



###### `init`

那么`ClassLoaderData`是如何初始化的呢 ？我们进入ClassLoaderData的构造器看看：

```java
public ClassLoaderData(ClassLoader classLoader) {
  if (classLoader == null) {
    throw new IllegalArgumentException("classLoader == null is not yet supported");
  }
  // 设置加载类使用的ClassLoader
  this.classLoader = new WeakReference<ClassLoader>(classLoader);
  // 声明了load函数
  Function<AbstractClassGenerator, Object> load =
    new Function<AbstractClassGenerator, Object>() {
    public Object apply(AbstractClassGenerator gen) {
      Class klass = gen.generate(ClassLoaderData.this);
      return gen.wrapCachedClass(klass);
    }
  };
  // 在这里对generatedClasses做了初始化
  generatedClasses = new LoadingCache<AbstractClassGenerator, Object, Object>(GET_KEY, load);
}
```

可以看出`classLoader，generatedClasses`两个对象被进行了初始化，在这里重点注意`load`这个函数 ，`这个函数就是创建代理类Class的关键`。



###### `ClassLoaderData.get`

了解了`ClassLoaderData`后，我们进入 `get()`方法一探究竟：

```java
public Object get(AbstractClassGenerator gen, boolean useCache) {
	// useCache默认值为true
  if (!useCache) {
    return gen.generate(ClassLoaderData.this);
  } else {
    // 从缓存中获取缓存的对象
    Object cachedValue = generatedClasses.get(gen);
    return gen.unwrapCachedValue(cachedValue);
  }
}
```

如果不修改useCache的值，代码最终会调用`generatedClasses.get`方法。到这里是不是感觉`generatedClasses`这个对象是不是非常眼熟，没错他就是`ClassLoaderData中存放生成过Class的对象`。

接着进入generatedClasses对象一探究竟。



###### `LoadingCache`

在如何`LoadingCache.get`方法前，我们先来看看`LoadingCache`的构造函数：

```java
// 上面ClassLoaderData构造器中最后一步会调用
public LoadingCache(Function<K, KK> keyMapper, Function<K, V> loader) {
    this.keyMapper = keyMapper;  // 用于获取 KEY_FACTORY.newInstance 创建的key的函数
    this.loader = loader;   // 用于生成代理类的Class函数
    this.map = new ConcurrentHashMap<KK, Object>();
}
```

LoadingCache构造函数主要是在对`自身变量进行赋值`操作。

`loader`：类型为函数，用于创建代理类Class

`keyMapper`：类型为函数，用于获取前面`Enhancer.create`方法中通过`KEY_FACTORY.newInstance`创建的`key`

`map`：key=`keyMapper函数获取到的key`，value=`loader函数生成的代理Class数据`。



###### `LoadingCache.get`

```java
public V get(K key) {
  // 获取到 KEY_FACTORY.newInstance 创建的key
  final KK cacheKey = keyMapper.apply(key);
  // 查看是否已经存在
  Object v = map.get(cacheKey);
  if (v != null && !(v instanceof FutureTask)) {
    return (V) v;
  }
  // 不存在，则创建
  return createEntry(key, cacheKey, v);
}
```



###### `LoadingCache.createEntry`

```java
protected V createEntry(final K key, KK cacheKey, Object v) {
  // key = AbstractClassGenerator
  // cacheKey = Enhancer.EnhancerKey
  
  FutureTask<V> task;
  boolean creator = false;
  
  if (v != null) {
    task = (FutureTask<V>) v;
  } else {
    // 创建一个Task
    task = new FutureTask<V>(new Callable<V>() {
      public V call() throws Exception {
        // 到这来我们终于看到了ClassLoaderData构造器中声明的load函数被执行了
        // (最后一步调用LoadingCache构造器，传递给LoadingCache.loader属性)
        return loader.apply(key);
      }
    });
    // 缓存Key和Task放入到map中缓存
    Object prevTask = map.putIfAbsent(cacheKey, task);
    if (prevTask == null) {
      creator = true;
      // 执行Task
      task.run();
    }
  }

  V result;
  try {
    // 获取结果
    result = task.get();
  } catch (InterruptedException e) {}

  if (creator) {
    // 将缓存Key和生成Class对象放入到map中
    map.put(cacheKey, result);
  }
  return result;
}
```

这个方法主要是创建FutureTask用于异步创建Class对象，并对其结果进行了缓存，提高性能。

接下来调用`load.apply`执行函数，最终调用至`AbstractClassGenerator.generate`方法中。

```java
Function<AbstractClassGenerator, Object> load = new Function<AbstractClassGenerator, Object>() {
  public Object apply(AbstractClassGenerator gen) {
    Class klass = gen.generate(ClassLoaderData.this);
    return gen.wrapCachedClass(klass);
  }
};
```



###### `AbstractClassGenerator.generate`

```java
protected Class generate(ClassLoaderData data) {
  Class gen;
  // 从ThreadLocal获取对象，默认应该为null
  Object save = CURRENT.get();
  // 设置ThreadLocal，保证此AbstractClassGenerator不被线程共享
  CURRENT.set(this);
  try {
    // 获取到加载Class字节码使用的ClassLoader
    ClassLoader classLoader = data.getClassLoader();

		// 步骤①
    byte[] b = strategy.generate(this);
    String className = ClassNameReader.getClassName(new ClassReader(b));
    ProtectionDomain protectionDomain = getProtectionDomain();
    synchronized (classLoader) { // just in case
      if (protectionDomain == null) {
        // 步骤②
        gen = ReflectUtils.defineClass(className, b, classLoader);
      } else {
        // 步骤②
        gen = ReflectUtils.defineClass(className, b, classLoader, protectionDomain);
      }
    }
    return gen;
  } catch (RuntimeException e) {
  } finally {
    // 置为null
    CURRENT.set(save);
  }
}
```

步骤①：

调用`strategy.generate`方法生成代理类字节码数组。

其默认实例为`GeneratorStrategy strategy = DefaultGeneratorStrategy.INSTANCE;`。

`strategy.generate`方法最终会调用到`Enhancer.generateClass(ClassVisitor v)`方法，这里面包含了生成代理类字节码具体步骤（这里了不做讲解，有兴趣的可自行查看）。



步骤②：

调用`ReflectUtils.defineClass`方法使用传入的ClassLoader加载生成的代理类字节码数组。

```java
public static Class defineClass(String className, byte[] b, ClassLoader loader, ProtectionDomain protectionDomain) throws Exception {
  Class c;
  if (DEFINE_CLASS != null) {
    Object[] args = new Object[]{className, b, new Integer(0), new Integer(b.length), protectionDomain };
    // 步骤①，使用ClassLoader加载字节码信息 
    c = (Class)DEFINE_CLASS.invoke(loader, args);
  } 
  // 忽略部分代码
  
  Class.forName(className, true, loader);
  return c;
}
```

```java
private static Method DEFINE_CLASS, DEFINE_CLASS_UNSAFE;
```

`DEFINE_CLASS`其实是`java.lang.ClassLoader.defineClass`对应的Method对象。



##### 步骤②

走完步骤①代理类的Class对象已生成，接下来就是通过该Class对象生成代理实例。

我们进入`nextInstance(obj);`方法查看实例化流程：

```java
protected Object nextInstance(Object instance) {
  EnhancerFactoryData data = (EnhancerFactoryData) instance;

  if (classOnly) {
    return data.generatedClass;
  }

  Class[] argumentTypes = this.argumentTypes;
  Object[] arguments = this.arguments;
  if (argumentTypes == null) {
    argumentTypes = Constants.EMPTY_CLASS_ARRAY;
    arguments = null;
  }
  // 步骤①
  return data.newInstance(argumentTypes, arguments, callbacks);
}
```

该方法在调用代理类Class构造函数前，处理好对应的构造函数参数类型和参数。

重点关注步骤①：

```java
public Object newInstance(Class[] argumentTypes, Object[] arguments, Callback[] callbacks) {
  setThreadCallbacks(callbacks);
  try {
    // Explicit reference equality is added here just in case Arrays.equals does not have one
    if (primaryConstructorArgTypes == argumentTypes ||
        Arrays.equals(primaryConstructorArgTypes, argumentTypes)) {
			// 创建代理实例
      return ReflectUtils.newInstance(primaryConstructor, arguments);
    }
    // 创建代理实例
    return ReflectUtils.newInstance(generatedClass, argumentTypes, arguments);
  } finally {
    setThreadCallbacks(null);
  }

}
```

至此Cglib创建代理对象流程分析完毕。



#### 使用Cglib手动生成Class文件

分析完Cglib整个创建流程，我还不能像Jdk动态代理一样了解到生成的代理类字节码到底是怎样的？接下来我们使用Cglib手动生成一个代理类的Class文件。

由于`strategy.generate`方法所需参数较为复杂，可`Debug`至`byte[] b = strategy.generate(this);`这行代码利用IDEA的`Evaluate Expression`功能手动输入以下代码：

```java
FileOutputStream out = new FileOutputStream(new File("ManualGeneratorProxyCglibProxy.class"));
out.write(b);
out.flush();
out.close();
```

生成文件如下：

```java
public class CglibBean$$EnhancerByCGLIB$$70184645 extends CglibBean implements Factory {
  private boolean CGLIB$BOUND;
  public static Object CGLIB$FACTORY_DATA;
  private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
  private static final Callback[] CGLIB$STATIC_CALLBACKS;
  private MethodInterceptor CGLIB$CALLBACK_0; // 我们自定义的MethodInterceptor
  private static Object CGLIB$CALLBACK_FILTER;
  private static final Method CGLIB$proxy$0$Method;  // CglibBean.proxy调用方法
  private static final MethodProxy CGLIB$proxy$0$Proxy;  // CglibBean.proxy代理方法
  private static final Object[] CGLIB$emptyArgs;

  static void CGLIB$STATICHOOK1() {
    CGLIB$THREAD_CALLBACKS = new ThreadLocal();
    CGLIB$emptyArgs = new Object[0];
    // 通过反射得到代理类的Class对象
    Class var0 = Class.forName("io.better.jdk.proxy.cglibproxy.CglibBean$$EnhancerByCGLIB$$70184645");
    Class var1;
    // 获取到被代理类所有的方法，找到proxy，返回类型为void的方法对应的Method对象
    CGLIB$proxy$0$Method = ReflectUtils.findMethods(new String[]{"proxy", "()V"}, (var1 = Class.forName("io.better.jdk.proxy.cglibproxy.CglibBean")).getDeclaredMethods())[0];
    // 为proxy方法生成MethodProxy对象
    // var1=被代理类的Class对象
    // var2=代理类的Class对象
    CGLIB$proxy$0$Proxy = MethodProxy.create(var1, var0, "()V", "proxy", "CGLIB$proxy$0");
  }

  final void CGLIB$proxy$0() {
    super.proxy();
  }

  final void proxy() {
    // 获取到我们自定义的MethodInterceptor实例
    MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
    if (var10000 == null) {
      CGLIB$BIND_CALLBACKS(this);
      var10000 = this.CGLIB$CALLBACK_0;
    }

    if (var10000 != null) {
      // 调用MethodInterceptor.intercept
      var10000.intercept(this, CGLIB$proxy$0$Method, CGLIB$emptyArgs, CGLIB$proxy$0$Proxy);
    } else {
      super.proxy();
    }
  }

  static {
    CGLIB$STATICHOOK1();
  }
}
```

上诉代码中忽略了`equals，hashCode，toString`等方法。感兴趣的同学可以自己操作一下。

## 总结

1. 静态代理实现较简单，只要代理对象对目标对象进行包装，即可实现增强功能，但静态代理只能为一个目标对象服务，如果目标对象过多，则会产生很多代理类。
2. JDK动态代理需要目标对象实现业务接口，代理类只需实现InvocationHandler接口。
3. 动态代理生成的类为 lass com.sun.proxy.$Proxy4，cglib代理生成的类为class com.cglib.UserDao$$EnhancerByCGLIB$$552188b6。
4. 静态代理在编译时产生class字节码文件，可以直接使用，效率高。
5. 动态代理必须实现InvocationHandler接口，通过反射代理方法，比较消耗系统性能，但可以减少代理类的数量，使用更灵活。
6. cglib代理无需实现接口，通过生成类字节码实现代理，比反射稍快，不存在性能问题，但cglib会继承目标对象，需要重写方法，所以目标对象不能为final类