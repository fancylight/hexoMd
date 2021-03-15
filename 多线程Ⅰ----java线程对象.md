---
layout: post
title: 多线程Ⅰ----java线程对象
date: 2021-01-13 10:16:47
cover: /img/java.png
top_img: /img/post.jpg
tags:
- 并发
categories:
- java
description: java并发学习
---
# 概述
描述java线程(并不单单指代Thread类)的概念
## 线程状态
{% asset_img 2021-01-13-10-17-57.png %}
关于java线程和操作系统线程的对应关系
{% asset_img 2021-03-02-11-08-05.png %}
### 虚拟机线程概念和实际操作系统线程概念
由于jvm是构建在实际机器上,实际上java线程状态是不能直接等价于实际系统线程的状态  
1. Thread.State定义(JAVA源码的注释定义)
    - NEW:线程对象创建未执行
    - RUNNABLE:此时真正的系统线程可能是运行(RUNNING)或者是等待(WAITING),后者产生的原因如IO中断状态,只是对于JVM来说抽象成了RUNNABLE状态
    - BLOCKED:实际上系统线程线程处于等待(WAITING)状态,jvm底层关于sync使用了monitor队列,类似于AQS的实现,java层面只有`synchronized`关键字才会导致BLOCKED状态
    - WAITING:该状态的线程等于阻塞停留,需要通过中断协作机制才能恢复执行,如`(wait|notify)`,`(park|unpark)`,`join()`
    - TIMED_WAITING:带有时间形参的WAITINIG
2. JVM的WAITING和BLOCKED状态
简单来说后者在系统线程状态就是WAITING,只是JVM更加具体描述这是由于锁导致的阻塞状态,一般来说BLOCKED的线程需要抢占到锁才有可能进入RUNABLE状态,而WAITTING状态的线程则需要一个中断信号来将其主动唤醒
3. WAITING状态的转变
    - 由于Wait()导致的

    ```java
  /**
     * 用来测试线程唤醒之后的State
     * 结果如下:
     *  WAITING
     *  BLOCKED
     *  RUNNABLE
     *  唤醒线程释放了锁
     *  若停顿点A未注释,则线程1会处于BLOCKED状态
     * wait()会导致线程进行WAITING状态并且此时必须释放持有的锁,
     *
     *
     *
     */
    @Test
    public void waitAfterState() throws InterruptedException {
        Object object = new Object();
        Thread thread1 = new Thread(() -> {
            synchronized (object) {
                try {
                    object.wait();
                    System.out.println(Thread.currentThread().getState());
                    System.out.println("唤醒线程释放了锁");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        Thread thread2 = new Thread(() -> {
            synchronized (object) {
                System.out.println(thread1.getState());
                object.notify();
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(thread1.getState());
                //停顿点A
//               while (true) {
//
//               }

            }
        });
        thread1.start();
        Thread.sleep(1000);
        thread2.start();
        thread1.join();
    }
    ```
    - 由Park()导致

    ```java
      public void lockSupportAfterState() throws InterruptedException {
        Thread thread1 = new Thread(()->{
            LockSupport.park();
            System.out.println(Thread.currentThread().getState());
            System.out.println("线程恢复");
        });
        Thread thread2 = new Thread(()->{
            System.out.println(thread1.getState());
            LockSupport.unpark(thread1);
            System.out.println(thread1.getState());
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(thread1.getState());
        });
        thread1.start();
        Thread.sleep(1000);
        thread2.start();
        thread1.join();
    }
    ```
    即WAIT调用之后的线程会进入BLOCKED状态进行锁的抢占
 4. WAIT()的生产者,消费者模型
 简化一下使用逻辑: 首先消费者抢占锁,接着判断条件满足条件释放锁,不满足条件则wait,并且释放锁,陷入WAITING状态;此时由另一个生产者抢占(由于多数消费者都由于资源不够陷入了WAITTING,那么消费者线程自然能够获得锁),并且补充资源,唤醒WAITTING的消费者线程.
 5. javaIo和线程状态关系
    ```java
      /**
     *  问题1:当java使用阻塞io时,Thread的状态
     *      一般来说io阻塞时,线程应该陷入waiting状态,而jvm的Thread状态则处于runnable状态
     *  问题2:线程中阻塞式io,并不会响应{@link InterruptedException},如何协作处理
     *      通过另一个线程来检测(条件,用户自身定义,如io多久未响应,或者其他条件),如下t2线程
     */
    @Test
    public void ioBlockThreadState() throws IOException, InterruptedException {
        //[1] 创建Socket监听线程
        ServerSocket serverSocket = new ServerSocket(8920);
        Thread t1 = new Thread(()->{
            while (true) {
                Socket accept = null;
                try {
                    //这里可以看出来,此种阻塞式操作并没有相应中断异常,那么如果要通过协作处理
                    //此种线程的长时间未相应,就必须在其他线程中关闭IO
                    accept = serverSocket.accept();
                    System.out.println("检测成功");
                } catch (IOException e) {
                    System.out.println("出现io异常,退出该线程");
                    return;
                }

            }
        });
        t1.start();
        System.out.println("阻塞Io线程状态为:"+t1.getState());
        Assertions.assertEquals(t1.getState(),Thread.State.RUNNABLE);
        //[2] 创建一个计时器,若时间超过4s,直接中断Socket
        Thread t2 = new Thread(()->{
            Timer timer = new Timer();
            timer.schedule(new TimerTask() {
                @Override
                public void run() {
                    try {
                        serverSocket.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }, 4000);
        });
        t2.start();
        t1.join();
        System.out.println("主线程中止");
    }
    ```
