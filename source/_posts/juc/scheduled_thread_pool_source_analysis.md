---
title: 调度线程池源码分析
categories: Juc
index_img: img/java-logo-480.png
tags:
  - Java
  - Juc
abbrlink: 50827
---

该文章主要讲解普通线程池子类调度线程池的源码分析

<!-- more -->

## 定时线程池源码分析



### 简介

> `ScheduledThreadPoolExecutor`继承了ThreadPoolExecutor, 并且可以延迟执行某个任务或定期执行一个任务



类继承图如下

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220528_ScheduledExecutorService_extend.png" alt="image-20220528091007019" style="zoom:50%;" />

### ScheduledExecutorService

作为ScheduledThreadPoolExecutor的父接口, 其定义了`schedule`等方法来向线程池提交定时任务.



```java
public interface ScheduledExecutorService extends ExecutorService {

  /**
    * 向线程池提交一个任务, 并在指定延迟时间后执行
    */
  ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);

  /**
    * 向线程池提交一个任务, 并在指定延迟时间后执行
    */
  <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);

  /**
    * 向线程池提交一个任务, 并周期性的执行.
    * <p>
    * preStartTime: 上一次任务执行开始时间
    * preEndTime: 上一次任务执行结束时间
    * nextTime: 下一次任务执行的时间
    * taskTime: task执行的时间
    * <p>
    * 下次任务触发时间:  preStartTime + delay
    * <p>
    * 如果 preEndTime < preStartTime + delay, 则preEndTime就是下次任务执行时间
    * 如果 preEndTime >= preStartTime + delay, 则延迟 preEndTime - (taskTime + delay) 时间后下次任务将执行
    */
  ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, 
                                         long period, TimeUnit unit);

  /**
    * 向线程池提交一个任务, 并周期性执行.
    * <p>
    * preStartTime: 上一次任务执行开始时间
    * preEndTime: 上一次任务执行结束时间
    * nextTime: 下一次任务执行的时间
    * <p>
    * 下次任务触发时间: nextTime = preEndTime + delay;
    */
  ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, 
                                            long delay, TimeUnit unit);
}
```

`scheduleAtFixedRate`和`scheduleWithFixedDelay`方法都是提交的周期性任务. 但是两个方法唯一的区别是参考的计算时间点不同. `一个是上次任务执行开始时间, 一个是上次任务的执行结束时间`. 



### 区别

和ThreadPoolExecutor相比, ScheduledThreadPoolExecutor使用了新的`ScheduledFutureTask`类来包装向线程池提交的任务, 重写了父类的`submit`方法.  并且是使用了`DelayQueue`的变体`DelayedWorkQueue`来存储任务.



### 核心构造函数

```java
// 初始化调度线程池, 使用DelayedWorkQueue来存储提交的任务
public ScheduledThreadPoolExecutor(int corePoolSize) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,new DelayedWorkQueue());
}

// 初始化调度线程池, 指定核心线程数和线程工厂
public ScheduledThreadPoolExecutor(int corePoolSize, ThreadFactory threadFactory) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
        new DelayedWorkQueue(), threadFactory);
}

// 初始化调度线程池, 指定核心线程数和拒绝策略
public ScheduledThreadPoolExecutor(int corePoolSize, RejectedExecutionHandler handler) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedWorkQueue(), handler);
}

// 初始化调度线程池, 指定核心线程数和线程工厂和拒绝策略
public ScheduledThreadPoolExecutor(int corePoolSize, ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
        new DelayedWorkQueue(), threadFactory, handler);
}
```

可以看出线程池的最大线程数都是`Integer.MAX_VALUE`, 且空闲线程的存活时间为 0 . 则表明`非核心线程存活时间为0, 即没有任务就会退出`, 因为 getTask 方法中的 poll 方法很快就会超时. 



### 核心方法

调度线程池核心方法为: `submit` 和 `schedule`两个, 由于调度线程池使用的ScheduledFutureTask, 所以submit方法最终都调用的是`schedule`方法.

#### submit方法原理

```java
public Future<?> submit(Runnable task) {
  return schedule(task, 0, NANOSECONDS);
}

public <T> Future<T> submit(Runnable task, T result) {
  return schedule(Executors.callable(task, result), 0, NANOSECONDS);
}

public <T> Future<T> submit(Callable<T> task) {
  return schedule(task, 0, NANOSECONDS);
}
```

