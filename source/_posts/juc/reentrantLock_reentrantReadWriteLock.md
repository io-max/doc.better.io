---
title: ReentrantLock和ReentrantReadWriteLock源码分析
categories: Juc
index_img: img/java-logo-480.png
tags:
  - Java
  - Juc
abbrlink: 266
---

该文章主要讲解ReentrantReadWriteLock和ReentrantLock的实现以及源码分析

<!-- more -->

## ReentrantLock/ReentrantReadWriteLock



### 本章简介

> 本文主要讲解`ReentrantLock`和`ReentrantReadWriteLock`锁实现.
>
> - `ReentrantLock`如何实现的可重入锁?
> - `ReentrantReadWriteLock`是如何设计的?
> - `ReentrantReadWriteLock`读锁数量是如何记录的?
> - `ReentrantReadWriteLock`中读锁和写锁的最大数量是多少?



### Lock

`ReentrantLock`实现了`Lock`接口.

```java
public interface Lock {

  // 获取锁
  void lock();

  // 获取锁, 支持中断
  void lockInterruptibly() throws InterruptedException;

  // 尝试获取锁, 没有锁时, 直接返回
  boolean tryLock();

  // 尝试获取锁, 没有锁时, 超时指定时间后返回
  boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

  // 解锁
  void unlock();

  // 创建一个条件队列
  Condition newCondition();
}
```



### Sync

该类为`ReentrantLock`的内部类, 是一个抽象同步器,继承了`AQS`, 其两个子类`NonfairSync`和`FairSync`分别代表了`非公平锁`和`公平锁`.

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
  // 抽象获取锁方法
  abstract void lock();

  final boolean nonfairTryAcquire(int acquires) {
    // 获取当前线程
    final Thread current = Thread.currentThread();
    // 获取状态
    int c = getState();
    // 未持有锁
    if (c == 0) {
      // 直接cas更新
      if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
    // 已经持有锁, 重复获取锁(可重入)
    else if (current == getExclusiveOwnerThread()) {
      // 增加操作
      int nextc = c + acquires;
      if (nextc < 0) // overflow
        throw new Error("Maximum lock count exceeded");
      // 更新状态
      setState(nextc);
      return true;
    }
    return false;
  }

  // 释放锁流程
  protected final boolean tryRelease(int releases) {
    // 获取释放锁后的状态值
    int c = getState() - releases;
    // 调用该方法的线程和持有锁的线程不一致
    if (Thread.currentThread() != getExclusiveOwnerThread())
      throw new IllegalMonitorStateException();

    boolean free = false;
    // 只有c等于0, 才真正释放锁. 可重入.
    if (c == 0) {
      free = true;
      setExclusiveOwnerThread(null);
    }
    // 更新状态
    setState(c);
    return free;
  }
  // 忽略其他方法
}
```



#### NonfairSync

非公平锁实现. 即获取锁时不会立即排队, 而是会先尝试获取锁.

```java
static final class NonfairSync extends Sync {

  // 实现了父类Sync的方法
  final void lock() {
    // 直接cas修改状态, cas成功表明获取锁成功
    if (compareAndSetState(0, 1))
      // 直接设置当前线程
      setExclusiveOwnerThread(Thread.currentThread());
    // cas失败说明有现成已经抢到锁并修改了state
    else
      // 排队AQS提供的方法获取锁
      acquire(1);
  }

  protected final boolean tryAcquire(int acquires) {
    // 调用父类Sync的方法获取锁
    return nonfairTryAcquire(acquires);
  }
}
```



#### FairSync

公平锁实现, 每次获取锁时都会排队, 知道其他线程释放锁并唤醒此线程才会继续抢锁.

```java
static final class FairSync extends Sync {

  final void lock() {
    // 获取锁
    acquire(1);
  }