---
## 线程组对象
线程组对象是java用来管理jvm的Thread一组数据结构,实质上是一颗树,根节点为jvm创建的,如下图所示.
{%codeblock  lang:cmd %}
        SYSTEM(tg对象)
  __________|____________
  |           |           |
  MAIN(tg对象) 系统线程1   系统线程2
  _____|________________________
  |             |               |
  SUBGROUP1    main(主线程)   SUBGROUP2
{%endcodeblock%}     
代码结构如下:
{%codeblock  lang:java %}
public
class ThreadGroup {
    Thread threads[];
    ThreadGroup groups[];
}
{%endcodeblock%}
1. 线程组的作用仅仅是为了管理Thread信息,并没有太多底层的作用,除了SYSTEM线程组除外,每个线程组都拥有父线程组
2. 任意线程都可以获得自身线程组信息,但是不能获得其父级和其他线程信息
3. 线程创建过程
    ```java
      ThreadGroup threadGroup = new ThreadGroup("test");
        Thread thread = new Thread(threadGroup, () -> {
            System.out.println("子线程组tg:  " + Thread.currentThread().getThreadGroup().getName());
            System.out.println("子线程组tg父:  " + Thread.currentThread().getThreadGroup().getParent().getName());
        });

        System.out.println("主线程组tg:  " + Thread.currentThread().getThreadGroup().getName());
        System.out.println("主线程组tg父:  " + Thread.currentThread().getThreadGroup().getParent().getName());
        thread.start();
        try {
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    ```
4. 线程组和线程异常处理机制
    {%codeblock  lang:java %}
     /**
     * 线程的异常处理机制
     * <p>
     * 通过debug可知,关系Thread的异常处理机制
     * 1.判断thread对象是否存在ExceptionHandler,存在则调用
     * 2.若不存在,则调用对应ThreadGroup的uncaughtException函数,因为该类本身就是ExceptionHandler
     * 2.1 ThreadGroup中逻辑是尝试调用Thread的defaultExceptionHandler
     */
    @Test
    public void threadExceptionTest() {
        Thread thread = new Thread(() -> {
            int a = 1 / 0;
        });
        thread.setUncaughtExceptionHandler((t, e) -> {
            System.out.println("异常");
        });
        Thread.setDefaultUncaughtExceptionHandler((t, e) -> {
            System.out.println("静态异常处理");
        });
        thread.start();
    }

    {%endcodeblock%} 
