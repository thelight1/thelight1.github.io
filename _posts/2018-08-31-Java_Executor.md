---
layout: post
title:  "Java并发之Executor框架"
categories: Java并发 
tags: Java并发 
author: thelight1
mathjax: true
---
* content
{:toc}

# Executor框架简介
Java的线程既是工作单元，也是执行机制。从JDK5开始，把工作单元和执行机制分离开来。

Executor框架由3大部分组成

* 任务。

  * 被执行任务需要实现的接口：Runnable接口或Callable接口

  * 异步计算的结果。Future接口和FutureTask类。

* 任务的执行。两个关键类ThreadPoolExecutor和SeheduledThreadPoolExecutor。

# 任务
包括Runnable接口或Callable接口、Future接口等。

## Runnable接口
不含有运行结果

``` java
public interface Runnable {
    public abstract void run();
}
```

## Callable接口
含有运行结果

``` java
public interface Callable<V> {
    V call() throws Exception;
}
```

Runnable和Callable的区别

(1)Callable规定的方法是call(),Runnable规定的方法是run().

(2)Callable的任务执行后可返回值，而Runnable的任务是不能返回值的

(3)call方法可以抛出异常，run方法不可以

(4)运行Callable任务可以拿到一个Future对象，Future 表示异步计算的结果。

## Future接口
Future接口代表异步计算的结果，通过Future接口提供的方法可以查看异步计算是否执行完成，或者等待执行结果并获取执行结果，同时还可以取消执行。

``` java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);//取消任务
    boolean isCancelled();//是否取消了
    boolean isDone();//任务是否完成
    //获取任务执行结果，如果任务还没完成则会阻塞等待直到任务执行完成
    V get() throws InterruptedException, ExecutionException;
    //等待一段时间尝试获取执行结果，
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

一个使用Runnable的简单例子

```java
import java.util.concurrent.*;
​
public class Test {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        final ExecutorService exec = Executors.newFixedThreadPool(5);
        Runnable task = new Runnable() {
            public void run() {
                try {
                    System.out.println(Thread.currentThread().getName() + " is running");
                    Thread.sleep(1000 * 10);//休眠指定的时间，此处表示该操作比较耗时
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        exec.submit(task);
        //关闭线程池
        exec.shutdown();
    }
}
```

一个使用Callable的简单例子

``` java
import java.util.concurrent.*;
 
public class Test {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        final ExecutorService exec = Executors.newFixedThreadPool(5);
        Callable<String> call = new Callable<String>() {
            public String call() throws Exception {
                Thread.sleep(1000 * 10);//休眠指定的时间，此处表示该操作比较耗时
                return "Other less important but longtime things.";
            }
        };
        Future<String> task = exec.submit(call);
        //重要的事情
        System.out.println("Let's do important things. start");
        Thread.sleep(1000 * 3);
        System.out.println("Let's do important things. end");
 
        //不重要的事情
        while(!task.isDone()){
            System.out.println("still waiting....");
            Thread.sleep(1000 * 1);
        }
        System.out.println("get sth....");
        String obj = task.get();
        System.out.println(obj);
        //关闭线程池
        exec.shutdown();
    }
}
```
运行结果

``` shell
Let's do important things. start
Let's do important things. end
still waiting....
still waiting....
still waiting....
still waiting....
still waiting....
still waiting....
still waiting....
get sth....
Other less important but longtime things.
``` 

# 任务的执行机制
## ThreadPoolExecutor
ThreadPoolExecutor是Executor框架最核心的类，是线程池的实现类。

核心配置参数包括

corePoolSize：核心线程池的大小

maximumPoolSize：最大线程池的大小

BlockingQueue：暂时保存任务的工作队列

RejectedExecutionHandler：当ThreadPoolExecutor已经饱和时（达到了最大线程池大小且工作队列已满）将执行的Handler。

``` java
    public ThreadPoolExecutor(int corePoolSize, 
                              int maximumPoolSize,
                              long keepAliveTime, 
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```
可以创建3种类型的ThreadPoolExecutor。

* FixedThreadPool

* SingleThreadExecutor

* CachedThreadPool

### 1.FixedThreadPool
FixedThreadPool是固定线程数的线程池，最多线程池中有nThreads个线程。

``` java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads, //nThreads为固定线程数
                                      0L, TimeUnit.MILLISECONDS, //空闲线程的等待时间为0ms，表示立刻被终止
                                      new LinkedBlockingQueue<Runnable>()); //工作队列
    }
