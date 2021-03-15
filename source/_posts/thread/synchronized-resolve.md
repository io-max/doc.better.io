---
title: Synchronized 解析
tags:
  - Java
  - Juc
  - Synchronized
categories: Synchronized
index_img: img/java-logo-480.png
abbrlink: 40532
---

> 本章讲解如下：
> - Synchronized 关键字的使用?
> - Synchronized 关键字在使用中锁的变化,及其升级过程?
> - Synchronized 的底层实现?
> - 锁的分类有些? 
> - 锁优化有哪些? 

<!-- more -->

## Synchronized

### 简介

在 Jdk1.5 之前 Synchronized 是一个重量级锁，但在 1.6 之后 Synchronized 经过优化后会经过一系列锁升级才会成为重量级锁。而 Synchronized 用的锁是存储在锁对象的`对象头`中。



### 特性

Synchronized 可以解决并发编程中的三个问题分别是：原子性（CPU 分片）， 有序性（编译器指令重拍），可见性（CPU 缓存）

- 原子性：确保线程互斥的访问同步代码
- 有序性：有效解决重排序问题
- 可见性：保证共享变量的修改能够及时可见



### 使用

Synchronized 可以使用任意`非 null `的对象来作为锁, 这些`锁`被称为`对象监视器(ObjectMonitor)`。

`Synchronized`可以修饰`静态方法（Class 实例）、实例方法（this 实例）、对象实例（括号范围）`，归根结底它能上锁的资源只有一类：就是`对象`

```java
//对象锁
public void synchronized demo() {}  // 此时锁的是 this 当前对象
public void demo() {
  synchronized(this) {}
}
public void demo() {
  synchronized(obj) {}  // 锁的范围仅仅在大括号范围
}

// 类锁
public statis void synchronized demo() {}  // 此时锁的是当前Class对象
public void  demo() {
  synchronized(Demo.class) {}
}
```



### 实现原理

Synchronized 是通过对象内部的一个叫做`监视器锁（moniter）`来实现的，而监视器锁本质依赖OS 底层的 `Mutex Lock`（互斥锁）来实现的。且 OS 实现线程的切换需要从用户态切换到内核态，成本太高，所以将这种依赖 Mutex Lock 实现的锁称为重量级锁。

任意对象都有一个Monitor与之关联，当且一个Monitor被持有后其关联的对象将处于锁定状态。Synchronized 就是基于`进入`和`退出` Monitor 对象来实现的。



### Monitor

#### 简介

在 Java 中任何对象都有可能成为 Monitor，当一个对象被 new 出来后都会携带一把看不见的锁：Monitor 锁



#### 结构

而 Monitor 的在 HotSpot 中是由 `ObjectMonitor` 实现的，其结构在`objectMonitor.hpp`中定义

![image-20210312145902426](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210312145902426.png)

其包含了三个非常的字段：WaitSet 和 EntryList 和 Owner。

- _WaitSet：用于存放处于 wait 状态的线程
- _EntryList：用于存放处于 blocked 状态的线程
- _owner：用于存储获取到锁的线程
- _count：记录个数



#### 执行流程

Monitor锁的获取流程紧紧围绕着 `_WaitSet、_EntryList、_owner` 三个字段展开；大致步骤如下：

- 当线程尝试获取锁时，会被包装成 ObjectWaiter，并进入 EntryList 中等待，判断 EntryList 是否为空
  - 为空：直接占领 Owner，执行同步代码块
  - 不为空：则和之前的线程一起等待
- 当线程持有 Monitor 时有两个选择：
  - 正常执行完代码块的代码，释放监视器
  - 执行一般等待某个条件的出现，调用 `wait() `进入 wait 状态，释放 _owner , count 减 1 ；并进入 WaitSet 中等待条件满足被唤醒。
- 当被唤醒后，继续获取监视器执行代码

#### 流程图

![](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200822151020.png)

