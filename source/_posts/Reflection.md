---
title: Reflection
date: 2020-05-06 16:34:51
tags: [Reflection,Java]
categories: [Java]
index_img: /images/java_270.png
---

该文章主要讲述Java中Reflection反射相关的API使用，以及一些常见问题。

<!-- more -->

# Jdk-Reflection

## 概述

Java反射机制可以让我们在编译期(Compile Time)之外的运行期(Runtime)检查类，接口，变量以及方法的信息。反射还可以让我们在运行期实例化对象，调用方法，通过调用get/set方法获取变量的值。



## Class

### 概述

`Class的实例表示正在运行的Java应用程序的类和接口`。 枚举是一种类和注释是一种接口。

每个数组都属于一个被映射成Class对象的类，该对象由具有相同元素类型和大小的所有数组共享。 

原始类型`(boolean， byte， char， short， int， long， float，double)`，以及关键字`void`也表示为Class对象。



### 特征

**Class没有公共构造函数。 相反， Class对象由Java虚拟机的类加载并通过调用自动构造defineClass的类加载器方法**。



### 获取Class

获取Class对象的方式一共有三种：通过`Class.forName("全路径包名")，类.class，类实例.getClass()`。

示例代码：

```java
Class<?> stringClass = Class.forName("java.lang.String");
Class<?> integerClass = String.class;
Class<?> stringObjClass = new String("1").getClass();

System.out.println(stringClass == integerClass);
System.out.println(stringObjClass == integerClass);
```

输出结果：

```txt
true
true
```

输出的结果证实了相同元素类型的Class实例是相同且共享的。



### 核心方法

在这个阶段会讲述Class类中常用方法的使用

`ReflectionBean`类：

```java
public class ReflectionBean extends ClassBean {

  private String name;
  private Integer age;
  String phone;
  protected String address;
  public Boolean sex;
  public void reflection() {
  }
  public void reflection(String name) {
  }
  private boolean validInfo() {
    return name != null && name.length() > 0 && Objects.nonNull(age);
  }
  public ReflectionBean(@NotNull String name) {
    this.name = name;
  }
  private ReflectionBean(String name, Integer age) {
    this.name = name;
    this.age = age;
  }
  public ReflectionBean() {
  }
  public static ReflectionBean instance() {
    return new ReflectionBean();
  }
}
```

`ClassBean`类：

```java
public class ClassBean {
    private String className;
    public boolean validClassName() {
        return className != null && className.length() > 0;
    }
    public ClassBean() {
    }
    public ClassBean(String className) {
        this.className = className;
    }
}
```

#### `Construtor`

一个`Constructor`对应着类中的一个构造函数，可以使用`Class.getConstructors()`获取指定类所有的构造函数(私有无法获取)。也可以使用`Class.getConstructor(Class<?> ... paramType)`获取指定参数的构造方法。



测试代码：

```java
@Test
public void testConstructor() throws NoSuchMethodException {

    Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;
    Constructor<ReflectionBean> specificConstructor = reflectionBeanClass.getConstructor(String.class);

    System.out.println(specificConstructor.getName());
    System.out.println(Arrays.toString(specificConstructor.getParameterAnnotations()));
    System.out.println(specificConstructor.getParameterCount());
    System.out.println(Arrays.toString(specificConstructor.getParameterTypes()));

    Constructor<?>[] constructors = reflectionBeanClass.getConstructors();
    System.out.println(Arrays.toString(constructors));
}
```



输出结果：

```txt
io.better.jdk.reflection.ReflectionBean
[[Ljava.lang.annotation.Annotation;@1b701da1]
1
[class java.lang.String]
[public io.better.jdk.reflection.ReflectionBean(java.lang.String), public io.better.jdk.reflection.ReflectionBean()]
```

上面代码中演示了`Class.getConstructors`和`Class.getConstructor(Class<?> ..params)`获取构造方法。

通过Constructor对象我们可以获取到构造方法中的`注解，参数，参数类型，参数数量`等信息，但这两种方式都不能获取私有构造方法。那如何获取私有的构造器呢？

