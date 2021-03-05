---
title: Juc-AQS 源码分析
tags:
  - Java
  - Juc
  - Aqs
  - 源码分析
categories: Juc
index_img: img/java-logo-480.png
abbrlink: 33783
---

该文章主要讲述 Java 中 Juc 包下的 AbstractQueuedSynchronizer 
主要讲述 AQS 的设计及其独占模式和共享模式的获取和释放流程

<!-- more -->

## 本章简介

>本章主要讲述 JUC 包下的 AQS 的设计与现实, 同时了解 AQS 中独占和共享模式的运转原理和机制
>
>1. AQS 的设计和实现
>2. AQS 中队列介绍及其变体
>3. AQS 中独占和共享模式的源代码分析





## AbstractOwnableSynchronizer

此类为AbstractQueuedSynchronizer的父类,  一个同步器框架有可能在一个时刻被某一个线程独占，`AbstractOwnableSynchronizer`就是为所有的同步器实现和锁相关实现提供了基础的保存、获取和设置独占线程的功能.

### 代码

```java
/**
 * 一个线程可以独占的同步器,  此类提供了创建锁和相关的同步器（可能涉及所有权概念）的基础
 *
 * AbstractOwnableSynchronizer 类本身并不管理或使用此信息。 
 * 但是，子类和工具可以使用适当维护的值来帮助控,制和监视访问并提供诊断。
 */
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {

  private static final long serialVersionUID = 3737899427754241961L;

  protected AbstractOwnableSynchronizer() { }

  /**
    * 独占模式同步的当前所有者
    */
  private transient Thread exclusiveOwnerThread;

  /**
    * 设置当前拥有独占访问权的线程
    * null参数表示没有线程拥有访问权限, 否则, 此方法不会强加任何同步或{@code volatile}字段访问
    */
  protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
  }

  protected final Thread getExclusiveOwnerThread() {
    return exclusiveOwnerThread;
  }
}
```



## `AbstractQueuedSynchronizer`

Juc 包下的多数同步器都是基于`AbstractQueuedSynchronizer（简称 AQS）`框架实现的，AQS为`同步状态`的`原子性管理`、`线程的阻塞`和`解除阻塞`以及`排队`提供了一种通用的机制。



### 功能

同步器一般包含两种方法，一种是`acquire`，另一种是`release`。acquire操作阻塞调用的线程，直到或除非同步状态允许其继续执行。而release操作则是通过某种方式改变同步状态，使得一或多个被acquire阻塞的线程继续执行。

但是 Juc 包中不同的同步器其相应的 API 页不相，`Lock.lock，Semaphore.acquire，CountDownLatch.await`（但本质都是`acquire`或`release`操作），但都支持下面的操作：

- 阻塞和非阻塞同步
- 可选的超时设置，让调用者可以放弃等待
- 通过中断实现任务取消，通常分为两个版本，一个`acquire`可取消，而另一个不可以。

同步器的实现根据其状态分为两种：`独占状态`和`共享状态`。

- 独占状态：同步器在同一时间允许一个线程执行（`Lock`）
- 共享状态：同步器在同一时间允许多个线程执行（`Semaphore`）



### 设计与实现

`acquire`和`release`简单的伪代码实现：

`acquire`操作：

```java
while (synchronization state does not allow acquire) {
    enqueue current thread if not already queued;
    possibly block current thread;
}
dequeue current thread if it was queued;
```

`release`操作：

```java
update synchronization state;
if (state may permit a blocked thread to acquire)
    unblock one or more queued threads;
```

实现上面的操作，需要下面三个操作：

- 同步状态的原子性管理；
- 线程的阻塞与解除阻塞；
- 队列的管理；



#### 同步状态

AQS类使用单个`int`（32位）来保存同步状态，并暴露出`getState`、`setState`以及`compareAndSet`操作来读取和更新这个状态。

基于AQS的具体实现类必须根据暴露出的状态相关的方法定义`tryAcquire`和`tryRelease`方法，以控制`acquire`和`release`操作。当同步状态满足时，`tryAcquire`方法必须返回`true`，而当新的同步状态允许后续`acquire`时，`tryRelease`方法也必须返回`true`。这些方法都接受一个`int`类型的参数用于传递想要的状态。



#### 阻塞

Juc包中提供了一个`LockSupport`类，其`LockSupport.park`和`LockSupport.unpark`用于替换传统的 `Thread.suspend`和 `Thread.resume`（同一产生死锁），`LockSupport.unpark`方法被提前调用也是可以的。