一个线程只有在持有Monitor时才能调用`wait()`方法进入`_WaitSet 队列`，而处于_WaitSet 队列的线程只有再次获得监视器才能退出等待状态。



#### 线程如何出入Monitor？

**JVM 使用 `monitorenter` 和 `monitorexit` 两个字节码指令来进入和退出 Monitor 对象** 



#### 线程如何找到 Monitor？

Monitor对象存在于每个Java对象的`对象头`Mark Word中（存储的指针的指向）。



#### 总结

Synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时`notify/notifyAll/wait`等方法会使用到Monitor 锁对象，所以必须在同步代码块中使用。



### `monitorenter`和`monitorexit`

上面将了Synchronized 的底层是通过`进入`和`退出`Monitor 对象来实现的，那么线程是如何进入和退出 Monitor 对象呢？

#### 示例

先使用 `javap -c -s -v -l` 来对一个编译好的只有一个同步方法的类进行反编译，结果如下：

**代码**

```java
public void demo() {
  synchronized (this) {
  }
}
```

结果如下图：

![image-20210311160653484](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210311160653484.png)



从图中可以看出 `synchronized` 底层使用 `monitorenter` 和 `monitorexit` 两个指令完成。

**使用 monitorenter 来进入 monitor 对象，而 monitorexit 来退出 monitor 对象**，而第二个 monitorexit 是为了防止程序出现异常锁不释放。



**当使用Synchronized加锁出现异常会不会释放锁？会释放，这里可以证明这点**



#### `monitorenter`

该 `monitorenter` 用于开启对`monitor`的监控, 并获取 monitor 的所有权，只有到获取到来 monitor 的所有权此 monitor 才会锁定。

执行`monitorenter`的线程获取 monitor 的流程：

1. 如果monitor的计数条目为0，则线程进入monitor，并将计数加 1；然后该线程是monitor所有者。（获取到锁）
2. 如果线程已经拥有计数器，那么它会重新进入monitor，增加计数条目。（可重入锁）
3. 如果另外一个线程拥有此monitor，则线程会被阻塞，直到monitor的计数条目变为 0，则再次获取monitor所有权。（为获取到锁，但尝试获取锁）



#### `monitorexit`

而 `monitorexit` 指令用于释放monitor的所有权, 执行`monitorexit`的线程必须是monitor的所有者。该线程减少monitor的条目计数。结果，如果条目计数的值为零，则线程退出monitor，并且不再是其所有者。其他被阻止进入monitor的线程也可以尝试这样做。



**特点: 一个`monitorenter`指令可以和多个`monitorexit`指令结合使用。**



**可重入性**

Synchronized 先天具有重入性。每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一. 即通过计数器实现的 .





当Synchronized修饰实例方法时会在方法标识上添加了`ACC_SYNCHRONIZED`, 当线程池调用此方法时会去检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。

```java
public synchronized void demo() {
}
```

反编译结果：

![image-20210312112504055](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210312112504055.png)



**在方法执行期间，其他任何线程都无法再获得同一个monitor对象。**



### 对象内存布局

知道了Synchronized 通过 monitorenter 和 monitorexit 来进入和退出 Monitor 来实现同步的功能，那么问题来了

**monitorenter 和 monitorexit 是如何找到 Monitor对象的呢？**



答案是`对象头`，要知道对象头在哪里就必须了解 Java 对象内存布局。



#### 对象布局

在 Java 中对象在内存中的布局大致分为三部分： `对象头，实例数据， 对齐填充`。

- 实例数据：存放类的属性信息和父类的属性信息

- 对齐填充：JVM 要求对象`起始地址`必须是 8 字节的整数倍
- 对象头中存储了：对象自身的运行时数据（Mark Word）、类型指针（Class Point）
  - 如果对象是数组类型则对象头还会存储：数组长度
  - 对象头在 32 位和 64 位操作系统分别占用 32bit 和 64bit



#### 对象头

对象头分为两部分：`Mark Word` 和 `Class Pointer`。我们重点关注 Mark Word。

