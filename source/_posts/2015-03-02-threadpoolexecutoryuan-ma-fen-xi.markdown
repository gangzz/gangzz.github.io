---
layout: post
title: "ThreadPoolExecutor源码分析"
date: 2015-03-02 21:05:29 +0800
comments: true
categories: ["concurrent", "others"]
---

## 设置
*	keepAliveTime：空闲线程的存活时间
*	allowCoreThreadTimeout：允许核心线程超时
*	maximumPoolSize：设置池的最大线程数
*	corePoolSize：设置池的核心线程数
*	rejectHandler：设置添加任务拒绝的处理策略

##	行为
###	keepAliveTime & allowCoreThreadTimeout：

设线程数为N，当（核心线程数 < N < 最大线程数）时，线程空闲超过设置的时间将会被关闭。如果设置allowCoreThreadTimeout，那么当(N > 0)时即存在上述行为。

### maximumPoolSize & corePoolSize

1.	当（N < corePoolSize）时新添加任务会创建一个线程。
2.	当（N >= corePoolSize)时新添加的任务会入队：
	1.	如果入队成功，还会检查线程池至少有一个线程是活动的（没有会尝试创建）。
	2.	如果入队失败，尝试创建一个线程，如果线程已达上限（或其他原因）不可创建，则拒绝当前任务。
3.	如果当前Executor已经Shutdown或停止，拒绝当前任务。


### prestartCoreThread、ensurePrestart、prestartAllThread

*	prestartCoreThread：不管有没有添加任务，确保线程池中有coreSize数量个线程。
*	ensurePrestart：和prestartCoreThread行为一致，唯一不同是即使coreSize设置为0也会启动一个空闲线程。
*	prestartAllThread：启动能启动的最大数量个线程。
### rejectHandler

ThreadPoolExecutor默认提供了四种处理任务被拒绝的策略：

*	AbortPolicy：拒绝，抛出异常，也是默认的处理策略。
*	DiscardPolicy：放弃当前。不抛出异常，放弃当前任务。
*	DiscardOldestPolicy：放弃最旧任务。不抛出异常，从任务队列中移除位于队首的任务。
*	CallerRunsPolicy：执行。在调用execute（submit）方法的当前线程执行该任务，该策略意在拉长提交任务的占用时间，从而降低提交频率。

### exception

被执行的Runnable或Callable在执行期间抛出异常，将导致当前线程中断，并将创建新的Thread弥补空缺（不管是不是超过了coreSize）。

### purge

线程池将遍历队列中所有的Task，清理已取消的任务。如果此时Executor已经Shutdown，此操作会尝试结束Executor。

### statistic

线程池会统计完成的任务总数和最大线程数。

##	源码分析
### 工作流程
![ThreadPoolExecutor工作流程](/images/blog/hold.png)


### 状态&状态变迁

#### 状态说明
ThreadPoolExecutor运行状态包括：

*	Running：Executor启动后的状态，接受并处理任务。
*	Shutdown：调用shutdown方法后，不接受新的任务，但是继续处理队列中的任务。
*	Stop：不接受不处理队列中的任务，并且中断正在处理的任务。
*	Tidying：所有任务已结束、WorkCount（线程数）归零时处于Tidying状态，开始调用terminated()的钩子。
*	Terminated：terminated()调用结束。

####	状态变迁：

*	Running -> Shutdown：调用了shutdown()。
*	(Running or Shtudown) -> Stop：调用了shutdownNow()。
*	Shutdown -> Tidying：队列和线程池已经空。
*	Stop -> Tidying：线程池已经空。
*	Tidying -> Terminated：terminated()方法已经调用。

#### ThreadPoolExecutor对状态和WorkCount的封装：
ThreadPoolExecutor使用一个AtomicInteger封装线程池状态和线程数

	    //ctl为状态和WorkCount的封装
		private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    	private static final int COUNT_BITS = Integer.SIZE - 3;
    	private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
		
		//编码ctl——状态和WorkCount按位与
		private static int ctlOf(int rs, int wc) { return rs | wc; }
按Integer.SIZE = 32计算，一个线程池低位(29位)是线程值可以容纳的Worker的最大数量，高位（30~32）用于表示状态——因为有5个状态必须要三位表示。

