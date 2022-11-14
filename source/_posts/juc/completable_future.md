---
title: CompletableFuture源码分析
categories: Juc
index_img: img/java-logo-480.png
tags:
  - Java
  - Juc
abbrlink: 49209
---

该文章主要讲解CompletableFuture的设计以及源码分析

<!-- more -->


## CompletableFuture源码分析



> 本文主要讲解CompletableFuture的设计以及源码分析
>
> - 是如何唤醒的后续Future?





### CompletionStage

一个`CompletionStage`代表了一个可异步计算的`阶段`. 在另一个`CompletionStage`完成时执行一个操作或者计算一个值.



一个`CompletionStage`的执行可以由另一个单独阶段完成后触发, 也可以由两个阶段都完成触发, 或者两个阶段中任意一个完成后触发. 



### CompletableFuture整体设计

![image-20220616171712930](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220616171712.png)

简易版

![image-20220616171528593](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220616171528.png)



### CompletableFuture

![image-20220614164755597](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220614_completableFuture.png)

一个CompletableFuture实现了`Future`和`CompletionStage`两个接口.

`所以每个CompletableFuture都是一个阶段`

核心变量

```java
// 运行结果
volatile Object result;
// 存储后续执行动作的栈
volatile Completion stack;
```



#### Completion

一个Completion是一个特殊的FutureTask, 可以由线程池提交



```java
abstract static class Completion extends ForkJoinTask<Void> 
  implements Runnable, AsynchronousCompletionTask {
  
  // 下一个Completion, 即Completion可以组成栈
  volatile Completion next;      

  // 如果触发则执行Completion的动作, 并返回一个可能需要传播的阶段(如果存在的话)
  // 触发的模式有三种: SYNC=同步, ASYNC=异步, NESTED=嵌套
  abstract CompletableFuture<?> tryFire(int mode);

	// 当前Completion是否能继续触发
  abstract boolean isLive();

  // 执行当前Completion, 提交到线程池中执行, 所以模式是异步
  public final void run()                { tryFire(ASYNC); }

  public final boolean exec()            { tryFire(ASYNC); return true; }

}
```



#### UniCompletion

```java
abstract static class UniCompletion<T,V> extends Completion {
  // 执行此Completion的线程池
  Executor executor;        
  // 下个阶段的CF
  CompletableFuture<V> dep; 
  // 上个阶段的CF
  CompletableFuture<T> src; 

  UniCompletion(Executor executor, CompletableFuture<V> dep,CompletableFuture<T> src) {
    this.executor = executor; 
    this.dep = dep; 
    this.src = src;
  }

  // 当前Completion能否执行
  final boolean claim() {
    Executor e = executor;
    // 设置Task的标志位
    if (compareAndSetForkJoinTaskTag((short)0, (short)1)) {
      if (e == null)
        return true;
      executor = null; // disable
      // 向线程池中提交当前Completion
      e.execute(this);
    }
    return false;
  }
	// 当前Completion是否关联下个阶段
  final boolean isLive() { return dep != null; }
}
```

UniCompletion子类如图:

<img src="https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220616174008.png" alt="image-20220616174008226" style="zoom:50%;" />

除去`AsyncSupply`和`AsyncRun`两个类其他类都是UniCompletion的子类.



### 测试代码

```java
public static void main(String[] args) throws Exception {        
  Integer result = CompletableFuture                            
    .supplyAsync(() -> 10)                        // step1      
    .thenApplyAsync(res -> res + 1)               // step2                                  
    .thenApplyAsync(res -> res + 2).get();        // step3                           
  System.out.println(result);    
}
```

此代码等效于

```java
public static void main(String[] args) throws Exception {        
  // step1  
  CompletableFuture<Integer> cf1 = CompletableFuture.supplyAsync(() -> 10);
  // step2
  CompletableFuture<Integer> cf2 = cf1.thenApplyAsync(res -> res + 1);
  // step3
  CompletableFuture<Integer> cf3 = cf2.thenApplyAsync(res -> res + 2);
  System.out.println(cf3.get());
}
```