```

FixedThreadPool的execute()方法内部执行过程

当新任务被提交时，如果当前运行线程数小于nTheads，创建新线程执行任务

如果当前运行线程数等于设置的最大线程数nThreads，将新任务加入到工作队列LinkedBlockingQueue中

线程执行完任务后会反复从LinkedBlockingQueue中获取新任务执行

LinkedBlockingQueue中没有新任务，线程空闲，线程将被终止。

注意点：

由于工作队列使用的是无界队列LinkedBlockingQueue，FixedThreadPool不会拒绝任务（不会调用RejectedExecutionHandler.rejectedExecution()方法）。

### 2.SingleThreadExecutor
SingleThreadExecutor是FixedThreadPool的特例，线程池中线程的固定数量为1，即最多有一个线程。

``` java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,  //nThreads为固定线程数
                                    0L, TimeUnit.MILLISECONDS, //空闲线程的等待时间为0ms，表示立刻被终止
                                    new LinkedBlockingQueue<Runnable>())); //工作队列
    }
```

SingleThreadExecutor的execute()方法内部执行过程与注意事项可参考FixedThreadPool的。

### 3.CachedThreadPool
CachedThreadPool是根据需要创建新线程的线程池。

``` java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE, //coolPoolSize为0，maxinumPoolSize为Integer.MAX_VALUE
                                      60L, TimeUnit.SECONDS, //空闲线程的等待时间，空闲60s后被终止
                                      new SynchronousQueue<Runnable>()); //工作队列
    }
```

CachedThreadPool的execute()方法的内部运行过程

当新任务被提交时，主线程将任务插入到工作队列中（SynchronousQueue的offer()方法），如果线程池有空闲线程在等待任务，新任务交给空闲线程处理。

如果线程池中没有在等待任务的空闲线程，创建新线程执行任务

线程执行完任务后，等待60s（SynchronousQueue.poll(60, TimeUnit.SECONDS)方法），如果没有等待新任务，线程终止

SynchronousQueue是一个没有容量的BlockingQueue。每一个插入操作必须等待另一个线程的移除操作。

CachedThreadPool使用SynchronousQueue，把主线程提交的任务传递给空闲线程。



注意点：

CachedThreadPool的线程池是无解的，没有限制数量，如果主线程提交任务的速度高于线程处理任务的速度，将不断创建新线程。

极端情况下，会因创建过多线程耗尽CPU和内存。

## ScheduledThreadPoolExecutor
ScheduledThreadPoolExecutor主要用于定时任务（定期执行，给定延迟后执行）

执行方式
### 1.scheduleAtFixedRate
任务按照固定周期执行，比如设定每10分钟执行一次，第8分钟时候执行了第一次，后续执行时间点为第18分钟，第28分钟，第38分钟

``` java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, //任务
                                                  long initialDelay, //初始延迟
                                                  long period, //任务执行周期
                                                  TimeUnit unit) 

```
### 2.scheduleWithFixedDelay
任务按照固定延迟执行，比如设定延迟时间为10是分钟，第8分钟时候执行了第一次，任务执行完成后，再等待10分钟，执行下一次。如果任务执行了2分钟，则下一次为第20分钟

``` java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,//任务
                                                     long initialDelay, //初始延迟
                                                     long delay, //任务执行延迟
                                                     TimeUnit unit)
```

###实现原理

数据结构
ScheduledThreadPoolExecutor：定时任务执行器

DelayQueue：使用DelayQueue作为任务队列，保存待调度的任务，任务按照执行的时间点排序。DelayQueue内部是用PriorityQueue实现。

ScheduledFutureTask：待调度任务。

ScheduledFutureTask的成员变量：

time：任务将被执行的具体时间

sequenceNumber：任务序号，time相同时，序号小的先执行

period：任务执行的间隔周期

运行机制
ScheduledThreadPoolExecutor内部运行过程

线程从DelayQueue中获取到到期的ScheduledFutureTask，到期是指time大于等于当前时间。

执行任务ScheduledFutureTask

修改ScheduledFutureTask的time为下次要执行的时间，放回到DelayQueue中