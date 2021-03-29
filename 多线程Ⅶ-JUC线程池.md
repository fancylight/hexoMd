---
title: 多线程Ⅶ---JUC线程池
cover: /img/java.png
top_img: /img/post.jpg
tags:
- 并发
categories:
- java
description: JUC线程池
---
# JUC线程池
`ThreadPoolExecutor`以及分治使用的`ForkJoinPool`,接口如下:
{% asset_img 2021-03-22-10-14-43.png %}
核心接口是`Executor`,这里涉及到`Runnable`,`Future`,`Callable`,,`FutureTask`的使用
{%codeblock lang:java AbstractExecutorService%}
//-------------所有的submit最终都是封装RunableFutre,之后调用execute----------------
 public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }    
{%endcodeblock%}
- 一般使用submit是为了获取线程执行的结果
  - callable直接调用
    ```java
    public void test() throws ExecutionException, InterruptedException {
      ExecutorService executor = Executors.newSingleThreadExecutor();
      Future future = executor.submit(()->{
         return 1;
      })
      Assertion.asserEqual(future.get(),1);
    }
    ```
   - FutureTask使用,该接口是`Future`和`Runnable`的接口,额外提供了判断线程是否执行结束的函数
        ```java
            public void test() throws ExecutionException, InterruptedException {
                ExecutorService executorService = Executors.newSingleThreadExecutor();
                FutureTask futureTask = new FutureTask(()->1);
                executorService.execute(futureTask);
                Assertions.assertEquals(futureTask.get(),1);
            }
        ```
## ThreadPoolExecutor实现
### 数据结构
该线程池内部维护一个`worker`工作线程集合,用来消耗由`Runnable`组成的阻塞队列`workQueue`,并且将`worker`分为核心线程和非核心线程,前者在从`workQueue`中取任务时会调用`take()`,后者会调用`poll()`,当任意时刻没有获取任务,则工作线程退出,即remove一个`worker`,明显能看出来核心线程正常情况会阻塞,非核心线程才会退出.
{%codeblock lang:java ThreadPoolExecutor%}   
    //----------------说明------------------------
    //ctl为32(可以扩展到long)位数字,使用前三位表示线程池状态,后29位表示worker数量,即线程池内部运行的线程数量     
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3; //29位,用来移位操作
    private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;//workerCout掩码
    //从RUNNING----->TERMINATED状态,数字字面值主键增大
    private static final int RUNNING    = -1 << COUNT_BITS; //111000...0000,数值是一个负数
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    
    private static int runStateOf(int c)     { return c & ~COUNT_MASK; } //计算状态
    private static int workerCountOf(int c)  { return c & COUNT_MASK; }  //计算ws
    private static int ctlOf(int rs, int wc) { return rs | wc; } //封装一个新的ctl

    //c一般是调用时读取的线程池ctl,s是状态的其中之一,用来判断当前线程池状态是否比s更加向RUNNING趋近
    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }
    //判断比s更加向TERMINATED趋近
    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }
    //检测是否运行
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }

    //这两个函数Atomic内部没有循环,调用者如果要保证执行,那么要自己循环使用
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

  
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

    //内部存在循环
    private void decrementWorkerCount() {
        ctl.addAndGet(-1);
    }
    //任务队列
    private final BlockingQueue<Runnable> workQueue;    
    //出现互斥操作时使用,如改变线程池状态,ctl,worker时
    private final ReentrantLock mainLock = new ReentrantLock();    
    //工作线程集合
    private final HashSet<Worker> workers = new HashSet<>();    
    private final Condition termination = mainLock.newCondition();
    //表示当前最大的工作线程数,理解为size
    private int largestPoolSize;    
    //完成任务数   
    private long completedTaskCount;    
    //用来创建worker内部Thread的工厂
    private volatile ThreadFactory threadFactory;    
    //runnable添加失败时执行的handler
    private volatile RejectedExecutionHandler handler;    
    //
    private volatile long keepAliveTime;    
    //ture表示核心线程在获取任务时,也调用poll,那么核心线程和非核心一样都会被remove
    private volatile boolean allowCoreThreadTimeOut;    
    //最大核心线程
    private volatile int corePoolSize;   
    //最大工作线程数 = 最大核心线程数+非核心线程 
    private volatile int maximumPoolSize;    
}    
{%endcodeblock%}
### 核心逻辑
- execute
    {%codeblock lang:java%}
    public void execute(Runnable command) {
            if (command == null)
                throw new NullPointerException();
            /*
            * Proceed in 3 steps:
            *
            * 1. 当线程池正常且woker数量小于核心数量,则尝试向wokers中添加一个
            * 含有初始任务的worker,添加函数可能会失败,由于多线程操作,也许线程池
            * 状态会改变,woker数量也会改变.
            *
            * 2. 如1未通过,则尝试将command加入任务队列,若入队成功,则再次判断线程
            * 池状态,若不正常则出队,并拒绝该任务;若未出队,且工作线程为空,则创建一个
            * 不带初始任务的非工作线程
            *
            *
            * 3. 若2未通过,即无法入队,则创建一个含有初始任务的非工作线程,否则拒绝此任务
            */
            int c = ctl.get();
            if (workerCountOf(c) < corePoolSize) {
                if (addWorker(command, true))
                    return;
                c = ctl.get();
            }
            if (isRunning(c) && workQueue.offer(command)) { //注意此处使用的时offer,不是put,因此不会阻塞
                int recheck = ctl.get();
                if (! isRunning(recheck) && remove(command))
                    reject(command);
                else if (workerCountOf(recheck) == 0)
                    addWorker(null, false);
            }
            else if (!addWorker(command, false))
                reject(command);
        }
    {%endcodeblock%}
