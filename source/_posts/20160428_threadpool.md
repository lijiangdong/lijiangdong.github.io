---
title: Java中的线程池
date: 2016-04-28 23:26
category: [Java]
tags: [ThreadPool]
comments: true
---

# **综述**

　　在我们的开发中经常会使用到多线程。例如在Android中，由于主线程的诸多限制，像网络请求等一些耗时的操作我们必须在子线程中运行。我们往往会通过new Thread来开启一个子线程，待子线程操作完成以后通过Handler切换到主线程中运行。这么以来我们无法管理我们所创建的子线程，并且无限制的创建子线程，它们相互之间竞争，很有可能由于占用过多资源而导致死机或者OOM。所以在Java中为我们提供了线程池来管理我们所创建的线程。<!--more-->

# **线程池的使用**

## **采用线程池的好处**

　　在这里我们首先来说一下采用线程池的好处。
　　1. 重用线程池中已经存在的线程，减少了线程的创建和消亡多造成的性能开销。
　　2. 能够有效控制最大的并发线程数，提高了系统资源的使用率，并且还能够避免大量线程之间因为相互抢占系统资源而导致阻塞。
　　3. 能够对线程进行简单管理，并提供定时执行、定期执行、单线程、并发数控制等功能。

## **ThreadPoolExecutor**

　　我们可以通过ThreadPoolExecutor来创建一个线程池。下面我们就来看一下ThreadPoolExecutor中的一个构造方法。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

### **ThreadPoolExecutor参数含义**

**1. corePoolSize**

　　线程池中的核心线程数，默认情况下，核心线程一直存活在线程池中，即便他们在线程池中处于闲置状态。除非我们将ThreadPoolExecutor的allowCoreThreadTimeOut属性设为true的时候，这时候处于闲置的核心线程在等待新任务到来时会有超时策略，这个超时时间由keepAliveTime来指定。一旦超过所设置的超时时间，闲置的核心线程就会被终止。

**2. maximumPoolSize**

　　线程池中所容纳的最大线程数，如果活动的线程达到这个数值以后，后续的新任务将会被阻塞。

**3. keepAliveTime**

　　非核心线程闲置时的超时时长，对于非核心线程，闲置时间超过这个时间，非核心线程就会被回收。只有对ThreadPoolExecutor的allowCoreThreadTimeOut属性设为true的时候，这个超时时间才会对核心线程产生效果。

**4. unit**

　　用于指定keepAliveTime参数的时间单位。他是一个枚举，可以使用的单位有天（TimeUnit.DAYS），小时（TimeUnit.HOURS），分钟（TimeUnit.MINUTES），毫秒(TimeUnit.MILLISECONDS)，微秒(TimeUnit.MICROSECONDS, 千分之一毫秒)和毫微秒(TimeUnit.NANOSECONDS, 千分之一微秒);

**5. workQueue**

　　线程池中保存等待执行的任务的阻塞队列。通过线程池中的execute方法提交的Runable对象都会存储在该队列中。我们可以选择下面几个阻塞队列。

 - ArrayBlockingQueue:基于数组实现的有界的阻塞队列,该队列按照FIFO（先进先出）原则对队列中的元素进行排序。
 - LinkedBlockingQueue:基于链表实现的阻塞队列，该队列按照FIFO（先进先出）原则对队列中的元素进行排序。
 - SynchronousQueue:内部没有任何容量的阻塞队列。在它内部没有任何的缓存空间。对于SynchronousQueue中的数据元素只有当我们试着取走的时候才可能存在。
 - PriorityBlockingQueue:具有优先级的无限阻塞队列。
 - 我们还能够通过实现BlockingQueue接口来自定义我们所需要的阻塞队列。
 
**6. threadFactory**

　　线程工厂，为线程池提供新线程的创建。ThreadFactory是一个接口，里面只有一个newThread方法。

**7. handler**

