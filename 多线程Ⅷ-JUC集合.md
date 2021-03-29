---
title: 多线程Ⅷ-JUC集合
cover: /img/java.png
top_img: /img/post.jpg
date: 2021-03-23 18:43:51
tags:
- 并发
categories:
- java
description: JUC集合
---
# 概述
{% asset_img 2021-03-26-19-26-46.png %}
## 队列实现
{% asset_img 2021-03-27-15-13-26.png %}
其中除了`ConcurrentLinkedQueue`和`ConcurrentLinkedDeque`,其他都是阻塞队列
### 阻塞队列
阻塞队列函数表
||抛出异常|返回特殊值|阻塞|超时阻塞|
|--|---|---|---|---|
|插入|add(e)|offer(e)|put(e)|offer(e,time,unit)|
|移除|remove()|poll()|take()|poll(time,unit)|
|检测|element()|peek()|---|---|
JUC中默认实现如下:
{% asset_img 2021-03-27-15-03-18.png %}
#### 有界阻塞队列
- `ArrayBlockingQueue`,`LinkedBlockingQueue`,`LinkedBlockingDeque`,`SynchronousQueue`都是有界阻塞队列,`LinkedBlockingQueue`默认容量为`Integer.Max`在使用上近似于无界队列,是一种可选有界阻塞队列.    
- 前两者放在一起是因为内部实现方式一致,区别在于前者的内部是通过数组实现,后者是通过节点实现,这里产生的问题就是对于数组结构,`take`和`put`操作是互斥的,而链表结构的入队操作是对于tail的操作,出队是对head的操作,因此`take`和`put`可以同时进行.    
- `LinkedBlockingDeque`则将内部take|put退化成了`ArrayBlockQueue`的单锁结构.    