可以看出调度线程池重写了所有的submit方法, 并最终调用了schedule, 但是延迟相关的参数都是0.

#### schedule方法原理

```java
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
  if (command == null || unit == null)
    throw new NullPointerException();
  // 使用 ScheduledFutureTask 包装任务
  RunnableScheduledFuture<?> t = decorateTask(command,
				new ScheduledFutureTask<Void>(command, null,triggerTime(delay, unit)));
	// 延迟执行  
  delayedExecute(t);
  return t;
}

public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {

  if (callable == null || unit == null)
    throw new NullPointerException();
	// 同上, 只是构建的ScheduledFutureTask参数不同
  RunnableScheduledFuture<V> t = decorateTask(callable,                                    
					new ScheduledFutureTask<V>(callable,triggerTime(delay, unit)));
  // 同上
  delayedExecute(t);
  return t;
}
```

使用ScheduledFutureTask对提交的任务进行了封装, 并调用`triggerTime`方法计算出延迟的时间.

可以看出核心方法应该存在于`delayedExecute`中.



#### delayedExecute方法原理

此方法为延迟或周期性执行任务的核心方法. 

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
  // 如果池处于关闭状态, 则直接拒绝任务
  if (isShutdown())
    reject(task);		// 拒绝任务
  // 池未处于关闭状态
  else {
    // 现将任务入队
    super.getQueue().add(task);
    // 池处于关闭状态
    if (isShutdown() && !canRunInCurrentRunState(task.isPeriodic()) && remove(task))
      // 当前任务不能执行, 且出队成功, 则取消当前任务
      task.cancel(false);
    // 池非关闭状态
    else
      ensurePrestart();
  }
}
```

需要注意`canRunInCurrentRunState`方法其判断的条件, 同时了解`ensurePrestart`方法执行细节.



#### ensurePrestart方法原理

此方法处于ThreadPoolExecutor类中.

```java
void ensurePrestart() {
  // 获取工作线程数量
  int wc = workerCountOf(ctl.get());
  // 小于核心线程数
  if (wc < corePoolSize)
    // 添加核心线程
    addWorker(null, true);
  // 工作线程为0
  else if (wc == 0)
    // 添加非核心线程
    addWorker(null, false);
}
```



#### scheduleAtFixedRate原理

```java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, 
                                              long period, TimeUnit unit) {
  // 忽略参数校验

  // 使用ScheduledFutureTask包装任务
  ScheduledFutureTask<Void> sft =
    new ScheduledFutureTask<Void>(command, null, triggerTime(initialDelay, unit), 
                                  unit.toNanos(period));

  RunnableScheduledFuture<Void> t = decorateTask(command, sft);

  // 
  sft.outerTask = t;
	// 同上
  delayedExecute(t);
  return t;
}
```



#### scheduleWithFixedDelay原理

```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay,
                                                 long delay, TimeUnit unit) {
  // 忽略参数校验

  // 使用ScheduledFutureTask包装任务
  ScheduledFutureTask<Void> sft =
    new ScheduledFutureTask<Void>(command, null, triggerTime(initialDelay, unit),
                                  unit.toNanos(-delay));
  
  RunnableScheduledFuture<Void> t = decorateTask(command, sft);
  sft.outerTask = t;
  delayedExecute(t);
  return t;
}
```



scheduleAtFixedRate和scheduleWithFixedDelay代码大致相同, 只是构建ScheduledFutureTask的参数有所差异.



#### triggerTime方法原理

该方法用于计算出周期性任务的执行时间, 此方法在`schedule`也有用调用.

```java
// 构造函数调用此方法
private long triggerTime(long delay, TimeUnit unit) {
  return triggerTime(unit.toNanos((delay < 0) ? 0 : delay));
}
```

调用下面的重载方法

```java
long triggerTime(long delay) {
  // 根据当前时间计算出延迟时间
  return now() +
    ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}

