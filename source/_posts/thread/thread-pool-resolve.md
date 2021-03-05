---
title: ThreadPoolExecutor 解析
tags:
  - Java
  - Juc
  - ThreadPool
  - 源码分析
categories: Juc
index_img: img/java-logo-480.png
abbrlink: 33784
---
该文章主要讲述 Java 中 Juc 包下的 ThreadPoolExecutor 
主要讲述 线程池 的核心设计以及其流程执行和源码分析

<!-- more -->
## 本章简介

>本章主要讲述线程池
>
>1. 线程池的核心配置参数
>2. 线程池任务提交执行流程
>3. 线程池中线程新增流程
>4. 线程池中线程回收流程
>5. 线程池核心参数动态调整
>6. 线程池队列动态调整



## Executor

### 继承图

我们先来看看线程池的类继承图

![image-20210304173122499](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210304173122499.png)



### 简介

Executor作为线程池的顶级接口, 定义了`task`的提交方法, 并将`task`的`提交`和`执行`进行了`解耦`.

### 核心方法

Executor 接口中只有一个 `execute` 方法, 用于提交任务.

```java
public interface Executor {
  void execute(Runnable command);
}
```



## ExecutorService

#### 简介

`Executor`提供了向线程池中提交任务的方式,但是却没有提供管理线程池的相关方法. 

`ExecutorService` 定义了基于`execute`的`submit`方法, 该方法会返回一个`Future`对象, 此对象可用于停止`task`的执行或等待`task`执行成功.

同时还定义了`shutdown`方法和`shutdownNow`方法来关闭线程池



#### 核心方法

```java
public interface ExecutorService extends Executor {

  // 关闭线程池, 如果线程池中有task, 则会执行这些task, 该方法不会等待先前提交的task执行完成, 
  // 线程池将不再接受新task
  void shutdown();

  // 尝试停止所有的task, 停止等待中的task, 并返回正在等待执行的task
  // 此方法不等待主动执行的任务终止, 除了尽最大努力尝试停止处理正在执行的任务之外，没有任何保证.
  // 典型的实现将通过Thread.interrupt取消，因此任何无法响应中断的任务都可能永远不会终止.
  List<Runnable> shutdownNow();

  // 返回当前线程池是否关闭
  boolean isShutdown();

  // 返回此线程池所有的task是否全部停止
  // 请注意，除非先调用shutdown或shutdownNow, 否则isTerminated永远不会为true.
  boolean isTerminated();

  // 阻塞，直到关闭请求后所有任务完成执行，或者发生超时，或者当前线程被中断（以先发生者为准）.
  boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException;

  // 提交一个task, 返回Future, Future.get方法可获取task的执行结果
  <T> Future<T> submit(Callable<T> task);

  // 同上
  <T> Future<T> submit(Runnable task, T result);

  // 同上
  Future<?> submit(Runnable task);

  // 执行集合中所有的task
  <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)

  // 超时执行集合中所有的task
  <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout,  
                                TimeUnit unit)

	// 执行集合中任意一个的task, 就返回
  <T> T invokeAny(Collection<? extends Callable<T>> tasks);

  // 超时执行集合中任意一个的task, 就返回
  <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit);
}
```



代码中的`shutdown`和`shutdownNow`都不保证已执行`task`的完成, 如果想要做到已执行的`task`完成后关闭线程池, 则可以使用`awaitTermination`方法.

## AbstractExecutorService

#### 简介

该类是ExecutorService的基础实现,  对现`submit `，` invokeAny`和`invokeAll`等方法进行的简单的实现.

对Future进行了再次封装, 使用RunnableFuture来间接的替换了Future.



#### 核心方法

```java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
  return new FutureTask<T>(runnable, value);
}

public Future<?> submit(Runnable task) {
  if (task == null) throw new NullPointerException();
  RunnableFuture<Void> ftask = newTaskFor(task, null);
  execute(ftask);
  return ftask;
}
```



## ThreadPoolExecutor

### 简介

此类为线程池的最终实现, 主要实现了 execute 方法,不论是 ExecutorService中的submit 方法 它们最终调用的都是 `execute` 方法.



### 核心参数配置

创建一个线程池必须要用其构造函数, 下面来看看 ThreadPoolExecutor 的构造函数

![image-20210304183341075](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210304183341075.png)

上面的三个构造函数最终都会调用到最后一个构造函数即参数最多的

```java
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
  // 忽略代码
}
```