### 静态方法

在CompletableFuture中可以使用一些静态方法来快捷的创建CompletableFuture.

`supplyAsync`方法

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
  // 包装步骤
  return asyncSupplyStage(asyncPool, supplier);
}

public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor) {
  
  return asyncSupplyStage(screenExecutor(executor), supplier);
}
```

`runAsync`方法

```java
public static CompletableFuture<Void> runAsync(Runnable runnable) {
  return asyncRunStage(asyncPool, runnable);
}
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor) {
  return asyncRunStage(screenExecutor(executor), runnable);
}
```



这两个方法的区别就是: `runAsync`返回的CompletableFuture没有结果, 即`get`返回null. 而`supplyAsync`方法则由返回值.



#### asyncSupplyStage方法原理

```java
static <U> CompletableFuture<U> asyncSupplyStage(Executor e, Supplier<U> f) {

  if (f == null) throw new NullPointerException();
  // 新建一个 CompletableFuture
  CompletableFuture<U> d = new CompletableFuture<U>();
  // 向线程池中提交一个任务, 注意此任务类型是 AsyncSupply
  e.execute(new AsyncSupply<U>(d, f));
  // 返回新创建的 CompletableFuture
  return d;
}
```

#### asyncRunStage方法原理

```java
static CompletableFuture<Void> asyncRunStage(Executor e, Runnable f) {
  if (f == null) throw new NullPointerException();
  // 新建一个CompletableFuture
  CompletableFuture<Void> d = new CompletableFuture<Void>();
  // 提交一个 AsyncRun 的任务执行
  e.execute(new AsyncRun(d, f));
  return d;
}
```

通过对两个方法的解析, 可以看看出代码的流程是一致的, 只是最终想线程池中提交的任务类型不一样.

`asyncSupplyStage ==> AsyncSupply` 而 `asyncRunStage ==> AsyncRun`.



查看AsyncSupply和AsyncRun两个内部类的`run`方法.

`AsyncSupply.run`代码如下

![image-20220617105321703](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220617_async_apply_run.png)



`AsyncRun.run`代码如下

![image-20220617105620095](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220617_async_run.png)

通过代码可以看出AsyncRun和AsyncSupply两个类的区别:

- 都存在`fn`字段, 用于存储当前阶段执行的动作, 但是一个是Supplier类, 一个是Runnable类型.

两个类的run方法在最后都调用了`CompletableFuture.postComplete()`方法.





在`supplyAsync()`或`runAsync()`执行完后会返回一个`CF`. 在其内部将需要执行的操作封装成了`AsyncSupply`或``AsyncRun`并与`代表当前阶段的CF`进行关联. 此`AsyncSupply`和`AsyncRun`都是一个`FutureTask`可以被线程池执行.

执行流程大致为: 

- 将执行的动作(`Supplier`或`Runnable`)和对应的`CF`通过`AsyncSupply`或`AsyncRun`进行封装
- 将`AsyncSupply`或`AsyncRun`并扔到线程池中执行. 
- 而`AsyncSupply`或`AsyncRun`通过判断`CF.result==null`是否完成. 为空执行动作,反之则设置结果.
- 最终调用`postComplete`方法唤醒后续依赖的CF.





### 实例方法

#### thenApplyAsync方法原理

```java
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn) {
  // 封装UniApply对象
  return uniApplyStage(asyncPool, fn);
}

```



##### uniApplyStage方法原理

此方法每次调用都会返回一个的CF, 但是每次调用的对象是不同的. 

执行`step2`时是调用的`CF1.uniApplyStage`, 而在执行`step3`时是调用的`CF2.uniApplyStage`.

`所以我们分析此段源码时 需要注意this所指向的对象`.