private long overflowFree(long delay) {
  // 获取到任务队列的头
  Delayed head = (Delayed) super.getQueue().peek();
  // 头不为空
  if (head != null) {
    // 获取到任务延迟
    long headDelay = head.getDelay(NANOSECONDS);
    // 使延迟时间大于Long.MAX_VALUE, 避免compareTo计算溢出
    if (headDelay < 0 && (delay - headDelay < 0))
      delay = Long.MAX_VALUE + headDelay;
  }
  return delay;
}
```



#### 总结

至此可以看出所有 schedule 开头的方法都是先构建ScheduledFutureTask对象, 然后在 `delayedExecute`方法将构建好的任务提交到线程池并添加一个Worker. 



### ScheduledFutureTask

看完池的任务提交代码, 可以知道池的核心逻辑在父类ThreadPoolExecutor已经实现好了, 唯一需要了解的就是ScheduledFutureTask, 此类决定了任务的执行逻辑, 和执行时间.

此类类似于FutureTask.



#### 核心变量

```java
// 序列号用于打断FIFO
private final long sequenceNumber;

// 重复执行任务的周期, 单位纳秒.
// 正数表示固定周期执行, 负数表示固定延迟执行. 0 标识非周期性任务
private final long period;

// 下一个周期需要入队的任务
RunnableScheduledFuture<V> outerTask = this;

// 延迟队列中的索引, 用于支持快速取消
int heapIndex;

// 任务开始执行的时间
private long time;
```



#### 核心构造函数

```java
ScheduledFutureTask(Runnable r, V result, long ns) {
  super(r, result);
  this.time = ns;
  this.period = 0;
  this.sequenceNumber = sequencer.getAndIncrement();
}

ScheduledFutureTask(Runnable r, V result, long ns, long period) {
  super(r, result);
  this.time = ns;
  this.period = period;
  this.sequenceNumber = sequencer.getAndIncrement();
}

