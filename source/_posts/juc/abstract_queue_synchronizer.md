---
title: AQS源码分析v2.0
categories: Juc
index_img: img/java-logo-480.png
tags:
  - Java
  - Juc
abbrlink: 48120
---

该文章主要讲解在向AQS的源码解析

<!-- more -->

## AbstractQueuedSynchronizer

### 本章简介

>本章主要讲述 JUC 包下的 AQS 的设计与现实, 同时了解 AQS 中独占和共享模式的运转原理和机制
>
>1. AQS 的设计和实现
>2. AQS 中独占和共享模式的源代码分析



### AbstractOwnableSynchronizer

该类为`AbstractQueuedSynchronizer`的父类,  其中包含了一个核心的字段:

```java
private transient Thread exclusiveOwnerThread;
```

此字段用于标识`独占模式下`当前持有锁的线程.





### AbstractQueuedSynchronizer.Node

在AQS中队列的节点由内部类`Node`来表示. 队列则是由一个个Node节点来形成的. 而从Node自身的变量可以知道当前节点的上一下节点(pre)和一下个节点(next). 同时每个节目都含有状态(唤醒时使用). 

由于需要知道具体抢锁的线程, 所以Node包含了`Thread`的引用. 



#### 变量

```java
static final class Node {
  // 当前节点是否是共享节点
  static final Node SHARED = new Node();
	// 当前节点是否是独立节点
  static final Node EXCLUSIVE = null;

  // 取消状态
  static final int CANCELLED =  1;
  // 唤醒状态
  static final int SIGNAL    = -1;
  // 条件队列状态
  static final int CONDITION = -2;
  // 传播状态, 共享模式下使用
  static final int PROPAGATE = -3;

  // 存储当前节目的状态
  volatile int waitStatus;

  // 当前节点的上一个节点
  volatile Node prev;

  // 当前节点的下一个节点
  volatile Node next;

  // 节点对应的线程
  volatile Thread thread;

  // 下一个等待节点, 此字段在条件等待队列中使用
  Node nextWaiter;

  // 是否是共享节点
  final boolean isShared() {
    return nextWaiter == SHARED;
  }

  // 返回上一个节点
  final Node predecessor() throws NullPointerException {
    Node p = prev;
    if (p == null)
      throw new NullPointerException();
    else
      return p;
  }
}
```

通过源码知道了每个节点的状态有四个取值分别是: `SIGNAL, CANCELLED, CONDITION, PROPAGATE`.

其中需要注意`CANCELLED`的取值为正数(大于0),  其他的状态值都为负数. 后续此条件会经常用到. 需要特殊注意.

`注意: waitStatus 默认初始化值为0.`

#### 构造函数

```java
Node(Thread thread, Node mode) {     
  // 赋值
  this.nextWaiter = mode;
  this.thread = thread;
}

Node(Thread thread, int waitStatus) { 
  // 赋值
  this.waitStatus = waitStatus;
  this.thread = thread;
}
```



### 核心变量

```java
// 队列头结点
private transient volatile Node head;

// 队列尾节点
private transient volatile Node tail;
```

AQS可以通过`head, tail`变量来操作队列中的元素. 此时队列已经完善了, 那么使用什么来表示锁呢?

AQS使用了一个`int`类型的字段来表示锁

```java
private volatile int state;
```

AQS中所有的操作都是围绕这个字段来展开的, 线程通过CAS来不断的修改这个值, 来模拟抢锁(修改成功则代表抢锁成功, 反之则抢锁失败).





### 核心方法

AQS的核心方法包括: 

- `acquire(int arg):` 独占模式下获取锁.
- `acquireInterruptibly(int arg): ` 独占模式下可中断获取锁, 当线程中断时直接抛出中断异常.
- `release(int arg):` 独占模式下释放锁
- `acquireShared(int arg):` 共享模式下获取锁
- `acquireSharedInterruptibly(int arg):` 共享模式下可中断获取锁
- `releaseShared(int arg):` 共享模式下释放锁



#### acquire方法原理

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

`tryAcquire`方法有具体的子类实现, 返回值代表抢锁是否成功: `true`表示抢锁成功, `false`表示抢锁失败.

当抢锁失败时, 会调用`addWaiter`方法来创建Node节点并入队. 注意`Node.EXCLUSIVE`是个空对象.  当入队完成时可能有其他线程释放锁了, 所以调用`acquireQueued`开启循环不断去获取锁.