- addWorker:添加工作线程
    {%codeblock lang:java%}
        private boolean addWorker(Runnable firstTask, boolean core) {

            //1. 判断能否加入到worker中,核心线程去和corePoolSize比较,非核心和maximumPoolSize比较,
            //for循环是为了处理clt的cas操作
            retry:
            for (int c = ctl.get();;) {
                // Check if queue empty only if necessary.
                if (runStateAtLeast(c, SHUTDOWN)
                    && (runStateAtLeast(c, STOP)
                        || firstTask != null
                        || workQueue.isEmpty()))
                    return false;

                for (;;) {
                    if (workerCountOf(c)
                        >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                        return false;
                    if (compareAndIncrementWorkerCount(c))
                        break retry;
                    c = ctl.get();  // Re-read ctl
                    if (runStateAtLeast(c, SHUTDOWN))
                        continue retry;
                    // else CAS failed due to workerCount change; retry inner loop
                }
            }
            //2.添加worker,并启动
            boolean workerStarted = false;
            boolean workerAdded = false;
            Worker w = null;
            try {
                w = new Worker(firstTask);
                final Thread t = w.thread;
                if (t != null) {
                    final ReentrantLock mainLock = this.mainLock;
                    mainLock.lock();//加锁
                    try {
                        int c = ctl.get();
                        //再次检查线程池状态,成功则加入worker到set中
                        if (isRunning(c) ||
                            (runStateLessThan(c, STOP) && firstTask == null)) {
                            if (t.getState() != Thread.State.NEW)
                                throw new IllegalThreadStateException();
                            workers.add(w);
                            workerAdded = true;
                            int s = workers.size();
                            if (s > largestPoolSize)
                                largestPoolSize = s;//记录当前最大工作线程数量
                        }
                    } finally {
                        mainLock.unlock();
                    }
                    if (workerAdded) {
                        t.start(); //启动Worker#thread.start()
                        workerStarted = true;
                    }
                }
            } finally {
                if (! workerStarted)
                    addWorkerFailed(w);
            }
            return workerStarted;
        }
    {%endcodeblock%}
    - worker:内部工作线程原理
    {%codeblock lang:java%}
    public class ThreadPoolExecutor extends AbstractExecutorService {
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {   
        //内部真正执行的线程,实际只是执行Worker#run函数
        final Thread thread;
        //初始任务
        Runnable firstTask;
        //该工作线程完成了多少次任务
        volatile long completedTasks;        
        
        Worker(Runnable firstTask) {
            setState(-1); // 注意此处将AQS的资源数设置为-1
            this.firstTask = firstTask; //设置初次任务
            this.thread = getThreadFactory().newThread(this);//创建执行线程
        }    

        //下边的逻辑和重入锁的互斥逻辑类似
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

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }    
    }
    }
    {%endcodeblock%}