  protected final boolean tryAcquire(int acquires) {
    // 获取当前线程
    final Thread current = Thread.currentThread();
    // 拿到状态值
    int c = getState();
    // 没有获取到锁
    if (c == 0) {
      // hasQueuedPredecessors=false: 表示队列为空 或 当前线程已经在排队. 不能重复入队
      // 然后直接抢锁
      if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
    // 获取到了锁, 即锁重入
    else if (current == getExclusiveOwnerThread()) {
      int nextc = c + acquires;
      if (nextc < 0)
        throw new Error("Maximum lock count exceeded");
      setState(nextc);
      return true;
    }
    return false;
  }
}
```



通过对`state`状态值的操作来判断锁是否获取, 且把`state`的值当做锁重入的次数(`大于0`).



### 构造函数

```java
// 同步器, 有非公平和公平两种实现, 默认是非公平
private final Sync sync;

public ReentrantLock() {
  // 默认创建非公平锁
  sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
  // 根据参数创建锁
  sync = fair ? new FairSync() : new NonfairSync();
}
```



### 方法

```java
public void lock() {
  sync.lock();
}

public boolean tryLock() {
  return sync.nonfairTryAcquire(1);
}

public void unlock() {
  sync.release(1);
}
```

可以看到大部分方法都是调用的`Sync`类中的方法.



## ReentrantReadWriteLock

### 介绍

读写锁

### ReadWriteLock

此接口为`ReentrantReadWriteLock`实现的接口. 定义了获取`读锁`和`写锁`方法.

```java
public interface ReadWriteLock {
  // 读锁
  Lock readLock();

  // 获取写锁
  Lock writeLock();
}
```



### 构造器

```java
public ReentrantReadWriteLock() {
  // 默认非公平锁
  this(false);
}

public ReentrantReadWriteLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
  // 读锁和写锁共享同一个同步器
  readerLock = new ReadLock(this);
  writerLock = new WriteLock(this);
}
```



### 内部类图

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220613_ReentrantReadWriteLock.png" alt="image-20220613144753945" style="zoom:50%;" />

可以看到`ReentrantReadWriteLock`也有`公平`和`非公平`的两种模式. 



### FairSync/NonfairSync

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220613_read_write_sync.png" alt="image-20220613163449700" style="zoom:50%;" />

可以看出`公平`和`非公平`主要是针对于`写锁`. 两个子类没有具体的获取锁和释放锁的代码, 全部都在Sync中.



### WriteLock/ReadLock

写锁代码

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220613_writeLock.png" alt="image-20220613162543592" style="zoom:50%;" />



读锁代码

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220613_readLock.png" alt="image-20220613162709108" style="zoom:50%;" />

从代码中可以看出, 读锁和写锁的加锁和释放锁的操作都是调用的sync中的方法来完成的. 所以我们重点关注`Sync`类的实现.



### Sync

抽象同步器, 封装了获取/释放读锁和写锁相关的代码.

#### 核心变量

```java
// 共享位移量
static final int SHARED_SHIFT   = 16;
// 共享单位
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 低16位最大值 65535
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
// 低16位 标记(即最大的写锁值) 65535
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

// 线程持有的读锁计数器, 用于计算每个线程持有了多少个读锁.
private transient ThreadLocalHoldCounter readHolds;
// 持有计数器
private transient HoldCounter cachedHoldCounter;

// 第一个获取读锁的线程, 用于优化操作
private transient Thread firstReader = null;
// 第一个线程持有的读锁次数, 用于优化操作
private transient int firstReaderHoldCount;
```

从`SHARED_UNIT`和`EXCLUSIVE_MASK`等变量可以看出, 同步器使用了`高16位`来存储读锁数量, `低16位`来存储写锁数量. 且每个`读线程`持有的`读锁数量`有各自线程进行管理.



#### 构造函数

```java
Sync() {
    readHolds = new ThreadLocalHoldCounter();
    setState(getState()); // ensures visibility of readHolds
}
```



#### HoldCounter原理

该类的实例被每个线程持有, 由ThreadLocal来绑定.

```java
static final class HoldCounter {
  // 线程ID, 后续用于比对使用
  final long tid = getThreadId(Thread.currentThread());
  // 计数器
  int count = 0;
}
```



#### ThreadLocalHoldCounter原理

ThreadLocal子类, 用于初始化与线程绑定的`HoldCounter`.

```java
static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
    // 每个线程都会有一个HoldCounter.
    // 即各自线程 记自己可重入的次数
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

