---
title: 线程池源码分析V1.0
categories: Juc
index_img: img/java-logo-480.png
tags:
  - Java
  - Juc
abbrlink: 44399
date: 2022-05-27 19:34:43
---

该文章主要讲解线程池在执行任务时的相关源码和关闭线程池的源码.

<!-- more -->

## 线程池源码分析

### Executor

> Executor作为ThreadPoolExecutor的顶级父类接口,仅仅只定义了执行任务的动作.

```java
public interface Executor {
  // 提交一个Runnable任务执行
  void execute(Runnable command);
}
```

``这里我们可以看出线程池可以执行Runnable及其子类实现的任务``.



### ExecutorService

> ExecutorService则定义了线程池的状态和一些管理方法, 例如线程池的



```java
public interface ExecutorService extends Executor {

  // 停止线程池, 会等待线程池中的任务完成
  void shutdown();

  // 立刻停止线程池, 返回线程池中为执行的任务, 该方法不会等待任务完成.
  List<Runnable> shutdownNow();
  
  // 线程池是否停止中
  boolean isShutdown();

  // 线程池是否终止
  boolean isTerminated();

  // 等待线程池终止(会处理完所有的任务)
  boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException;

  // 提交一个任务,只不过类型为callable
  <T> Future<T> submit(Callable<T> task);

  // 提交一个任务,返回一个具体的类型Future
  <T> Future<T> submit(Runnable task, T result);

  // 提交一个任务, 返回一个Future
  Future<?> submit(Runnable task);

  // 执行所有任务
  <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException;

	// 执行所有任务, 具有超时
  <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                long timeout, TimeUnit unit)
    throws InterruptedException;

  // 执行任迎春一个任务即可
  <T> T invokeAny(Collection<? extends Callable<T>> tasks)
    throws InterruptedException, ExecutionException;

  // 执行任迎春一个任务即可, 存在超时
  <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                  long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
}
```



> `该接口定义了反应线程池状态和向线程池提交任务的方法`
>
> 关于上面代码中的Future, 可以当成其是一个`代表任务执行的回调(接收通知)`



### AbstractExecutorService

> 此抽象类对ExecutorService中的部分方法进行了简单的实现. 
>
> 重点关注`submit`方法



```java
public Future<?> submit(Runnable task) {
  if (task == null) throw new NullPointerException();
  // 将Runnable类型的task封装成了RunnableFuture
  RunnableFuture<Void> ftask = newTaskFor(task, null);
  // 调用execute方法执行任务
  execute(ftask);
  // 返回Future
  return ftask;
}

// 同上
public <T> Future<T> submit(Callable<T> task) {
  if (task == null) throw new NullPointerException();
  RunnableFuture<T> ftask = newTaskFor(task);
  execute(ftask);
  return ftask;
}
```

`RunnableFuture是一个组合接口, 分别集成了Runnable和Future接口, 所以它包含了Runnable和Future所有的功能`



### 思考-线程池设计



看了上面的ExecutorService接口中的方法,我们可以知道,线程池中需要有个变量来表明线程池的状态.



> - `状态变量`,标识线程池的`状态`
> - `任务队列`,存储线程池执行的`任务`
> - `线程队列`,存储线程池执行任务的`线程`



### 核心变量



#### 状态

```java
// 整形值变量, 高4位控制线程池状态, 低28位控制线程数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
// 线程容量就是剩余的28位
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// 运行状态, 区别于其他状态, 该值为负数
private static final int RUNNING    = -1 << COUNT_BITS;
// 关闭状态, 就是 0
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 停止状态
// 0010 0000 0000 0000 0000 0000 0000 0000
private static final int STOP       =  1 << COUNT_BITS;
// 转换状态
// 0100 0000 0000 0000 0000 0000 0000 0000
private static final int TIDYING    =  2 << COUNT_BITS;
// 终止状态
// 0110 0000 0000 0000 0000 0000 0000 0000
private static final int TERMINATED =  3 << COUNT_BITS;

// ----------分割线----------

// 存储任务的队列
private final BlockingQueue<Runnable> workQueue;
// 存储执行任务的线程
private final HashSet<Worker> workers = new HashSet<Worker>();
// 线程工厂
private volatile ThreadFactory threadFactory;
// 拒绝策略
private volatile RejectedExecutionHandler handler;
// 空闲线程的存活时间
private volatile long keepAliveTime;
// 是否允许核心线程超时存活, 默认为false, 即核心线程空闲时也会存活
private volatile boolean allowCoreThreadTimeOut;
// 线程池核心线程数
private volatile int corePoolSize;
// 线程池最大线程数
private volatile int maximumPoolSize;
// 默认的拒绝策略, 即丢弃策略
private static final RejectedExecutionHandler defaultHandler =
  new AbortPolicy();
```