- runWorker:真正调用逻辑
当一个新的worker被加入集合后,调用内部Thread.start后会正式执行该函数
{%codeblock lang:java%}    
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); //在此处还是可以被别的线程进行中断
        boolean completedAbruptly = true;
        try {
            //此处的getTask就是用来决定工作线程是否退出的
            //1.非工作线程或者允许核心线程退出的情况,没有在一定时间内获得task,则会退出线程
            //2.如果在getTask队列过程中,线程被中断,则会退出(中断操作会导致此处条件不成立,从而退出线程)
            while (task != null || (task = getTask()) != null) { 
                //获取锁,防止其他线程中断该线程
                w.lock();
                // (1||(2&&3))&4 --,此处的逻辑要和shutdown结合来看
                //这里可能会发生shutdown和此处的并发执行,很有可能发生
                //1. (即(ture||(3&&4))&&true),shutdown设置了状态,但是后续是不可能调用中断(因为锁),因此需要自我中断
                //2. 当(flase||(ture||ture))&&true,说明线程池执行了shutdown,但是由于此处获得lock,shutdown函数无法设置中断,因此需要自我设置
                //但是无论如何此次的任务还是会执行下去,下一次getTask就会中断异常,从而退出线程
                if ((runStateAtLeast(ctl.get(), STOP) ||  //1 
                     (Thread.interrupted() &&  //2
                      runStateAtLeast(ctl.get(), STOP))) && //3
                    !wt.isInterrupted()) //4
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    try {
                        task.run(); //执行任务,环绕了一些调用点
                        afterExecute(task, null);
                    } catch (Throwable ex) {
                        afterExecute(task, ex);
                        throw ex;
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
    //删除工作线程
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }


    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();

            //1. 如果线程池处于SHUTDOWN状态,并且队列还有剩余,可以执行
            //2. 如果线程池处于STOP,则不能处理队列
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize; //这里决定了是否采用take还是poll

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //这两个函数都是可以响应中断的
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
{%endcodeblock%}
- `Worker`实现AQS的原因,以及内部不使用重入锁的原因
```
private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                //这里尝试获取锁,若通过才中断线程
                //1.假设Worker加上了锁,那么此处必然不会通过,防止了线程工作过程中被中断
                //2.此处w内部没有使用重入锁的原因就是,如果runWork内部调用了该函数(beforeExecute)可以调用,
                //由于不是重入锁实现,此处也不会通过,也防止了线程工作过程中被中断
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
```
### 线程池退出
- shutdown
```java
   public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess(); //检查权限
            advanceRunState(SHUTDOWN);//cas循环设置shutdown状态
            interruptIdleWorkers();//给所有未获得lock的worker设置中断标志
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

       private void checkShutdownAccess() {
        // assert mainLock.isHeldByCurrentThread();
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkPermission(shutdownPerm);
            for (Worker w : workers)
                security.checkAccess(w.thread);
        }
    }

     private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }

    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) { //只有worker未获得lock的情况下才能设置中断,那么这个中断会被worker自己设置
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

```
### 线程池状态说明
- RUNNING:正常
- SHUTDOWN:该状态下,无法添加新的任务,但是可以添加新的工作线程(即不能添加还有初始任务的工作线程);可以继续处理任务
- STOP:该状态,无法添加工作线程;也不能继续处理任务
```java
//-------------addWorker---------------
private boolean addWorker(Runnable firstTask, boolean core) {
     retry:
        for (int c = ctl.get();;) {
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP)
                    || firstTask != null
                    || workQueue.isEmpty()))
                return false;
                //........
        }
}    
private Runnable getTask() {
          boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();

            // Check if queue empty only if necessary.
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            //..........
        }
}    
```
### ThreadPoolExecutor的三种常见使用
使用`Executors.newxx`这个api除了`newWorkStealingPool`和`newScheduledThreadPool`实际上就是创建参数不同的`ThreadPollExecutor`,构造器如下
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
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler= handler;
    }
```
- threadFactory的作用是用来创建Thread,该thread会调用Woker中的run函数从而执行runWork,假设需要获取线程异常信息,可以通过重写一个threadFactory
    ```java
    //这是Exectuors中默认实现之一,并没有给线程增加异常处理
    private static class DefaultThreadFactory implements ThreadFactory {
            private static final AtomicInteger poolNumber = new AtomicInteger(1);
            private final ThreadGroup group;
            private final AtomicInteger threadNumber = new AtomicInteger(1);
            private final String namePrefix;

            DefaultThreadFactory() {
                SecurityManager s = System.getSecurityManager();
                group = (s != null) ? s.getThreadGroup() :
                                    Thread.currentThread().getThreadGroup();
                namePrefix = "pool-" +
                            poolNumber.getAndIncrement() +
                            "-thread-";
            }

            public Thread newThread(Runnable r) {
                Thread t = new Thread(group, r,
                                    namePrefix + threadNumber.getAndIncrement(),
                                    0);
                if (t.isDaemon())
                    t.setDaemon(false);
                if (t.getPriority() != Thread.NORM_PRIORITY)
                    t.setPriority(Thread.NORM_PRIORITY);
                //可以加上异常处理
                t.setUncaughtExceptionHandler(()->{
                    //捕捉异常,将信息输出,或者保存下来
                })    
                return t;
            }
        }
    ```
- newCachedThreadPool:适用于任务耗时非常短的场景
    ```java
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
            return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                        60L, TimeUnit.SECONDS,
                                        new SynchronousQueue<Runnable>(),
                                        threadFactory);
        }
    ```
`corePoolSize`为0,`keepAliveTime`为60秒,`blockQueue`为`SynchronousQueue`,使用过程中不会创建核心线程,并且任务队列最多只能由一个任务,若任务队列被阻塞,则会伴随着非核心线程的创建,每个非核心线程的阻塞等待时间是60s,适用场景就是不确定线程数量,每个任务非常短的情况使用
- newFixedThreadPool:固定工作线程
    ```java
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
            return new ThreadPoolExecutor(nThreads, nThreads,
                                        0L, TimeUnit.MILLISECONDS,
                                        new LinkedBlockingQueue<Runnable>(),
                                        threadFactory);
        }
    ```
`corePoolSize`和`maximumPollSize`相同,阻塞队列为无界阻塞队列`LinkedBlockingQueue`,不会创建非核心线程,并且核心线程正常情况不会退出,任务队列是无界的,适用场景是用户要确定任务量,和线程数量
- newSingleThreadExecutor:单一线程
    ```java
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
            return new FinalizableDelegatedExecutorService
                (new ThreadPoolExecutor(1, 1,
                                        0L, TimeUnit.MILLISECONDS,
                                        new LinkedBlockingQueue<Runnable>(),
                                        threadFactory));
        }
    ```
仅仅创建一个核心线程,任务队列采用无界阻塞队列,当然当该线程退出(异常),会有新的核心工作线程来替代它,适用于执行顺序任务.
## ScheduledExecutorService
JUC中实现的用来延迟执行和定时执行的线程池
{% asset_img 多线程Ⅶ-JUC线程池/2021-03-23-19-40-48.png %}
```java
//延迟执行,无结果
public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);
//延迟执行,有结果
 public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);                                       