```java
private <V> CompletableFuture<V> uniApplyStage(Executor e, 
                                               Function<? super T,? extends V> action) {
  
  if (action == null) throw new NullPointerException();
  // 假设step2创建的CF2, step3创建的是CF3
  CompletableFuture<V> cf =  new CompletableFuture<V>();

  // cf.uniApply 方法可以简单理解成执行一个action参数的动作, 后面会分析.
  // step2执行时, this == CF1; step3执行时, this == CF2
  if (e != null || !cf.uniApply(this, action, null)) {
    // 创建Completion对象关联当前阶段CF和下阶段CF
    UniApply<T,V> c = new UniApply<T,V>(e, cf, this, action);
    // 入栈, 每个CF都有一个stack变量存储Completion对象
    push(c);
    // 尝试触发
    // 疑问: 为什么会有同步和异步? 因为UniApply的父类Completion实现了Runnable接口
    c.tryFire(SYNC);
  }
  return cf;
}
```

看完这段代码可能非常疑惑.

- uniApply干了什么?
- push方法做了什么?
- tryFire干了什么?
- UniApply是用于干什么的?



但是大致流程是: 

- 创建了一个`UniApply`对象绑定了`新CF(cf对象)`和`当前CF(this)`同时还有新CF执行的`动作(action参数)`
  - 此步骤体现: `new UniApply<T,V>(e, cf, this, action);`
- 然后将`UniApply`对象推送到`当前CF`的某个`地方`
  - 此步骤体现: `push(c);`
- 最后尝试触发`UniApply`对象.
  - 此步骤体现: `c.tryFire(SYNC);`



##### uniApply方法原理

step2执行时调用此方法的CF对象为: `CF2`. 所以参数a表示的是CF1.

```java
final <S> boolean uniApply(CompletableFuture<S> a, Function<? super S,? extends T> f,
                           UniApply<S,T> c) {
  Object r; Throwable x;
  // 依赖的上一个CF还未执行完, 直接返回
  if (a == null || (r = a.result) == null || f == null)
    return false;

  // 当前CF结果为空
  tryComplete: if (result == null) {
    // 依赖的上一个CF执行出现异常, 则当前CF也设置异常
    if (r instanceof AltResult) {
      if ((x = ((AltResult)r).ex) != null) {
        completeThrowable(x, r);
        break tryComplete;
      }
      r = null;
    }
    // 依赖的上一个CF正常结束
    try {
      if (c != null && !c.claim())
        return false;
      @SuppressWarnings("unchecked") S s = (S) r;
      // 当前CF能执行则直接执行
      completeValue(f.apply(s));
    } catch (Throwable ex) {
      completeThrowable(ex);
    }
  }
  return true;
}
```

`可以看出此方法就是想尝试的执行一下当前CF的动作.` 

如果依赖的上一个CF还未执行完或出现异常, 则当前CF的动作不会执行, 会直接返回.



##### push/tryPushStack方法原理

```java
final void push(UniCompletion<?,?> c) {
  if (c != null) {
    while (result == null && !tryPushStack(c))
      lazySetNext(c, null); // clear on failure
  }
}

final boolean tryPushStack(Completion c) {
	// 获取当前CF的栈顶对象
  Completion h = stack;
  // 将c.next 指针指向了当前 Completion.stack
  lazySetNext(c, h);
  // 将当前 CF.stack 更新成 c
  return UNSAFE.compareAndSwapObject(this, STACK, h, c);
}
```

简而言之push方法就是将传递的Completion压入当前CF的stack变量的顶部.



当上述测试代码执行完, CF的执行结构如下图:

![image-20220616154723907](https://better-io-blog.oss-cn-beijing.aliyuncs.com/20220616154723.png)



### 唤醒操作

唤醒操作由大致有两类: 一类是CompletableFuture的唤醒, 一类是Completion的唤醒



通过整体的代码分析, 得知了触发后续CF的方法目前有两个:  `tryFire`和`postComplete`.



#### postComplete方法原理

该方法属于CompletableFuture

```java
final void postComplete() {
  
  CompletableFuture<?> src = this;
  // 当前CF的栈顶对象
  Completion completion;
  
  // 当step1执行时, src就是CF1
  while ((completion = src.stack) != null 
         || (src != this && (completion = (src = this).stack) != null)) {

    CompletableFuture<?> dep; Completion nextCompletion;

    // 将当前CF的栈顶Completion出栈
    if (src.casStack(completion, nextCompletion = completion.next)) {
      // 不为空, 说明当前CF调用了多次thenXXX, 生成了多个Completion
      if (nextCompletion != null) {
        // src已经变化
        if (src != this) {
          // 将completion压入当前阶段CF的栈顶
          pushStack(completion);
          continue;
        }
        // 便于垃圾回收
        completion.next = null;    // detach
      }
      // Completion.tryFire 返回代表下个阶段的CF, 所以每次循环
      // NESTED: 表示内嵌触发, 即在一个CF中触发另一个CF
      src = (dep = completion.tryFire(NESTED)) == null ? this : dep;
    }
  }
}
```

使用上面的测试代码分析流程: 

第一次循环: src = CF1. while循环第一个条件满足, 此时将CF1的栈顶Completion出栈. 由于栈中只有一个Completion所以`nextCompletion != null`不成立.

通过Completion的tryFire方法唤醒与此绑定的后续阶段(CF2). 然后dep!=null, 返回dep, 此时src = CF2

第二次循环: src = CF2, while循环第一个条件满足, 此时将CF2的栈顶Completion出栈, 同样的`nextCompletion != null`不成立.

通过Completion的tryFire方法唤醒与此绑定的后续阶段(CF3). 然后dep!=null, 返回dep, 此时src = CF3

第三次循环: src=CF3, while循环第一个条件不满足(CF3没有发生thenXXX调用, 所以它的栈是空的), 此时会走while循环第二个条件, 此时src被重新切回到了CF1. 但是CF1的栈也只有一个, 所以while循环终止(第一次循环时唯一的Completion出栈了).



`注意这三次循环都是在CF1.postCompele方法中完成的.`



当第二次循环后, CF2开始执行.



#### tryFire方法原理

此方法属于Completion, 用于触发依赖此Completion的阶段(dep字段即代表下个阶段的CF).

```java
final CompletableFuture<V> tryFire(int mode) {
	
  CompletableFuture<V> d; CompletableFuture<T> a;
  // 参数合法的情况下执行一下 当前阶段. dep为当前阶段, src为上个阶段
  if ((d = dep) == null || !d.uniApply(a = src, fn, mode > 0 ? null : this))
    return null;	// 参数不合法 或者 执行失败 直接返回

  // 字段置为空, 则当调用Completion.isLive时返回false. 便于清理栈中死亡的Completion
  dep = null; src = null; fn = null;

  // 能执行, 则继续向后触发
  return d.postFire(a, mode);
}
```

该方法用于尝试触发下个阶段的执行, 此时上个阶段`已执行完成`,或者还`未执行完成`.



**postFire方法原理**

```java
final CompletableFuture<T> postFire(CompletableFuture<?> a, int mode) {
  // a为当前阶段所依赖的上一个阶段
  if (a != null && a.stack != null) {
    // 上阶段的CF还未执行完
    // 内嵌模式 或 非内嵌且上阶段的结果为空.
    if (mode < 0 || a.result == null)
      // 清楚栈
      a.cleanStack();
    // 上阶段CF执行结束
    else
      // 触发上阶段CF的后续阶段
      a.postComplete();
  }
  // 当前阶段的CF执行完成, 且有下阶段CF
  if (result != null && stack != null) {
    // 内嵌模式, 返回当前阶段
    if (mode < 0)
      return this;
    else
      // 触发下阶段的CF
      postComplete();
  }
  return null;
}
```

此方法作用就是: 

1. 检查是否存在另一个阶段和当前阶段一样, 依赖上阶段完成时触发. 
2. 如果当前阶段已完成则触发后续阶段





### 总结

CompletableFuture的核心就是Completion, 通过Completion就前后阶段的CF连接了起来. 