##### acquireQueued方法原理

该方法就是开启一个循环, 不断的去获取锁同时判断前置节目的状态. 

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    // 开启自旋
    for (;;) {
      // 当前节点的上一个节点
      final Node p = node.predecessor();
      // 如果是头结点, 则再次尝试获取锁
      if (p == head && tryAcquire(arg)) {
        // 设置头结点
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
      }
      // 当获取锁失败时, 则根据前置节点的状态来判断是否需要阻塞当前线程
      if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      // 获取失败则取消当前节点
      cancelAcquire(node);
  }
}
```

注意此时循环的结束条件: `只有当前节点获取锁成功才会结束循环. 即p == head && tryAcquire(arg)条件成立`.

`shouldParkAfterFailedAcquire`方法就是根据前置节点的状态来判断当前线程是否应该被阻塞. 

而`parkAndCheckInterrupt`方法就是阻塞当前线程并检查线程中断标志位. 底层调用`LockSupport.park`方法.



#### addWaiter方法原理

```java
private Node addWaiter(Node mode) {
  // 创建Node节点, 并传递当前线程池
  // 当独占模式时, mode==null, 共享模式时 mode==Node.SHARED
  Node node = new Node(Thread.currentThread(), mode);
  Node pred = tail;
  // 尾节点不为空, 则将当前节点置为尾节点
  if (pred != null) {
    node.prev = pred;
    // cas 将当前节点设置成尾节点, 如果失败则会执行enq方法
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  // 到这里两种情况: 1. tail==null 2. compareAndSetTail(pred, node) 失败
  enq(node);
  // 返回当前节点
  return node;
}
```

此方法比较简单, 就是创建`Node`节点并通过`cas修改tail节点`. 如果`cas修改失败`或者`队列为初始化`, 则调用`enq`方法开启循环进行队列初始化和`cas`修改`tail`节点.



#### enq方法原理

```java
private Node enq(final Node node) {
  // 开启循环
  for (;;) {
    Node t = tail;
    if (t == null) { // 尾节点为空, 则头结点必然也为空
      if (compareAndSetHead(new Node()))	// 初始化头结点
        tail = head;
    } 
    // 不为空
    else {
      node.prev = t;
      // 设置node节点为尾节点
      if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
      }
    }
  }
}
```

可以看出此方法的作用有两个: `1. 初始化队列头(head) 和 尾(tail); 2.将指定节点添加到尾部(即tail)`

循环的终止条件为: 通过cas将tail节点修改成当前节点成功才会结束.



#### release方法原理

```java
public final boolean release(int arg) {
  // 尝试释放锁
  if (tryRelease(arg)) {
    Node h = head;	
    if (h != null && h.waitStatus != 0)
      // 解锁后继者
      unparkSuccessor(h);
    return true;
  }
  // 失败直接返回
  return false;
}
```



#### acquireShared方法原理

```java
public final void acquireShared(int arg) {
  // 尝试获取共享锁, 小于0说明共享锁已经没了.
  if (tryAcquireShared(arg) < 0)
    // 再次尝试获取
    doAcquireShared(arg);
}
```

##### doAcquireShared方法原理

```java
private void doAcquireShared(int arg) {
  // 添加一个共享节点
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    boolean interrupted = false;
    // 循环, 也叫自旋
    for (;;) {
      // 前置节点
      final Node p = node.predecessor();
      if (p == head) {	// 如果前置节点为头结点
        // 再次尝试获取共享锁. 此时两种情况: 头结点释放了锁 或者 没有释放锁.
        int r = tryAcquireShared(arg);
        // 大于0, 说明还有锁资源
        if (r >= 0) {
          // 设置头结点, 并传播
          setHeadAndPropagate(node, r);
          p.next = null; // help GC
          // 如果线程设置过中断标志位, 则直接抛出中断异常
          if (interrupted)
            selfInterrupt();
          failed = false;
          return;
        }
      }
      // 前置节目非头结点 或者 前置节点是头节点但确没有释放锁
			// shouldParkAfterFailedAcquire参考上面的解析
      if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      // 失败则取消节点
      cancelAcquire(node);
  }
}
```

当执行此方法时, 存在的场景如下:

1. 成功获取到锁, 执行完`setHeadAndPropagate`返回.
2. 继续循环, 不断调用`shouldParkAfterFailedAcquire`方法修改前置节目的状态(修改成SIGNAL).
3. 已经调用`parkAndCheckInterrupt`阻塞.



##### setHeadAndPropagate方法原理

```java
private void setHeadAndPropagate(Node node, int propagate) {
  // 该方法可能同时有多个线程进入
  Node h = head;
  // 设置并更新头结点
  setHead(node);

  // propagate > 0: 说明还有多余的锁资源. 后续节点可以被唤醒
  // propagate = 0: 说明没有多余的所资源. 需要根据头结点状态来进行判断
  // h.waitStatus < 0: 后续线程可以被唤醒
  // h.waitStatus = 0: 等于0说明可能有其他线程在执行释放锁操作(即调用doReleaseShared方法)
  // 因为可能有人在释放锁, 此时head节点可能已经更新了, 所以再次获取head节点用于判断
  if (propagate > 0 || h == null || h.waitStatus < 0 ||
      (h = head) == null || h.waitStatus < 0) {
    // 获取下一个节点
    Node s = node.next;
    // 处于共享状态, 则执行共享释放.
    if (s == null || s.isShared())
      doReleaseShared();
  }
}
```

由于`releaseShared`方法最终调用的也是`doReleaseShared`方法, 所以将其解析放在下面.



#### releaseShared方法原理

```java
public final boolean releaseShared(int arg) {
  // 尝试释放共享锁
  if (tryReleaseShared(arg)) {
    doReleaseShared();
    return true;
  }
  return false;
}
```



##### doReleaseShared方法原理

```java
private void doReleaseShared() {
  // 调用此方法有两处: 1. 获取锁时调用, 2. 释放锁时调用

  // 开启自旋
  for (; ; ) {
    Node h = head; // 获取头
    // 条件成立的情况: 队列中还有节点
    if (h != null && h != tail) {
      int ws = h.waitStatus;    // 获取节点状态
      // 头结点为SIGNAL, 说明有线程在执行doAcquireShared方法中的循环(且获取共享锁失败)
      if (ws == Node.SIGNAL) {
        // cas更新状态, 告知其他线程有现成在释放锁
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) {
          continue;
        }
        unparkSuccessor(h);
      }

      // ws == 0 表明有可能其他线程也在释放锁
      else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))  {
        continue;
      }
    }
    // 到这里说明, 执行完 unparkSuccessor 或 compareAndSetWaitStatus(h, 0, Node.PROPAGATE) 成功后
    // 如果有后续节目被唤醒且获取锁成功, 则head势必会更改, 反之则不会. 结束循环.
    if (h == head)
      break;
  }
}
```

看完此方法可能有点疑惑, 举个例子: 此时有 A, B, C三个线程, 只有一把共享锁, 假设此时B和C在队列中排队等待A释放锁.

此时队列如下:`head -> B(blocked) -> C(blocked)`  /   `A -- lock`

当A线程调用`doReleaseShared`方法时,  此时head的状态必然是`SIGNAL`, 所以会cas修改状态并唤醒B线程. 

注意此时会出现两种情况:

1. A线程执行到`h == head`时, head还未更改, A线程结束循环, 完成释放是操作. 情况如下:
   1. 出现新的D线程抢走了锁
   2. B线程抢到了锁,但是还未执行`setHead`操作即更新头结点.
2. A线程执行到`h == head`时, head已经更改, A线程继续循环. 情况如下:
   1. B线程执行完了`setHead`操作



当B线程执行`setHeadAndPropagate`方法后最终进入`doReleaseShared`方法, 同A线程一样执行起了相同的唤醒操作.

此时唤醒的线程有A和B两个线程, 注意此时B还未释放锁. 当C线程被唤醒后, 尝试去获取锁, 结果失败. head节点不会更改, 此时`h == head`成立, A线程和B线程结束循环. 退出释放的操作.

后续B线程调用`releaseShared`方法释放锁,则和A线程当初一样. 继续循环唤醒.



简单来说就是: 只要有线程释放锁或获取到锁, 此线程都会加入唤醒后续线程的`队伍`. 直到后续线程唤醒后无法抢锁. 导致head不更新, `队伍`中的唤醒线程才会停止唤醒操作.



#### shouldParkAfterFailedAcquire方法原理

此方法无论是在独享锁或共享锁的情况下都会有调用, 且都是在获取锁失败的情况下.

该方法主要是判断前置节点去状态来对当前节点进行操作: 1. 继续循环获取锁 2. 直接阻塞. 由返回结果标识.

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  // 获取上一个节点的状态
  int ws = pred.waitStatus;
  // 状态为SIGNAL, 说明前置节点也需要被唤醒
  // 疑问: 前置节点的状态在哪里修改的?
  if (ws == Node.SIGNAL)
    return true;
  // 大于0, 说明是前置节点为取消状态, 则跳过, 直到找到非取消状态的节点
  if (ws > 0) {
    do {
      // 跳过取消的前置节点
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } 
  // 非取消状态
  else {
    // cas修改前置节点的状态, 可能成功或失败
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```