Class Pointer 即指向当前对象的类的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

#### Mark Word

而Mark Word 存储了`哈希码`、`分代年龄`、`锁标志位`、`偏向线程ID`、`偏向时间`戳等信息。



在 OpenJdk 的 Hotspot 源码的 markOop.hpp 文件中对 Mark Word 进行了描述

![image-20210312162706709](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210312162706709.png)

结果如下图：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200822150953.png"  />

在对象的最后两个 bit 为存储了锁的标识位， 默认是 01 即正常对象， 而随着锁等级的不同。存储的数据如下列表：

| 锁状态           | 锁标识位 | 存储内容                               |
| ---------------- | -------- | -------------------------------------- |
| 无锁             | 001      | 对象 HashCode(如果有调用)              |
| 偏向锁           | 101      | 持有线程的线程ID                       |
| 轻量级锁、自旋锁 | 00       | 指向持有线程栈帧中的 Lock Record 指针  |
| 重量级锁         | 10       | 指向互斥量（向需要向内核申请锁）的指针 |



而 monitorenter 和 monitorexit 操作的 monitor 就是对象在内存中的`对象头`(通过指针)



#### Lock Record

线程要想获取到锁，则必须和对象头中的 Mark Word 建立关联，而这个关联就是通过 Lock Record 来实现的。

Lock Record 顾名思义为`锁记录`, 它主要存在于线程的栈帧中，每个线程都有属于自己的 Lock Record 列表。

主要用于`存储对象头中的 Mark Word 的拷贝`，且每个Lock Record 同一时间只能与一个 Mark Word 关联；标识该对象锁被当前线程持有。

Lock Record 中有个 `owner` 字段用于存放`拥有该锁的线程的唯一标识`或`锁对象的 Mark Word`。



### 总结

上面的内容大致讲述了 Synchronized 的底层实现，以及 Lock Record 和 Mark Word 和 Monitor 的关系。

Lock Record 通过存储对象头中 Mark Word 的拷贝来建立关联，并开始抢占 Monitor 锁。

- 获取成功则修改 Mark Work 中的锁标志位，并修改其中的指针。

- 获取失败则进行阻塞（ObjectMonitor 的_EntreyList）。



## 锁优化

由于 Jdk 在 1.5 之前Synchronized是非常重的锁，在 1.6 之后对Synchronized进行了优化和调整，加入了`自适应的CAS自旋、锁消除、锁粗化、偏向锁、轻量级锁这些优化策略`。

锁主要存在四种状态，依次是**：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态**，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁。但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级。

在 JDK 1.6 中默认是开启偏向锁和轻量级锁的，可以通过`-XX:-UseBiasedLocking`来禁用偏向锁。



### 自旋锁

#### 为什么会有自旋锁？

线程的阻塞和唤醒需要CPU从用户态转为核心态，而频繁的阻塞唤醒非常消耗 CPU 资源。

#### 简介

所谓自旋锁就是当一个线程尝试获取锁时，如果该锁被其他线程占用，则该线程就`一直循环`检测锁是否被释放，而不是进行`睡眠`或`挂起`状态。

#### 适用场景

自旋锁适用于锁保护的临界区很小的情况，所谓临界区就是指访问`共享资源`的`程序片段`，当临界区很小时，锁占用的时间很短。为什么？

因为自旋锁是占用 CPU 资源的，如果临界区太大，执行时间长，则 CPU 资源会被占用很长时间，典型占着茅坑不拉屎。

在 Jdk1.6 自旋锁默认开启，自旋的默认次数为 `10`，可通过`-XX:PreBlockSpin`指令修改，但 Jvm加入了`自适应`，可自适应性的修改自旋次数。



#### **自适应自旋**

如果自旋成功，则自旋的次数会增加，反之如果失败，则会减少自旋的次数，避免过度浪费 CPU 资源。



### 锁消除 

在某些情况下当同步代码中的对象不存在被多个线程竞争时，JVM会对这些同步锁进行锁消除。