　　他是RejectedExecutionHandler对象，而RejectedExecutionHandler是一个接口，里面只有一个rejectedExecution方法。当任务队列已满并且线程池中的活动线程已经达到所限定的最大值或者是无法成功执行任务，这时候ThreadPoolExecutor会调用RejectedExecutionHandler中的rejectedExecution方法。在ThreadPoolExecutor中有四个内部类实现了RejectedExecutionHandler接口。在线程池中它默认是AbortPolicy，在无法处理新任务时抛出RejectedExecutionException异常。下面是在ThreadPoolExecutor中提供的四个可选值。

 - CallerRunsPolicy:只用调用者所在线程来运行任务。
 - AbortPolicy:直接抛出RejectedExecutionException异常。
 - DiscardPolicy:丢弃掉该任务，不进行处理
 - DiscardOldestPolicy:丢弃队列里最近的一个任务，并执行当前任务。
 - 我们也可以通过实现RejectedExecutionHandler接口来自定义我们自己的handler。如记录日志或持久化不能处理的任务。

### **ThreadPoolExecutor执行规则**

 1. 如果在线程池中的线程数量没有达到核心的线程数量，这时候就回启动一个核心线程来执行任务。
 2. 如果线程池中的线程数量已经超过核心线程数，这时候任务就会被插入到任务队列中排队等待执行。
 3. 由于任务队列已满，无法将任务插入到任务队列中。这个时候如果线程池中的线程数量没有达到线程池所设定的最大值，那么这时候就会立即启动一个非核心线程来执行任务。
 4. 如果线程池中的数量达到了所规定的最大值，那么就会拒绝执行此任务，这时候就会调用RejectedExecutionHandler中的rejectedExecution方法来通知调用者。

### **ThreadPoolExecutor的使用**

　　上面说了那么多，我们现在就来看一下到底是如何使用这个ThreadPoolExecutor。首先我们通过ThreadPoolExecutor创建一个一个线程池。

```java
ExecutorService executorService = new ThreadPoolExecutor(5,10,10,TimeUnit.MILLISECONDS,new LinkedBlockingQueue<>());
```
　　对于ThreadPoolExecutor有多个构造方法，对于上面的构造方法中的其他参数都采用默认值。我们创建完一个线程池以后，下面就再来看一下如何向线程池提交一个任务。我们可以通过execute和submit两种方式来向线程池提交一个任务。

**execute**

　　当我们使用execute来提交任务时，由于execute方法没有返回值，所以说我们也就无法判定任务是否被线程池执行成功。

```java
executorService.execute(new Runnable() {
	
	@Override
	public void run() {
		// doSomething
		
	}
});
```

**submit**

　　当我们使用submit来提交任务时,它会返回一个future,我们就可以通过这个future来判断任务是否执行成功，通过future的get方法来获取返回值，get方法会阻塞住直到任务完成，而使用get(long timeout, TimeUnit unit)方法则会阻塞一段时间后立即返回，这时候有可能任务并没有执行完。

```java
Future<Object> future = executorService.submit(new Callable<Object>() {
	@Override
	public String call() throws Exception {
		// TODO Auto-generated method stub
		return null;
	}
});

try {
	Object object = future.get();
} catch (InterruptedException e) {
	// 处理中断异常
	e.printStackTrace();
} catch (ExecutionException e) {
	// 处理无法执行异常
	e.printStackTrace();
}
```
### **关闭线程池**

　　我们可以通过shutdown方法或者shutdownNow方法来关闭线程池。对于这两种关闭线程池的方式他们都是通过遍历线程池中所有的线程，然后依次调用线程的interrupt方法来中断线程。当然对于这两种关闭线程池的方法也是有一定区别的（具体区别见下面注释）。
　　当我们调用了下面任何一个关闭方法时，isShutdown方法就会返回true。而当线程池关闭成功以后isTerminaed方法会返回true。对于线程池中的正在执行的任务如果我们希望他们执行完成以后再去关闭线程池则调用shutdown方法；而我们希望在关闭线程池的时候中断线程池内正在执行的任务，则调用shutdownNow方法。
```java
/**
* 首先将线程池的状态设置成SHUTDOWN状态，然后中断所
* 有没有正在执行任务的线程。
*/
executorService.shutdown();

/**
* 首先将线程池的状态设置为STOP，然后开始尝试停止所有的正在
* 工作或暂停任务的线程
*/
executorService.shutdownNow();
```

