---
layout: post
title: 多线程Ⅳ-----基于AQS-Condition
date: 2021-03-12 09:39:10
cover: /img/java.png
top_img: /img/post.jpg
tags:
- 并发
categories:
- java
description: 条件协作机制
---
# Condition
JDK中使用ConditionObject(AQS)中内部类来实现,实际上和`Object.wait|Object.notify`,都是线程协作机制
- 将AQS中CLH队列成为`同步队列`,将ConditionObject中单向队列称为`等待队列`
- 需要在获取lock的情况使用,说明调用该api是当前线程一定是AQS中,因此该线程会对应同步线程中的队首节点
- 内部维护了一个单向的等待队列,当线程陷入wait状态便会进入该队列,该队列中元素实际上就是AQS中的Node
核心思想就是,等线程获取lock之后,执行wait操作,会创建新的Node加入等待队列,同时relase同步队列队首,并尝试自身陷入park;当
其他线程调用signal操作,会将等待线程队首出队,同时将该Node加入同步队列队尾,那么如果之前park的线程会由于其前驱的释放而被唤醒.
## 数据结构
{%codeblock lang:java%}
    public class ConditionObject implements Condition, java.io.Serializable {
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
    }    
{%endcodeblock%}
- 等待队列使用了Node.nextWait当作下一个连接点
- 等待队列中不使用Node.pre和Node.next
- Node.waitStatus基本处于-2,0,1
{%asset_img 等待队列简单图示.png%}
## 执行逻辑
### await
{%codeblock lang:java%}
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter(); //加入新的Node到等待线程队尾
            int savedState = fullyRelease(node);//释放锁
            int interruptMode = 0;
            //此处循环判断node是否存在于同步队列中,如果存在,说明没有线程执行signal操作,
            //那么就将当前线程park
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                //如果由于中断退出,则不再参与循环
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) 
                    break;
            }
            //当线程恢复,则需要继续参与AQS同步竞争,只有通过竞争才能真正从await函数结束
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
        //入队操作
            private Node addConditionWaiter() {
            if (!isHeldExclusively()) //这里操作和Object.wait相似,判断是否锁正确
                throw new IllegalMonitorStateException();
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters(); //清除等待队列中取消的节点
                t = lastWaiter;
            }

            Node node = new Node(Node.CONDITION); //创建新的节点,加入队列

            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
          //释放锁
          final int fullyRelease(Node node) {
        try {
            int savedState = getState();
            if (release(savedState)) //如果该步骤不能释放锁,那么就说明出了问题,直接取消等待,置为取消状态
                return savedState;
            throw new IllegalMonitorStateException();
        } catch (Throwable t) {
            node.waitStatus = Node.CANCELLED;
            throw t;
        }
    }

    //---------该函数中AQS中的函数,用来判断任意node是否是同步队列中的元素--------------------
      final boolean isOnSyncQueue(Node node) {
          //condition好理解, node.prev==null的判断是因为,在同步队列中,任意状态下
          //都不会尝试修改prev=null这个操作
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
          //若存在后继说明一定是同步队列中的节点  
        if (node.next != null) // If has successor, it must be on queue
            return true;
          //遍历判断
        return findNodeFromTail(node);
    }
{%endcodeblock%}
### Signal操作
{%codeblock lang:java%}
  public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

          private void doSignal(Node first) {
             
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);  //尝试出队队首,只要成功,便退出
        }

           private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null); //不断出队,一直等到等待队列为空
        }

           final boolean transferForSignal(Node node) {
        //如果CAS失败,说明该等待节点可能成为了取消节点,那么跳过该节点
        if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
            return false;
        Node p = enq(node); //将等待节点加入同步队列队尾,并返回前驱
        int ws = p.waitStatus;
        //如果等待前驱为取消或者置为-1失败,则主动唤醒node
        if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL)) 
            LockSupport.unpark(node.thread);
        return true;
    }
{%endcodeblock%}
### 简易图示
{%asset_img 出入队伍操作.png%}
- 图一,时刻t1AQS存在线程A,线程B
- 图二,时刻t2,由于线程A,B分别获取lock,并且将调用await,将自身加入了等待队列
- 图三,时刻t3,线程C加入队列,并且等待获取锁
- 图四,时刻t4,线程C获得锁,并且调用了signal操作,从而将A节点加入了同步线程,当线程C执行release时,会将线程A唤醒
从而继续执行await中代码,使得A线程重新参与AQS竞争
### 原生Wait和LOCK.AWAIT使用差别
共性:
- 都需要在获取锁的情况下使用(这是协作机制从逻辑上的保证)
- 都可以响应中断
考虑以下情况,分别使用wait和Condition分别实现阻塞队列
{%codeblock lang:java 基于wait的阻塞队列%}
public class SimpleWaitBlockArray<T> {
    private volatile int size;
    private Object[] arrays = new Object[100];//默认最大值为100

    public T take() {
        T object = null;
        synchronized (arrays) {
            try {
                while (size == 0) {
                    arrays.wait();
                }
                object = dequeue();
                arrays.notify();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return object;
    }

    public void put(T ele) {
        synchronized (arrays) {
            try {
                while (size == 100) {
                    arrays.wait();
                }
                enqueue(ele);
                arrays.notify();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}    
{%endcodeblock%}
下边为JDK的ArrayBlockQueue实现
{%codeblock lang:java ArrayBlockQueue%}
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;

 public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    public void put(E e) throws InterruptedException {
        Objects.requireNonNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }  
}    
{%endcodeblock%}
可以明显看到原生的结构必须是
```java
  synchronized (obj) {
      obj.wait();
      //....
      obj.notify();
  }
```
这个结构在上边的实现中所带来的问题是,假设当所有生产者处于waitting状态,如果此时任意生产被唤醒,
向队列里插入了数据,并调用notify,此时并不能保证唤醒的是消费者,缺乏效率,也就说jdk原生结构过于死板,不够灵活.
反而使用AQS结构的Condition可以做到唤醒指定方.