状态读取：

	private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }

WorkCount的操作（增删）：
	
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }
对状态和WorkerCount的操作都是使用Compare-And-Set的方式，无锁。

###	interruptIdleThread

ThreadPoolExecutor有几个中断空闲的线程的动机：

*	重新设置池大小的时候——CoreSize、MaximumSize。
* 	重新设置空闲线程存活时间的时候——KeepAliveTime、AllowCoreThreadTimeout。
*  调用shutdown的时候。

中断空闲线程需要两个动作：识别空闲线程、移除空闲线程（从队列中<Workers>），其中WorkerQueue的实现是HashSet，非线程安全的。这里有两个锁的实现：

**不可重进的排它锁——Worker类继承AbstractQueuedSynchronizer实现**

1.	用于空闲线程识别，线程允许获得的Task前会获得自己的锁，中断线程根据能够获得锁来识别线程是否空闲。
2. 为什么是不可重进的？为了避免线程本身修改线程池Size时导致重获得自己的锁。

	    private void interruptIdleWorkers(boolean onlyOne) {
        	final ReentrantLock mainLock = this.mainLock;
	        mainLock.lock();
	        try {
	            for (Worker w : workers) {
	                Thread t = w.thread;
	                //tryLock失败视为线程在忙碌
	                if (!t.isInterrupted() && w.tryLock()) {
	                    try {
	                        t.interrupt();
	                    } catch (SecurityException ignore) {
	                    } finally {
	                        w.unlock();
	                    }
	                }
	                if (onlyOne)
	                    break;
	            }
	        } finally {
	            mainLock.unlock();
	        }
    	}
    

**可重进排它锁：mainlock**

1.	mainlock用于保证WorkerQueue的线程安全，需要获得该锁的操作包括：修改Workers、更新LargestPoolSize、更新CompletedTaskCount。
2. 为什么LargestPoolSize和CompleteTaskCount不使用Atomic*？没有必要，更新这两个值的动机都是要对workers操作。
	1. largestPoolSize：添加worker时触发。
	2. CompletedTaskCount：每个worker会统计自己的CompleteTasks，在移除该worker时Executor会累加其值。	

			private final HashSet<Worker> workers = new HashSet<Worker>();
			
			private final Condition termination = mainLock.newCondition();
			
			private int largestPoolSize;
			
			private long completedTaskCount;

### 性能CAS & Lock

Executor尽可能追求使用CAS，只有必要的地方才会使用lock。

使用CAS的地方：

*	poolSize计数
* 	Executor的状态变迁

使用lock的地方：

*	workers的操作（包括LargestPoolSize、CompletedTaskCount）。
* 	获取LargestPoolSize值的操作。

### When to terminate

大前提是Executor被调用了shutdown方法，但是从行为中得知shutdown方法不会立即终止Executor，而是要经历几个过程：Queue中task消费完、Pool为空。

tryTerminate方法的目的在于确保条件到达后，Executor可以及时退出，因此在会触发条件的位置都调用了tryTerminate方法：

* shutdown、shutdownNow方法被调用
* queue的清理：purge、remove
* woker的清理：processWorkerExist、addWorkerFailed

### When to interrupt idel thread

interruptIdelThread意在清理空闲的可被关闭的线程，在可能引起线程清理的位置对其调用：

* shutdown被调用
* 设置CorePoolSize
* 设置MaximumPoolSize
* 设置KeepAliveTime
* AllowCoreThreadTimeout
* tryTerminate调用的同时

### Why is worker queue not concurrent collection

Executor中使用mainlock和HashSet来存储线程池内的worker，而不是使用Concurrent Collection。这样做可以强制对worker的操作是顺序进行的，避免造成中断风暴（interrupt storm），特别是在shutdown的时候。同时简化了largestThreadSize统计实现，因为线程的退出和加入也是顺序化进行的。

### keyword "volatile" useage in Executor
Executor中所有用户控制的参数都使用了“volatile”关键字，作者的注释是说处于执行的动作能够实时读到最新的数据，同时没有使用锁的必要——内部机制不需要立即对变化做出响应。
