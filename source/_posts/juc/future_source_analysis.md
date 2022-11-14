---
title: Future及FutureTask的源码分析
categories: Juc
index_img: img/java-logo-480.png
tags:
  - Java
  - Juc
  - Future
abbrlink: 31883
date: 2022-05-28 18:34:43
---

该文章主要讲解在向线程池提交任务时, 返回的Future的源码分析.

<!-- more -->

## Future



### 简介

一个Future代表一个异步计算的结果,  提供了检查计算是否完成, 等待其完成以及检索计算结果的方法.

在向ThreadPoolExecutor中提交任务时, 会返回一个Future.

### 接口方法

```java
public interface Future<V> {

  // 取消Future, 当Future已完成或者已经取消, 此方法返回false. 
  // 当此方法调用后, isDone方法将始终返回true
  // false: 表示Future取消失败, true: 表示Future取消成功
  boolean cancel(boolean mayInterruptIfRunning);

  // Future是否取消
  boolean isCancelled();
  
  // Future 是否完成
  boolean isDone();

  // 等待Future完成并获取结果
  // 当Future被取消时,会抛出CancellationException
  // 当Future出现异常时,会抛出ExecutionException
  V get() throws InterruptedException, ExecutionException;

  // 超时等待Future完成并获取结果
  // 异常结果如上
  V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
}
```



### FutureTask

来看看Future的实现类`FutureTask`, 该类实现了 `RunnableFuture` ,而 RunnableFuture 继承了 Future 了.



#### 简介

FutureTask提供了对Future的基础实现. 一个可取消的异步计算. 且可以使用FutureTask包装Runnable和Callable.



#### 状态变量

```java
public class FutureTask<V> implements RunnableFuture<V> { 
  // 同步状态
  private volatile int state;

  // 初始化状态
  private static final int NEW          = 0;
  private static final int COMPLETING   = 1;
  private static final int NORMAL       = 2;
  private static final int EXCEPTIONAL  = 3;
  private static final int CANCELLED    = 4;
  private static final int INTERRUPTING = 5;
  private static final int INTERRUPTED  = 6;
}
```

`使用state来控制FutureTask的状态变化, 可行的状态变化`

```text
NEW -> COMPLETING -> NORMAL
NEW -> COMPLETING -> EXCEPTIONAL
NEW -> CANCELLED
NEW -> INTERRUPTING -> INTERRUPTED
```



#### 核心变量

```java
public class FutureTask<V> implements RunnableFuture<V> { 
  // 潜在的callable, 当Future运行结束后清空
  private Callable<V> callable;
  // 返回的结果 或者 执行任务中抛出的异常(调用get方法时)
  private Object outcome; 
  // 执行callable的线程
  private volatile Thread runner;
  // 阻塞队列
  private volatile WaitNode waiters;
}
```



#### 构造函数

```java
public class FutureTask<V> implements RunnableFuture<V> { 
  
  // 初始化状态和赋值callable
  public FutureTask(Callable<V> callable) {
    if (callable == null)
      throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;
  }

  // 包装 Runnable
  public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;
  }
}
```



### 核心方法

由于 FutureTask 实现了 Runnable 接口, 所以优先查看`run`方法



#### run方法原理

run方法继承自Runnable, 当FutureTask执行时会调用其run方法.

```java
public void run() {
  // 条件成立的情况: 1. Future已经结束(状态未知) 
  // 2. Future处于NEW状态 且 cas修改执行当前FutureTask的线程失败
  if (state != NEW ||
      !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
    return;

  // FutureTask处于NEW状态 且 cas 当前 FutureTask 的执行线程成功.
  try {
    // 得到要执行的任务
    Callable<V> c = callable;
    // 不为空切状态为初始化状态
    if (c != null && state == NEW) {
      V result;
      boolean ran;
      try {
        // 调用call方法执行挼是
        result = c.call();
        ran = true;
      } catch (Throwable ex) {
        result = null;
        ran = false;
        // 设置异常
        setException(ex);
      }
      if (ran)
        // 设置结果
        set(result);
    }
  } finally {
    // 疑问: 问什么不cas更新? 此时线程安全, 只能有一个线程执行此操作
    runner = null;
    // 重新读取状态, 以免错过中断状态
    int s = state;
    if (s >= INTERRUPTING)
      handlePossibleCancellationInterrupt(s);
  }
}

protected void setException(Throwable t) {
	// 此方法可能被多个方法调用, 所以使用cas  
  if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
    outcome = t;
    UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); 
    // 完成计算, 主要是释放waiters链表存储的线程
    finishCompletion();
  }
}

protected void set(V v) {
  // 此方法可能被多个方法调用, 所以使用cas
  if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
    outcome = v;
    UNSAFE.putOrderedInt(this, stateOffset, NORMAL);
    // 完成计算, 主要是释放waiters链表存储的线程
    finishCompletion();
  }
}
```

在`setException`和`set`方法中唯一不同的就是修改的状态不同. 大致如下: 

`setException`状态变化: `NEW -> COMPLETING -> EXCEPTIONAL`