我们可以看到ThreadPoolExecutor的构造函数一共有四个, 但每个函数至少会有四个参数,分别是:

- `corePoolSize:` 此参数表示当前线程池有多少`核心线程`
- `maximumPoolSize:` 此参数表示当前线程池`最大`能创建多少`线程`
- `keepAliveTime: ` 此参数表示`超过核心线程数量的线程存活的时间`
- `unit:` 时间单位,需要结合 `keepAliveTime` 使用
- `workQueue:` 此参数用于存放想线程池提交的任务

而 `ThreadFactory` 和 `RejectedExecutionHandler` 分别用于创建线程和拒绝任务(`当队列满,且线程池中存活线程达到最大线程池`)

在 ThreadPoolExecutor 中默认提供了四种拒绝策略, 都已内部类的形式.

![image-20210304184245365](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210304184245365.png)



#### 创建新线程

线程的创建就是交给 ThreadFactory 参数实例来完成的, 线程池 默认使用`Executors.defaultThreadFactory`, 该工厂创建的线程拥有同一个ThreadGroup,且拥有相同的优先级和非守护进程状态. 也可以通过自定义线程池来定义线程名称和修改优先级和守护线程状态.

```java
public interface ThreadFactory {
	// 创建一个新线程 ,具有相同的ThreadGroup和优先级
  Thread newThread(Runnable r);
}
```

如果要实现创建不同优先级或守护线程状态, 可自定义 ThreadFactory.



#### 队列

线程池使用了队列来存储调用线程提交的`task`. 

队列的使用和线程池的大小有关系:

- 如果线程池中运行的线程小于corePoolSize, 会直接创建线程执行task
- 如果线程池中运行的线程大于corePoolSize,
  - 队列未满直接入队,并创建一个 Worker 执行 task (此 Worker 不一定马上执行此 task)
  - 队列已满时, 则会创建线程执行, 如果创建后的线程数大于`maximumPoolSize`, 则会执行拒绝策略.



而排队的策略有以下三种:

1. 同步队列, 比较好的队列是`SynchronousQueue`
2. 无限队列
   1. 当线程池中执行线程达到corePoolSize时, 新提交的task将会直接排队.
   2. maximumPoolSize属性的设置将没有意义
3. 有界队列
   1. 当maximumPoolSizes有限时, 可使用有界队列, 防止资源耗尽



队列的具体实现由:

1. `ArrayBlockingQueue` : 有界的数组队列, 初始化时指定大小
2. `LinkedBlockingQueue` : 有界的链表队列, 默认值为 Integer.MAX
3. `DelayQueue` : 延迟队列, 只要延迟时间到期才能获取
4. `SynchronousQueue` : 同步队列, 获取和放入是同步完成的
5. `PriorityBlockingQueue` : 优先级队列, 可通过compareTo 来排序, 同级别的不能保证



#### 拒绝策略

拒绝策略在`队列已满时`且`线程达到maximumPoolSize`时将会执行, 当达到前面两种情况时线程池会通过`RejectedExecutionHandler.rejectedExecution(Runnable, ThreadPoolExecutor)`来拒绝task的提交.

`RejectedExecutionHandler `提供了四种拒绝策略实现:

1. AbortPolicy 实现: 拒绝策略在拒绝时会抛出`RejectedExecutionException`
2. CallerRunsPolicy 实现: 使用调用execute方法的调用线程(`自身而非线程池中的线程`)来执行task.
3. DiscardPolicy 实现: 删除队列中无法执行的task
4. DiscardOldestPolicy 实现: 丢弃队列的`头`任务, 然后重试执行(该操作可能再次失败)



也可以通过实现RejectedExecutionHandler接口来实现自定义拒绝策略.



### 任务执行

了解完线程池的创建参数后, 就可以创建一个线程池了, 这时就需要关注线程池是如何提交任务. 在线程池中submit 无论是提交 Runnable 还是 Callable 最终都会调用 execute 方法.

#### `execute`

##### **源代码**

先来看看源代码, 分析一下 execute 方法具体执行流程