可以看出此方法做了两件事:

- 清理队列中的取消节点
- 设置前置节点的状态为`SIGNAL`, 以表示当前节点可以被安全的阻塞

此方法仅当前置节点的状态为`SIGNAL`返回`true`, 其余情况返回`false`. 



#### unparkSuccessor方法原理

注意此方法的调用有多处: 1. 独享锁的释放`release` 2. 共享锁的获取或释放`releaseShared` 3. 取消节点

此方法根据传入的node节点, 唤醒node的后继节点.

```java
private void unparkSuccessor(Node node) {

  int ws = node.waitStatus;
  // 节点
  if (ws < 0)
    // 尝试cas更新节点状态, 可能成功或失败
    compareAndSetWaitStatus(node, ws, 0);

  // 获取到当前节点的下一个节点
  Node s = node.next;
  // 如果节点状态为CANCELLED状态
  if (s == null || s.waitStatus > 0) {
    s = null;
    // 从尾部开始向前寻找, 直到找到一个正常节点.
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  if (s != null)
    // 解锁下一个节点中的线程
    LockSupport.unpark(s.thread);
}
```



#### cancelAcquire方法原理

```java
private void cancelAcquire(Node node) {

  if (node == null)
    return;
  node.thread = null;

  // 向前找到状态小于0的节点(即未取消的节点)
  Node pred = node.prev;
  while (pred.waitStatus > 0){
    node.prev = pred = pred.prev;
  }

  // 获取到前置节点的next节点
  Node predNext = pred.next;

  // 将状态更新为取消, 下次节点入队会清除掉此节点
  node.waitStatus = Node.CANCELLED;

  // 当前节点为尾节点则直接cas更新tail
  if (node == tail && compareAndSetTail(node, pred)) {
    // 此时pred已经是尾节点, 将其next引用设置为null
    compareAndSetNext(pred, predNext, null);
  }
  // 当前节点非tail节点
  else {
    int ws;
    // pred非头结点
    if (pred != head &&
        // 且判断前置节点是否为SIGNAL, 不是则通过CAS修改为SIGNAL
        // 此时pred的状态可能是: SIGNAL, CONDITION, PROPAGATE, 0
        ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL)))
        // 且前置节点线程不为空
        && pred.thread != null
       ) {
      // 获取当前节点的next
      Node next = node.next;
      // 当前节点的后继节点有效
      if (next != null && next.waitStatus <= 0)
        // 则将前置节点的next指针指向当前节点的后继节点
        compareAndSetNext(pred, predNext, next);
    }
    // pred为头结点, 则说明当前节点是第二个. 直接唤醒后续即可.
    else {
      unparkSuccessor(node);
    }

    node.next = node; // help GC
  }
}
```