使用`getDeclaredConstructors()`和`getDeclaredConstructor(Class<?> ..params)`可以获取到类中所有的构造方法，包括私有的。

```java
@Test
public void testGetDeclaredConstructors() throws NoSuchMethodException {
  Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;

  // 获取所有构造方法，包括私有
  Constructor<?>[] declaredConstructors = reflectionBeanClass.getDeclaredConstructors();
  // 获取指定构造方法
  Constructor<ReflectionBean> specificDeclaredConstructor = reflectionBeanClass.getDeclaredConstructor(String.class, Integer.class);
}
```

介绍完`Constructor`对象的获取方式后，接下来通过`Constructor`对象来实例化目标对象。

我们可以通过`Constructor.newInstance(Object ...params)`方法传入实际的参数来创建目标对象。

```java
@Test
public void testConstructorInstance() throws Exception {

  Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;
  // 获取指定参数的公共构造方法
  Constructor<ReflectionBean> stringConstructor = reflectionBeanClass.getConstructor(String.class);
  // 实例化
  ReflectionBean reflectionBean = stringConstructor.newInstance("Constructor");
  
  System.out.println(reflectionBean);
  // 获取指定参数的私有构造方法
  Constructor<ReflectionBean> declaredConstructor = reflectionBeanClass.getDeclaredConstructor(String.class, Integer.class);
  // 设置成可访问，不设置，会出现访问私有构造方法
  declaredConstructor.setAccessible(true);
	// 实例化
  reflectionBean = declaredConstructor.newInstance("DeclaredConstructor", 1);
  System.out.println(reflectionBean);
}
```





#### `Method`

讲完`Constructor`的使用，已经能通过其创建对象了，那如何调用目标对象中的方法呢？

与Constructor类似，Class提供了`getMethods，getMethod(String methodName, Class<?> ...paramType)`方法分别获取执行类的`所有方法(包括静态方法，父类继承下来的方法)`和`指定方法`。



```java
@Test
public void testGetMethod() throws Exception {
    Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;

    Method[] methods = reflectionBeanClass.getMethods();

    Method method = reflectionBeanClass.getMethod("reflection", String.class);
    System.out.println("指定方法: " + method.getName());

    for (Method item : methods) {
        System.out.println(item);
    }
}
```

执行结果：

![image-20200506141809485](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200506163345.png)

从结果可以看出，私有方法未被获取到，和Constructor类似这两个方法不能获取私有方法，要想获取私有方法必须使用



```java
@Test
public void testGetDeclaredMethod() throws Exception {
    Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;
		// 获取所有方法，包括私有方法
    Method[] methods = reflectionBeanClass.getDeclaredMethods();
		// 获取指定私有方法
    Method method = reflectionBeanClass.getDeclaredMethod("validInfo");
    System.out.println(method.getName());
		// 获取到所有方法的方法名
    List<String> methodName = Stream.of(methods).map(Method::getName).collect(Collectors.toList());
    System.out.println(methodName);
}
```

执行结果：

```txt
validInfo
[equals, toString, hashCode, getName, setName, instance, reflection, reflection, validInfo, canEqual, getAge, setAge]
```

获取类中方法已经了解，接下一来尝试调用执行一下方法。

与Constructor不太一样，要想执行方法需要调用`invoke(Object obj, Object... args)`。该方法接受两个参数，第一个为目标对象(即拥有此方法的类对象)，第二为方法所需的参数。当调用私有方法时做法与Constructor一致。

```java
@Test
public void testInvokedMethod() throws Exception {
  Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;
  Constructor<ReflectionBean> constructor = reflectionBeanClass.getDeclaredConstructor(String.class, Integer.class);
  constructor.setAccessible(true);
  ReflectionBean object = constructor.newInstance("TEST", 1);

  Method method = reflectionBeanClass.getDeclaredMethod("validInfo");
  method.setAccessible(true);
	// 执行方法，获取结果
  Object methodResult = method.invoke(object);
  System.out.println(methodResult);
}
```



#### `Field`

