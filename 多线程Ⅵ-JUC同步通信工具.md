---
layout: post
title: 多线程Ⅵ-JUC同步通信工具
date: 2021-03-19 10:01:09
cover: /img/java.png
top_img: /img/post.jpg
tags:
- 并发
categories:
- java
description: 并发包部分结构
---
# JUC通信工具
## CountDownLatch
一般用来是线程A等待指定多个线程结束某种条件,使用较为简单
### 原理
内部是AQS实现,原理非常简单,内部实现SYNC
{%codeblock lang:java CountDownLatch%}
public class CountDownLatch {
     private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
        //Sync初始化时,默认接收state
        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c - 1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
    private final Sync sync;
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    //在state>0的情况,调用改函数,线程会进入wait状态
     public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    //减少state
     public void countDown() {
        sync.releaseShared(1);
    }
}
{%endcodeblock%}
## CyclicBarrier
用来实现多个线程之间相互等待,在某个点时停止,当所有线程到达该点后再开启所有线程,内部是使用Condition.wait实现的,`Cyclic`的语义是当一组线程使用完之后,`CyclicBarrier`可以复用,代码用例:
```java
class Solver {
    final int N;
    final float[][] data;
    final CyclicBarrier barrier;
 
    class Worker implements Runnable {
      int myRow;
      Worker(int row) { myRow = row; }
      public void run() {
        while (!done()) {//使用循环是为了说明CyclicBarrier当一组使用结束后可以再次使用
          processRow(myRow);
 
          try {
            //当最后一个线程执行到此处之前其他N-1要等待  
            barrier.await();
          } catch (InterruptedException ex) {
            return;
          } catch (BrokenBarrierException ex) {
            return;
          }
        }
      }
    }
 
    public Solver(float[][] matrix) {
      data = matrix;
      N = matrix.length;
      Runnable barrierAction = () -> mergeRows(...);
      barrier = new CyclicBarrier(N, barrierAction);
 
      List<Thread> threads = new ArrayList<>(N);
      for (int i = 0; i < N; i++) {
        Thread thread = new Thread(new Worker(i));
        threads.add(thread);
        thread.start();
      }
 
      // wait until done
      for (Thread thread : threads)
        thread.join();
    }
  }}
```
### 原理
简要来说就是内部使用`ReentrantLock`,以及`Condition`,当线程获取到锁时,减少计数器,当计时器>0时,线程陷入waiting,当最后一个线程修改计数器为0之后,就会调用`signalAll`唤醒所有等待线程
{%codeblock lang:java%}
public class CyclicBarrier {
    //为了实现组,当一组线程等待结束后会创建一个新的Generation
        private static class Generation {
            Generation() {}                 
            boolean broken;                
        }
        private final ReentrantLock lock = new ReentrantLock();//内部的锁
        private final Condition trip = lock.newCondition();//计数器不满足时的条件
        private final int parties;//表示一组线程的数量,此值固定
        private final Runnable barrierCommand;//当所有线程到达等待点后,可以执行的线程,可选参数
        private Generation generation = new Generation();//表示当前的一组线程
        private int count;//计算器,表示当前等待的线程
        
