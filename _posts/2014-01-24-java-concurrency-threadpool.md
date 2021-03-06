---
layout: post
category: Java Concurrency
description: 介绍多线程编程中的一个基本工具线程池ThreadPool的内部原理解析，包含ExecutorService原理和Executor原理。
keywords: java concurrency, java的多线程处理，ThreadPool的用法，ThreadPool的源代码分析和ThreadPool的主要方法以及原理分析，Executors的用法，Executors的源代码分析和Executors的主要方法以及原理分析，
title: Java线程池Executor的分析
tags: [java concurrency]
summary: Java 多线程基本工具之ThreadPool的用法和原理分析
---


###一.无限制创建线程的不足
并不是说多线程就一定能带来性能上的优化，过多的线程也会有它自己的劣势。

1.	线程生命周期的开销非常高，创建和销毁线程具有一定的代价。
2.	资源消耗：大量的空线程会带来内存消耗，如果已有线程已经能让CPU保持忙碌状态，那么再创建更多地线程反而会降低性能。
3.	稳定性。个别线程的异常，能会导致OOM等，从而影响整体。

###二.Executor框架
Executor是典型的生产者消费者模式，它将线程的提交和执行解耦。

Executors是创建Executor的工具类，它有如下几种创建ExecutorService的方法：

1.	newCachedThreadPool 创建可缓存的线程池，池中的线程将被循环利用。线程池的最大线程数为Integer.MAX_VALUE
2.	newFixedThreadPool  创建固定大小的线程池。
3.	newSingleThreadExecutor 创建单线程的线程池。
4.	newScheduledThreadPool 创建可以定时执行线程的线程池

除了定时执行的线程池外，其它线程池创建方式内部均创建了ThreadPoolExecutor这个线程池实现。

接口ExecutorService继承了接口Executor。

而Executor只有一个execute方法：

    void execute(Runnable command);
    

ExecutorService在Executor的基础上添加了生命周期管理。

1.	shutdown() 等线程执行完后关闭
2.	shutdownNow 暴力关闭
3.	isTerminated

Executor的提交的任务的生命周期为：创建、提交、开始和完成。

只有已经提交，但是尚未开始的任务可以取消。

Executors创建的ExecutorService基本上可以指定线程池的基本大小和指定线程工厂类。

####ThreadPoolExecutor
ThreadPoolExecutor的构造函数包含几个参数：

1.	int corePoolSize 基本线程池大小，即在没有任务执行时线程池的大小，并且只有在工作队列满了的情况下才会创建这个超出数量的线程。
2.	int maxiumPollSize 线程池最大大小，可活动的线程数量上限。
3.	long keepAliveTime 线程的最大存活时间
4.	TimeUnit timeUnit时间单位
5.	`BlockingQueue<Runnable>workqueue` 等待任务的工作队列
6.	ThreadFactory threadFactory 线程工厂
7.	RejectedExecutionHandler handler 拒绝执行的策略

newFixedThreadPool返回的是基本线程大小和线程池最大大小的ThreadPoolExecutor。

newCachedThreadPool 返回的则是最大大小为Integer.MAX_VALUE的ThreadPoolExecutor。


newFixedThreadPool和newSingleThreadExecutor返回的是一个无界的LinkedBlockingQueue，如果所有工作线程都处于忙碌状态，那么任务将在队列中等待。

如果任务快速到达，并且超过处理速度，那么队列将无限制地增加。

newCachedThreadPool使用的则是SynchronousQueue,这个是个虚拟的队列，它将需要执行的线程交给执行它的线程，而不是放在等待队列中。

SynchronousQueue可以有效避免任务排队。

####饱和策略
RejectedExecutionHandler设置的为线程池满后的饱和策略，

JDK提供了四种实现：

1.	AbortPolicy 直接舍弃，抛出RejectedExecutionException
2.	CallerRunsPolicy 调用者执行，由调用者直接执行这个线程。
3.	DiscardPolicy 舍弃策略，这个线程将被舍弃。源码实现中，什么都没干。
4.	DiscardOldestPolicy 舍弃下一个即将被执行的任务，然后执行被提交的任务。

####线程工厂
当线程池需要创建新的线程时，会调用ThreadFactory的newThread方法，返回新的线程。

通过自定义线程工厂，可以为线程增加一些额外的功能。

比如自定义线程名，线程的创建和销毁数量，自定义UncaughtExceptionHanlder等。

由于ThreadPoolExecutor的所有参数在后期都可以通过set方法修改，如果需要将Executor暴露给其它不可信的调用方时，

可以通过调用Executors的unconfigurableExecutorService来包装成不可配置的形式。



####Runnable\Callable\Future

Runnable接口只有一个run方法：

	public abstract void run();
	
它有很大的局限性，没有返回值，也不抛出受检查的异常。


Callable是一种更好地抽象，它认为任务具有返回值，并可以抛出受检异常。

	V call() throws Exception;
	
Future代表一个任务的生命周期，可以用相应的方法来判断任务是否已经完成，或者取得任务的结果。

1.	boolean cancel(boolean mayInterruptIfRunning);
2.	boolean isCancelled();
3.	boolean isDone();
4.	V get() throws InterruptedException, ExecutionException;
5.	V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
        
####提交任务
ExecutorService的任务提交submit()方法可以将一个Callable或者Runnable提交，然后返回一个Future。

		1.	Future<?> submit(Runnable task);
		2.	<T> Future<T> submit(Callable<T> task);

代码实例:[ExecutorType.java](https://github.com/llohellohe/cp/blob/master/src/yangqi/jcp/executor/ExecutorType.java)

Future 的get方法会一直阻塞直到任务完成或者到达指定超时时间。

### 三.中断
Java并没有直接提供安全的终止另个线程的方法，stop方法由于会导致对象的状态不一致，因此不推荐使用。

但是它提供了中断机制,每个线程都有自己的中断状态（native实现）。

线程A可以将线程B的中断变量置为true，但这并不是说线程B马上会终止。

而是线程B在下个时间内，可以检查这个中断状态，而后再决定是否响应这个中断。

将中断状态置为true的方法只有一个:interrupt()。

判断是否中断：isInterrupt()。

重置中断状态：interrupted()。


当收到InterruptedException这个异常后，不该捕获它后什么都不做，而该采取相应措施，比如调用当前线程的interrupt方法。

####为何不用stop方法
stop失败后，将直接抛出ThreadDeth这个Error的子类。

stop后对象的锁将被释放，会导致其它用到锁的地方失效，从而导致状态的不一致。

参考资料：

1.	
2.	[详
细分析Java中断机制](http://www.infoq.com/cn/articles/java-interrupt-mechanism)

### 四.线程饥饿死锁
如果线程1的执行依赖线程2的执行，那么线程池中将出现线程饥饿死锁Thread Starvation Deadlock。

同样，如果线程的执行时间过长，也将影响线程池的效率。

####线程池大小
对于计算密集型的任务，在有N个CPU的情况下，线程池的大小为N+1，通常能实现最优的利用率。

IO密集型时，可以适当增大线程池大小。

Runtime.availableProcessors方法可以获得CPU的数量。

### 五.线程池Executor相关类图
![线程池的相关类图](https://raw2.github.com/llohellohe/llohellohe.github.com/master/readers/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%AE%9E%E6%88%98/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%B1%BB%E5%9B%BE.png)