此类在Sync类初始化时被创建.



#### tryRelease方法原理

```java
protected final boolean tryRelease(int releases) {
  // 独占线程不是当前线程,抛出异常
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  // 计算state
  int nextc = getState() - releases;
  // 计算写锁数量
  boolean free = exclusiveCount(nextc) == 0;
  if (free)	// 重入次数还未到0
    setExclusiveOwnerThread(null);
  // 重新设置state
  setState(nextc);
  return free;
}
```

此方法用于写线程释放锁. 只有当重入的次数为0, 才会将持有锁线程置为空.



#### tryAcquire方法原理

```java
protected final boolean tryAcquire(int acquires) {
  // 获取当前线程
  Thread current = Thread.currentThread();
  int c = getState();
  // 写线程数量
  int w = exclusiveCount(c);
  // 此时有线程获取到了锁, 但是不知道是读锁还是写锁?
  if (c != 0) {
    // c != 0 and w == 0 表明一定有线程抢到了读锁(修改了state), 即此时不允许获取写锁
    // w != 0 and current != getExclusiveOwnerThread() 表明其他线程获取到了读锁
    if (w == 0 || current != getExclusiveOwnerThread())
      return false;   // 已经有其他线程获取到了读锁, 直接返回

    // w != 0 and current == getExclusiveOwnerThread() 表明其他线程获取到了读锁
    // w == 0 没有线程获取到读锁
    if (w + exclusiveCount(acquires) > MAX_COUNT)
      throw new Error("Maximum lock count exceeded");

    // 此时就是可重入获取锁, 直接修改数即可
    setState(c + acquires);
    return true;
  }
  // 此时没有线程获取到锁, 所以可以直接获取写锁
  // 在非公平模式下永远是false, 直接修改state变量
  if (writerShouldBlock() || !compareAndSetState(c, c + acquires))
    return false;
  // 设置线程
  setExclusiveOwnerThread(current);
  return true;
}
```



#### tryWriteLock方法原理

```java
final boolean tryWriteLock() {
  // 获取当前线程
  Thread current = Thread.currentThread();
  int c = getState();
	// 有线程持有锁
  if (c != 0) {
    // 写锁数量
    int w = exclusiveCount(c);
    // c != 0 and w == 0 表明有线程获取到了读锁, 此时不允许获取写锁
    if (w == 0 || current != getExclusiveOwnerThread())
      return false;
    if (w == MAX_COUNT)
      throw new Error("Maximum lock count exceeded");
  }
  if (!compareAndSetState(c, c + 1))
    return false;
  setExclusiveOwnerThread(current);
  return true;
}
```

此方法和`tryAcquire`方法差不多, 只不过少了`writerShouldBlock`方法的调用.  由于其通常被`tryLock`方法调用. 因此调用线程不用阻塞.



#### tryAcquireShared方法原理

```java
protected final int tryAcquireShared(int unused) {
  // 当前线程
  Thread current = Thread.currentThread();
  int c = getState();
  // 如果已经有线程持有写锁且持有锁线程不是当前线程. 不允许获取读锁
  if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current)
    return -1;

  // 读锁数量
  int r = sharedCount(c);

  // 
  if (!readerShouldBlock() && r < MAX_COUNT && compareAndSetState(c, c + SHARED_UNIT)) {
    // 当前还没有读锁, 则当前线程就是第一个读锁
    if (r == 0) {
      firstReader = current;
      firstReaderHoldCount = 1;
    }
    // 重复获取读锁
    else if (firstReader == current) {
      firstReaderHoldCount++; // 直接增加计数
    }
    // 非第一个获取锁的读锁线程
    else {
      HoldCounter rh = cachedHoldCounter;
      // 为空, 或者不是当前线程绑定的HoldCounter, 则直接初始化一个
      if (rh == null || rh.tid != getThreadId(current))
        cachedHoldCounter = rh = readHolds.get();
      // 更新当前线程绑定的HoldCounter
      else if (rh.count == 0)
        readHolds.set(rh);
      rh.count++;
    }
    return 1;
  }

  // cas失败 或 r > MAX_COUNT 或 readerShouldBlock=true
  return fullTryAcquireShared(current);
}
```