        //满足条件后的激活函数
        private void nextGeneration() {
        // 唤醒其他线程
        trip.signalAll();
        // 恢复条件,可以供下一组使用
        count = parties;
        generation = new Generation();
        //异常情况
        private void breakBarrier() {
            generation.broken = true;
            count = parties;
            trip.signalAll();
        }
    //实现等待的逻辑    
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock(); //获取锁
        try {
            final Generation g = generation;
            //异常处理
            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            //计数器减少
            int index = --count;
            //说明最后一个线程进入,尝试唤醒
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null) //可以执行一个命令线程
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction) //这过程如果出错则唤醒其他,并设置
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            // 若计数器未归零,说明最后一个线程未进入,则循环执行等待
            // 这里的循环只有超时和中断异常才会再次执行
            for (;;) {
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }        
    }
    //接收线程数量和等待点执行线程
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }    
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }    
}    
{%endcodeblock%}
## Semaphore
此工具用来控制同时能够执行的线程,如连接池限制同时并发执行的线程数,内部使用共享锁实现,代码用例如下:
```java
class Pool {
    private static final int MAX_AVAILABLE = 100;
    private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);
 
    public Object getItem() throws InterruptedException {
      available.acquire(); //当MAX_AVALIABLE+1的线程进入,并且前100个线程未执行putItem函数,此线程阻塞
      return getNextAvailableItem();
    }
 
    public void putItem(Object x) {
      if (markAsUnused(x))
        available.release();
    }
```
### 原理
这个类的结构和`ReentrantLock`一致,不再赘述
{%codeblock lang:java%}
public class Semaphore implements java.io.Serializable {
     private final Sync sync;
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;
        //这里将state当作了计数器
        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires; //一般的共享实现中都是增加state,此时则是减少,在于把 state等同于了计数器
                //当remaing=0,说明是最后一个线程进入,该线程可以执行,但是不会做任何唤醒操作
                //当remaing<0,说明超过permits的线程进入,那么就会排队在CLH中
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases; //增加state(计数器)
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }
        static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors()) //典型的公平锁
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
     public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
     public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
     public void release() {
        sync.releaseShared(1);
    }
}    
{%endcodeblock%}
## Phaser
该工具是JDK7提供的同步工具,可以替代`CountDownLatch`和`CyclicBarrier`,和后者相比它将能够随意注册和取消`parties`,在等待栅栏的时候,可以阻塞,也可以不阻塞.相对于`CyclicBarrier`,`CyclicBarrier`是所有的`parties`到达栅栏之后更新计数器,到达下一代`Generation`,并且每一代的参与者`parties`是不能变化的;对于`Phaser`,用`PHASE`这个概念表示阶段,并且会记录每个阶段的值(就是递增值),并且在不同的阶段,参与者可以不同.
### 数据结构分析
`Phaser`采用了父子结构,存在一个`root`节点,所有的新`Phaser`都持有`root`,并且指向其`Parent`,这样的作用是因为假设所有的操作都集中在通过一个`Phaser`,当有大量参与者`parties`的情况会导致内部Cas操作竞争激烈,因此采用如此的结构.
`Phaser`内部维护了两个单项队列,被称为`Treiber stack `无锁栈,由所有父子`Phaser`共享,`Phaser`内部wait是通过空转`onSpinWait`(JDK9),或者通过该结构实现的`LockSupport.park`
- 运行图示
```               
                   屏障A                  屏障B
    ThreadA          |           ThreadA   |
    ThreadB          |           ThreadB   |
    ThreadC          |           ThreadC   |
    ThreadD          |                     |

```
首先多个屏障这个行为并不是`Pahser`独特的,`CyclicBarrier`也能完成,只是后者每次参与者是一样的,举个例子,假设3个阶段,第一个有4个参与者,第二次一个参与者退出,第三次增加三个参与者.
```java
public class MultiplyPhasePhaserTest {
    static class Task implements Runnable {
        Phaser phaser;
        Phaser phaserMain;

        public Task(Phaser phaser, Phaser phaserMain) {
            this.phaser = phaser;
            this.phaserMain = phaserMain;
        }

        @Override
        public void run() {
            //阶段一
            System.out.println(Thread.currentThread().getName() + "--执行阶段" + phaser.getPhase());
            if (Thread.currentThread().getName().equals("1")) {
                phaser.arriveAndDeregister();
                phaserMain.arriveAndAwaitAdvance();
                return;
            } else {
                phaser.arriveAndAwaitAdvance();
            }
            //阶段二
            System.out.println(Thread.currentThread().getName() + "--执行阶段" + phaser.getPhase());
            phaser.arriveAndAwaitAdvance();
            //阶段三
            phaser.register();
            Thread thread = new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "--执行阶段" + phaser.getPhase());
                phaser.arriveAndAwaitAdvance();
            });
            thread.setName(Thread.currentThread().getName() + "XXX");
            thread.start();
            System.out.println(Thread.currentThread().getName() + "--执行阶段" + phaser.getPhase());
            phaser.arriveAndAwaitAdvance();
            //等待终结
            phaserMain.arriveAndAwaitAdvance();
        }
    }

    private static int TASK_COUNT = 4;

    @Test
    public void taskTest() {
        Phaser phaser = new Phaser(TASK_COUNT);
        Phaser phaser1 = new Phaser(1);
        for (int i = 0; i < TASK_COUNT; i++) {
            phaser1.register();
            Thread thread = new Thread(new Task(phaser, phaser1));
            thread.setName(i+"");
            thread.start();
        }
        phaser1.arriveAndAwaitAdvance();
    }
}
//某次的输出如下:
3--执行阶段0
2--执行阶段0
1--执行阶段0
0--执行阶段0
2--执行阶段1
0--执行阶段1
3--执行阶段1
2--执行阶段2
0--执行阶段2
3--执行阶段2
2XXX--执行阶段2
3XXX--执行阶段2
0XXX--执行阶段2
```
- 内部结构图示
{% asset_img 多线程Ⅵ-JUC同步通信工具/2021-03-25-13-19-12.png %}
```java
public class Phaser {



    //将long变量分成4部分,0-15表示未达到的parites,16-31表示parites总数,32-62表示phase当前代数,63表示phaser是否终止
    private volatile long state;

    private static final int  MAX_PARTIES     = 0xffff; //最大的parties数,16位
    private static final int  MAX_PHASE       = Integer.MAX_VALUE;//最大的phase
    private static final int  PARTIES_SHIFT   = 16; //计算parites时的移码
    private static final int  PHASE_SHIFT     = 32;//计算phase的移码
    private static final int  UNARRIVED_MASK  = 0xffff;      // to mask ints
    private static final long PARTIES_MASK    = 0xffff0000L; // to mask longs
    private static final long COUNTS_MASK     = 0xffffffffL;
    private static final long TERMINATION_BIT = 1L << 63;

    // some special values
    private static final int  ONE_ARRIVAL     = 1;
    private static final int  ONE_PARTY       = 1 << PARTIES_SHIFT;
    private static final int  ONE_DEREGISTER  = ONE_ARRIVAL|ONE_PARTY;
    private static final int  EMPTY           = 1; //注意空并不是0   

    //计算未到达数量,取低16位
    private static int unarrivedOf(long s) {
        int counts = (int)s;
        return (counts == EMPTY) ? 0 : (counts & UNARRIVED_MASK);
    }
    //计算总parties值
    private static int partiesOf(long s) {
        return (int)s >>> PARTIES_SHIFT;
    }
    //获取phaser
    private static int phaseOf(long s) {
        return (int)(s >>> PHASE_SHIFT);
    }
    //计算到达的parties,总-未到达
    private static int arrivedOf(long s) {
        int counts = (int)s;
        return (counts == EMPTY) ? 0 :
                (counts >>> PARTIES_SHIFT) - (counts & UNARRIVED_MASK);
    }
    private final Phaser parent;//父pahser
    private final Phaser root;//树结构的根
    //奇偶无锁栈
    private final AtomicReference<QNode> evenQ;
    private final AtomicReference<QNode> oddQ;            
}    
```
{% asset_img 多线程Ⅵ-JUC同步通信工具/2021-03-25-13-25-14.png %}
phaser内部结构可以是如此,需要使用父子结构,并且采用QNode作为阻塞方式
### 运行逻辑
Phaser的运行逻辑基本就是对于任意一个phaser,如果它内部的所有parites到达了屏障,如果是子`phaser`,则通知其父,递归执行,最终是root,则修改`state`的`phase`部分完成`advance`操作,并且会使所有阻塞的线程正确执行.
- 构造
```java
   public Phaser(Phaser parent, int parties) {
        if (parties >>> PARTIES_SHIFT != 0)
            throw new IllegalArgumentException("Illegal number of parties");
        int phase = 0;
        this.parent = parent;
        if (parent != null) { //存在parent,则设置一部分数据
            final Phaser root = parent.root;
            this.root = root;
            this.evenQ = root.evenQ;
            this.oddQ = root.oddQ;
            if (parties != 0)
                phase = parent.doRegister(1); //注意此处,parent仅仅注册一个
        }
        else {
            this.root = this;
            this.evenQ = new AtomicReference<QNode>();
            this.oddQ = new AtomicReference<QNode>();
        }
        this.state = (parties == 0) ? (long)EMPTY :
            ((long)phase << PHASE_SHIFT) |
            ((long)parties << PARTIES_SHIFT) |
            ((long)parties);
    }
```
当子phaser第一次注册(通过构造器或者第一次调用register)时,都会调用一次`parent.doRegister`,原因在于`Phaser`的机制是,分层结构,当子`Phaser`的所有任务达到屏障点时,会递归调用父类,那么从父类的角度来看它仅仅需要知道子`Pahser`就可以了,并不需要知道子`Pahser`的任务,所以是1.
```
      _______p__________
    |    |   |   |   |  |  
    s1  s2   s3  s4  s5 s6
```
假设`s1`到`s6`都是子`Pahser`,那么对于`p`节点来说,它的`parties`数量就是6,`s1`到`s6`自身的任务或者子`phaser`,p并不需要知道,这是一种递归
- 注册
```java
private int doRegister(int registrations) {
        // adjustment to state
        long adjust = ((long)registrations << PARTIES_SHIFT) | registrations;
        final Phaser parent = this.parent;
        int phase;
        for (;;) {
            //reconcileState函数的作用是同步子pahser和root的phase状态,实际上root的phase部分是最早更新的,而子pahse的phase部分是可以延后的
            long s = (parent == null) ? state : reconcileState();
            int counts = (int)s;
            int parties = counts >>> PARTIES_SHIFT;
            int unarrived = counts & UNARRIVED_MASK;
            //数量检测
            if (registrations > MAX_PARTIES - parties)
                throw new IllegalStateException(badRegister(s));
            phase = (int)(s >>> PHASE_SHIFT);
            if (phase < 0)
                break;
            if (counts != EMPTY) {                  // not 1st registration
                //counts不是EMPTY说明已经调用过一次Register了,此次调用并不是第一次
                //parent==null,说明是root调用不需要执行reconcileState函数
                if (parent == null || reconcileState() == s) {  
                    if (unarrived == 0)             // wait out advance,说明当前pahser的下的所有parites都到达了屏障点,因此需要进行阻塞等待操作
                        root.internalAwaitAdvance(phase, null);
                    else if (STATE.compareAndSet(this, s, s + adjust)) //正常修改
                        break;
                }
            }
            else if (parent == null) {// 1st root registration
                //这是最常见的情况,即root pahser第一次调用注册函数,仅仅修改root.state就可以
                long next = ((long)phase << PHASE_SHIFT) | adjust;
                if (STATE.compareAndSet(this, s, next))
                    break;
            }
            else {
                synchronized (this) {               // 1st sub registration
                    //子pahser加锁修改本身的状态
                    if (state == s) {               // recheck under lock
                        phase = parent.doRegister(1);
                        if (phase < 0)
                            break;
                        // finish registration whenever parent registration
                        // succeeded, even when racing with termination,
                        // since these are part of the same "transaction".
                        while (!STATE.weakCompareAndSet
                               (this, s,
                                ((long)phase << PHASE_SHIFT) | adjust)) {
                            s = state;
                            phase = (int)(root.state >>> PHASE_SHIFT);
                            // assert (int)s == EMPTY;
                        }
                        break;
                    }
                }
            }
        }
        return phase;
    }
        //实际上就是将root的pahse部分同步到当前pahse身上,并且重置了当前phaser的等待部分
        private long reconcileState() {
        final Phaser root = this.root;
        long s = state;
        if (root != this) {
            int phase, p;
            // CAS to root phase with current parties, tripping unarrived
            while ((phase = (int)(root.state >>> PHASE_SHIFT)) !=
                   (int)(s >>> PHASE_SHIFT) &&
                   !STATE.weakCompareAndSet
                   (this, s,
                    s = (((long)phase << PHASE_SHIFT) |
                         ((phase < 0) ? (s & COUNTS_MASK) :
                          (((p = (int)s >>> PARTIES_SHIFT) == 0) ? EMPTY :
                           ((s & PARTIES_MASK) | p))))))
                s = state;
        }
        return s;
    }
```
逻辑如下:
    1. 检测并非首次注册,尝试调整pahse(子),若当前phase的所有parites到达临界点,此注册操作需要等待pahse升级,否则正常修改
    2. 是root的首次调用则,正常修改
    3. 子phaser的首次则加锁,cas修改

