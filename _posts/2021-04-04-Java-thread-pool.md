---
layout: post
title:  "Java 线程池详解"
date:   2021-04-04 10:13:11 +0800
categories: java thread pool executor
abstract: "详细分析 java 线程池的构造和几种常用线程池的设计"
---

# Java 线程池详解

## 一、ThreadPoolExecutor 构造

大部分的线程池底层都是由同一个构造方法实现的。

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
这个构造函数包含了线程池需要的所有参数：
* `corePoolSize` - 核心线程数
* `maximumPoolSize` - 最大线程数
* `keepAliveTime` - 线程存活时间
* `unit` - 时间单位
* `workQueue` - 用于存放任务的阻塞工作队列
* `threadFactory` - 创建线程的工厂类
* `handler` - 线程池满时使用的拒绝策略

#### 线程池的工作原理

在一个线程池中提交任务（`Task`）时分为以下几种情况：
1. 当前线程池线程数小于核心线程数的情况下，由线程工厂类创建新线程，并执行该任务。
2. 当前线程池线程数大于等于核心线程数且工作队列未满的情况下，将任务放入工作队列尾部。
3. 当前线程池线程数小于最大线程数且工作队列已满的情况下，由线程工厂类创建新线程，并执行该任务。
4. 当前线程池线程数等于于最大线程数且工作队列已满的情况下，执行当前线程池拒绝策略。
当线程完成任务时从工作队列最前端获取任务执行。

如果线程完成任务后工作队列为空，则线程进入空闲状态。
当空闲时间超过线程存活时间时且当前线程数大于核心线程数，线程会被销毁，以保持线程池当前线程数等于核心线程数。

#### 常用的工作队列使用策略

##### 1. 直接传递（`Direct handoffs`）
官方提供了一个阻塞队列的实现类 `SynchronousQueue`，这是一个没有容量的队列，每一次插入操作后都必须同步等待对应的删除操作，连续插入会报错。也就是说，使用这种队列作为工作队列，每提交一个新任务都必须有一个线程去处理任务（从队列中删除任务）。队列的作用只是传递任务。需要注意的是，如果任务的创建速度大于任务的平均处理速度，那么线程池的线程总数将很快会达到最大线程数。

##### 2. 无限长度队列（`Unbounded queues`）
使用不设定初始长度的 `LinkedBlockingQueue` 就是一种无限制长度的工作队列。在这种情况下设置最大线程数是不生效的，因为所有的任务都会存入工作队列中，线程池中的线程数永远是核心线程数。同样的，当任务的创建速度大于任务的平均处理速度，工作队列的长度将会无限增长直到 OOM。

##### 3. 有限长度队列（`Bounded queues`）
使用设定长度的 `ArrayBlockingQueue` 队列在最大线程数的配合下可以有效的避免 OOM 的问题。由于队列长度有限，当任务的创建速度大于任务的平均处理速度时，线程池中的线程数会很快扩展到最大线程数。当线程数达到最大线程数以后将执行线程池拒绝策略，来避免出现资源耗尽的情况。

#### 官方提供的拒绝策略
官方提供了四种 `RejectedExecutionHandler` 接口的实现类作为线程池的拒绝策略。
##### 1. AbortPolicy
在执行拒绝策略时，直接抛出 `RejectedExecutionException` 异常。
##### 2. CallerRunsPolicy
使用当前线程执行此任务。
##### 3. DiscardPolicy
直接扔掉此任务。
##### 4. DiscardOldestPolicy
扔掉工作队列的队首任务，并将此任务加入队尾。

或者可以实现 `RejectedExecutionHandler` 接口自定义拒绝策略。

## 二、ExecutorService 接口设计
ExecutorService 接口主要由以下几个接口组成

#### 1. shutdown
shutdown()方法会关闭当前线程池，拒绝接收新的任务，当前线程中的任务会继续执行。但是 shutdown()方法不会等待已有任务执行结束。

