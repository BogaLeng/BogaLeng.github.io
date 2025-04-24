---
title: Java线程池
tags:
  - Java
createTime: 2025/04/25 00:53:21
permalink: /article/exgeihpe/
---

## 一、什么是线程池

线程池其实是一种**池化**的技术实现，池化技术的核心思想就是**实现资源的复用**，避免资源的重复创建和销毁带来的性能开销。线程池可以管理一堆线程，让线程执行完任务之后不进行销毁，而是继续去处理其它线程已经提交的任务。

这种池化的思想不止应用在线程这里，我们还有：**内存池**、**常量池**；计算机网络中，还有**连接池**，等等。

## 二、线程池的七大参数

源码中，`ThreadPoolExecutor`的其中一个构造方法如下：

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
		......
}
```

为什么说其中一个？因为它还有很多**重载**的构造方法，可以省略某些参数，使用默认值，比如使用默认的`RejectedExecutionHandler`,这个参数表示当阻塞队列满了的时候，新进入任务的处理方法，默认是直接丢弃`AbortPolicy`。

JDK 自带的 `RejectedExecutionHandler` 实现有 4 种

- AbortPolicy：丢弃任务，抛出运行时异常
- CallerRunsPolicy：由提交任务的线程来执行任务
- DiscardPolicy：丢弃这个任务，但是不抛异常
- DiscardOldestPolicy：从队列中剔除最先进入队列的任务，然后再次提交任务

参数解释如下：

- corePoolSize：线程池中用来工作的核心线程数量。
- maximumPoolSize：最大线程数，线程池允许创建的最大线程数。
- keepAliveTime：超出 corePoolSize 后创建的线程存活时间或者是所有线程最大存活时间，取决于配置。
- unit：keepAliveTime 的时间单位。
- workQueue：任务队列，是一个阻塞队列，当线程数达到核心线程数后，会将任务存储在阻塞队列中。
- threadFactory ：线程池内部创建线程所用的工厂。项目中我们通常对其重写，为了给线程按一定规则命名，便于出问题时排错。
- handler：拒绝策略；当队列已满并且线程数量达到最大线程数量时，会调用该方法处理任务。

## 三、线程池的原理

### 3.1 刚创建

刚创建出来的线程池中只有一个构造时传入的阻塞队列，里面并没有线程，如果想要在执行之前创建好核心线程数，可以调用 `prestartAllCoreThreads` 方法来实现，默认是没有线程的。

`prestartAllCoreThreads`的源码如下：

```java
    /**
     * Starts all core threads, causing them to idly wait for work. This
     * overrides the default policy of starting core threads only when
     * new tasks are executed.
     *
     * @return the number of threads started
     */
    public int prestartAllCoreThreads() {
        int n = 0;
        while (addWorker(null, true))
            ++n;
        return n;
    }
```

### 3.2 传入任务

当有线程通过 execute 方法提交了一个任务，会发生什么呢？

首先会去判断当前线程池的线程数是否小于**核心线程数**，也就是线程池构造时传入的参数 `corePoolSize`。如果小于，那么就直接通过 `ThreadFactory` 创建一个线程来执行这个任务。

接下来如果又提交了一个任务，也会按照上述的步骤去判断是否小于核心线程数，如果小于，还是会创建线程来执行任务，执行完之后也会从阻塞队列中获取任务。

这里有个细节，就是提交任务的时候，就算有线程池里的线程从阻塞队列中获取不到任务，如果线程池里的线程数还是小于核心线程数，那么依然会**继续创建线程**，而不是复用已有的线程。

如果线程池里的线程数不再小于核心线程数呢？那么此时就会尝试将任务放入**阻塞队列**中。

### 3.3 创建非核心线程

随着任务越来越多，队列已经满了，任务放入失败，怎么办呢？

此时会判断当前线程池里的线程数是否小于最大线程数，也就是入参时传入的`maximumPoolSize` 参数

如果小于最大线程数，那么也会创建非核心线程来执行提交的任务。

### 3.4 拒绝策略

随着任务越来越多，队列已经满了，任务放入失败。同时假如线程数已经达到最大线程数量。这个时候怎么办呢？

此时就会执行拒绝策略，也就是构造线程池的时候，传入的 RejectedExecutionHandler 对象，来处理这个任务。

## 四、实际使用

### 4.1 注意线程数

线程数的设置主要取决于业务是 IO 密集型还是 CPU 密集型。

CPU 密集型：指的是任务主要使用来进行大量的计算，没有什么导致线程阻塞。一般这种场景的线程数设置为 `CPU 核心数+1`。

IO 密集型：当执行任务需要大量的 io，比如磁盘 io，网络 io，可能会存在大量的阻塞，所以在 IO 密集型任务中使用多线程可以大大地加速任务的处理。一般线程数设置为 2*CPU 核心数

Java 中用来获取 CPU 核心数的方法是：`Runtime.getRuntime().availableProcessors();`

## 4.2 ThreadFactory

一般建议自定义线程工厂，构建线程的时候设置线程的名称，这样在查日志的时候就方便知道是哪个线程执行的代码。

### 4.3 有界队列

一般需要设置有界队列的大小，比如使用`LinkedBlockingQueue`在构造的时候可以传入参数来限制队列中任务数据的大小，这样就不会因为无限往队列中扔任务导致系统的Out of Memory。
