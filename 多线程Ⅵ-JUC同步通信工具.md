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