通过二进制的可以看出所有的状态都存储在`高四位`.

>  `使用一个原子性的整形值将线程池的运行状态和线程数量统一管理.`
>
> `整形值一共有32位,有用高四位来存储线程池的运行状态,低28位来存储线程数量`
>
> `RUNNING: 运行状态, 线程池可以接收并处理新的任务; `
>
> `SHUTDOWN:不接受新的任务,但是会处理队列中的任务; `
>
> `STOP: 停止状态, 不接受新任务, 不处理队列中的任务;`
>
> `TERMINATED: 终止状态;`
>
> `TIDYING:过渡状态,中断所有任务`
>
> 
>
> 从变量中可以看出, 使用了阻塞队列(`BlockingQueue`)来存储向线程池中提交的任务. 并且使用了`Worker`来对执行线程进行了封装. 



### Worker



```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {

  // 工作线程
  final Thread thread;
  // 初始化线程
  Runnable firstTask;
  // 完成的任务数量
  volatile long completedTasks;

  Worker(Runnable firstTask) {
		// 疑问: 为什么要设置同步状态?
    setState(-1);
    // 赋值
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
  }

  public void run() {
    // 运行Worker
    runWorker(this);
  }

  // 是否持有独占锁
  protected boolean isHeldExclusively() {
    return getState() != 0;
  }
	
  protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
      setExclusiveOwnerThread(Thread.currentThread());
      return true;
    }
    return false;
  }

  protected boolean tryRelease(int unused) {
    setExclusiveOwnerThread(null);
    setState(0);
    return true;
  }

  // 上锁和解锁等操作
  public void lock()        { acquire(1); }
  public boolean tryLock()  { return tryAcquire(1); }
  public void unlock()      { release(1); }
  public boolean isLocked() { return isHeldExclusively(); }
}
```



#### 疑问

- 为什么在Worker的构造函数中会设置同步状态为-1?
  - 避免线程还没启动就被中断, 无论是在shutdown或shutdownNow方法中设置中断都需要获取到锁.

- `runWorker`方法干了什么事?
  - 大致为开启循环, 不断调用getTask方法获取任务并执行.




### 核心构造函数

```java
// 核心构造函数
public ThreadPoolExecutor(int corePoolSize, // 核心线程数量
                          int maximumPoolSize, // 最大线程数量
                          long keepAliveTime,	// 空闲线程存活时间
                          TimeUnit unit, // 时间单位
                          BlockingQueue<Runnable> workQueue, // 存放任务的队列
                          ThreadFactory threadFactory,	// 线程工厂
                          RejectedExecutionHandler handler // 任务拒绝策略) {                     
	// 校验参数                          
  if (corePoolSize < 0 || maximumPoolSize <= 0 || maximumPoolSize < corePoolSize ||
      keepAliveTime < 0)
    throw new IllegalArgumentException();

  if (workQueue == null || threadFactory == null || handler == null)
    throw new NullPointerException();
	
  this.acc = System.getSecurityManager() == null ? null : AccessController.getContext();
	// 简单的变量赋值 
  this.corePoolSize = corePoolSize;
  this.maximumPoolSize = maximumPoolSize;
  this.workQueue = workQueue;
  this.keepAliveTime = unit.toNanos(keepAliveTime);
  this.threadFactory = threadFactory;
  this.handler = handler;
}
```



> 其他构造函数最终都是调用了此函数来初始化线程池.



### 线程池执行



在抽象父类`AbstractExecutorService`中可以通过`submit`向线程池中提交任务执行. 而在`submit`方法最终调用了`execute`方法来执行任务.



在了解execute方法之前, 先来看几个前置方法.



```java
// CAPACITY 取反就是高四位
// 查看线程池运行状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 查看线程数量
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```



#### execute 方法原理