```java
public void execute(Runnable command) {
  if (command == null)
    throw new NullPointerException();
  // 获取线程池的状态字段
  int c = ctl.get();
  // step1: 如果工作线程数小于核心线程数, 则可以直接创建一个 Worker
  if (workerCountOf(c) < corePoolSize) {
    // 尝试添加一个工作线程并执行task
    if (addWorker(command, true))
      return;
    // 失败重新获取ctl
    c = ctl.get();
  }
	// 可能出现的情况:
  // 1. 线程池未在运行状态(忽略)
  // 2. task入队失败(队列已满)
  if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    // 再次检查线程池状态, 如果未运行, 则从队列中删除此 task, 并执行拒绝策略
    if (! isRunning(recheck) && remove(command))
      reject(command);
    // 线程池运行且工作线程数量为 0, 则添加一个空任务
    else if (workerCountOf(recheck) == 0)  
      addWorker(null, false);
  }
	// 执行此 else if 的情况
  // 1. 队列已满, 会导致上面的入队失败
  else if (!addWorker(command, false))
    reject(command);
}
```

##### **执行流程**

- 步骤1: 获取线程数并判断线程池中的线程是否小于核心线程数
  - 小于则添加一个Worker(addWorker方法)
  - 大于则执行步骤2
- 步骤2: 判断线程池是否运行, 且task是否能插入队列成功?
  - 成功: 双重检查线程池状态
    - 未运行: 则将刚刚入队的 task 移除,并执行拒绝策略
    - 运行中: 如果线程池工作线程为 0, 则添加一个 Worker(一个 Worker 就是一个线程)
  - 失败: 执行步骤3
- 步骤3: 再次尝试添加Worker, 如果失败则执行拒绝策略



注意上面的代码多次调用了`addWorker`方法, 顾名思义该方法添加了一个工作者去执行用户提交的task. 

**且addWorker方法的第二个参数在第一次调用和后面一次调用时值不一样** 

**该值在 addWorker 方法中用于区分比较的值(true: 比较的是核心线程, false: 比较的是最大线程)**



#### Worker

##### 简介

Worker 主要维护线程运行任务的中断控制状态，以及其他次要记录。同时扩展了AbstractQueuedSynchronizer来简化获取和释放围绕每个任务执行的锁。

这可以防止旨在唤醒工作线程等待任务的中断，而不是中断正在运行的任务。

我们实现了一个简单的`非可重入互斥锁`，而不是使用ReentrantLock，因为我们不希望辅助任务在调用诸如setCorePoolSize之类的池控制方法时能够重新获取该锁。
另外，为了抑制直到线程真正开始运行任务之前的中断，我们将锁定状态初始化为负值，并在启动时将其清除（在runWorker中）。



##### 核心字段

```java
/** Worker运行的线程 */
final Thread thread;
/** 初始化Worker时要执行的task */
Runnable firstTask;
```



##### 构造函数

```java
Worker(Runnable firstTask) {
  setState(-1); 
  this.firstTask = firstTask;
  this.thread = getThreadFactory().newThread(this); // 调用 ThreadFactory 创建线程
}
```



看完 Worker 类,我们知道了 Worker 就是一个工作线程, 而线程池中的 task 也是由 Worker 来执行的.

接下来回到 execute 方法中继续分析 addWorker 方法

#### `addWorker`

##### 源代码

```java
private boolean addWorker(Runnable firstTask, boolean core) {
  // 开启死循环
  retry:
  for (;;) {
    int c = ctl.get();
    int rs = runStateOf(c); // 获取运行状态

    // step1: 检查队列和线程池状态和firstTask参数
    if (rs >= SHUTDOWN &&
        ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
      return false;

    // 死循环
    for (;;) {
      int wc = workerCountOf(c);  // 获取线程池中的工作线程
      // step2: 判断工作线程是否达到阈值
      // 这里的 core 就解释了上面为什么了 workerCoun>corePoolSize 传递的参数为 false
      if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
        return false;

      // 工作线程自增 1, 失败说明其他线程也调用了此方法
      if (compareAndIncrementWorkerCount(c)) {
        break retry;
      }

      c = ctl.get();
      if (runStateOf(c) != rs)  // 线程池状态发生改变(调用了 shutdown 方法), 继续自旋
        continue retry;
    }
  }
  
  // 执行到此,说明workerCount已经自增成功
  boolean workerStarted = false;
  boolean workerAdded = false;
  Worker w = null;
  try {
    // 实例化Worker
    w = new Worker(firstTask);
    final Thread t = w.thread; // 获取到Worker的thread, 此线程使用线程池的ThreadFactory创建
    if (t != null) {
      // 获取到锁(线程池级别)
      final ReentrantLock mainLock = this.mainLock;
      mainLock.lock();
      try {
        // 忽略部分代码
        workers.add(w);        // 将Worker添加到hash表中, 方便后期线程释放 回收处理
        // 忽略部分代码
      } finally {
        mainLock.unlock(); // 解锁
      }
      if (workerAdded) {
        t.start();      // 添加成功启动Worker的线程
        workerStarted = true;
      }
    }
  } finally {
    if (! workerStarted)  // worker启动失败, 执行
      addWorkerFailed(w);  
  }
  return workerStarted;
}
```



