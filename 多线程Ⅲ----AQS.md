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
# AQS
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
 状态0:初始状态
 状态1:表示节点取消,即请求资源的线程取消动作  
 状态-1:表示自身释放时需要`唤醒`后继节点(此时后继节点未必陷入了park状态)
 状态-2:条件等待
 状态-3:只有head才有可能是这个状态,表示SHARE模式下,后继节点未进行shouldPark操作
 ---
 ## AQS执行逻辑
 {% asset_img 多线程Ⅲ----AQS/2021-03-08-22-29-21.png %}
 AQS的实现类都会维护资源,当NODE能够`获得`资源时就会进行执行状态,否则就会处于阻塞状态
 首先说明一下,AQS源码在jdk8,11,13中都有一些变化,这里以11为说明
 ps:
 - AQS中采用的LockSupport的park和unpark是采取标记使用的,并不是一定要先park,再unpark,unpark相当于设置了一个标志,park相当于消费该标志
 - 本文中采用`唤醒`表示unpark不是很合适,将这个操作理解为设置一次通行标志,线程是否能够通过acquire取决于`资源`是否能够获取
 - AQS中CLH队列节点状态由于多线程访问的原因,某一时刻状态可能存在多种情况,不是很好分析
 - AQS中dummy节点表示是第一个获取到资源的线程,队列第二位置是下一个能够获取到资源的线程节点
 ### 入队操作
 AQS中CLH队列使用pre来稳定的表示节点关系,next实际上是一种优化,思路是这样,当多个线程同时插入,通过循环加cas维护tail的正确性,此时新的节点对于不同线程
 来说就是局部变量,修改pre一定是正确
 {%codeblock lang:java%}
    private Node enq(Node node) {
        for (;;) { //循环
            Node oldTail = tail;
            if (oldTail != null) {
                node.setPrevRelaxed(oldTail); //1   1.8是node.pre=oldTail
                if (compareAndSetTail(oldTail, node)) { //2  cas操作
                    oldTail.next = node;//3
                    return oldTail;
                }
            } else {
                initializeSyncQueue();
            }
        }
    }
 {%endcodeblock%}
如下图:
{%asset_img enq.png%}
- 若当线程A,B竞争过程,插入代码执行到1,2之间,TAIL还是原来的队尾,此时队首.next==null,如果发生旧队尾唤醒操作,从前向后寻找就是null,但是这是很少发生的
{%codeblock lang:java%}
private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) { //这里next==null就是上述竞争导致的
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev) //
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
{%endcodeblock%}
- 当开始反向遍历的时候,`for (Node p = tail; p != node && p != null; p = p.prev)`,这里的p如果还是oldTail,说明线程A的插入过程未结束,那么唤醒它没有意义,unparkSuccessor的s==null,直接退出;若p=线程A,则说明线程A竞争过程的2步骤已经执行结束,直接`唤醒`线程A,由于park的性质,不用等待线程A的步骤3执行结束
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
 - addWaiter的逻辑:即插入指定模式的节点到队尾
    {%asset_img 初始化节点状态.png %}
    - 图1为线程A通过tryAcquire,获取资源,并未在队列中创建节点;当线程B执行tryAcquire失败时,则会创建代表线程A的虚拟节点和代表本身的节点,节点状态都为0
    - 图2表示存在三个线程,第一个线程在执行,第二个线程处于park状态或刚执行完入队函数,第三个线程为刚进入队尾的状态,处于这种状态的不是一个稳定状态
    - 由于AQS本身会被多线程调用,很多时候队列的状态是时刻发生改变的,如图二中第二个线程可能处于刚进入tail,并未park的时候,第三个线程也进入队尾
 - acquireQueued循环的作用:当前线程循环尝试获取资源,以及park
     {%asset_img 节点循环获取资源.png %}
    - 当图2中的队尾线程执行完shouldparkAfter函数之后,就会变成图3的情况
    - 当队首的next节点能够获取到资源时,就会执行出队操作,替代队首,并且
 -  shouldParkAfterFailedAcquire作用
    - 情况2:修正取消节点,将当前节点的pre连接到最前方正常的节点,并且取消正常前置的next
    - 情况3:即前置线程节点处于0或者-3,前者表示当前线程是刚进入队列第一次执行,那么就需要将前置节点置为-1,此时返回false;当前线程会尝试再次获取资源,如果不能获取,
    会进入情况1,进而park;也有可能会由于前置变为1,而进入取消节点修正
    - 情况1:即前置线程节点处于-1,第二次进入该函数,说明当前节点应该直接执行park函数,由于park的标志特性不用去管这个判断过程中前置唤醒和该节点park的顺序
    - 情况2和情况3都相当于进行了二次检查(可以理解为自我激活)
        {%asset_img 自我激活.png%}
    - 如图5所示,若存在线程A,B,如果线程A快速获取并释放资源,线程B未进入到shouldParkAfterFailedAcquire函数,那么在release中是不会执行唤醒后续线程的操作(因为队首waitStatus=0),
    那么此时线程B就需要自我唤醒,便出现了在shouldparkAfter函数中先执行情况3,然后再尝试获取资源,此时会将队首出队
    - 如图6所示,当线程A,B,C快速进入队列,并且B,C未执行should函数就会是这种状态,当B线程出现异常被取消后,会变成图6状态,此时如果C线程执行了should函数之后会变成图7状态,如果该过程中
    线程A释放了,那么就和图5的逻辑是一样,因此此时线程C也需要一个自我再次检查的过程      
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
- release()函数过程中,若head.waitStatus==0,说明有线程B进入到队列,但是在执行到shouldParkAfterFailedAcquire中修改pre.waitStatus=-1之前线程A就执行结束了,即线程A还没有执行过修改
前置线程的状态,那么他还会去执行should函数的3情况,还有机会再次去获取资源,因此该线程也可以安全释放
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
若不存在异常,没有节点处于取消状态,那么队列状态是比较规整的,如下:
{%asset_img 共享无取消状态图.png 共享无取消状态%}
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
        //这样是为了在unpark过程中,从后向前找一定能够找到第一个有效的待激活节点,和should中不同的是
        //没有改变任意节点的next状态
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
- 取消操作过程,节点一般处于三种位置,分别为队尾,队中,第二个位置(不为队尾)
- 取消操作的第一个步都是从维护当前节点的pre一定指向前驱有效节点,原因是AQS中虽然是多线程操作,但是队列一旦成立,不会随便有中间出队的操作,
这也是从后向前遍历的原因.
- 若为队尾,则会尝试直接删除队尾节点,但是此操作可能由于新的线程节点加入而失败,若失败,该节点会随着后继节点的should函数抛弃
  {%asset_img 队尾取消操作.png%}
  - 前两张图表示无新线程节点导致cas失败,那么后续过程就和一般流程一致
  - 后三张图表示当出现cas失败,那么取消节点的清除就要延迟到存在新线程节点执行shouldParkAfterFailedAcquire函数