```java
public void execute(Runnable command) {
  if (command == null)
    throw new NullPointerException();

  int c = ctl.get();
  // 如果工作线程小于核心线程
  if (workerCountOf(c) < corePoolSize) {
    // 则添加一个工作线程
    if (addWorker(command, true))
      return;
    c = ctl.get();
  }
  // 走到这步说明: 工作线程大于核心线程, 或者添加Worker失败
  if (isRunning(c) && workQueue.offer(command)) {   // 线程池处于运行状态且任务入队成功
    
    int recheck = ctl.get();
    
    // 再次检查线程池状态, 因为上面的操作执行完,可能有人调用了shutdown方法
    if (! isRunning(recheck) && remove(command))
      reject(command); // 拒绝任务

    // 到这里有两种状况: 
    // 1. 线程池是运行状态 
    // 2. 线程池非运行状态且remove方法执行失败
    else if (workerCountOf(recheck) == 0)    // 如果工作线程为0
      addWorker(null, false);       // 疑问? 工作线程为0, 为什么还要创建一个线程
  }
  // 到此出现的情况: 线程池非运行状态或者任务入队失败
  // 再次尝试入队(此时线程池可能处于shutdown或stop状态或队列满了)
  else if (!addWorker(command, false))
    reject(command);
}
```

根据源码分析可得出流程如下:

> - 第一步: 先判断工作线程是否大于核心线程数
>   - 小于: 则添加一个Worker(`核心线程`)
>     - 添加成功, 直接返回
>     - 添加失败, 执行第二步
>   - 大于, 执行第二步
> - 第二步: 判断线程池是否处于运行状态
>   - 运行: 调用队列的`offer`方法将任务入队
>     - 入队成功: 再次检查线程池状态(此时线程池可能关闭), 并添加一个Worker(`非核心线程`)
>     - 入队失败: 执行第三步
>   - 其他状态: 执行第三步
> - 第三步: 再次尝试添加Worker, 此时线程池可能处于shutdown或stop状态 或者任务队列满了
>   - 添加Worker成功: 返回 (`非核心线程`)
>   - 添加Worker失败: 拒绝任务





#### addWorker方法原理



```java
private boolean addWorker(Runnable firstTask, boolean core) {
  retry:
  for (;;) {
    int c = ctl.get();  
    int rs = runStateOf(c); // 获取运行状态

		// 终止条件
    // 线程池非运行状态(SHUTDOWN, STOP...)
    if (rs >= SHUTDOWN && 
        // 注意此条件, 当此条件返回true, 需要注意到任务队列不为空, 这里和processWorkerExit方法最后的
        // addWorker 相呼应
        ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()) 
    )
      return false;

    // 自旋
    for (;;) {

      int wc = workerCountOf(c);												 // 获取工作线程数量
      if (wc >= CAPACITY ||															 // 工作线程数大于容量
          wc >= (core ? corePoolSize : maximumPoolSize)) // 大于核心线程或最大线程
        return false;
      
      if (compareAndIncrementWorkerCount(c)) 						 // cas增加工作线程数
        break retry; // 结束

      c = ctl.get();  																	 // cas失败, 其他现在也在提交任务

      if (runStateOf(c) != rs) 													 // 线程池状态以改变, 重新整个大循环
        continue retry;
    }
  }
  
  // 工作线程数修改成功,但是 Worker 可能新增失败, 怎么办?

  boolean workerStarted = false;
  boolean workerAdded = false;
  Worker w = null;
  try {

    w = new Worker(firstTask); // 创建Worker对象
    final Thread t = w.thread;

    if (t != null) {
      final ReentrantLock mainLock = this.mainLock; 
      mainLock.lock();
      try {
       
        int rs = runStateOf(ctl.get());

        // 出现两种情况:
				// 1. 线程池处于 RUNNING 或 SHUTDOWN 状态
        // 2. 如果线程池处于SHUTDOWN 且 firstTask为空
        if (rs < SHUTDOWN ||  (rs == SHUTDOWN && firstTask == null)) { 
          if (t.isAlive()) // 线程是否存活
            throw new IllegalThreadStateException();

          workers.add(w); // 添加Worker的引用
          int s = workers.size();
          if (s > largestPoolSize)	// 修改最大线程池数量值
            largestPoolSize = s;
          workerAdded = true;
        }
      } finally {
        mainLock.unlock();
      }
      if (workerAdded) {	// 添加成功, 启动线程
        t.start();
        workerStarted = true;
      }
    }
  } finally {
    if (! workerStarted)
      addWorkerFailed(w);
  }
  return workerStarted;
}
```