#### **示例代码**

```java
public class _02_Lock_Clean {

  Integer lock = 0;

  public static void main(String[] args) {
    _02_Lock_Clean lock_clean = new _02_Lock_Clean();

    for (int i = 0; i < 10; i++) {
      lock_clean.lockClean(i);
    }
  }

  public synchronized void lockClean(int source) {
    lock += source;
    System.out.println(lock);
  }
}

```

在 main 方法 lock_clean 对象没有逃逸出 main 方法，所以 JVM 认为不存在竞争关系。

方法生成的字节码

```c
 0 aload_0
 1 aload_0
 2 getfield #3 <io/better/jdk/_synchronized/_02_Lock_Clean.lock>
 5 invokevirtual #7 <java/lang/Integer.intValue>
 8 iload_1
 9 iadd
10 invokestatic #2 <java/lang/Integer.valueOf>
13 putfield #3 <io/better/jdk/_synchronized/_02_Lock_Clean.lock>
16 getstatic #8 <java/lang/System.out>
19 aload_0
20 getfield #3 <io/better/jdk/_synchronized/_02_Lock_Clean.lock>
23 invokevirtual #9 <java/io/PrintStream.println>
26 return
```



**锁消除的依据是逃逸分析的数据支持**



### 锁粗化

在使用 Synchronized 时我们一般会加在临界区很小的地方，缩短锁的占用时间，等待锁的线程尽可能快的获取锁。但是如果一系列的连续加锁解锁操作，可能会导致不必要的性能损耗，所以引入锁粗化的概念。

锁粗化：就是将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁。



### 偏向锁

#### **为什么会有偏向锁？**

因为在大多数程序中，锁不仅不存在多线程竞争，而且总是被同一个线程多次获得，为了让线程获取锁的代价更低，加入了偏向锁。



#### 特性

偏向锁是在单线程执行代码块时使用的机制，如果在多线程并发的环境下（即线程A尚未执行完同步代码块，线程B发起了申请锁的申请），则一定会转化为轻量级锁或者重量级锁。



#### **获取流程**

当锁对象被线程持有时，Mark Word 存储了持有锁对象的线程的线程ID，所以偏向锁的获取就是：线程通过 CAS 操作不断的比对并修改锁对象的 Mark Word 中存储的线程 ID。



大致流程如下：

- 步骤1：检查锁对象 Mark Word 偏向标识是否为 `1`
  - `=1`：不可偏向：说明其他线程正在拥有锁，执行步骤 2
  - `!=1`：可偏向：直接将当前线程 ID 存储到锁对象的 Mark Word中并将偏向标识设置为 1，执行步骤 5
- 步骤2：判断存储的线程 ID 是否与当前线程 ID 匹配
  - 匹配，则执行步骤 5
  - 不匹配，则执行步骤 3
- 步骤 3：通过 Cas 操作将 Mark Word 中的线程 ID 替换成自己的线程 ID
  - 成功：执行步骤 5
  - 失败：执行步骤 4
- 步骤 4：竞争失败，说明有多线程竞争，当到达全局安全点，将获取锁的线程挂起，偏向锁升级为轻量级锁，阻塞的线程在安全点继续竞争
- 步骤 5：执行同步代码块



#### **释放流程：偏向锁撤销**

偏向锁使用了一种等待竞争出现才会释放锁的机制。所以当其他线程尝试获取偏向锁时，持有偏向锁的线程才会释放锁。

但是偏向锁的撤销需要等到全局安全点(就是当前线程没有正在执行的字节码)。

流程如下：

- 步骤 1：暂停拥有锁对象的线程，并判断锁对象是否处理锁定状态
  - 否：恢复到无锁状态，偏向标识位设置为`0`；
  - 是：执行步骤 2
- 步骤 2：挂起持有锁的线程，并在线程栈帧中创建 Lock Record 记录，并将其指针拷贝到锁对象的 Mark Word 的中，将锁标识位改为 `00`，升级为`轻量级锁`。