##### 执行流程

大致分为两个阶段: 

1. 修改`workerCount`数量
   1. 开启一个死循环, 获取到线程池运行状态, 判断状态和队列及入参是否合法,不合法直接返回
   2. 在开启一个死循环, 比较workerCount是否超过maximumPoolSize或corePoolSize, 超过直接返回.
   3. 对工作线程数进行自增+1 操作成功,结束第一阶段.
      1. 自增失败的情况: 1. 其他线程修改了workerCount 2. 线程池状态发生改变
      2. 如果其他线程修改了 workerCount,则继续执行内层循环, 直到修改 workerCount成功
      3. 如果是线程池状态改变, 则继续外层循环
2. 创建 Worker
   1. 创建一个 Worker 实例, 并获取到其线程, 如果Worker 中的线程为空, 说明 ThreadFactory 创建线程失败
   2. 获取到线程池的锁, 将 Worker 实例放入到 workers 集合中, 方便后续线程销毁
   3. 启动 Worker 中的线程, 如果启动失败, 执行步骤 4
   4. 执行`addWorkerFailed`方法



#### addWorkerFailed

当 Worker 创建完成后, 如果其线程启动失败则会执行`addWorkerFailed`方法来对线程池做一个回滚操作

```java
private void addWorkerFailed(Worker w) {
  final ReentrantLock mainLock = this.mainLock;
  mainLock.lock();
  try {
    if (w != null)
      workers.remove(w);   // 从队列中删除 task
    decrementWorkerCount(); // 自减 workCount
    tryTerminate();    // 尝试终止线程池
  } finally {
    mainLock.unlock();
  }
}
```

##### 线程池终止

`tryTerminate` 大致逻辑是: 当线程池处于 `SHUTDOWN 且队列为空`或处于` STOP 且队列为空`时将线程池状态转变成`TERMINATED`. 如果可以终止,但 workerCount 不为 0 则中断一个空闲 Worker 保证传播关闭信号.



#### Worker.run

##### 描述

从上面的代码中我们看到了当Worker 新建成功后会调用其 `Thread.start`方法. 那 Runnable 是在何时传递给这个线程的呢?

查看 Worker 的构造函数得知:

```java
this.thread = getThreadFactory().newThread(this);
```

因为 Worker 本身继承了 AQS 且实现了 Runnable 接口, 所以当调用了 Thread.start 方法会执行 Worker.run 方法.



##### 代码

```java
public void run() {
    runWorker(this);
}
```

在 Worker.run 方法调用了 runWorker 方法



##### 运行 Worker

先来看看 runWorker 的代码, 简单梳理一下流程:

```java
final void runWorker(Worker w) {
  // step1: 获取Worker的thread和task, 并解锁
  Thread wt = Thread.currentThread();
  Runnable task = w.firstTask;
  w.firstTask = null;
  w.unlock(); // 疑问: 为什么要解锁?
  boolean completedAbruptly = true;
  try {
    // step2: 循环获取任务
    while (task != null || (task = getTask()) != null) {
      w.lock(); 
      // 线程池如果停止,保证线程中断,反之保证线程非中断
      if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() &&
					runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted()){
         wt.interrupt();  // 中断Worker线程
      }

      try {
        // step3: 执行前操作
        beforeExecute(wt, task);
        Throwable thrown = null;
        try {
          task.run();  // 调用Runnable.run方法
        } catch (RuntimeException x) {
          thrown = x; throw x;
        } catch (Error x) {
          thrown = x; throw x;
        } catch (Throwable x) {
          thrown = x; throw new Error(x);
        } finally {
          afterExecute(task, thrown);  // step4: 执行后操作
        }
      } finally {
        task = null;        // 将task的引用置为空, 方便回收
        w.completedTasks++;
        w.unlock();
      }
    }
    completedAbruptly = false;
  } finally {
    // setp5: 当 task.run 出现异常时执行, 销毁当前 Worker
    processWorkerExit(w, completedAbruptly);
  }
}
```



##### 执行流程