//延迟第一个,周期执行之后的,无结果
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);   
//每个任务调用直接存在延迟                                                         
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);                                                                                   
```
简单使用:
```java
  @Test
    public void ScheduledThreadPoolExecutorTest() throws ExecutionException, InterruptedException {
        ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(4);
        AtomicInteger count = new AtomicInteger();
        //延迟执行
        ScheduledFuture<Integer> schedule = scheduledExecutorService.schedule(count::incrementAndGet, 5, TimeUnit.SECONDS);
        Assertions.assertEquals(schedule.get(),1);
        //周期执行
        scheduledExecutorService.scheduleAtFixedRate(()-> System.out.println(count.incrementAndGet()), 2, 2, TimeUnit.SECONDS);
        Thread.currentThread().sleep(1000*16);
    }
```
## FutreTask实现原理
### 无锁栈Treiber stack
通过CAS循环实现的最简单的栈,可以进行并发操作,并且能够保证线程安全,结构就是循环取旧值,cas操作失败再循环执行,这本身就是一个数据结构,并不是起到像AQS替代锁的作用.
```java
public class FutureTaskTest {
    //实现一个无锁栈
    static class Node<T> {
        private Object object;
        private Node next;

        public Node(Object object) {
            this.object = object;
        }
    }

    static class NonLockStack<T> {
        private AtomicReference<Node<T>> head =new AtomicReference<>();
        private AtomicInteger atomicInteger =new AtomicInteger();

        public void put(T ele) {
            Node<T> tempNode = null;
            Node newNode = new Node(ele);
            do {
                //明显可以看出来这个循环内部就是临界区,由于CAS循环的作用,这个区域是线程安全的
                //如果此次有第二个成员变量,如size,这个for循环没有办法处理,该操作,原因是cas只能一次处理一个变量,无法将两个变量同时处理

                tempNode = head.get();
                newNode.next = tempNode;
            } while (!head.compareAndSet(tempNode, newNode));
            //此处内部和上班while循环的实质是一样的
            atomicInteger.getAndIncrement();
        }
    }