addWorker的代码大致分成了两步骤:

1. 通过`cas+自旋`来修改`workCount`的数值.
2. 创建Worker对象, 并启动`Worker`中的`thread`.



但是启动线程的代码由一个`workerAdded`变量控制, 而只有满足`if (rs < SHUTDOWN ||  (rs == SHUTDOWN && firstTask == null))`条件才会将其置为`true`.

满足此条件只会有两种情况:

1. 线程池处于运行状态(正常情况)
2. 线程池处于SHUTDOWN状态, 且`firstTask==null`, 说明此时线程池正在关闭(异常情况)

代码如下:

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220525_thread_pool.png" alt="image-20220525111316849" style="zoom:50%;" />





前提是线程池状态必须是运行状态.

Worker自己本身就是一个Runnable, 所以在线程启动时会调用Worker类中的run方法,而在run方法中调用了`runWorker`方法

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220524154640.png" alt="image-20220524154640310" style="zoom:50%;" />



#### runWorker方法原理

执行Worker, 注意此时Worker中的线程已经启动



```java
final void runWorker(Worker w) {
  Thread wt = Thread.currentThread();
  Runnable task = w.firstTask;
  w.firstTask = null;
  w.unlock(); // 允许中断? 疑问: 中断在哪里体现? 目前来看不知道.(可能在线程池关闭)
  boolean completedAbruptly = true;	 // 记录线程执行是否出现异常
  try {
    while (task != null || (task = getTask()) != null) {
      w.lock(); // 获取锁
      // 线程池状态是否是TERMINATED或TIDYING
      if ((runStateAtLeast(ctl.get(), STOP) ||
           // 线程是否设置中断标示位
        (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted())
        wt.interrupt();	 // 中断线程, 

      try {
        
        beforeExecute(wt, task); // 前置钩子函数
        Throwable thrown = null;
        try {
          task.run();	// 执行task
        } catch (RuntimeException x) {
          thrown = x; throw x;
        } catch (Error x) {
          thrown = x; throw x;
        } catch (Throwable x) {
          thrown = x; throw new Error(x);
        } finally {
          afterExecute(task, thrown); // 后置钩子函数
        }
      } finally {
        task = null;
        w.completedTasks++;
        w.unlock();
      }
    }
    // 当从队列中获取的task==null时, 此变量才会被修改. 而当task.run出现异常时, 此变量不会被修改
    completedAbruptly = false;
  } finally {
    processWorkerExit(w, completedAbruptly); // 处理工作线程退出
  }
}
```



Worker工作线程内容比较简单, 开启循环不断调用`getTask`方法从任务队列中获取任务, 如果`getTask`返回null说明当前无任务执行, 终止循环将调用`processWorkerExit`方法处理.



注意`completedAbruptly`变量, 此变量默认为true, 而只有当while循环正常结束时才会将其改为false, 然而当`task.run`运行出现异常异常时, 会直接跳过此修改, 直接走到`processWorkerExit`方法.

`所以completedAbruptly变量就是用来标识用户的任务是否会出现异常`.



#### getTask方法原理

该方法用于从线程池中获取任务执行

```java
private Runnable getTask() {
  boolean timedOut = false; // Did the last poll() time out?

  // 启动自旋, 获取到任务结束
  for (;;) {
    int c = ctl.get();
    int rs = runStateOf(c);

    // 条件成立的情况: 1. 线程池状态处于STOP或TERMINATED状态且任务队列为空
    if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
      decrementWorkerCount();
      return null;
    }

    // 上面条件不成立的情况: 1. 线程池处于RUNNING 2. 线程池处于SHUTDOWN且任务队列不为空
    // 此种情况用于处理当调用shutdown方法后保留线程继续处理任务队列中的任务
    int wc = workerCountOf(c);

    // 假设allowCoreThreadTimeOut=false, 则此条件有wc变量控制
    // timed=true: 工作线程数大于核心线程数
    // timed=false: 工作线程数小于核心线程数
    boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

    // 条件成立的情况: 
    // (工作线程数大于核心线程数 或 非核心线程且此线程获取任务超时过) 
    // 且 
    // (任务队列为空或工作线程数大于1)
    if ((wc > maximumPoolSize || (timed && timedOut))
        && (wc > 1 || workQueue.isEmpty())) {
      if (compareAndDecrementWorkerCount(c))
        return null;
      continue;
    }

    // timed=true: 说明此时Worker为核心线程且会阻塞在任务队列上, 反之则为非核心线程.
    try {
      Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
      workQueue.take();
      if (r != null)
        return r;
      // 到此表示任务获取超时且任务为空, 此Worker空闲了.
      timedOut = true;
    } 
		// 响应中断异常, shutdown方法或shutdownNow方法, 此时线程必定阻塞在任务队列中
    // 此时任务队列一定为空.则getTask循环将会终止返回null
    catch (InterruptedException retry) {      
      timedOut = false;
    }
  }
}
```



