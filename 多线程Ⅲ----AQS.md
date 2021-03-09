---
layout: post
title: 多线程Ⅲ----AQS
date: 2020-12-30 15:12:55
cover: /img/java.png
top_img: /img/post.jpg
tags:
- 并发
categories:
- java
description: java并发学
---
# 概述
本文用来描述并发包中AQS的原理,AbstractQueuedSynchronizer内部使用了双向队列实现了阻塞锁以及相关的同步器
## AQS数据结构
- 内部节点
{% asset_img 2020-12-30-16-17-58.png %}
```java
//这是AQS中的静态类
static final class Node {
 //两种模式独占和共享
 static final Node SHARED = new Node();   
 static final Node EXCLUSIVE = null;
 //5种状态常量,默认为0
 static final int CANCELLED =  1;
 static final int SIGNAL    = -1;
 static final int CONDITION = -2;
 static final int PROPAGATE = -3;
 //----属性变量----------
 Node nextWaiter; //和condition有关
 volatile int waitStatus; //当前节点状态
 volatile Node prev; //前驱
 volatile Node next; //后继
 volatile Thread thread; //持有的线程对象
    // VarHandle mechanics 这是VarHandle,内部通过native,操作指定变量的函数,包含了读写,volatile操作,以及compare-and-set操作
    // 也就是说节点内部状态的改变都是通过CAS处理的
 private static final VarHandle NEXT;
 private static final VarHandle PREV;
 private static final VarHandle THREAD;
 private static final VarHandle WAITSTATUS;
 //常量,用来表示节点的模式,赋值给waitWaiter变量
 static final Node SHARED = new Node();      
 static final Node EXCLUSIVE = null;
 static {
     try {
         MethodHandles.Lookup l = MethodHandles.lookup();
                NEXT = l.findVarHandle(Node.class, "next", Node.class);
                PREV = l.findVarHandle(Node.class, "prev", Node.class);
                THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
                WAITSTATUS = l.findVarHandle(Node.class, "waitStatus", int.class);
            } catch (ReflectiveOperationException e) {
                throw new ExceptionInInitializerError(e);
            }
        }
 
}/*  */
```
nextWaiter和节点模式:
  - 当nextWaiter为指向下一个等待条件的节点时候,这就是条件队列,即一个单向链表
  - 当nextWaiter=NODE.EXCLUSIVE,表示处于独享模式
  - 当nextWaiter=NODE.SHARED,表示处理共享模式


- AQS结构
```java
private transient volatile Node head; 
private transient volatile Node tail;
private volatile int state; //改变量表示AQS本身的资源,并非是节点状态

```
- 节点状态说明
 状态0:表示节点刚加入队列  
 状态1:表示节点取消,即请求资源的线程取消动作  
 ---
 ## AQS执行逻辑
 {% asset_img 多线程Ⅲ----AQS/2021-03-08-22-29-21.png %}
 AQS的实现类都会维护资源,当NODE能够`获得`资源时就会进行执行状态,否则就会处于阻塞状态
 ### 独占模式
 #### 获取
 {%codeblock lang:java %} 
   public final void acquire(int arg) {
        if (!tryAcquire(arg) && 
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    
    //----------------插入队列尾操作--------------------------------
    private Node addWaiter(Node mode) {
        Node node = new Node(mode); //指定节点的模式,此时为独享模式
        //这里采用自选加原子操作保证会将新节点放置到队尾
        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
                node.setPrevRelaxed(oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return node;
                }
            } else {
                initializeSyncQueue();//当队首为空时,初始化一个空节点
            }
        }
    }
     private final void initializeSyncQueue() {
        Node h;
        if (HEAD.compareAndSet(this, null, (h = new Node()))) //此时队首Node是一个虚拟节点,没有实际含义,但必须存在
            tail = h;
    }
    //----------------队列中的元素获取行为------------------
    //首先能够进入这个函数,说明当前节点之前必定存在前驱节点,可能是初始化的虚拟节点,
    //也可能是其他排队线程节点
    final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            //通过循环和内部的park将当前线程停留在此处
            for (;;) {
                //当且仅当前驱为head,即当且节点为第二个元素,并且能够获取到资源
                //将当前节点置为head,并释放
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                //若不满足上述条件则判断前驱的状态,是否应该直接park当前节点
                if (shouldParkAfterFailedAcquire(p, node))
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted) 
                selfInterrupt();  //此处只是调用了Thread.currentThread().interrupt(),保留了线程中断信息,AQS并不做任何处理,用户可进行判断并处理
            throw t;
        }
    }
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL) //若前驱为SIGNAL,说明前驱正常,则直接park当前节点即可
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
            //下两种状态都没有直接进入park,而是再次尝试获取资源
        if (ws > 0) { //若前驱为CANCELL状态,则将当前节点的pre和之前一个正常节点相连
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;//这里断开了正常节点的next,相当于会将那些cancelled节点GC
        } else { //此时前驱一定处于0或者-3,此时将前驱改成-1
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
        }
        return false;
    }
    //park,注意这里返回并清除了中断信号
     private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
 {%endcodeblock%}
 - 队列初始化时队列可能的情况
    - 线程A通过tryAcquire函数,队列为空;线程B进入,创建了 空NODE<---->线程BNODE,若此时线程A未释放,则线程Bpark
    - 当线程A释放之后,线程B会充当head,并且此时状态为0
 - 当后续线程进入shouldParkAfterFailedAcquire,说明前驱节点很多,或者head节点处于持有资源的状态
    - 当前驱为0,说明当前节点第一次进入此函数,置前驱为-1,并使当前节点再次尝试获取资源
    - 当前驱为1,说明当前节点第二次进入此函数,直接进入park状态
 #### 释放
{%codeblock lang:java%} 
    
    public final boolean release(int arg) {
        if (tryRelease(arg)) { //若能够返回资源
            Node h = head; //当且仅当head状态非0
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
{%endcodeblock%}
- release()函数过程中,若队首为null,说明在线程A获取到释放的过程中没有任意另外线程尝试获取过资源,那么AQS直接退出就行
- release()函数过程中,若head.waitStatus==0,说明有线程B进入到队列,但是在执行到shouldParkAfterFailedAcquire中修改pre.waitStatus=-1之前线程A就执行结束了
{%codeblock lang:java%} 
 private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
         //此处从后向前找的原因就是,AQS里next链除了出队的时候基本不变,也就是说下一个节点可能是取消的节点,
         //因此通过从后向前找可以找后继中第一个有效的节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }

{%endcodeblock%}
### 取消动作
AQS中取消发生于超时,或者异常,并不是对外的Api,如独占模式下获取acquireQueued函数过程中异常,则会取消该节点(即等待过程中出现异常)
{%codeblock lang:java%}
private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;

        node.thread = null;

        // Skip cancelled predecessors
        //修改目前要取消的节点pre为前驱中正常节点,也就是说取消的节点不会通过pre相连接
        //这样是为了在unpark过程中,从后向前找一定能够找到第一个有效的待激活节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary, although with
        // a possibility that a cancelled node may transiently remain
        // reachable.
        Node predNext = pred.next;

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        node.waitStatus = Node.CANCELLED;

        // If we are the tail, remove ourselves.
        if (node == tail && compareAndSetTail(node, pred)) {
            pred.compareAndSetNext(predNext, null);
        } else {
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && pred.compareAndSetWaitStatus(ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    pred.compareAndSetNext(predNext, next);
            } else {
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
{%endcodeblock%}