### ConditionObject

#### Condition

Condition是ConditionObject实现的接口, 定义了ConditionObject包含了那些操作

```java
public interface Condition {

  // 使当前线程阻塞, 直到有信号唤醒或中断
  void await() throws InterruptedException;

  // 使当前线程阻塞, 直到有信号唤醒, 不响应中断
  void awaitUninterruptibly();
  
  // 使当前线程阻塞指定时间
  long awaitNanos(long nanosTimeout) throws InterruptedException;

  // 使当前线程阻塞指定时间
  boolean await(long time, TimeUnit unit) throws InterruptedException;

  // 使当前线程阻塞
  boolean awaitUntil(Date deadline) throws InterruptedException;

  // 唤醒一个等待的线程
  void signal();

  // 唤醒所有等待的线程. 所有唤醒的线程都要重新获取锁才会返回.
  void signalAll();
}
```





#### 设计

在ConditionObject内部其实也是使用的队列来管理等待的线程. 它和AQS使用的队列隔离.



#### 变量

```java
// 条件队列的头
private transient Node firstWaiter;
// 条件队列的尾
private transient Node lastWaiter;
/** Mode meaning to reinterrupt on exit from wait */
private static final int REINTERRUPT =  1;
/** Mode meaning to throw InterruptedException on exit from wait */
private static final int THROW_IE    = -1;
```