`getTask返回null`的情况有两种:

1. 任务队列中没有任务
2. 线程池调用了shutdown方法或shutdownNow方法.



在最后的代码中捕捉了`InterruptedException`异常, 这是因为在调用`shutdown`方法时, 会设置线程的中断标志位.代码会进行下一次循环, 但是此时线程池的状态已经为`SHUTDOWN`状态, 所以`getTask`方法会直接返回null, 调用`processWorkerExit`方法处理worker退出的逻辑.



`核心线程会阻塞在阻塞队列中.`



#### processWorkerExit方法原理

注意此方法由执行Worker的线程调用.

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
  if (completedAbruptly)		// 任务执行出现了异常
    decrementWorkerCount();	// 减少工作线程数量

  final ReentrantLock mainLock = this.mainLock;
  mainLock.lock();
  try {
    completedTaskCount += w.completedTasks;	// 计算任务数量
    workers.remove(w);											// 删除Worker引用
  } finally {
    mainLock.unlock();
  }

  // 尝试终止
  tryTerminate();

  int c = ctl.get();
  if (runStateLessThan(c, STOP)) {	// 线程池处于 SHUTDOWN 或 RUNNING 状态
    if (!completedAbruptly) {		// 此条件成立, 说明队列中无任务, 此时Worker已经空闲
      int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
      // allowCoreThreadTimeOut=true 此条件才有可能成立
      // 需要保留一个线程来处理队列中的任务.
      if (min == 0 && ! workQueue.isEmpty())
        min = 1;
      if (workerCountOf(c) >= min)		// 工作线程数还充足(最坏的情况是只有一个工作线程)
        return; 
    }
    // 执行任务出现异常(completedAbruptly=true)或者工作线程为0(workerCountOf(c) >= min -> false)
    // 由于上面workers已经删除对其的引用, 所以此时这个Worker会在下次垃圾回收时被回收掉
    // 此时需要添加一个新的工作线程
    addWorker(null, false);						
  }
}
```



在此方法处理中, 知道了在开启了`所有线程超时(allowCoreThreadTimeOut=true)`和`关闭线程池(状态为SHUTDOWN)`时需要至少保留一个线程来处理队列中的任务.



#### 总结

线程池中线程执行任务的线程由Worker对象进行封装, 每个线程都会开启一个循环来从任务队列中获取任务执行,

当执行任务出现异常时, 此时Worker



可以看出在`addWorker.addWorkerFailed`和`runWorker.processWorkerExit`方法中都会调用`tryTerminate`方法.

在`addWorker`方法中会调用`addWorkerFailed`方法来处理添加Worker失败的情况. 同时在runWorker方法中也会调用`processWorkerExit`来处理Worker空闲退出线程池的问题.



### 线程池终止

线程池终止的方法有两个, 分别是: `shutdown` / `shutdownNow`. 它俩的区别是 一个是停止线程池, 但是会继续处理任务队列中的任务, 另一个是停止线程池, 不会处理任务队列中的任务.



#### shutdown方法原理

```java
public void shutdown() {
  final ReentrantLock mainLock = this.mainLock;
  mainLock.lock();
  try {
    checkShutdownAccess();		// 检查权限
    advanceRunState(SHUTDOWN);	// 修改线程池状态为SHUTDOWN
    interruptIdleWorkers();		// 中断Worker中的
    onShutdown(); // 钩子函数
  } finally {
    mainLock.unlock();
  }
  // 尝试终止
  tryTerminate();
}