ScheduledFutureTask(Callable<V> callable, long ns) {
  super(callable);
  this.time = ns;
  this.period = 0;
  this.sequenceNumber = sequencer.getAndIncrement();
}
```



#### 核心方法



##### run方法原理

```java
public void run() {
  // 是否是周期性执行任务
  boolean periodic = isPeriodic();
  // 当前池状态能否执行周期
  if (!canRunInCurrentRunState(periodic))
    // 不能则取消任务
    cancel(false);
  // 不能周期执行
  else if (!periodic)
    // 调用父类
    ScheduledFutureTask.super.run();
  // 可以周期性执行
  // runAndReset 此方法只会执行Future但是不会设置结果
  else if (ScheduledFutureTask.super.runAndReset()) {
    setNextRunTime();   // 设置下一次运行时间
    reExecutePeriodic(outerTask);   // 再次执行周期任务
  }
}
```

##### setNextRunTime方法原理

设置Task下次执行的事件

```java
private void setNextRunTime() {
    long p = period;
    if (p > 0)  // 固定周期执行
        time += p;  // 替换下次task执行时间
    else        // 固定延迟周期执行
        time = triggerTime(-p);     // 根据当前时间计算出下次延迟的时间, 此时任务已经执行完成
}
```



### DelayedWorkQueue

此队列用来存储向调度线程池提交的任务. 底层使用数组来存储任务, 并将任务在数组中的索引存储在任务中.



#### 核心变量

```java
static class DelayedWorkQueue extends AbstractQueue<Runnable>
  implements BlockingQueue<Runnable> {

  // 初始化容量
  private static final int INITIAL_CAPACITY = 16;
  // 任务数组
  private RunnableScheduledFuture<?>[] queue = new RunnableScheduledFuture<?>[16];
  // 锁
  private final ReentrantLock lock = new ReentrantLock();
  // 数组大小
  private int size = 0;
  // 阻塞在队列的第一个线程
  private Thread leader = null;
  // 当队列头部有新任务可用或新线程可能需要成为领导者时发出条件信号
  private final Condition available = lock.newCondition();
}
```



#### 核心方法

在线程池的源码分析中, 我们知道了线程池操作队列的基本方法: 

- offer: 向队列中添加任务
- poll: 超时阻塞获取队列中的任务
- take: 阻塞获取队列中的任务
- remove: 删除队列中的任务

所以重点关注这几个方法.



##### offer方法原理

```java
public boolean offer(Runnable x) {
  // 忽略参数非空校验
  
  RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
  final ReentrantLock lock = this.lock;
  lock.lock();
  try {
    int i = size;
    // 数组容量太小
    if (i >= queue.length)
      // 重新计算数组大小
      grow();             

    size = i + 1;
    if (i == 0) {   // 第一次添加
      queue[0] = e;
      setIndex(e, 0);
    }
    // 非第一次添加
    else {
      // 入队, 该方法从数组的后面向前找, 会对数组中任务的延迟时间进行排序
      // 避免任务延迟时间到了, 但是未执行的情况.
      siftUp(i, e);
    }
    // 任务处于第一个
    if (queue[0] == e) {
      leader = null;
      available.signal();     // 唤醒等待的线程, 调用take
    }
  } finally {
    lock.unlock();
  }
  return true;
}
```



##### poll方法原理

```java
public RunnableScheduledFuture<?> poll(long timeout, TimeUnit unit)
  throws InterruptedException {

  long nanos = unit.toNanos(timeout);
  final ReentrantLock lock = this.lock;
  // 可中断
  lock.lockInterruptibly();
  try {
    // 循环
    for (;;) {
      // 获取数组头的任务
      RunnableScheduledFuture<?> first = queue[0];
      // 为空, 表示数组中无任务
      if (first == null) {
        if (nanos <= 0)
          return null;
        // 等待指定时间
        else
          nanos = available.awaitNanos(nanos);
      }
      // 不为空
      else {
        // 获取到任务的延迟时间
        long delay = first.getDelay(NANOSECONDS);
        // 可以立即执行
        if (delay <= 0)
          return finishPoll(first);
        // 延迟时间未到就已经超时了
        if (nanos <= 0)
          return null;

        first = null;

        // 超时时间小于延迟时间, 或者 超时时间大于延迟时间且等待线程已经存在
        if (nanos < delay || leader != null)
          // 延迟等待
          nanos = available.awaitNanos(nanos);
        // 超时时间大于延迟时间且等待线程不存在, 则将当前线程设置上
        else {
          Thread thisThread = Thread.currentThread();
          // 复制
          leader = thisThread;
          try {
            // 阻塞延迟时长
            long timeLeft = available.awaitNanos(delay);
            // 重新计算超时时间
            nanos -= delay - timeLeft;
          } finally {
            if (leader == thisThread)
              leader = null;
          }
        }
      }
    }
  } finally {
    // 没有等待头线程,且队列头任务不为空
    if (leader == null && queue[0] != null)
      available.signal();     // 唤醒一个等待线程
    lock.unlock();
  }
}
```

在线程池的`runWorker`源码中, 只有当工作线程大于核心线程数时才会使用poll方法来超时获取任务.

而此方法的逻辑, 则是开启循环, 不断计算任务的延迟时间, 当任务的延迟时间到了, 则出队. 反之则一直循环.  

##### take方法原理

```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
  final ReentrantLock lock = this.lock;
  lock.lockInterruptibly();
  try {
    for (;;) {
      // 获取到队列中的头任务
      RunnableScheduledFuture<?> first = queue[0];
      if (first == null)
        // 为空则阻塞调用线程
        available.await();
      else {
        // 不为空,则获取到任务的延迟时间
        long delay = first.getDelay(NANOSECONDS);
        if (delay <= 0)
          // 延迟时间已过, 直接获取任务
          return finishPoll(first);
        // 延迟时间未过
        first = null;
        if (leader != null)
          // 已经有线程在等待了,则阻塞当前线程
          available.await();
        // 没有线程等待
        else {
          Thread thisThread = Thread.currentThread();
          leader = thisThread;
          try {
            // 延迟指定时间
            available.awaitNanos(delay);
          } finally {
            // 线程被唤醒后, 如果相等
            if (leader == thisThread)
              leader = null;
          }
        }
      }
    }
  } finally {
    if (leader == null && queue[0] != null)
      available.signal(); // 唤醒一个等待线程
    lock.unlock();
  }
}
```

take方法和poll比较类似, 只是判断的条件少了很多.



### 总结

调度线程池复用了很多线程池的代码. 特别注意在初始化调度线程池时部分参数的特殊性. 且是使用的队列和普通线程池不同.

调度线程池执行的任务由 `ScheduledFutureTask` 封装, 且如果任务为周期性任务, 则任务的再次入队操作在 `run` 方法中完成提交. 同时使用了 `time` 和 `period` 变量来分别存储`任务的执行时间`和`任务执行间隔`.

调度线程池使用 `DelayedWorkQueue` 来存储提交的任务, 底层使用数组结构. 并且在 `poll` 和 `take` 方法中不断读取 `ScheduledFutureTask` 的延迟时间以便执行此任务(循环+条件队列). 同时可以使用 `offer` 方法将任务入队, 同时唤醒等待的线程和对队列中的任务进行排序. `(保证任务能够执行)`.