5. join()和wait()关系
    - wait必须在sync的前提下才能使用
        ```java
           //消费线程
           while(!condition) {
               obj.wait();
           }
           //生成线程
           if(!condition) {
               condition = true;
               obj.notify();
           }
           //明显可以看出来如果,当!condition成立时,必须保证消费者线程陷入wait,如果不采
           //取同步机制,此时cpu如果执行生产者线程则会无效的唤醒,那么消费者线程可能就入
           //了死循环状态

        ```
    - join函数本身就是wait(),其对应的notify在jvm调用,[参考](https://blog.csdn.net/shangguoli/article/details/108453551)    
        ```java
        public final synchronized void join(long millis)
    throws InterruptedException {  //本身是一个sync函数
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);  //暂停线程
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
        ```

---

## java中断协作机制
java的中断机制是进程间通信的一种方式,通过其他线程设置任意线程的中断变量,来通知该线程来进行`处理`,可以通过抛出中断异常,或者内部处理,或者将异常向上层继续抛出.
### java内部本身响应中断异常的Api,且会使线程陷入挂起状态的Api
- 例如Thread内部的会使得线程陷入阻塞状态的api都会抛出中断异常,调用应当捕获该异常,并处理或向上层抛出
    ```java
       run(){ //并非一定是用户自定义的Runnable,可以理解为任意一个线程
            //业务
            try {
            Thread.currentThread().sleep(100);
            } catch (InterruptedException e) {
                //1.此处处理
                //2.继续抛出,即让更上层的调用处理
                throw xxException();
            }
       }
    ```
上述无论是直接处理还是抛出,用户都是察觉到了中断,并作出了处理
### java内部不响应中断异常,且会导致线程陷入挂起状态的Api
- java内部并不会响应抛出中断异常的api,需要用户自我检查并作出处理
    {%codeblock lang:java AbstractQueuedSynchronizer#doAcquireSharedNanos %}
     private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return true;
                    }
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted()) //此处检验可能在park是由于中断而返回的情况
                    throw new InterruptedException();//处理方式是向上抛出异常
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    {%endcodeblock%}
## 并发相关概念   
- 可见性:可由volatile,同步等方式处理
- 原子性:同步,原子性操作(CAS)处理
- 有序性:和指令重排关系
java中volatile具有原子性和可见性的能力,通过内存屏障实现的
### 指令重排
- 定义:
    - 指令重排:编译器优化,处理器重排,内存系统重排
    - 指令重排仅仅保证单线程之间的语义正确性
- JMM的happens-before(JSR-133)    
这实际上就是java内存模型给程序员的保证,按照如此规则下的程序都是具有顺序性,可见性的
  - 程序顺序规则:单个线程中无论如何重排,结果不变
  - 锁规则:解锁位于加锁之后,java中加锁临界区重排对于后续线程是可见结果
  - volatile变量规则:可见性规则,该种变量的读取发生在写之后
  - 传递性:如果A happens-before B，且B happens-before C，那么A happens-before C
  - start()规则： 如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作
  - join()规则： 如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回
  - 中断规则:响应中断函数的操作,一定晚于中断操作
  - 对象终结规则:就是一个对象的初始化的完成，也就是构造函数执行的结束一定 happens-before它的finalize()方法
----
## 参考
**[Java 线程状态之 WAITING](https://my.oschina.net/goldenshaw/blog/802620)**  
**[java中的指令重排](https://xiaolyuh.blog.csdn.net/article/details/103289570)**  
**[volatile的使用场景](https://www.cnblogs.com/ouyxy/p/7242563.html)**