private void advanceRunState(int targetState) {
  for (;;) {
    int c = ctl.get();
    if (runStateAtLeast(c, targetState) || // 线程池非SHUTDOWN, 则cas成SHUTDOWN
        ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
      break;
  }
}

private void interruptIdleWorkers() {
  interruptIdleWorkers(false);
}

private void interruptIdleWorkers(boolean onlyOne) {
  final ReentrantLock mainLock = this.mainLock;
  mainLock.lock();
  try {
    for (Worker w : workers) {	// 遍历所有的Worker
      Thread t = w.thread;	
      // 线程没有中断过且Worker此时未工作(Worker在执行task时会获取锁)
      if (!t.isInterrupted() && w.tryLock()) {	
        try {
          t.interrupt(); 	// 设置中断标志位
        } catch (SecurityException ignore) {
        } finally {
          w.unlock();	// 解锁
        }
      }
      if (onlyOne)
        break;
    }
  } finally {
    mainLock.unlock();
  }
}
```

`shutdown`方法会修改线程池的状态为SHUTDOWN, 且会中断所有的空闲线程(即没有执行任务的线程). 此时addWorker执行将不会被允许, 即不会添加新的线程. 此时只需要把任务队列中的任务执行完即可.

此时线程池的状况如下: 

1. 任务队列没有任务, 存在阻塞的线程, 此时这些线程响应中断. 最终getTask会返回null, 最终调用runWorker中的processWorkerExit销毁此Worker
2. 任务队列中有任务, 不存在阻塞的线程, 此时线程正常获取任务, 下个循环再次调用getTask获取任务, 此时任务队列中就没有任务了, 此时线程会响应中断 抛出异常, 再次循环.getTask返回null

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220526_getTask.png" alt="image-20220526110855543" style="zoom:50%;" />

详情参考`getTask`方法.



#### shutdownNow方法原理

```java
public List<Runnable> shutdownNow() {
  List<Runnable> tasks;
  final ReentrantLock mainLock = this.mainLock;		// 获取锁
  mainLock.lock();
  try {
    checkShutdownAccess();	// 同上
    advanceRunState(STOP);	// 修改线程池状态为STOP
    interruptWorkers();			// 中断线程
    tasks = drainQueue();		// 获取队列中未执行的任务
  } finally {
    mainLock.unlock();
  }
  tryTerminate();						// 尝试终止
  return tasks;
}

private void interruptWorkers() {
  final ReentrantLock mainLock = this.mainLock;
  mainLock.lock();
  try {
    for (Worker w : workers)	
      w.interruptIfStarted();		// 设置中断位, 由Worker内部实现
  } finally {
    mainLock.unlock();
  }
}
```

shutdownNow的方法实现和shutdown方法差不多, 只是将线程池的状态改为了STOP状态, 同时该方法会中断正在执行任务的线程,通过Worker内部实现的`interruptIfStarted`方法设置中断标志位.

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220526_worker_interrupt.png" alt="image-20220526111455759" style="zoom:50%;" />

而被设置中断标志位的线程在执行完当前任务后, 再次调用`getTask`方法获取任务时将会得到null. 此时Worker的退出流程和上述一致.



#### tryTerminate方法原理

此方法在多个方法中都会被调用,所以会存在线程安全问题.

```java
final void tryTerminate() {
  for (;;) {
    int c = ctl.get();

    // 条件成立的情况: 1. RUNNING状态 2.TERMINATED状态 3.SHUTDOWN状态队列不为空(队列中的任务需要处理)
    if (isRunning(c) ||
        runStateAtLeast(c, TIDYING) ||
        (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
      return;

    // 到这里 线程池状态为STOP或队列为空, 当时还有线程在执行任务
    if (workerCountOf(c) != 0) {
      interruptIdleWorkers(ONLY_ONE); // 中断一个空闲线程
      return;
    }

    // 到这里说明工作线程已经没有,即wc==0, 此时可以将状态设置为TIDYING
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
      // cas将线程池状态修改为TIDYING, 标识线程池在向TERMINATED状态转换
      if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
        try {
          terminated();
        } finally {
          // 设置线程池状态为终止状态
          ctl.set(ctlOf(TERMINATED, 0));
          termination.signalAll();    // 唤醒所有等待线程池终止的线程
        }
        return;
      }
    } finally {
      mainLock.unlock();
    }
  }
}
```



#### 总结

关于线程池的关闭一共有两个方法, 分别是shutdown和shutdownNow. 而这两者的区别就是一个会执行任务队列中的任务, 另一个则不会执行. 

两个方法都是通过设置中断标志位来达到关闭Worker的目的. 而对中断异常的捕捉实在`getTask`方法, 或者是用户自行提交的任务中处理. 