#### **偏向锁关闭**

在 Jdk1.6 和 1.7后默认开启，想要关闭偏向锁可使用`-XX:-UseBiasedLocking=false`指令。



### 轻量级锁

#### **为什么会有轻量级锁？**

当偏向锁存在多线程竞争时会升级为轻量级锁，偏向锁是为了某些单线程执行同步代码块的场景下使用，而轻量级锁会通过自旋的方式获取锁，不会阻塞。



#### 使用场景

**轻量级锁所适应的场景是线程交替执行同步块的情况，如果存在同一时间访问同一锁的情况，必然就会导致轻量级锁膨胀为重量级锁。**



#### **获取流程**

大致流程如下：

- 步骤 1：判断锁对象的状态
  - 无锁状态执行步骤 2
  - 偏向锁状态：则挂起持有锁对象的线程，执行步骤 2（对应偏向锁升级轻量级锁的过程）
- 步骤 2：在当前线程的栈帧中创建 Lock Record（锁记录）空间，用于存储锁对象的 Mark Word
- 步骤 3：拷贝锁对象的 Mark Word 到当前线程的 Lock Record 中
- 步骤 4：通过 CAS 操作替换锁对象的 Mark Word 中存储的 Lock Record ，并把 MarkWord 的指针设置到 LockRecord 的 `owner` 字段中。
  - 成功：更新锁标识位为 `00`，标识当前锁状态为轻量级锁
  - 失败：执行步骤 5
- 步骤 5：当 CAS 更新失败后，Jvm 查看锁对象的 MarkWord 中的 Lock Record 是否指向当前线程的栈帧，如果是，说明当前线程已经拥有了这个对象的锁，可直接进入同步块执行。否则说明多个线程竞争，进入自旋，若自旋结束依旧未获取到锁，轻量级锁就要膨胀为重量级锁，锁标志位变为 10，锁对象的 MarkWord 中存储指向重量级锁的指针，当前线程以及后面等待的线程进入阻塞状态。



步骤 2：

![image-20200821193905959](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200822151013.png)

步骤 4：

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200822151017.png" alt="image-20200821193950404"  />



#### **释放流程**

当持有锁对象的线程执行完同步代码块时将通过 CAS 操作将锁对象的 Mark Work 中存储的 Lock Record 指针置为 NULL。如果成功，则表示没有发生竞争关系。如果失败，表示当前锁存在竞争关系。锁就会膨胀成重量级锁。 



#### 注意

对于轻量级锁，其性能提升的依据是 “对于绝大部分的锁，在整个生命周期内都是不会存在竞争的”，如果打破这个依据则除了互斥的开销外，还有额外的CAS操作，因此在有多线程竞争的情况下，轻量级锁比重量级锁更慢。



#### 轻量级锁膨胀为重量级锁流程



![](https://upload-images.jianshu.io/upload_images/2062729-b952465daf77e896.png)



**为什么轻量级锁会在 MarkWord 中保存 LockReacord 记录？**

一方面可以用于 CAS 比较，其次在升级为重量级锁时，持有锁的线程和等待的线程都会被阻塞，便于唤醒线程。



## 总结

关于不同锁的使用及其有点场景如下：

| 锁       | 优点                     | 缺点                             | 场景                       |
| -------- | ------------------------ | -------------------------------- | -------------------------- |
| 偏向锁   | 加锁和解锁不用消耗时间   | 锁存在竞争会带来额外的锁撤销消耗 | 单线程访问同步代码块       |
| 轻量级锁 | 竞争的线程不会长时间阻塞 | 自旋浪费 CPU                     | 追求响应时间，代码块执行快 |
| 重量级锁 | 不会自旋消耗 CPU         | 线程阻塞响应时间慢               | 代码块执行时间长           |



**推荐文章**

[深入分析Synchronized原理(阿里面试题)](https://www.cnblogs.com/aspirant/p/11470858.html)