- 若为第二个位置,则会直接尝试唤醒取消节点的后续节点,等待进入一次shouldParkAfterFailedAcquire,修正后续节点的pre正确性
- 若为队中,则尝试改变pred.waitstatus=-1,并且尝试连接pred.next为取消节点.next
### 共享模式
共享和独占模式的区别在于后者每次唤醒是不会唤醒后续节点,也就是如果由于资源不够而进入CLH队列的节点,每次只能存在一个线程被唤醒;
而前者则会尝试反复唤醒,直到资源不够使用
#### 获取
{%codeblock lang:java%}
 public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

  private void doAcquireShared(int arg) {
      //加入队列的是SHARED类型的节点
        final Node node = addWaiter(Node.SHARED);
        boolean interrupted = false;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) { 
                        //当处于第二位置的线程能够获取资源,会出队,并且尝试是否要唤醒后继
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node))
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        } finally {
            if (interrupted)
                selfInterrupt();
        }
    }  
{%endcodeblock%}
#### 释放
{%codeblock lang:java%}
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
 private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }    
{%endcodeblock%}
#### 共享状态下状态分析
##### 理想状态下
{%asset_img 一般流程中的共享状态图.png %}
- 假设资源为2,线程依次进入队列,并逐步释放,为上图
##### 共享模式下Propagate作用
{%asset_img 共享模式下Propagate作用.png%}
设置资源为2,可能会出现一种极端情况,假设没有状态Propagate的参与,[AQS1.73和1.74的改变](http://gee.cs.oswego.edu/cgi-bin/viewcvs.cgi/jsr166/src/main/java/util/concurrent/locks/AbstractQueuedSynchronizer.java?r1=1.73&r2=1.74)
- 状态1为时刻1,当A线程释放后,oldHead.waitStatus=0,并且会使线程C进入setHeadAndPropagate,进行出队操作,
- 若出队的操作未执行时,线程D开始了释放操作,并在它执行完成之前入队操作都没有结束,由于此刻线程D看到的head实际是oldHead,那么如果不存在Propagate状态,它所看到的就是0状态,那么应该退出,如状态2
- 到了状态3,线程C执行setHeadAndPropagate不会唤醒线程D(因为在线程C的执行过程中它所看到的剩余资源是0,即线程B和线程C持有,它没有观测到线程C的释放)
在加入了Propagate状态后,通过`  if (propagate > 0 || h == null || h.waitStatus < 0 ||(h = head) == null || h.waitStatus < 0) `,可以判读oldHead可能存在的0->-3的转变,以此观测到新的资源释放,
来唤醒后续资源.
- 这种状态可能会引起该节点永久挂起,假设状态3,再有一个节点进入队列,那么新节点是可以获取资源的,就会造成中间有节点没有被唤醒,线程C和线程D的节点会被出队,线程D还处于阻塞状态
-----
## 参考
[Propagate作用](https://blog.csdn.net/zy353003874/article/details/110535122)