    @Test
    public void pushTest() {
        CountDownLatch countDownLatch = new CountDownLatch(1000);
        NonLockStack<Integer> nonLockStack = new NonLockStack<>();
        for (int i = 0; i < 1000; i++) {
            int finalI = i;
            new Thread(() -> {
                nonLockStack.put(finalI);
                countDownLatch.countDown();
            }).start();
        }
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Assertions.assertEquals(nonLockStack.atomicInteger.get(), 1000);
    }
}
```
### FutureTask内部结构
```java
public class FutureTask<V> implements RunnableFuture<V> {
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
    private Callable<V> callable; //真正的任务
    //最终结果,该变量没有被volatile修饰是因为happend-before,state的volatile传递性,和AQS能够保证可见性的原因一致
    private Object outcome; // non-volatile, protected by state reads/writes            
    private volatile Thread runner;//执行命令的线程
    private volatile WaitNode waiters;//无锁栈
}    
```
### 运行逻辑
- run:`FutureTask`被线程池`Worker`调用时真正的任务执行的逻辑
```java
  public void run() {
      //判断状态
        if (state != NEW ||
            !RUNNER.compareAndSet(this, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call(); //调用任务,等待返回值
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex); //异常结束,设置异常状态
                }
                if (ran)
                    set(result); //正常结束,设置正常结束状态
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

     protected void setException(Throwable t) {
        if (STATE.compareAndSet(this, NEW, COMPLETING)) {  //call返回后,未最终设置状态时,为ing状态
            outcome = t;
            STATE.setRelease(this, EXCEPTIONAL); // final state
            finishCompletion();//唤醒那些由于get函数而阻塞的线程
        }
    }

     protected void set(V v) {
        if (STATE.compareAndSet(this, NEW, COMPLETING)) {
            outcome = v;
            STATE.setRelease(this, NORMAL); // final state
            finishCompletion();
        }
    }
```
- get:通过无锁栈结构阻塞线程
```java
 public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING) //等待状态,尝试阻塞
            s = awaitDone(false, 0L); 
        return report(s);
    }


private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        long startTime = 0L;    // Special value 0L means not yet parked
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            int s = state;
            if (s > COMPLETING) { //返回结果状态
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) //让出cpu等待状态设置完毕
                // We may have already promised (via isDone) that we are done
                // so never return empty-handed or throw InterruptedException
                Thread.yield();
            else if (Thread.interrupted()) { //中断了则可能从park状态通过,出队,并且抛出中断异常
                removeWaiter(q);
                throw new InterruptedException();
            }
            else if (q == null) { //创建node
                if (timed && nanos <= 0L)
                    return s;
                q = new WaitNode();
            }
            else if (!queued) //压栈
                queued = WAITERS.weakCompareAndSet(this, q.next = waiters, q);  //无锁栈入队操作
            else if (timed) { //超时等待
                final long parkNanos;
                if (startTime == 0L) { // first time
                    startTime = System.nanoTime();
                    if (startTime == 0L)
                        startTime = 1L;
                    parkNanos = nanos;
                } else {
                    long elapsed = System.nanoTime() - startTime;
                    if (elapsed >= nanos) {
                        removeWaiter(q);
                        return state;
                    }
                    parkNanos = nanos - elapsed;
                }
                // nanoTime may be slow; recheck before parking
                if (state < COMPLETING) 
                    LockSupport.parkNanos(this, parkNanos);
            }
            else
                LockSupport.park(this); //阻塞
        }
    }
    //出队操作
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!WAITERS.compareAndSet(this, q, s))
                        continue retry;
                }
                break;
            }
        }
    }    