#### fullTryAcquireShared方法原理

```java
final int fullTryAcquireShared(Thread current) {
    // 执行到此方法的情况: cas修改state失败, readerShouldBlock=true即当前读线程应该阻塞
    HoldCounter rh = null;
    // 循环
    for (;;) {
        int c = getState();
        // 有线程持有写锁且非当前线程
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;  // 直接返回
        }
        // 没有线程持有写锁
        // 写锁是否应该被阻塞, 因为同步队列可能已经发生改变
        else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            }
            // 当前线程非首个线程
            else {
                // 操作每个线程的计数器
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                // 当前线程持有的读锁为0
                if (rh.count == 0)
                    return -1;
            }
        }
        // 到此的情况: 读锁不应该被阻塞

        // 判断最大读锁数
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");

        // cas更新状态
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            }
            else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```



从代码中可以看出此方法和`tryAcquireShared`方法代码类似, 只不过`fullTryAcquireShared`方法开启了一个循环, 不断的去获取锁.



可以将`tryAcquireShared`方法看做读锁的优化操作. 类似于提前尝试一下, 不行在开启循环.



#### tryReadLock方法原理

```java
final boolean tryReadLock() {
  // 当前线程
  Thread current = Thread.currentThread();
  // 循环
  for (;;) {
    int c = getState();
    // 有线程持有了写锁
    if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current)
      return false;
    
    int r = sharedCount(c);
    if (r == MAX_COUNT)
      throw new Error("Maximum lock count exceeded");
    
    // cas修改状态
    if (compareAndSetState(c, c + SHARED_UNIT)) {
      // cas成功后 修改计数器
      if (r == 0) {
        firstReader = current;
        firstReaderHoldCount = 1;
      } else if (firstReader == current) {
        firstReaderHoldCount++;
      } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
          cachedHoldCounter = rh = readHolds.get();
        else if (rh.count == 0)
          readHolds.set(rh);
        rh.count++;
      }
      return true;
    }
  }
}
```

`tryReadLock`方法相较于`fullTryAcquireShared`方法只是少了`readerShouldBlock`调用来判断当前读线程是都应该被阻塞; 由于此方法是在`tryLock`中调用, 调用线程不应该阻塞. 没有锁直接返回即可, 不用阻塞



#### tryReleaseShared方法原理

```java
protected final boolean tryReleaseShared(int unused) {
  // 获取到当前线程
  Thread current = Thread.currentThread();
  // 当前线程是第一个读取的线程
  if (firstReader == current) {
    // 对读数量进行减1操作
    if (firstReaderHoldCount == 1)
      firstReader = null;
    else
      firstReaderHoldCount--;
  }
  // 当前线程非第一个读取的线程
  else {
    HoldCounter rh = cachedHoldCounter;
    // 不为空且线程ID不等
    if (rh == null || rh.tid != getThreadId(current))
      // 获取当前线程的HoldCounter
      rh = readHolds.get();

    int count = rh.count;
    if (count <= 1) {
      // 解除当前线程的HoldCounter
      readHolds.remove();
      if (count <= 0)
        throw unmatchedUnlockException();
    }
    // 反之则count自减1
    --rh.count;
  }
  // 开启自旋
  for (;;) {
    int c = getState();
    int nextc = c - SHARED_UNIT;
    if (compareAndSetState(c, nextc))
      // 读锁和写锁是否都为0
      return nextc == 0;
  }
}
```



### 总结

`ReentrantReadWriteLock`巧妙的使用了`int`变量32位的设计, 将其拆分成两个16位, 低16位存储写锁, 高16位存储读锁. 

由于写锁是互斥的, 所以不用记录. 但是读锁是共享的, 所以需要知道线程获取读锁的次数. 

使用`HoldCounter`来记录每个线程获取读锁的次数, 且此类实例与获取读锁的线程绑定. 通过`ThreadLocalHoldCounter`对象实现.



读锁每次操作状态变量 都是以`SHARED_UNIT`变量为单位来进行操作. 