`set状态`变化: `NEW -> COMPLETING -> NORMAL`



##### finishCompletion方法原理

```java
private void finishCompletion() {
  // assert state > COMPLETING;
  // 遍历等待节点
  for (WaitNode q; (q = waiters) != null;) { 
    // 更新waiters==null
    if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
      // cas成功
      for (;;) {
        Thread t = q.thread;
        if (t != null) {
          q.thread = null;
          LockSupport.unpark(t);	// 解锁线程
        }
        WaitNode next = q.next;		// 获取下一个节点
        if (next == null)
          break;
        q.next = null;
        q = next;
      }
      break;
    }
  }
	// 钩子函数
  done();
	// 将执行的任务置为空
  callable = null;
}
```



##### handlePossibleCancellationInterrupt方法原理

该方法只有当FutureTask的状态为`INTERRUPTING`或`INTERRUPTED`才会被调用.

```java
/**

 */
private void handlePossibleCancellationInterrupt(int s) {

  if (s == INTERRUPTING)  // FutureTask状态为中断中
    while (state == INTERRUPTING)   // 说明此时cancel方法还没结束, 所以让出cpu时间
      Thread.yield();

  // assert state == INTERRUPTED;

  // We want to clear any interrupt we may have received from
  // cancel(true).  However, it is permissible to use interrupts
  // as an independent mechanism for a task to communicate with
  // its caller, and there is no way to clear only the
  // cancellation interrupt.
  //
  // Thread.interrupted();
}
```



注意:

只有当调用cancel(true)方法并传递true时, FutureTask的状态才会被更改为INTERRUPTING.

调用此方法的场景:

 * FutureTask还未被执行, 调用了cancel方法, 当再次执行run方法时则无法执行.
 * FutureTask正在被执行, 调用了cancel方法.
   * 如果执行此Future的线程阻塞, 则会响应中断异常, 此方法会被调用
   * 如果执行此Future的线程未阻塞, 此方法不会被调用





#### cancel方法原理

该方法用于取消FutureTask

```java
public boolean cancel(boolean mayInterruptIfRunning) {
  // 条件成立情况: 状态非NEW 或 状态为NEW但cas状态失败
  if (!(state == NEW &&
        UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                                 mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
    return false;

  // 状态为NEW且cas状态成功
  try {
    // true: 标识中断线程
    if (mayInterruptIfRunning) {
      try {
        Thread t = runner;
        // 当此FutureTask在线程池中排队时, 此时Thread就是空, 此时中断没有意义
        if (t != null)
          // 设置中断标志位
          t.interrupt();
      } finally {
        // 更新状态为已中断
        // 当此FutureTask在线程池中获取执行将不会成立.
        UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
      }
    }
  } finally {
    // 唤醒状态
    finishCompletion();
  }
  return true;
}
```

注意:

 该方法只有`mayInterruptIfRunning`参数为true时,才会设置执行线程的中断标志位,  这就意味着如果执行该FutureTask的线程被阻塞, 就会响应中断. 反之如果不阻塞,则不会响应中断. FutureTask正常执行.

#### get方法原理

此方法用于获取`FutureTask`异步计算的结果, 当FutureTask未计算完成时, 此方法会阻塞调用线程.

```java
public V get() throws InterruptedException, ExecutionException {
  int s = state;
  if (s <= COMPLETING)
    s = awaitDone(false, 0L);
  return report(s);
}
```



#### awaitDone方法原理

该方法用于等待任务完成

```java
private int awaitDone(boolean timed, long nanos) throws InterruptedException {
  // 计算超时时间, 纳秒
  final long deadline = timed ? System.nanoTime() + nanos : 0L;
  WaitNode q = null;
  boolean queued = false;
  // 开启循环
  for (;;) {
    // 线程是否中断过并清除中的标志位
    if (Thread.interrupted()) {
      // 删除等待节点
      removeWaiter(q);
      throw new InterruptedException();
    }

    int s = state;
    // FutureTask已经完成
    if (s > COMPLETING) {
      if (q != null)
        q.thread = null;
      return s;
    }
    // 如果FutureTask正在计算中
    else if (s == COMPLETING)
      Thread.yield();       // 让出线程
    // FutureTask处于NEW
    else if (q == null)
      // 初始化等待节点, 存储当前线程
      q = new WaitNode();
    // 等待节点是否入队
    else if (!queued)
      // 通过cas更新waiters引用
      queued = UNSAFE.compareAndSwapObject(this, waitersOffset, q.next = waiters, q);
    // 如果超时获取,则调用parkNanos方法
    else if (timed) {
      nanos = deadline - System.nanoTime();
      if (nanos <= 0L) {
        removeWaiter(q);
        return state;
      }
      LockSupport.parkNanos(this, nanos);
    }
    // 非超时获取,则调用park方法
    else
      LockSupport.park(this);
  }
}
```



### 总结

FutureTask由于实现了Runnable接口, 让其自身具备了被执行的能力. 且本身缓存了执行当前FutureTask的线程. 同时又实现了Future接口, 具备了操作异步计算的能力.

其通过一个state变量来控制FutureTask的状态变化.