- 等待阻塞
```java
public int arriveAndAwaitAdvance() {
        // Specialization of doArrive+awaitAdvance eliminating some reads/paths
        final Phaser root = this.root;
        for (;;) {
            long s = (root == this) ? state : reconcileState(); //重置子节点
            int phase = (int)(s >>> PHASE_SHIFT);
            if (phase < 0)
                return phase;
            int counts = (int)s;
            int unarrived = (counts == EMPTY) ? 0 : (counts & UNARRIVED_MASK);
            if (unarrived <= 0)
                throw new IllegalStateException(badArrive(s));
            //到达一个,修改state    
            if (STATE.compareAndSet(this, s, s -= ONE_ARRIVAL)) {
                if (unarrived > 1) //非最后一个到达
                    return root.internalAwaitAdvance(phase, null);
                if (root != this) //最后一个到达,但是当前phaser不是root,则递归通知父
                    return parent.arriveAndAwaitAdvance();
                //root pahser直接子通过,则开始调用onAdvance    
                long n = s & PARTIES_MASK;  // base of next state
                int nextUnarrived = (int)n >>> PARTIES_SHIFT;
                if (onAdvance(phase, nextUnarrived)) //若该函数返回ture,则表示pahser要终止
                    n |= TERMINATION_BIT;
                else if (nextUnarrived == 0)
                    n |= EMPTY;
                else
                    n |= nextUnarrived;
                int nextPhase = (phase + 1) & MAX_PHASE;
                //修改state,进行phase的升级
                n |= (long)nextPhase << PHASE_SHIFT;
                if (!STATE.compareAndSet(this, s, n))
                    return (int)(state >>> PHASE_SHIFT); // terminated
                releaseWaiters(phase); //释放由于node而等待的线程
                return nextPhase;
            }
        }
    }  
```
逻辑如下:
    1. 任意非phaser到达会修改未到达数量
    2. 当当前parties全部到达则通知parent,若无parent则说明是root,则修改root的state的phase以及重置它的paeties部分,子pahser的state重置会延迟到子调用regiester或者wait相关函数通过reconcileState处理
    3. 调用releaseWaiters,释放那些由于node阻塞的线程