可以看到`ConditionObject`也使用了`Node`内部类来封装节点. 



由于AQS中也存在队列所以我们称之为: `同步队列`. 而ConditionObject也存在队列所以称为: `条件等待队列`



#### await方法原理

调用此方法时, 线程必须持有锁.

```java
public final void await() throws InterruptedException {
  // 线程由中断标志位,直接抛出异常
  if (Thread.interrupted())
    throw new InterruptedException();
	// 添加一个新的条件等待节点
  Node node = addConditionWaiter();
  // 全部释放锁
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  // 当前节点是否在 同步队列 中
  while (!isOnSyncQueue(node)) {
    // 阻塞
    LockSupport.park(this);
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  // 循环结束u
  // acquireQueued该方法和独享模式下获取锁调用的方法一致
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```



##### addConditionWaiter方法原理

```java
private Node addConditionWaiter() {
  Node t = lastWaiter;
  // 节点是取消状态
  if (t != null && t.waitStatus != Node.CONDITION) {
    // 将取消状态的节点提出队列
    unlinkCancelledWaiters();
    t = lastWaiter;
  }
  Node node = new Node(Thread.currentThread(), Node.CONDITION);
  if (t == null)
    firstWaiter = node;
  else
    t.nextWaiter = node;
  lastWaiter = node;
  return node;
}
```

从这个方法可以看出 `条件等待队列` 是一个`单向`的链表, 只能`从头向尾`遍历.



#### signal方法原理

```java
public final void signal() {
  // 非独占模式抛出异常
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  // 获取到第一个节点
  Node first = firstWaiter;

  if (first != null)
    // 唤醒一个节点
    doSignal(first);
}
```

从`signal`方法可以看出使用`ConditionObject`必须要在独占模式下才能使用.



##### doSignal方法原理

```java
private void doSignal(Node first) {
  do {
    // 此时first节点已经出队, 即条件等待队列中是否还有节点
    if ((firstWaiter = first.nextWaiter) == null)
      lastWaiter = null;
    first.nextWaiter = null;
  }
  // 唤醒条件等待队列的第一个线程, 且条件队列后续节点不为空
  // transferForSignal 返回false: 说明first节点已经取消, 跳下一个节点继续执行
  // transferForSignal 返回true: 说明first节点已经进入同步队列, 结束循环.
  while (!transferForSignal(first) && (first = firstWaiter) != null);
}
```



#### signalAll方法原理

```java
public final void signalAll() {
  // 非独占模式抛出异常
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  // 获取到第一个节点
  Node first = firstWaiter;
  if (first != null)
    // 唤醒所有节点
    doSignalAll(first);
}
```



##### doSignalAll方法原理

```java
private void doSignalAll(Node first) {
  // 将头和尾节点都置为空
  lastWaiter = firstWaiter = null;
  do {
    // 获取第二个节点
    Node next = first.nextWaiter;
    first.nextWaiter = null;
    // 添加节点到 同步队列 中, 并解锁线程(可能发生)
    transferForSignal(first);
    first = next;
  } while (first != null);
}
```



#### transferForSignal方法原理

```java
final boolean transferForSignal(Node node) {

  // 当前节点已经取消, 单值cas修改当前节点状态失败.
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
    return false;

  // 调用enq方法进入 同步队列. 此时p指针指向的是node节点的前置节点
  Node p = enq(node);
  int ws = p.waitStatus;

  // 前置节点为取消或cas修改前置节点失败则唤醒当前线程
  if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
    LockSupport.unpark(node.thread);
  return true;
}
```

如果`p`节点如果被阻塞, 那么cas必然是能成功的, 只有`p`节点是活动的(可能在获取锁 或者 刚好被唤醒), 才会导致cas修改失败.

此时`node`节点有可能获取到锁, 所以需要唤醒node节点的线程.