1. 获取 Worker 的线程, 并解锁(Worker 构造时已经加锁了), 初始 task 为空则调用 `getTask` 获取任务
2. task 执行前先加锁, 避免线程池状态更改时(`SHUTDOWN`), task 执行了
3. 调用 `beforeExecute` 前置方法
4. 执行 task, 是否出现异常, 出现异常执行最后一步
5. 执行完成调用`afterExecute`后置方法, 并对数据进行自增
6. 出现异常,则调用`processWorkerExit`方法回收当前 Worker



<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210305112210966.png" alt="image-20210305112210966"  />



#### `getTask`

在 Worker 的启动代码中知道了 Worker 是如何执行 task 的, 缺不太了解是如何获取 Task 的, 而获取 Task 则是调用 getTask 方法实现的.

##### 方法简介

该方法根据当前线程池的配置来设置阻塞或定时获取任务, 但出现以下一些情况则会返回 null:

1. 当 workerCount > maximumPoolSize , 即超过了最大线程数, 不能再创建 Worker 了
2. 线程池状态为 `STOP`
3. 线程池状态为 `SHUTDOWN` 或`队列为空`
4. Worker 等待 task 的时间超过了 `keepAliveTime`(workerCount > corePoolSize 的情况)



##### 源代码

```java
private Runnable getTask() {
  boolean timedOut = false; // Did the last poll() time out?

  for (;;) { // 自旋等待task
    int c = ctl.get();
    int rs = runStateOf(c);

    // 线程池处于 SHUTDOWN 且 (线程池处于 STOP 或队列为空)
    if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) { 
      decrementWorkerCount();  // 自减 workerCount, 并返回 null
      return null;  
    }

    int wc = workerCountOf(c);  // 获取 workerConut

		// allowCoreThreadTimeOut: 表示是否开启核心线程过期销毁
    boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

    // 如果超过了最大线程数 或 已超时
    if ((wc > maximumPoolSize || (timed && timedOut))
        && (wc > 1 || workQueue.isEmpty())) { // 队列为空
      if (compareAndDecrementWorkerCount(c)) // 自减 workerCount
        return null;
      continue;
    }

    try {
      // 根据当前 workerCount 来判断是否应该超时从队列中获取 task
      Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS):workQueue.take();
      if (r != null)
        return r;
      timedOut = true;   // 超时, 继续下一次自旋(最终会退出)
    } catch (InterruptedException retry) {
      timedOut = false;
    }
  }
}
```



##### 执行流程

1. 开启自旋, 判断线程池状态, 如果处于 SHUTDOWN 或 STOP 或 队列为空 则直接返回 null
2. 获取到 workerCount, 判断是否开启核心线程超时(`allowCoreThreadTimeOut`)
   1. 未开启则比较 workerCount > coolPoolSize 是否成立
   2. 开启则从队列获取 task 时为超时获取
3. 判断 workerCount > maximumPoolSize 和 队列为空 和 超时过(`timedOut`) 等条件是否成立
   1. 成立则修改 workerCount , 成功返回 null , 失败(其他 Worker 可能也在修改 workerCount)则继续自旋
4. 根据步骤2 的结果判断中队列中获取是`超时`获取还是`阻塞`获取, 如果是超时获取且结果为空, 则会进入下一次自旋再次执行步骤 1,2,3中的判断逻辑.



![image-20210305141210441](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210305141210441.png)



#### `processWorkerExit`

##### 简介

该方法用于销毁当前 Worker, 当 `getTask 返回 null`, 或 `workerCount > maximumPoolSize `就会销毁当前 Worker