```
- 唤醒和完成操作:
```java
    private void finishCompletion() {
        // assert state > COMPLETING;
        //唤醒所有等待的线程
        for (WaitNode q; (q = waiters) != null;) {
            if (WAITERS.weakCompareAndSet(this, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done(); //回调函数

        callable = null;        // to reduce footprint
    }

```
## ForkJoinPool
## CompletionService
考虑一下场景,使用线程池创建多个任务,并且要获取任务结果,那么代码如下:
```java
  static class FutureTask1 implements Callable<Integer> {
        int result;

        public FutureTask1(int result) {
            this.result = result;
        }

        @Override
        public Integer call() throws Exception {
            synchronized (this) {
                this.wait(1000);
            }
            return result;
        }
    }


public void futureTest() {
        for (int i = 0; i < FUTURE_TASK_SIZE; i++) {
            taskList.add(executorService.submit(new FutureTask1(i)));
        }
        taskList.forEach(f-> {
            try {
                Object result = f.get();
                doSomething(result);//假设存在一个任务执行时间过长,那么就会阻塞后边的任务
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
    }
```
假设任务`FutureTask1`的每次调用占用时间不同,那么for循环结构必须按照顺序等待任务结束,若一个任务执行时间过程则会导致后边的任务阻塞等待,那么如果采用轮询和超时等待方式:
```java
  @Test
    public void futurePollTest() {

        for (int i = 0; i < FUTURE_TASK_SIZE; i++) {
            taskList.add(executorService.submit(new FutureTask1(i)));
        }
        //轮询
        while (!taskList.isEmpty()) {
            Iterator<Future<Integer>> iterator = taskList.iterator();
            while (iterator.hasNext()) {
                try {
                    Future<Integer> next = iterator.next();
                    next.get(0, TimeUnit.SECONDS);
                } catch (InterruptedException e) {
                    System.out.println("等待过程线程被中断,退出");
                    return;
                } catch (ExecutionException e) {
                    e.printStackTrace();
                    System.out.println("数据获取过程发生异常,删除该任务");
                    iterator.remove();
                    continue;
                } catch (TimeoutException e) {
                    System.out.println("等待超时,重新尝试");
                    continue;
                }
                iterator.remove();
            }
        }
    }
```
这样也是不能解决上述问题,假设每次获取iterator的时候去打乱list的顺序,也不是一个好方法,因为打乱的执行没有逻辑,为了解决该问题,使用`CompletionService`
### 接口定义
```java
public interface CompletionService<V> {

    Future<V> submit(Callable<V> task);
    Future<V> submit(Runnable task, V result);
    //获取执行完成的结果,若未有完成的则阻塞
    Future<V> take() throws InterruptedException;
    Future<V> poll();
    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
}
```
由`take`和`poll`就可以看出来,这个玩意内部使用了阻塞队列实现,通过submit设置任务,当任务完成进入完成阻塞队列(生产者),调用线程使用`take().get()`来获取最新完成的(消费者),例子如下:
```java
  @Test
    public void completionServiceTest() {
        CompletionService<Integer> completionService = new ExecutorCompletionService(executorService);
        for (int i = 0; i < FUTURE_TASK_SIZE; i++) {
            completionService.submit(new FutureTask1(i));
        }
        for (int i = 0; i < FUTURE_TASK_SIZE; i++) { //再非异常的情况下,应该执行FUTURE_TASK_SIZE次
            try {
                System.out.println(completionService.take().get()); //此处永远会获取最新完成的
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
    }
```

### ExecutorCompletionService实现
该类实现非常简单,就是使用阻塞队列实现
```java
public class ExecutorCompletionService<V> implements CompletionService<V> {
    private final Executor executor; //实际的执行器
    private final AbstractExecutorService aes;
    private final BlockingQueue<Future<V>> completionQueue; //阻塞队列

    private static class QueueingFuture<V> extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task,
                       BlockingQueue<Future<V>> completionQueue) {
            super(task, null);
            this.task = task;
            this.completionQueue = completionQueue;
        }
        private final Future<V> task;
        private final BlockingQueue<Future<V>> completionQueue;
        protected void done() { completionQueue.add(task); } //注意此处将真正的task封装进了阻塞队列
    }    
    public ExecutorCompletionService(Executor executor,
                                     BlockingQueue<Future<V>> completionQueue) {
        if (executor == null || completionQueue == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = completionQueue;
    }    
    public Future<V> submit(Callable<V> task) {
        if (task == null) throw new NullPointerException();
        //封装创建Future,让线程池执行
        RunnableFuture<V> f = newTaskFor(task);
        executor.execute(new QueueingFuture<V>(f, completionQueue));
        return f;
    }    
    //阻塞队列消费
    public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }

    public Future<V> poll() {
        return completionQueue.poll();
    }    
```


----
[CompletionService参考](https://segmentfault.com/a/1190000023587881)