##### ArrayBlockQueue
```java
    final Object[] items; //内部结构
    读写互斥锁
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;  

      public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) //空等待
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

        private E dequeue() {
        // assert lock.isHeldByCurrentThread();
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E e = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal(); //唤醒put队列
        return e;
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
    private void enqueue(E e) {
        // assert lock.isHeldByCurrentThread();
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = e;
        if (++putIndex == items.length) putIndex = 0;
        count++;
        notEmpty.signal();
    }    
  
```
`ArrayBlockQueue`结构简单,属于普通的有界阻塞队列,读写是互斥的,关于两个Condition解释在[原生Wait和LOCK-AWAIT使用差别](/2021/03/12/多线程Ⅳ-基于AQS-Condition/#原生Wait和LOCK-AWAIT使用差别)
##### LinkedBlockingQueue
该结构内部采用链表实现,结构本复杂,但是采取了一些特别的处理方式提高读写性能.
```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    //内部节点            
    static class Node<E> {
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        Node<E> next;

        Node(E x) { item = x; }
    }   
    private final int capacity;

    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

    //该链表结构使用了哨兵节点,有一个虚拟头
    /**
     * Head of linked list.
     * Invariant: head.item == null
     */
    transient Node<E> head;

    /**
     * Tail of linked list.
     * Invariant: last.next == null
     */
    private transient Node<E> last;

    //这里该结构使用了两个锁,因此读写可以同时进行,读读和写写之间是互斥的
    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();             
        }            

    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }   
//-------------阻塞读操作----------------------      
    public E take() throws InterruptedException {
        final E x;
        final int c;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue(); //基本的出队操作
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }
//-------------阻塞写操作----------------------------    
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        final int c;
        final Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node); //基本的入队操作
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }    
}        
```
上述链表结构的take和put和数组结构最大的不同是这部分,以take说明
```java
       takeLock.lockInterruptibly();
       try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue(); //基本的出队操作
            //操作1
            c = count.getAndDecrement();//返回此次cas操作之前的count值,即最终-1之前的值
            //操作2
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        //操作3
        if (c == capacity)
            signalNotFull();//该函数获取写锁,并且唤醒所有被阻塞的写线程
```
这些操作的原因都是读写可以同时进行,两把锁导致的,假设按照array的实现方式,采取一把锁效率效率低,采取两把锁,则不能简单的使用读写互相激活的方式(操作2的作用),解释如下:
1. A,B为读线程,C,D为写线程,假设A被阻塞,此时count==0
2. 同时B,C线程执行,由于C先,因此B通过了,接着D,E线程进入,在B执行操作1的过程中,D增加,那么操作2返回的c==2
3. 此时A线程应该实际可以被唤醒,所以线程B执行了操作2,如果将这种情况的唤醒交予c,d写线程那么可能会有新的读线程存在等导致读锁,而线程B就是已经获取读锁的线程,那么它来激活是效率最高的
操作3的行为,c表示的是该线程执行-1操作前的值,那么说明在该读线程执行过程中,**可能**存在写线程被阻塞,因此此处尝试唤醒
关于take的操作是一致,这几个操作的目的都是读写是可以同时进行的,读线程唤醒读线程,尝试使容器向着0趋近,写线程相反,并且读写不会随意唤醒对方,仅仅是保证对方不会因为全部阻塞而导致无法唤醒的情况
#### 无界阻塞队列
##### PriorityBlockingQueue
基于最大堆实现的优先阻塞队列,`PriorityQueue`的逻辑和这个一致,关于堆参考[堆和堆排序](https://time.geekbang.org/column/article/69913)
```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    private transient Object[] queue;
    private transient int size;
    private transient Comparator<? super E> comparator; //比较元素之间大小
    //这里就能看出来优先队列和ArrayBlockQueue一致是读写互斥
    private final ReentrantLock lock = new ReentrantLock();
    //由于无界,因此写线程不会阻塞,因此只需要一个Condition
    private final Condition notEmpty = lock.newCondition();        
    //读        
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        E result;
        try {
            while ( (result = dequeue()) == null)
                notEmpty.await();
        } finally {
            lock.unlock();
        }
        return result;
    }    
    }        
    public void put(E e) {
        offer(e); // never need to block
    }
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        final ReentrantLock lock = this.lock;
        lock.lock();
        int n, cap;
        Object[] es;
        while ((n = size) >= (cap = (es = queue).length))
            tryGrow(es, cap);
        try {
            final Comparator<? super E> cmp;
            //最大堆向上堆化操作
            if ((cmp = comparator) == null)
                siftUpComparable(n, e, es);
            else
                siftUpUsingComparator(n, e, es, cmp);
            size = n + 1;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
        return true;
    }    
```
##### DelayQueue
基于`PriorityQueue`实现的延迟阻塞队列
```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    private Thread leader; //当前最前排队的线程
    private final Condition available = lock.newCondition();                
    }  

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek(); //查看优先队列堆顶元素
                if (first == null) //队列为空,直接阻塞
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS); //判断是否延迟
                    if (delay <= 0L) //无延迟则直接返回
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null) //若此时有线程阻塞在  available.awaitNanos(delay),说明其在等待first的超时,那么说明那个任务未处理,因此该线程应该陷入等待
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
    //写线程不会阻塞
    public void put(E e) {
        offer(e);
    }
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) { //说明当前加入的元素就是堆顶元素,那么就唤醒等待的
            读线程
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }                  
```
##### SynchronousQueue
该队列是一种配对机制的阻塞队列,在消费者和生产者之间配对,当任意一方无法有配对者时则会陷入阻塞,适用于生产速度和消费速度近似的场景,若有一方线程过快,则会导致另一方阻塞.采用了成为`dual stack and dual queue algorithms`的无锁算法,分别使用栈和双向队列实现内部结构
该队列提供了公平模式和非公平模式,这影响匹配的获取结果顺序,原理在于内部的数据结构实现,如下测试:
```java
    @ParameterizedTest
    @ValueSource(booleans = {true,false})
    public void synchronousQueueTest(boolean fair) throws InterruptedException {
        System.out.println("模式为:"+(fair?"公平":"非公平"));
        SynchronousQueue<String> synchronousQueue = new SynchronousQueue<>(fair);
        Phaser phaser = new Phaser(1);
        phaser.register();
        new Thread(() -> {
            try {
                synchronousQueue.put("A");
                phaser.arrive();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        Thread.sleep(1000);
        new Thread(() -> {
            try {
                synchronousQueue.put("B");
                phaser.arrive();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        //消费者
        Thread.sleep(1000);
        new Thread(() -> {
            try {
                System.out.println(synchronousQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        phaser.arriveAndAwaitAdvance();
    }
//-------输出如下----------------
模式为:公平
A
模式为:非公平
B    
```
###### 结构和栈实现
```java
public class SynchronousQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
        //两种结构的抽象接口
    abstract static class Transferer<E> {
        abstract E transfer(E e, boolean timed, long nanos);
    }           
    }        
    //构造,非公平模式使用栈,非公平使用队列
    public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }
    //阻塞队列函数
    public E take() throws InterruptedException {
        E e = transferer.transfer(null, false, 0); //都是调用transfer函数
        if (e != null)
            return e;
        Thread.interrupted();
        throw new InterruptedException();
    }    
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        if (transferer.transfer(e, false, 0) == null) {
            Thread.interrupted();
            throw new InterruptedException();
        }
    }
```
该函数并未使用AQS锁,也没有使用sync关键字,内部是还是使用的cas自选,和`phaser`内部无锁栈一样都是自己实现
###### 栈结构TransferStack
- 内部结构
```java
    static final class TransferStack<E> extends Transferer<E> {
        //内部节点
        static final class SNode {
            volatile SNode next;        // 下一个元素
            volatile SNode match;       // 与之匹配的节点
            volatile Thread waiter;     // 阻塞的线程,和无锁栈结构一致,都是通过自旋空转先,如果不行则采用park
            Object item;                // 真正的数据
            int mode;   //模式表示消费者或生产者,还有一个临界状态即匹配状态    
            //匹配函数
            boolean tryMatch(SNode s) {
                if (match == null &&
                    SMATCH.compareAndSet(this, null, s)) {
                    Thread w = waiter;
                    if (w != null) {    // waiters need at most one unpark
                        waiter = null;
                        LockSupport.unpark(w);
                    }
                    return true;
                }
                return match == s;
            }                           
        }                    
        //表示消费者状态        
        static final int REQUEST    = 0;
        //表示生产状态
        static final int DATA       = 1;
        //该节点和另一个互补未匹配节点可以匹配
        static final int FULFILLING = 2;
        //实际上节点mode 会处于 10  11   0  1  (二进制) 四种 ,高位1标记该几点为FULFILLING状态
        static boolean isFulfilling(int m) { return (m & FULFILLING) != 0; }        
        //栈顶
        volatile SNode head;        
    }        
```
简单描述该栈的工作方式,当通过take或者put调用队列时,分别会产生mode=0和mode=1的节点,假设能够遇到当前栈顶为互补元素,则尝试匹配,并且将二者出栈;如栈顶为相同元素,则入栈,并且阻塞,原理看着简单,但是由于多线程的原因,这个栈状态也是不好分析,如AQS和无锁栈一样
- transfer实现
工作原理和刚才描述得一致,但是这个函数考虑很多多线程调用时的临界状态,因此就变得复杂了很多
```java
 E transfer(E e, boolean timed, long nanos) {
            /*
             * Basic algorithm is to loop trying one of three actions:
             *
             * 1. 若栈顶为空或栈顶为相同元素,则入栈等待,若栈顶为取消元素则将栈顶出栈,并重新循环
             *
             * 2. 若栈顶为互补元素,则进行匹配,并将两者出栈.出栈这个动作可能会由于多线程而由3触发.
             *
             * 3. 若栈顶就是一个10或11元素,则将两者出栈(help fulfill)
             */

            SNode s = null; // constructed/reused as needed
            int mode = (e == null) ? REQUEST : DATA; //明显take等函数调用为REQUEST

            for (;;) {
                SNode h = head;
                //情况一:空栈或相同模式
                if (h == null || h.mode == mode) {  // empty or same-mode  
                    if (timed && nanos <= 0L) {     // can't wait ,当poll函数调用时timed=ture&&nanos=0,不等待
                        if (h != null && h.isCancelled())
                            casHead(h, h.next);     // pop cancelled node,将取消的节点出栈
                        else
                            return null;    //说明此次poll或者offer操作之前也是相同性质操作,因此直接退出
                    } else if (casHead(h, s = snode(s, e, h, mode))) { //put或take操作,直接入栈
                        SNode m = awaitFulfill(s, timed, nanos); //自选或阻塞等待
                        //节点被唤醒
                        //m实际上返回的是互补节点,只有取消的情况m == s,默认为null,互补成功就应该为另一个节点
                        if (m == s) {               // wait was cancelled 由于节点取消,因此清楚该节点以及.next后续的取消节点
                            clean(s);
                            return null;
                        }
                        //这里说明当前节点和其互补节点,即此刻的栈顶没有被出队,因此此处出队
                        if ((h = head) != null && h.next == s)
                            casHead(h, s.next);     // help s's fulfiller
                        return (E) ((mode == REQUEST) ? m.item : s.item); //返回,注意如果当前节点为0,说明m为1,那么m.item就是一个元素;如果当前节点为1,那么就应该返回null,实际上s.item即它自身就是null
                    }
                    //情况二:栈顶不是一个fulfinglling节点,但是和当前互补
                } else if (!isFulfilling(h.mode)) { // try to fulfill 判断
                    if (h.isCancelled())            // already cancelled
                        casHead(h, h.next);         // pop and retry
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                        for (;;) { // loop until matched or waiters disappear
                            SNode m = s.next;       // m is s's match
                            //mark:特殊清除栈操作
                            if (m == null) {        // all waiters are gone
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                break;              // restart main loop
                            }
                            SNode mn = m.next;
                            //s和s.next尝试匹配,成功就尝试将两者出栈
                            if (m.tryMatch(s)) {
                                casHead(s, mn);     // pop both s and m
                                return (E) ((mode == REQUEST) ? m.item : s.item);
                            } else                  // lost match 当m节点发生如中断导致tryMatch的cas失败,就会这样,那么就需要将m请出,并重新开始循环
                                s.casNext(m, mn);   // help unlink
                        }
                    }
                    //情况三:栈顶已经是一个fulfilling节点,这种情况只有并发的时候才会出现,即情况二还在进行当中,由于节点加入
                } else {                            // help a fulfiller
                    SNode m = h.next;               // m is h's match
                    //mark:特殊清除栈操作
                    if (m == null)                  // waiter is gone
                        casHead(h, null);           // pop fulfilling node
                    else { 
                        //这里和情况二的操作一致
                        SNode mn = m.next;
                        if (m.tryMatch(h))          // help match
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }
//-----------------阻塞的实现----------------------------------------
  SNode awaitFulfill(SNode s, boolean timed, long nanos) {
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Thread w = Thread.currentThread();
            int spins = shouldSpin(s)
                ? (timed ? MAX_TIMED_SPINS : MAX_UNTIMED_SPINS)
                : 0;
            for (;;) {
                if (w.isInterrupted())
                    s.tryCancel(); //注意,如果发生中断,那么就取消该节点
                SNode m = s.match;
                if (m != null)
                    return m;
                if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) { //超时等待也会取消该节点
                        s.tryCancel();
                        continue;
                    }
                }
                if (spins > 0) { //和无锁栈一样的自选
                    Thread.onSpinWait();
                    spins = shouldSpin(s) ? (spins - 1) : 0;
                }
                //park操作
                else if (s.waiter == null)
                    s.waiter = w; // establish waiter so can park next iter
                else if (!timed)
                    LockSupport.park(this);
                else if (nanos > SPIN_FOR_TIMEOUT_THRESHOLD)
                    LockSupport.parkNanos(this, nanos);
            }
        }    
            //SNODE的函数,这个函数会和 tryMatch由于中断而发生竞争
            void tryCancel() {
                SMATCH.compareAndSet(this, null, this);
            }        
//----------------------匹配逻辑----------------------------------
            boolean tryMatch(SNode s) {
                //匹配并不是自选cas操作,仅仅尝试一次,如果这个过程由于中断导致this节点(阻塞节点)被唤醒而执行的tryMatch,那么该函数就会返回false
                if (match == null &&
                    SMATCH.compareAndSet(this, null, s)) {
                    Thread w = waiter;
                    if (w != null) {    // waiters need at most one unpark
                        waiter = null;
                        LockSupport.unpark(w);
                    }
                    return true;
                }
                return match == s;
            }
```
解释`mark:特殊清除栈操作`,为何会有清除栈和s.next == null的情况发生,如下:
{%asset_img TransferStack清除栈.png%}
代码对应:
```java
  for (;;) { // loop until matched or waiters disappear
                            SNode m = s.next;       
                            //mark:特殊清除栈操作
                            if (m == null) {       
                                casHead(s, null);   //注意由于casHead会判断head == s是否成立,所以这个清栈操作是安全的
                                s = null;         
                                break;              
                            }
                            SNode mn = m.next;
                         
                            if (m.tryMatch(s)) {
                                casHead(s, mn);     
                                return (E) ((mode == REQUEST) ? m.item : s.item);
                             //当m节点发生如中断导致tryMatch的cas失败,就会这样,那么就需要将m请出,并重新开始循环,假设mn就是null,那么再次进入循环就会发生上述情况,   
                            } else                  
                                s.casNext(m, mn);   
                        }
```