#### 2. shutdownNow
shutdownNow()方法和 shutdown()方法最大的区别在于 shutdownNow()方法会立刻中断正在执行的所以任务，并以 List 形式进行返回。
但是由于绝大部分的 shutdownNow()方法是使用 Thread.interrupt()方法进行中断线程，所以存在没有执行或响应 Thread.interrupt()方法的任务无法中断。

#### 3. awaitTermination
awaitTermination(long, TimeUnit)方法会一直阻塞直到关闭请求 (shutdown 或 shutdownNow 方法) 执行完成或阻塞直到超过设定的等待时间。
如果关闭请求后任务执行完成返回 true，如果达到等待时间则返回 false。

#### 4. submit
submit 的多个重载方法用于提交任务并指定返回值类型。由于返回值类型为指定泛型的 Future 类，可以通过 Future.get()方法获取任务返回值。

#### 5. invokeAll
invokeAll 方法可以提交一组任务，当任务 ` 全部 ` 完成或者到达设定的超时时间后返回 List<Future>。

#### 6. invokeAny
invokeAll 方法可以提交一组任务，当任务 ` 有任意一个 ` 完成或者到达设定的超时时间后返回 List<Future>。

## 三、常用线程池
Excutors类中提供了几种常用线程池的默认实现，供开发人员使用。但是在阿里巴巴java开发手册中却强制要求不要使用Executors工具类，其目的是要让开发人员更加熟悉线程池的创建过程和更加合理的配置参数。
但是我们可以通过查看源码来了解常用的线程池具体是如何实现的。

#### 1. Executors.newCachedThreadPool()

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

newCachedThreadPool创建了一个核心线程数为0，最大线程数为int最大值的ThreadPoolExecutor。并且使用SynchronousQueue作为工作队列，前文中提到过，SynchronousQueue是一个没有长度的阻塞队列，他的作用只是将任务传递给线程。而newCachedThreadPool中的最大线程数为Integer.MAX_VALUE，说明这个线程池可以尽可能多的启动线程来处理任务，而不是在工作队列中等待线程执行任务。当线程空闲一分钟以上时线程池会销毁线程直至线程池中没有线程。所以这个“cached”线程池是一个很有弹性的线程池。

#### 2. Executors.newFixedThreadPool()

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

newCachedThreadPool使用了一个int参数来控制核心线程的数量，并且使用了前文中提到的无限长度的LinkedBlockingQueue队列在作为工作队列。由于工作队列的容量为无限大（直到内存不足），所以最大线程数没有作用，于是使用和核心线程数相同的值作为最大线程数。也因此不会有任何非核心线程被创建，于是设置keepAliveTime为0在默认情况下(executor.allowCoreThreadTimeOut(false))也不会有任何线程会被销毁。所以这个“fixed”线程池是一个拥有固定线程容量一直工作的线程池。

#### 3. Executors.newScheduledThreadPool(int)

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```

newScheduledThreadPool创建的是一个ThreadPoolExecutor的子类ScheduledThreadPoolExecutor。

```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {
        
    //省略其他代码
    public ScheduledThreadPoolExecutor(int corePoolSize) {
            super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
                  new DelayedWorkQueue());
        }
}
```

DelayedWorkQueue是一个基于二叉堆结构实现的工作队列，可以根据传入执行任务的延时时间调整队列顺序。所以“scheduled”线程池是一个可以提供延时执行时间的固定线程数线程池。


#### 4. Executors.newSingleThreadExecutor()

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

newSingleThreadExecutor内部使用代理类实现了一个newFixedThreadPool(1)，而这个代理类重写了finalize()方法，使线程在被gc时执行shutdown方法。

```java
static class FinalizableDelegatedExecutorService
        extends DelegatedExecutorService {
        FinalizableDelegatedExecutorService(ExecutorService executor) {
            super(executor);
        }
        protected void finalize() {
            super.shutdown();
        }
    }
```

所以“single”线程池是一个固定单线程的线程池，可以保证任务的执行顺序。