`LockSupport.unpark`的调用是没有被计数的，因此在一个`park`调用前多次调用`unpark`方法只会解除一个`park`操作。

另外，它们作用于每个线程而不是每个同步器。一个线程在一个新的同步器上调用park操作可能会立即返回，因为在此之前可能有“剩余的”`unpark`操作。

但是，在缺少一个`unpark`操作时，下一次调用`park`就会阻塞。



#### 队列

整个框架的关键就是如何管理被阻塞的线程的队列，该队列是严格的FIFO队列，因此，框架不支持基于优先级的同步。

同步队列的最佳选择是自身没有使用底层锁来构造的非阻塞数据结构，已知 MCS 和 CLH 两种锁队列，但在 AQS 中使用了 CLH 锁队列，因为CLH 更容易实现`超时`和`取消`功能。AQS 基于 CLH 进行了修改和 CLH有较大的出入。



![](http://ifeve.com/wp-content/uploads/2013/01/CLHNode.png)



第一个对CLH队列主要的修改是添加了 next 字段，来用于唤醒后继节点

第二个对CLH队列主要的修改是将每个节点都有的状态字段用于控制阻塞而非自旋。



而AQS中同步队列的基础实现是其内部类`Node`



##### Node

```java
static final class Node {

  // 标记共享模式
  static final Node SHARED = new Node();

  // 标记排他模式
  static final Node EXCLUSIVE = null;

  // 取消状态
  static final int CANCELLED = 1;

  //表示后续节点需要释放
  static final int SIGNAL = -1;

  // 节点出去等待状态
  static final int CONDITION = -2;

  // 下一个acquireShared应该无条件传播
  static final int PROPAGATE = -3;

  // 状态字段
  volatile int waitStatus;
  
  // 前继节点, 在入队期间分配，并且仅在出队时将其清空, 前置节点取消时, 会短路
  volatile Node prev;

  // 前继节点, 在排队过程中分配，在绕过取消的前任对象时进行调整，并在出队时清零
  // 被取消节点的next字段设置为指向节点本身而不是null
  volatile Node next;

  // 使该节点排队的线程。在构造上初始化，使用后消失
  volatile Thread thread;

  // 链接到等待条件的下一个节点，或者链接到特殊值SHARED
  // 由于条件队列仅在以独占模式保存时才被访问，因此我们只需要一个简单的链接队列即可在节点等待条件时保存节点。
  // 然后将它们转移到队列中以重新获取
  // 由于条件只能是互斥的，因此我们使用特殊值来表示共享模式来保存字段
  // 下一个等待节点
  Node nextWaiter;

  Node() {    // Used to establish initial head or SHARED marker
  }

  Node(Thread thread, Node mode) {     // Used by addWaiter
    this.nextWaiter = mode;
    this.thread = thread;
  }

  Node(Thread thread, int waitStatus) { // Used by Condition
    this.waitStatus = waitStatus;
    this.thread = thread;
  }

  // 返回ture表示当前节点处于共享模式等待
  final boolean isShared() {
    return nextWaiter == SHARED;
  }

  // 返回前一个节点, 如果为null抛出空指针异常
  final Node predecessor() throws NullPointerException {
    Node p = prev;
    if (p == null)
      throw new NullPointerException();
    else
      return p;
  }
}
```

`waitStatus`的取值:

- `CANCELLED`: 由于超时或中断导致该节点被取消,被取消节点的线程永远不会再次阻塞
- `CONDITION`: 该节点当前在`条件队列`中, 在传输之前, 它不会用作`同步队列`节点, 此时状态将设置为`0`.
- `SIGNAL`: 当前节点的后继节点被阻塞, 如果当前节点释放或取消时必须唤醒其后继节点
- `PROPAGATE`:  此状态值通常只设置到调用了`doReleaseShared()`方法的头节点，确保`releaseShared()`方法的调用可以传播到其他的所有节点，简单理解就是共享模式下节点释放的传递标记。

非负值表示节点不需要发信号, 对于常规同步节点，该字段初始化为0, 对于条件节点，该字段初始化为CONDITION
使用CAS对其进行修改



`nextWaiter`该字段字面意思是:下一个等待节点，其实有三个取值:

- `Node.EXCLUSIVE`: 独占模式
- `Node.SHARED`: 共享模式
- 其他值: **代表Condition等待队列中当前节点的下一个等待节点**

## 队列变体

知道了AQS 的队列是使用的CLH队列的变体, 所以有必要看看 CLH 和 MCS 两个队列.

### CLH 锁队列

```java
public class _CLH_Lock {

  final AtomicReference<Node> tail = new AtomicReference<>(new Node());
  final ThreadLocal<Node> current;

  public _CLH_Lock() {
    this.current = new ThreadLocal<Node>() {
      @Override
      protected Node initialValue() {
        Node node = new Node();
        System.out.println("构造器: " + Thread.currentThread().getName() + "-" + node);
        return node;
      }
    };
  }

  public static void main(String[] args) {

    _CLH_Lock lock = new _CLH_Lock();

    Runnable runnable = () -> {
      try {
        lock.lock();
        System.out.println(Thread.currentThread().getName() + "获取到了锁");
        TimeUnit.SECONDS.sleep(5);
        System.out.println(Thread.currentThread().getName() + "释放到了锁");
        lock.unlock();
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    };

    new Thread(runnable, "线程 A").start();
    new Thread(runnable, "线程 B").start();
    new Thread(runnable, "线程 C").start();
  }

  public void lock() throws InterruptedException {

    Node own = this.current.get();
    System.out.println(Thread.currentThread().getName() + "-" + own);
    own.locked = true;
    Node preNode = tail.getAndSet(own);

    while (preNode.locked) {
      System.out.println(Thread.currentThread().getName() + "开始自旋....");
      TimeUnit.SECONDS.sleep(2);
    }
  }

  public void unlock() {
    current.get().locked = false;
  }

  class Node {
    volatile boolean locked = false;
  }
}

```

CLH 队列类似于一个伪链表，每个线程在通过 CAS 操作替换 `tail` 节点引用后，拿到上一个线程节点引用，不断循环检测该线程节点中的 blocked 字段（这个操作就是自旋）。

![](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20200831111943196.png)







### MCS锁队列

```java
public class _MCS_Lock {

  final AtomicReference<Node> tail = new AtomicReference<>(null);
  ThreadLocal<Node> current;

  public _MCS_Lock() {
    this.current = new ThreadLocal<Node>() {
      @Override
      protected Node initialValue() {
        return new Node();
      }
    };
  }

  public void lock() throws InterruptedException {

    Node own = current.get();

    Node preNode = tail.getAndSet(own);

    if (Objects.nonNull(preNode)) {
      preNode.next = own;
      own.locked = true;
      while (own.locked) {
        System.out.println(Thread.currentThread().getName() + "开始自旋....");
        TimeUnit.SECONDS.sleep(2);
      }
    }
  }

  public void unlock() {

    Node own = current.get();
    if (Objects.isNull(own.next)) {
      // CAS操作失败，说明当前节点后面新加入了节点
      if (tail.compareAndSet(own, null)) {
        return;
      }
      // 条件不成立，执行下面的解锁流程代码
      while (own.next == null) {
      }
    }
    own.next.locked = false;
    own.next = null;
  }

  class Node {
    volatile Node next;
    volatile boolean locked;
  }

  public static void main(String[] args) throws InterruptedException {
    _MCS_Lock lock = new _MCS_Lock();

    Runnable runnable = () -> {
      try {
        lock.lock();
        System.out.println(Thread.currentThread().getName() + "获取到了锁");
        TimeUnit.SECONDS.sleep(5);
        System.out.println(Thread.currentThread().getName() + "释放到了锁");
        lock.unlock();
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    };

    new Thread(runnable, "线程 A").start();
    new Thread(runnable, "线程 B").start();
    TimeUnit.SECONDS.sleep(3);
    new Thread(runnable, "线程 C").start();
  }
}
```

MCS 是一个真正的链表通过 `next` 字段来关联下一个线程节点，但是相对于 CLH 它的 CAS 操作多了



![](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20200831144343389.png)





### 总结

CLH 适用于SMP 系统架构，不适用于NUMA架构(内存分隔)，如果前一个节点的内存过远会导致性能下降。



CLH 对比 MCS: 

（1）从代码实现来看，CLH比MCS要简单得多。

（2）从自旋的条件来看，CLH依靠前驱节点自旋，而MCS是依靠自身自旋。

（3）从链表队列来看，CLH的队列是隐式的，MCS的队列是物理存在的，通过 next 字段。

（4）CLH 锁释放时只需要改变自己的属性，MCS锁释放则需要改变后继节点的属性。

（5）CLH 适合CPU个数不多的计算机硬件架构上，MCS则适合拥有很多CPU的硬件架构上

（6）CLH和MCS实现的自旋锁都是不可重入的



## 独占模式

### 获取

```java
public final void acquire(int arg) {
  // 如果tryAcquire获取失败, 且添加队列
  if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

`acquire`方法在独占模式下修改`同步状态`, 会至少调用一次`tryAcquire`方法, 如果`tryAcquire`返回`true`代表状态修改成功, 反之则尝试调用`acquireQueued`方法入队. 

`addWaiter`是入队操作, 而`acquireQueued`则是从队列中获取操作.

```java
// 1. 获取到 tail 节点, 如果不为空,则调用 cas 将 node 入队
// 2. 如果为空,则调用 enq 方法初始化队列,且将 node 节点入队
private Node addWaiter(Node mode) {
  Node node = new Node(Thread.currentThread(), mode);
  Node pred = tail;
  if (pred != null) {
    node.prev = pred;
    // 入队操作
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  // 队列为空,则进行初始化
  enq(node);
  return node;
}
```

#### 队列初始化

在调用`addWaiter`进行入队操作时, 可能会出现队列为初始化的情况, 即`pred == null`.此时调用了 `enq` 来对队列进行初始化

```java
private Node enq(final Node node) {
  for (;;) {
    Node t = tail;
    if (t == null) { // 如果为空,则进行处处华
      if (compareAndSetHead(new Node()))
        tail = head;
    } 
    // 不为空
    else {
      node.prev = t; // 先设置尾节点的前置节点,保证队列的完整性
      if (compareAndSetTail(t, node)) { // 在 CAS 设置尾节点
        t.next = node;
        return t;
      }
    }
  }
}
```

#### 自旋获取

```java
// 入队成功后,再次从队列中获取锁
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      // step1: 获取到当前节点的前置节点
      final Node p = node.predecessor();
      // step2: 如果前置节点是头节点,则说明前置节点已经获取到锁, 且再次调用tryAcquire方法获取锁
      if (p == head && tryAcquire(arg)) {
        // 成功,说明前置节点已经释放了搜
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
      }
      // step3: 此时前置节点可能还未释放锁, 则判断是否应该阻塞当前节点
      if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

在自旋时, 前置节点的状态: 

1. waitStatus=0: 说明当前节点可以继续自旋, 说不定下次自旋就能获取到锁
2. waitStatus>0: 说明前置节点已经取消, 更新当前节点的前置节点, 并进行下次自旋
3. waitStatus<0: 说明当前节点可以阻塞, 这个状态可能是当前节点在执行`shouldParkAfterFailedAcquire`修改的.

#### 获取失败后更新状态

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL) // 前置状态为 SIGNAL,说明当前节点需要被唤醒, 可以安全的的被阻塞
    return true; // 返回 true,则会执行 parkAndCheckInterrupt 方法
	
  // 大于 0,说明前置节点已经取消, 则从前置节点向前查找,直到遇到未取消的节点,
  // 将当前节点的 prev 指向为取消的节点
  if (ws > 0) {
    do {
      // 寻找未取消的前置节点
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    // 更新当前节点的 prev
    pred.next = node;
  } 
  // 此时状态只能是 0 或 PROPAGATE, 表明当前节点需要一个唤醒信号
  // 而 PROPAGATE 是在共享模式下使用的, CONDITION 则是在条件队列中时的状态
  else {
    // 将前置节点状态更新成 SINGAL
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```



**当前节点的唤醒是保存在前置节点中的**

### 释放

#### 源代码

```java
public final boolean release(int arg) {
  if (tryRelease(arg)) {
    Node h = head;
    if (h != null && h.waitStatus != 0) // 头节点不为空,且状态不等于0
      unparkSuccessor(h);
    return true;
  }
  return false;
}

// 唤醒后续节点
// 回顾获取的代码, 在后续节点获取失败后可能会修改前置节点的状态(shouldParkAfterFailedAcquire方法)
private void unparkSuccessor(Node node) {

  // 如果状态为正数(可能需要唤醒)
  int ws = node.waitStatus;
  // 此时waitStatus是SINGAL,
  if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0); // 更新成 0, 利于后续节点获取锁

  // 后继节点
  Node s = node.next;
  // 疑问: 后继节点为 null 或已取消为啥还要从尾节点开始遍历
  if (s == null || s.waitStatus > 0) {
    s = null;
    // 当上面判断成立时的一瞬间, 可能有新节点入队了, 这么做就是为了避免新节点的加入,而被忽略
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  if (s != null)
    // 解锁后继线程
    LockSupport.unpark(s.thread);
}
```



#### 后继节点状态

在执行释放时, 后继节点可能处于以下几种状态:

1. 已经阻塞.
2. 还在`acquireQueued`方法中执行自旋,还未阻塞.
3. 已经取消.



#### 优化点

在`unparkSuccessor`中将当前节点的状态更新成了 0

```java
if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);
```

而后继节点在获取失败后可能在执行 spaf 将前置节点置为 SIGNAL

```java
compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
```

**为什么这是个优化操作呢?**

假设队列 : `A -> B -> C`

假设 B 已经自旋了一次(获取失败), 调用`spaf`方法将 A 的状态设置成了 SIGNAL, 进行下一次自旋, 此时 A 会有两种操作:

1. 调用release方法, 此时 A 的状态会变成 0, B 在下一次自旋后发现 A 的状态变成了 0, 则会继续自旋
2. 未调用release方法, 此时 A 的状态还是 SIGNAL(被 B 修改的), 此时 B 在下一次自旋后就可以被阻塞了

这样可以加速锁的获取, 避免了一次没必要的 park. 



### 取消获取

```java
private void cancelAcquire(Node node) {
  // 为空忽略
  if (node == null)
    return;
	
  node.thread = null; // 线程置为空

	// 跳过取消的节点
  Node pred = node.prev;
  while (pred.waitStatus > 0)
    node.prev = pred = pred.prev;

	// 获取点未取消前置节点的前置节点
  Node predNext = pred.next;

	// 修改 ws
  node.waitStatus = Node.CANCELLED;

	// 如果是尾节点,则直接 CAS 更新
  if (node == tail && compareAndSetTail(node, pred)) {
    compareAndSetNext(pred, predNext, null);
  } 
  // 非尾节点
  else {
    // If successor needs signal, try to set pred's next-link
    // so it will get one. Otherwise wake it up to propagate.
    int ws;
    if (pred != head &&  // 前置节点不是头节点
        ((ws = pred.waitStatus) == Node.SIGNAL || // pred.ws = SIGNAL
         (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) && // 
        pred.thread != null) {
      
      Node next = node.next; // 获取当前节点的后继节点

      if (next != null && next.waitStatus <= 0) // 后继节点有效
        compareAndSetNext(pred, predNext, next); // 将 pred 前置节点的 next 修改成 node.next
    } 
    // 1. pred 为头节点s
    else {
      unparkSuccessor(node);  // 解锁后继者
    }

    node.next = node; // help GC
  }
}
```



### 总结



## 共享模式

共享模式下的`获取`和`释放`和独占模式有一些区别, 共享模式下,锁是可以被多个线程锁持有的.

下面的会涉及到源代码的分析, 再看每行代码时, 都要记住一点:`所有的方法都有可能并发执行`



### 获取

```java
public final void acquireShared(int arg) {
  if (tryAcquireShared(arg) < 0)
    doAcquireShared(arg);
}
```

`tryAcquireShared`返回值有以下几种情况:

- `<0`: 表示获取失败
- `=0`:表示获取成功,但是后继节点不能获取成功
- `>0`:表示获取成功



#### `doAcquireShared`

##### 源代码

```java
// 执行共享获取操作
private void doAcquireShared(int arg) {
  // step1: 入队
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      // step2: 前继节点
      final Node p = node.predecessor();
      if (p == head) {
        int r = tryAcquireShared(arg);
        if (r >= 0) {
					// step2.1: 设置头节点并传播
          setHeadAndPropagate(node, r);
        }// 忽略部分代码
      }
			// step3: 修改当前节点状态,并根据状态阻塞
      if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

##### 流程

1. 调用`addWaiter`进行入队操作, 只不过是共享模式
2. 判断前置节点是否是头节点
   1. 是头节点:  再次尝试获取, 如果成功, 调用`setHeadAndPropagate`方法传播, 反之执行步骤 3
   2. 非头节点:  执行步骤 3
3. 获取失败后, 根据前置节点状态 判断是否应该阻塞当前节点



##### `setHeadAndPropagate`

我们知道只要当当前节点的前继节点为头节点且再次 `tryAcquireShared` 成功后再回执行 `shp` 方法



注意: shp 是`setHeadAndPropagate`方法的简称

```java
// propagate 为 tryAcquireShared 的返回值
private void setHeadAndPropagate(Node node, int propagate) {
  Node h = head; 
  setHead(node);
	
  if (propagate > 0 || h == null || h.waitStatus < 0 ||
      (h = head) == null || h.waitStatus < 0) {
    Node s = node.next;  // 获取后继节点
    if (s == null || s.isShared())  // 如果处于共享模式
      doReleaseShared();		// 执行释放操作
  }
}
```

调用 `setHead` 方法后, 此前的 `head` 已经不在队列中了

如果 node.next 不为空且处于共享模式, 调用 `doReleaseShared()` 方法, 此方法在 `releaseShared` 会一起讲解



### 释放

#### 源代码

```java
public final boolean releaseShared(int arg) {
  if (tryReleaseShared(arg)) {
    doReleaseShared();
    return true;
  }
  return false;
}
```

可以看到在调用了`tryReleaseShared`方法后,如果成功则会调用`doReleaseShared`, 这和上面`共享获取`时`setHeadAndPropagate`方法中调用的方法是一致的.



#### `doReleaseShared`

此时我们知道在 `acquireShared` 和 `releaseShared` 中都会调用此方法, 该方法至少会被一个节点调用两次.

在分析这个方法前, 假设现在有三个节点为别是: `A, B, C` 且假设入队顺序是 `A -> B -> C`

```java
private void doReleaseShared() {
  // 如果需要信号, 头节点通常将会以这种方式唤醒后继节点, 如果没有则将状态更新为PROPAGATE,以确保传播
  // 此外，在执行此操作时，必须循环以防添加新节点。
  for (;;) {
    Node h = head;
    // 该判断只能保证在此时此刻队列中至少有两个节点, 可能在执行下面的代码中,队列从可能就只有一个节点了
    if (h != null && h != tail) {
      int ws = h.waitStatus;
      // 疑问:A.ws为什么是 SIGNAL
      // A 成为头节点, B 获取锁失败在执行 spaf 方法时修改
      if (ws == Node.SIGNAL) {
        // 疑问: 为什么会更新失败? 
        // B 在执行完 spaf 方法可能没阻塞, 再次自旋获取到锁了, 并成为了头节点, 所以造成此时A执行CAS失败
        // 更新成 0 是为了加速后续节点在执行 spaf 后不被阻塞
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) {
          continue;
        }
        // 解锁后继者
        unparkSuccessor(h);
      }
      // 后继节点不需要信号,则更新状态为PROPAGATE, 失败则自旋
      // 疑问: 什么情况会进入此 else if
      // A刚成为头节点(默认为 0), B执行 spaf 还未将 A.ws修改成 SIGNAL 此时 A 会进入 else if
      // 疑问: compareAndSetWaitStatus(h, 0, Node.PROPAGATE)为什么失败?
      // 当 ws==0 成立时,此时 B 在 spaf 将 A.ws 修改了 SIGNAL, 所以会失败
      // 注意: 这个 else if 主要就是处理并预防有新节点加入
      else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        continue;
    }
		// 执行此判断的条件, 成功调用unparkSuccessor方法或队列中没有两个节点
    if (h == head)
      break;
  }
}
```



#### `unparkSuccessor`

解锁后继线程, 该方法只有在 CAS 将 head.ws修改成 0 成功时才会执行

```java
private void unparkSuccessor(Node node) {
  /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
  int ws = node.waitStatus;
  if (ws < 0)
    // 疑问: 为什么要将 ws 更新成 0
    // 后继线程在执行完 shp 没有被阻塞, 继续执行下次自旋, 
    // 下次在执行 shp 时后继节点发现前置节点的 ws改变了, 继续自旋, 避免了阻塞
    compareAndSetWaitStatus(node, ws, 0);  

  // 跳过取消节点
  Node s = node.next;
  if (s == null || s.waitStatus > 0) {
    s = null;
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  // 不为空
  if (s != null)
    // 解锁线程, 则后继节点继续自旋, 获取锁
    LockSupport.unpark(s.thread);
}
```

## 总结

至此 AQS 中独占和共享模式下的 acquire 和 release 操作的细节都已分析完毕, 了解 AQS 对学习JUC 包下的类非常有帮助, 如果看完本章还有疑惑, 可以查看一下其它博客. 

[CLH 锁](https://cloud.tencent.com/developer/article/1187386)

[同步框架说明文档](https://www.cnblogs.com/dennyzhangdd/p/7218510.html)

[AQS讲解](https://www.cnblogs.com/micrari/p/6937995.html)

[AQS讲解](https://segmentfault.com/a/1190000016447307)