# **Java线程池**

## **Java中的线程池分类**

　　在这里我们介绍一下Java中四种具有不同功能常见的线程池。他们都是直接或者间接配置ThreadPoolExecutor来实现他们各自的功能。这四种线程池分别是newFixedThreadPool,newCachedThreadPool,newScheduledThreadPool和newSingleThreadExecutor。这四个线程池可以通过Executors类获取。下面分别介绍这四种线程池。
　　 1. **newFixedThreadPool**
　　我们可以通过Executors中的newFixedThreadPool方法来创建，该线程池是一种线程数量固定的线程池。在这个线程池中所容纳最大的线程数就是我们设置的核心线程数。如果线程池的线程处于空闲状态的话，它们并不会被回收，除非是这个线程池被关闭。如果所有的线程都处于活动状态的话，新任务就回处于等待状态，直到有线程空闲出来。由于newFixedThreadPool只有核心线程，并且这些线程都不会被回收，也就是它能够更快速的响应外界请求。从下面的newFixedThreadPool方法的实现可以看出，newFixedThreadPool只有核心线程，并且不存在超时机制，采用LinkedBlockingQueue，所以对于任务队列的大小也是没有限制的。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

　　 2. **newCachedThreadPool**
　　我们可以通过Executors中的newCachedThreadPool方法来创建，通过下面的newCachedThreadPoolfan'f在这里我们可以看出它的核心线程数为0，线程池的最大线程数Integer.MAX_VALUE。而Integer.MAX_VALUE是一个很大的数，也差不多可以说这个线程池中的最大线程数可以任意大。当线程池中的线程都处于活动状态的时候，线程池就会创建一个新的线程来处理任务。该线程池中的线程超时时长为60秒，所以当线程处于闲置状态超过60秒的时候便会被回收。这也就意味着若是整个线程池的线程都处于闲置状态超过60秒以后，在newCachedThreadPool线程池中是不存在任何线程的，所以这时候它几乎不占用任何的系统资源。对于newCachedThreadPool他的任务队列采用的是SynchronousQueue，上面说到在SynchronousQueue内部没有任何容量的阻塞队列。SynchronousQueue内部相当于一个空集合，我们无法将一个任务插入到SynchronousQueue中。所以说在线程池中如果现有线程无法接收任务,将会创建新的线程来执行任务。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

　　 3. **newScheduledThreadPool**
　　我们可以通过Executors中的newScheduledThreadPool方法来创建，它的核心线程数是固定的，对于非核心线程几乎可以说是没有限制的，并且当非核心线程处于限制状态的时候就会立即被回收。

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

　　 4. **newSingleThreadExecutor**
　　我们可以通过Executors中的newSingleThreadExecutor方法来创建，在这个线程池中只有一个核心线程，对于任务队列没有大小限制，也就意味着这一个任务处于活动状态时，其他任务都会在任务队列中排队等候依次执行。newSingleThreadExecutor将所有的外界任务统一到一个线程中支持，所以在这个任务执行之间我们不需要处理线程同步的问题。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

## **四种线程池的使用**

　　下面我们就来看一下对于上面四种线程池是如何使用的。

```java
Runnable command = new Runnable() {
	public void run() {
		//doSomething
	}
};
		
ExecutorService fixedThreadPool = Executors.newFixedThreadPool(10);
fixedThreadPool.execute(command);

ExecutorService  cachedThreadPool= Executors.newCachedThreadPool();
cachedThreadPool.equals(command);

ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(10);
//1000毫秒后执行coommand
scheduledThreadPool.schedule(command, 1000, TimeUnit.MILLISECONDS);
//延时5毫秒后，每隔100毫秒执行一次command
scheduledThreadPool.scheduleAtFixedRate(command, 5, 100, TimeUnit.MILLISECONDS);

ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
singleThreadExecutor.execute(command);
```

# **总结**

　　对于Java中的线程池概念同样适用于Android，例如在我们开发一个app时，我们可以创建一个线程池，将所有的子线程任务交由线程池来处理，于是我们便可以通过这个线程池来管理维护我们的子线程。减少了应用的开销。