##### 源代码

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
  // completedAbruptly: 标识当前 Worker 是否是因为task 执行异常而需要销毁的
  if (completedAbruptly)
    decrementWorkerCount();

  // step1: 统计所有 worker 完成的 task 数量
  final ReentrantLock mainLock = this.mainLock;
  mainLock.lock(); 
  try {
    completedTaskCount += w.completedTasks;  // 统计完成的 task 数量
    workers.remove(w); 			// 删除此 Worker 的引用
  } finally {
    mainLock.unlock();		
  }

  // step2: 尝试终止此 Worker
  tryTerminate();   		// 此方法在 addWorker 失败时调用的一致

  int c = ctl.get();
  if (runStateLessThan(c, STOP)) {   // 线程池未处于 STOP 状态
    if (!completedAbruptly) { // 条件成立, 说明不是因为异常导致此 Worker 销毁, 可能是队列没有任务
      int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
      if (min == 0 && ! workQueue.isEmpty())   // 如果队列不为空
        min = 1;
      if (workerCountOf(c) >= min)
        return; // replacement not needed
    }
    // 进入到这里有两个可能:
    // 1. completedAbruptly=false, 在 runWorker 中只有 while 循环条件不满足才会执行
    // 2. 上面的if (workerCountOf(c) >= min) 判断不成立, 即 workerCount<corePoolSize
    
    // 此时创建一个新的 Worker 来替换此 Worker
    addWorker(null, false);
  }
}
```



### 线程池关闭

线程池关闭有两个方法可以进行操作: `shutdown` `shutdownNow`

它两的区别是: 

- `shutdown` : 等待正在执行任务的 Worker 执行完成, 不接受新的 task 提交

- `shutdownNow` : 尝试停止所有正在执行的任务, 从队列中删除等待执行的 task 并返回
  - 此实现通过`Thread.interrupt`取消任务，因此任何无法响应中断的任务都可能永远不会终止。



### 动态参数配置

虽然网上有线程池配置的公式, 但是公司不一定适合所有场景, 因此线程池提供了动态修改线程池的方法.

![image-20210305152123194](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210305152123194.png)

其中我们比较关心的是核心线程, 最大线程, 队列大小的设置



#### 核心线程设置

##### 简介

核心线程的配置可以通过 `setCorePoolSize()` 来设置.

此方法用于设置核心线程数。 这将覆盖在构造函数中设置的任何值。 如果新值小于当前值，则多余的现有线程将在下次空闲时终止。 如果更大，将在需要时启动新线程以执行任何排队的任务。

##### 代码

```java
public void setCorePoolSize(int corePoolSize) {
  if (corePoolSize < 0)
    throw new IllegalArgumentException();
  // 计算核心线程差值, 利于后面判断
  int delta = corePoolSize - this.corePoolSize;
  this.corePoolSize = corePoolSize;  // 先赋值

  // workerCount 超过 corePoolSize
  if (workerCountOf(ctl.get()) > corePoolSize)
    interruptIdleWorkers(); // 中断Worker, 调用 Worker 中 Thread.interrupt 方法实现
  // workerCount 还未达到 corePoolSize
  else if (delta > 0) { 
    int k = Math.min(delta, workQueue.size()); 
		// 如果新 corePoolSize 还未达到 和 workQueueSize 至少有一个task, 则创建一个 Worker
    while (k-- > 0 && addWorker(null, true)) {
      if (workQueue.isEmpty())
        break;
    }
  }
}
```

##### 流程

1. 计算 `新corePoolSize` 和`旧corePoolSize` 的差值, 并修改线程池的 corePoolSize
2. 判断 workerCount 是否大于新 corePoolSize 是否成立, 成立则中断多余的 Worker, 反之继续执行
3. 判断是否需要新增 Worker(delta>0), 成立则开启自旋创建, 当 delta<=0 或队列为空时结束

![image-20210305172152265](https://better-io-blog.oss-cn-beijing.aliyuncs.com/image-20210305172152265.png)



#### 最大线程设置

##### 简介

设置允许的最大线程数。 这将覆盖在构造函数中设置的任何值。 如果新值小于当前值，则多余的现有线程将在下次空闲时终止。

##### 代码

```java
public void setMaximumPoolSize(int maximumPoolSize) {
  if (maximumPoolSize <= 0 || maximumPoolSize < corePoolSize)
    throw new IllegalArgumentException();
  this.maximumPoolSize = maximumPoolSize;
  // workerCount > 新maximumPoolSize, 则中断多余的 Worker
  if (workerCountOf(ctl.get()) > maximumPoolSize)
    interruptIdleWorkers();
}
```



#### 队列大小设置

线程池没有提供修改队列大小的方法, 当时提供了获取队列的方法: ` getQueue`, 该方法返回的类型为 `BlockingQueue`

常用的是 `LinkedBlockingQueue` 但是其 capacity 是 final 类型的, 不支持修改,  可以自行拷贝一份源代码,将其 `capacity` 修改成非 final 的, 并提供 `get set` 方法



## 总结

到此线程池的核心参数和以及动态调整, 实际场景中参数的配置可能是根据场景 QPS 进行变化的, 所以一般都会使用线程池监控, 来监控线程池的状态. 

本文主要偏向于源码的分析, 和对线程池执行流程的分析, 需要将整个流程串起来. 才能更好的理解线程池.

一些看法和理解如有错误,请指出, 多多交流. 



推荐文章:

[美团](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html) 

[博客园](https://www.cnblogs.com/thisiswhy/p/12690630.html)