- 阻塞的实现
```java
private int internalAwaitAdvance(int phase, QNode node) {
        // assert root == this;
        releaseWaiters(phase-1);          // ensure old queue clean
        boolean queued = false;           // true when node is enqueued
        int lastUnarrived = 0;            // to increase spins upon change
        int spins = SPINS_PER_ARRIVAL;
        long s;
        int p;
        while ((p = (int)((s = state) >>> PHASE_SHIFT)) == phase) { //可以看到,如果root的state被改变了,那么自选的线程就会突破这个循环
            if (node == null) {           // spinning in noninterruptible mode
                //假设调用的不是带有中断的api,如arriveAndAwaitAdvance,则是通过自选等待,而不是无锁栈
                int unarrived = (int)s & UNARRIVED_MASK;
                if (unarrived != lastUnarrived && //说明由于新的到达者的
                    (lastUnarrived = unarrived) < NCPU) //如果此次未达到的人小于cpu逻辑数,则增加自选次数
                    spins += SPINS_PER_ARRIVAL;
                boolean interrupted = Thread.interrupted();
                if (interrupted || --spins < 0) { // need node to record intr,如果自旋数<0,则开始使用无锁栈
                    node = new QNode(this, phase, false, false, 0L);
                    node.wasInterrupted = interrupted;
                }
                else
                    Thread.onSpinWait(); //空函数,被hotspot优化
            }
            else if (node.isReleasable()) // done or aborted , 判断节点是否应该被释放
                break;
            else if (!queued) {           // push onto queue   新的节点压栈
                AtomicReference<QNode> head = (phase & 1) == 0 ? evenQ : oddQ; //根据phase取奇偶队列
                QNode q = node.next = head.get();
                if ((q == null || q.phase == phase) &&
                    (int)(state >>> PHASE_SHIFT) == phase) // avoid stale enq
                    queued = head.compareAndSet(q, node);
            }
            else {
                try {
                    ForkJoinPool.managedBlock(node); //阻塞,ForkJoinPool先不研究,在Pahser类中只是为了调用了QNODE.block进行阻塞
                } catch (InterruptedException cantHappen) {
                    node.wasInterrupted = true;
                }
            }
        }

        if (node != null) {
            if (node.thread != null)
                node.thread = null;       // avoid need for unpark()
            if (node.wasInterrupted && !node.interruptible)
                Thread.currentThread().interrupt();
            if (p == phase && (p = (int)(state >>> PHASE_SHIFT)) == phase)
                return abortWait(phase); // possibly clean up on abort
        }
        releaseWaiters(phase);
        return p;
    }

    //作用是将和当前phase不同的节点唤醒,并且将内部thread变量置为为空
    private void releaseWaiters(int phase) {
        QNode q;   // first element of queue
        Thread t;  // its thread
        AtomicReference<QNode> head = (phase & 1) == 0 ? evenQ : oddQ;
        while ((q = head.get()) != null &&
               q.phase != (int)(root.state >>> PHASE_SHIFT)) {
            if (head.compareAndSet(q, q.next) &&
                (t = q.thread) != null) {
                q.thread = null;
                LockSupport.unpark(t);
            }
        }
    }
```
逻辑如下:
`internalAwaitAdvance`函数在任何一个子phaser都是`root.internalAwaitAdvance`调用,为的就是当`root.state`的升级之后,可以把那些自选或阻塞的线程解开