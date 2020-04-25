---
title: java并发
cover: /img/java.png
top_img: /img/post.jpg
date: 2020-03-31 22:20:29
tags:
- 并发
categories:
- java
description: java并发学习
---
### juc
#### CAS
描述CAS是通过指令集完成的操作,一共为三个值,`L`表示内存中的值,`E`表示期待比较的值,`V`当`L=E`时将`V`替换内存值
- Atomic类
{%codeblock lang:java AtomicInteger%}
//valueOffset表示atomicInteger中value属性的偏移量
ublic final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }
//=================unsafe实现====================
public final int getAndSetInt(Object var1, long var2, int var4) {
       int var5;
       do {
           var5 = this.getIntVolatile(var1, var2);//获取当前内存值
       } while(!this.compareAndSwapInt(var1, var2, var5, var4));//CAS操作

       return var5;
   }    
{%endcodeblock%}
- AtomicMarkableReference
ABA问题:表示线程1修改变量A时,cas操作过程中,若线程2导致A->B->A,而线程1的判断发生在线程2的最后一部,那么就是ABA问题了
{%codeblock lang:java AtomicMarkableReference%}
private static class Pair<T> {
       final T reference;
       final boolean mark;
       private Pair(T reference, boolean mark) {
           this.reference = reference;
           this.mark = mark;
       }
       static <T> Pair<T> of(T reference, boolean mark) {
           return new Pair<T>(reference, mark);
       }
   }

   private volatile Pair<V> pair; //vlolatile禁止重排,线程可见

   public boolean compareAndSet(V       expectedReference,
                                 V       newReference,
                                 boolean expectedMark,
                                 boolean newMark) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedMark == current.mark &&
            ((newReference == current.reference &&
              newMark == current.mark) ||
             casPair(current, Pair.of(newReference, newMark)));
    }

    private boolean casPair(Pair<V> cmp, Pair<V> val) {
       return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
   }
{%endcodeblock%}
- 说明
  - 线程1执行到return语句,线程2发生了A->B->A,如果这个A.mark,A.ref不变,那么不会执行cas
    ,如果变了,那么就应该执行cas,这样就避免的aba问题.
  - 如果线程1执行到了||后边即cas语句,线程2发生了A-B->A,此时这个A不可能是线程A中的current,
  因为Pair.of是new出来的,这也就避免aba问题.
  - 总而言之,使用cas就是就是为了提高性能,而不是直接使用悲观锁   
