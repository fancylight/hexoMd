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
 