与Method类似，我们可以使用`getFields，getField(String name)`两个方法来获取类中所有非私有的字段。

```java
@Test
public void testGetField() throws Exception {
    Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;

    Field[] fields = reflectionBeanClass.getFields();
    for (Field field : fields) {
        System.out.println(field);
    }
}

// 结果：public java.lang.Boolean io.better.jdk.reflection.ReflectionBean.sex
```

这两个方法只能获取`公共字段(public修饰)`，`private，default，protected`都获取不到。

我们可以使用`getDeclaredFields`，`getDeclaredField`两个方法来获取所有的字段和指定字段，

```java
@Test
public void testGetDeclaredField() throws Exception {
    Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;

    Field[] fields = reflectionBeanClass.getDeclaredFields();
    for (Field field : fields) {
        System.out.println(field);
    }
    Field age = reflectionBeanClass.getDeclaredField("age");
    System.out.println(age);
}
```

执行结果：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200506163353.png" alt="image-20200506150309477" style="zoom:50%;" />

如果想要获取某个字段对应的值，则需要调用`get(Object obj)`方法，传入包含此字段的目标类对象，私有字段需要调用`setAccessible()`方法。

```java
@Test
public void testFieldMethod() throws Exception {
    Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;

    Constructor<ReflectionBean> constructor = reflectionBeanClass.getDeclaredConstructor(String.class, Integer.class);
    constructor.setAccessible(true);
    ReflectionBean object = constructor.newInstance("TEST", 1);

    Field age = reflectionBeanClass.getDeclaredField("age");
    age.setAccessible(true);
    System.out.println(age.getDeclaringClass());
    System.out.println(age.get(object));
}
// 执行结果
// class io.better.jdk.reflection.ReflectionBean
// 1
```



#### 总结

`Constructor，Method，Field`三个类分别 对应类中 的`构造方法、方法、字段`，分别提供了`getDeclared`开头的方法来获取对应所有的信息。



### 其他方法

#### `newInstance()`

在获取到类的Class对象时，可以通过调用此方法来创建目标类对象，该方法会默认调用类的`无参构造方法`来创建对象。

```java
@Test
public void testNewInstance() throws Exception {
  Class<ReflectionBean> reflectionBeanClass = ReflectionBean.class;
  ReflectionBean reflectionBean = reflectionBeanClass.newInstance();
  System.out.println(reflectionBean);
}
```



#### `isInstance(Object obj)`

该方法用于断定指定Object是赋值兼容与此表示的对象Class ，等效`instanceof`运算符。

```java
Class<String> stringClass = String.class;

stringClass.isInstance("test"); // 结果=true，类型匹配
stringClass.isInstance(123456); // 结果=false，类型不匹配
```





## 使用场景

### 反射进行数组扩容

```java
@Test
public void testArrayExtension() {
  String[] stringArray = new String[]{"1", "2", "3", "4", "5"};
  System.out.println("原始数组长度: " + stringArray.length);

  Object newStringArray = ArrayExtension(stringArray, stringArray.length * 2);
  System.out.println("扩容后数组长度: " + Array.getLength(newStringArray));
}

public Object ArrayExtension(Object sourceArray, int extensionLength) {
  if (extensionLength < Array.getLength(sourceArray))
    throw new IllegalArgumentException("扩展长度不合法");

  Class<?> componentType = sourceArray.getClass().getComponentType();
  Object newStringArray = Array.newInstance(componentType, extensionLength);
  System.arraycopy(sourceArray, 0, newStringArray, 0, Array.getLength(sourceArray));

  return newStringArray;
}
```



### 反射跳过泛型检查

```java
@Test
public void testJumpTypeCheck() throws Exception {
  List<String> stringList = new ArrayList<>();

  stringList.add("a");
  stringList.add("b");
  stringList.add("c");

  Class<? extends List> listClass = stringList.getClass();

  Method addMethod = listClass.getMethod("add", Object.class);
  // 向集合中添加了一个int类型的值
  addMethod.invoke(stringList, 1);
  // 结果为4
  System.out.println(stringList.size());
}
```