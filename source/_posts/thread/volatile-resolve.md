---
title: Volatile 解析
tags:
  - Java
  - Juc
  - Volatile
categories: volatile
index_img: img/java-logo-480.png
abbrlink: 40532
---

> 本章讲解如下：
>
> - Volatile 关键字的作用？
> - Volatile 是如何保证有序性？如何保证可见性？

<!-- more -->


## _Volatile_

### 简介

知道了 Synchronized 关键字可以保证在高并发下的`可见性，有序性，原子性`，今天来看看 Volatile 这个关键字的作用。

### 作用

保证在多线程的情况下程序的`可见性`和`有序性`。



对 Volatile 修饰变量的单次读写是可以保证原子性的，复合操作则不行例如：++i 或 i+=1



### 实现原理

从两个方面讲解 Volatile 的实现原理，分别是：`可见性的实现`和`有序性的实现`



#### 可见性实现原理

> 可见性的实现是基于`内存屏障`

##### **什么是内存屏障？**

所谓内存屏障就是内存栅栏，即一道 CPU 指令。



##### lock 指令

而 Volatile 就是使用 lock 前缀指令来实现的，在多核 CPU 下遇到 lock 前缀会引发触发下拉事件：

- 将当前处理器缓存行中的数据写回内存。
- 写回内存的操作会使缓存了此内存地址的数据无效

汇编代码如下：

![image-20210315143739503](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210315143739503.png)



如果对声明了 volatile 的变量进行写操作，JVM 就会向处理器发送一条 lock 前缀的指令，将这个变量所在缓存行的数据写回到系统内存。

为了保证各个处理器的缓存是一致的，实现了`缓存一致性协议(MESI)`，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是否过期，当处理器发现自己缓存行对应的内存地址被修改（Modify），就会将当前处理器的缓存行设置成无效状态（Invalid），当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。



##### 缓存行

在早期的CPU 中 lock 前缀指令会产生一个`LOCK#信号`，会导致锁住整个`总线`，其他CPU 对内存的读写都将阻塞。

这会大大的影响性能，后来的 CPU 都将范围缩小到各个 CPU 的高速缓存。通过缓存一致性协议实现。



而缓存是分段（一行一行）的，一个段对应一块存储空间，称之为缓存行，它是 CPU 缓存中可分配的最小存储单元，大小 32 字节、64 字节、128 字节不等，通常来说是 64 字节。



**示例代码**

```java
public static T[] arr = new T[2];

static {
  arr[0] = new T();
  arr[1] = new T();
}
public static void main(String[] args) throws Exception {
  Thread one = new Thread(() -> {
    for (long i = 0; i < 1000_0000L; i++) {
      arr[0].x = i;
    }
  });

  Thread two = new Thread(() -> {
    for (long i = 0; i < 1000_0000L; i++) {
      arr[1].x = i;
    }
  });

  final long start = System.nanoTime();
  one.start();
  two.start();

  one.join();
  two.join();
  System.out.println((System.nanoTime() - start) / 100_0000);
}

public static class T {
  // public volatile long p1, p2, p3, p4, p5, p6, p7;
  public volatile long x = 0L;
}
```

上面的代码中两个线程都对 x 这个 Volatile 变量进行缓存（各自 CPU 的高速缓存），当线程修改时会有以下情况：

- 注释打开，此时一个 T 对象刚好占用一个缓存行，线程发生读写，两个线程互不影响。
- 注释关闭，此时两个 T 对象占用一个缓存行，线程发生读写，会导致两个线程不断重置并获取最新值。





#### 有序性实现原理

> 有序性的实现是基于 happens-before 和 内存屏障

##### happens-before

happens-before 中有一个 Volatile 规则即：所有对这个 Volatile 域前的写操作都 happens-before 后面对这个 Volatile 域的读操作



```java
public int a;
public volatile boolean flag;

public void read() {
  System.out.println(flag);  // step3
  int b = a;								 // step4
}

public void write() {
  a++;           // step1
  flag = true;   // step2 
}
```

假设先调用 write 方法在调用 read 方法，则会有以下规则：

- 程序次序规则：step1 happens-before step2 ，step3 happens-before step4
- Volatile 规则：setp1 happens-before step3， step2 happens-before step4
- 传递性规则：step1 happens-before step4



重排序有两种方式：`编译器重排序`和`处理器重排序`，为了实现volatile内存语义，JMM会对volatile变量限制这两种类型的重排序。下面是JMM针对volatile变量所规定的重排序规则表：

- 编译器重排序：编译器级别即 Java 代码通过 Javac 编译出的字节码顺序和源代码不一致。
- 处理器重排序：硬件和 CPU 级别的重排序，为了充分利用 CPU 的性能。

规则表：

![image-20210315173747043](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210315173747043.png)



##### 内存屏障

而处理器重排序则使用了内存屏障来完成有序性，规则如下：

- 在每个 Volatile 写操作的`前面`插入一个 StoreStore 屏障
- 在每个 Volatile 写操作的`后面`插入一个 StoreLoad 屏障
- 在每个 Volatile 读操作的`后面`插入一个 LoadLoad 屏障
- 在每个 Volatile 读操作的`后面`插入一个 LoadStore 屏障

**Volatile 写操作是在前面和后面分别插入内存屏障，而 Volatile 读操作是在后面插入了两个内存屏障。**

![image-20210315174008943](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210315174008943.png)

总结如下：

- 当进行完普通读写时再次进行 Volatile 写时不支持重排序
- 当进行完 Volatile 读后再次进行普通读写或 Volatile 读或 Volatile 写时不支持重排序
- 当进行完 Volatile 写后再次进行 Volatile 读和 Volatile 写时不支持重排序



可见性：LoadBarrier 和 StoreBarrier，LoadBarrier 作用是将其他线程对于共享变量的更新从其他处理器同步到当前线程的处理器，StoreBarrier 保证写线程对共享变量的更新对于其他读线程的处理器是可同步的。



**不要使用 String 类型的对象来加锁**



## 面试题

### DCL 单例需要加 Volatile 关键字吗？

答案：

需要加 Volatile 关键字，因为对象的创建分为三个步骤：

1. 申请内存（赋予默认值，int = 0）

2. 对象赋值（用户指定的值）

3. 对象指向目标（）

而这三个步骤可能会被 Jvm 重排序，可能 步骤 1 执行完直接就执行步骤 3