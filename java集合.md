---
title: java集合
date: 2019-01-14 10:39:17
tags: 数据结构|算法
categories: java
---
### 架构
**immutable Object:**
1.final 修饰
2.如String这种,内部pro被final修饰,本身没有直接修改该属性的能力
3.jcf中强调是否为可变对象,即改变自身会产生新的对象,从而不影响数据本身的对象为不可变对象,如string
{%asset_img CollectionInterface.png 接口%}
**view 视图**
java中默认都是引用逻辑,因此在集合实现中,数据实际是引用,迭代器,sub等方法返回的数据属于引用值,没有进行值拷贝,
jdk将这种称之为视图.
##### Collection接口
- **概述:**
1.collection不作为直接父类
2.juc中collection子类都包含了一个空构造器和有参构造器
3.子类不支持的操作,抛出  UnsupportedOperationException
4.某些子类显示某些元素的,如对null,或特定元素的限制
5.子类决定线程同步策略
6.collection中许多函数是通过Object::equals判断的
7.集合递归遍历自身可能会失败
- **视图集合 View Collections**
定义:视图集合指的是本身不储存元素,由其后备集合存储元素,对于视图集合的修改会显现在源集合中
如: Collections.checkedCollection, Collections.synchronizedCollection, and Collections.unmodifiableCollection返回的包装集合; List.subList, NavigableSet.subSet, or Map.entrySet;以及迭代器.
- **不可修改集合 Unmodifiable Collections**
若集合不支持修改,如add操作,那么称为不可修改集合,若对此集合执行上述操作,则抛出异常或不进行任何修改;其视图集合应该符合相同性质
根据注释所言,Unmodifiable Collections不一定是immutable,若不可修改集合包含了可变元素,那么当元素"mutator"时,集合本身就不是immutable,若集合内部元素是immutable元素那么该集合就是ummutable
{%asset_img Colleciton.png%}
{%codeblock lang:java collection%}
/*
* 该函数的逻辑是通过generator生成数组,当数组大小小于colleciton.size(),返回一个包含所有元素的数组对象;当生成器数组.length>Collection.size(),则拷贝返回形参数组对象
*/

default <T> T[] toArray(IntFunction<T[]> generator) {
        return toArray(generator.apply(0));
    }
/*
* 本来该函数没有什么疑问,我开始看到的时候疑问在于fast-fail问题上
*/
default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove(); //这里调用的是iterator.remove(),不是collection.reomve() 因此modCount由迭代器维护,不会触发fast-fail
                removed = true;
            }
        }
        return removed;
    }	
{%endcodeblock%}
###### Set接口
- **概述**
 set表示不含重复元素(Object::equal相同的情况),以及最多一个null
 set接口对于继承于collection的有些函数做了自己的规定
 当set包含可以mutable元素时要注意,因为当元素值改变时equal结果不同
{%asset_img Set.png%}
{%codeblock lang:java Set%}
/**
 返回不可修改set
*/
 static <E> Set<E> copyOf(Collection<? extends E> coll) {
        if (coll instanceof ImmutableCollections.AbstractImmutableSet) {
            return (Set<E>)coll;
        } else {
            return (Set<E>)Set.of(new HashSet<>(coll).toArray());
        }
    }
	
 static <E> Set<E> of(E... elements) {
        switch (elements.length) { // implicit null check of elements
            case 0:
                return ImmutableCollections.emptySet();
            case 1:
                return new ImmutableCollections.Set12<>(elements[0]);
            case 2:
                return new ImmutableCollections.Set12<>(elements[0], elements[1]);
            default:
                return new ImmutableCollections.SetN<>(elements);
        }
    }	
{%endcodeblock%}
###### SortedSet
- **概述:**
在创建时按照自然顺序或比较器排序,在SortedMap中定义了更多的函数
1.插入的元素必须是可以比较的
2.子类都提供四种构造器,例如:

```java
  public TreeSet() {
        this(new TreeMap<>());
    }
  public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }
  public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }
  public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }	
```
{%asset_img SortedSet.png %}
{%codeblock lang:java SortedSet.png%}
//[}左闭右开
SortedSet<E> subSet(E fromElement, E toElement);
//包含参数元素,当参数所在范围不在set[]中,则按照边界取
SortedSet<E> headSet(E toElement);
SortedSet<E> tailSet(E fromElement);
{%endcodeblock%}
###### NavigableSet
- **概述:**
该接口是SortedSet子类,表示该set可以进行higher() lower()等函数
- **函数**
{%asset_img NavigableSet.png%}
###### Queue
- **概述**
典型的队列,LIFO原则,该接口提供了成对的操作,一组失败抛出异常,一组失败时返回null|false

| |抛出异常|返回特殊值|
|-|-|-|
|Insert|add(e)|offer(e)|
|Remove|remove()|poll()|
|Examine|element()|peek()|
- **函数**
{%asset_img Queue.png%}
###### Deque
- **概述**
Deque->"Double End Queue"双向队列,读作"deck",对于线性Collection支持双向操作,具有FILO(stack性质),和FIFO(队列性质)
对于两端的处理也各提供了两组,抛出异常和不抛出异常的函数

| |Head异常|Head特殊|Tail异常|Tail特殊|
|--|--|--|--|--|
|Insert|addFirst(e)|offerFirst(e)|addLast(e)|offerLast(e)|
|Remove|removieFirst()|pollFrist()|removeLast()|pollLast()|
|Examine|getFirst()|peekFirst()|getLast()|peekLast()|
Deque接口继承了Queue,因此若将deque按照queue使用,则可以使用queue中的等效函数

|queue|等效Deque|
|-|-|
|add(e)|addLast(e)|
|offer(e)|offerLast(e)|
|remove()|removeFirst()|
|poll()|pollFirst()|
|element|getFirst()|
|peek()|peekFirst()|

若将Deque当作stack使用,也具有等效函数,该使用方式优先于遗留的Stack类,并且注意,此处将Deque的**Head**视为**Stack栈顶**
|Stack|等效Deque|
|-|-|
|push(e)|addFirst(e)|
|pop()|removeFirst()|
|peek()|getFirst()|
- **函数**
{%asset_img Deque.png%}
##### Map接口
- **概述**
map表示一组key-value映射,map接口提供了三组视图,key|value|key-value视图
若使用可变对象,则要注意当对象改变时,是否能保证equals相同
每个map子类都提供两个构造器,空|map(map)
map子类有些通过hascode进行比较
*注意:* stream()这个函数是Collection接口定义,因此map不能创建stream,也就无法使用stream接口中的各种函数;不过collect()可以用来创建map,比较复杂,因此jdk8加入了一些default函数补偿|map也没有iterator接口,而使用另外三种视图接口
- **要点**
内部interface
{%codeblock lang:java Map.Entry%}
//内部的entry,entry的逻辑非常容易理解,一个节点但是具有key和value两个属性
//即使是用数组实现也可以用来当作map,如最大堆,使用数组实现,每个节点使用map也可以当作map操作
//该接口不是面向使用者的
interface Entry<K, V> {
K getKey();
V getValue();
V setValue(V value);
boolean equals(Object o);
int hashCode();
//返回一个升序比较器,比较key
 public static <K extends Comparable<? super K>, V> Comparator<Map.Entry<K, V>> comparingByKey() {
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> c1.getKey().compareTo(c2.getKey());
        }
}
//比较value
 public static <K, V extends Comparable<? super V>> Comparator<Map.Entry<K, V>> comparingByValue() {
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> c1.getValue().compareTo(c2.getValue());
        }
//两个自定义比较		
public static <K, V> Comparator<Map.Entry<K, V>> comparingByKey(Comparator<? super K> cmp) {
            Objects.requireNonNull(cmp);
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> cmp.compare(c1.getKey(), c2.getKey());
        }

 public static <K, V> Comparator<Map.Entry<K, V>> comparingByValue(Comparator<? super V> cmp) {
            Objects.requireNonNull(cmp);
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> cmp.compare(c1.getValue(), c2.getValue());
        }
}		
{%endcodeblock%}

- 其他操作


{%codeblock lang:java Map%}

//jdk8加入的default
 
//当不存在key返回defaultValue
default V getOrDefault(Object key, V defaultValue) {
        V v;
        return (((v = get(key)) != null) || containsKey(key))
            ? v
            : defaultValue;
    }
//替代stream中的foreach,使用BiConsumer接口而不是Consumer
default void forEach(BiConsumer<? super K, ? super V> action) {
        Objects.requireNonNull(action);
        for (Map.Entry<K, V> entry : entrySet()) {
            K k;
            V v;
            try {
                k = entry.getKey();
                v = entry.getValue();
            } catch (IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }
            action.accept(k, v);
        }
    }	
//替换
default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        Objects.requireNonNull(function);
        for (Map.Entry<K, V> entry : entrySet()) {
            K k;
            V v;
            try {
                k = entry.getKey();
                v = entry.getValue();
            } catch (IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }

            // ise thrown from function is not a cme.
            v = function.apply(k, v);

            try {
                entry.setValue(v);
            } catch (IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }
        }
    }	
	
default V putIfAbsent(K key, V value) {
        V v = get(key);
        if (v == null) {
            v = put(key, value);
        }

        return v;
    }
//当key存在,并且对应的curValue==value时溢出,注意此处没有使用泛型	
default boolean remove(Object key, Object value) {
        Object curValue = get(key);
        if (!Objects.equals(curValue, value) ||
            (curValue == null && !containsKey(key))) {
            return false;
        }
        remove(key);
        return true;
    }
//存在key,并且对应的curValue==oldValue是替换	
default boolean replace(K key, V oldValue, V newValue) {
        Object curValue = get(key);
        if (!Objects.equals(curValue, oldValue) ||
            (curValue == null && !containsKey(key))) {
            return false;
        }
        put(key, newValue);
        return true;
    }
//存在key就替换
default V replace(K key, V value) {
        V curValue;
        if (((curValue = get(key)) != null) || containsKey(key)) {
            curValue = put(key, value);
        }
        return curValue;
    }
// 当key不存在时,调用mapper逻辑,替代stream中mapper()函数
default V computeIfAbsent(K key,
            Function<? super K, ? extends V> mappingFunction) {
        Objects.requireNonNull(mappingFunction);
        V v;
        if ((v = get(key)) == null) {
            V newValue;
            if ((newValue = mappingFunction.apply(key)) != null) {
                put(key, newValue);
                return newValue;
            }
        }

        return v;
    }	
//当key存在时,根据key和value进行设置新值
default V computeIfPresent(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        V oldValue;
        if ((oldValue = get(key)) != null) {
            V newValue = remappingFunction.apply(key, oldValue);
            if (newValue != null) {
                put(key, newValue);
                return newValue;
            } else {
                remove(key);
                return null;
            }
        } else {
            return null;
        }
    }
//当存在旧值时,通过oldvalue和value进行创建新值并替换,否则newValue=vlaue,当newValue==null,移除该节点	
default V merge(K key, V value,
            BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        Objects.requireNonNull(value);
        V oldValue = get(key);
        V newValue = (oldValue == null) ? value :
                   remappingFunction.apply(oldValue, value);
        if (newValue == null) {
            remove(key);
        } else {
            put(key, newValue);
        }
        return newValue;
    }	
/**
jdk9新增
*/	
//返回不可修改视图,注意接口实现的static函数必须通过接口本身调用,Map.of 不能是HashMap.of
static <K, V> Map<K, V> of() {
        return ImmutableCollections.emptyMap();
    }
static <K, V> Map<K, V> of(K k1, V v1, K k2, V v2) {
        return new ImmutableCollections.MapN<>(k1, v1, k2, v2);
    }	
//.... Map中实现了多个该函数,类似于Set中	


//通过enteries,放置多个, 看到entries就想起c++中的pair<>()
 static <K, V> Map<K, V> ofEntries(Entry<? extends K, ? extends V>... entries) {
        if (entries.length == 0) { // implicit null check of entries array
            return ImmutableCollections.emptyMap();
        } else if (entries.length == 1) {
            // implicit null check of the array slot
            return new ImmutableCollections.Map1<>(entries[0].getKey(),
                    entries[0].getValue());
        } else {
            Object[] kva = new Object[entries.length << 1];
            int a = 0;
            for (Entry<? extends K, ? extends V> entry : entries) {
                // implicit null checks of each array slot
                kva[a++] = entry.getKey();
                kva[a++] = entry.getValue();
            }
            return new ImmutableCollections.MapN<>(kva);
        }
    }
	
 static <K, V> Entry<K, V> entry(K k, V v) {
        // KeyValueHolder checks for nulls
        return new KeyValueHolder<>(k, v);
    }

  static <K, V> Map<K, V> copyOf(Map<? extends K, ? extends V> map) {
        if (map instanceof ImmutableCollections.AbstractImmutableMap) {
            return (Map<K,V>)map;
        } else {
            return (Map<K,V>)Map.ofEntries(map.entrySet().toArray(new Entry[0]));
        }
    }	
{%endcodeblock%}


###### SortedMap
**概述**
该接口定义了顺序排序的map,默认按照自然顺序,或者按照指定的比较器排序,提供返回三个视图
jdk中Map的具体实现基本都对应了一个Set,将value置为固定值就能当作set使用
该接口的Key规定:当存在compartor时,key可以不是comparable子类,这里可以看TreeMap.put实现
提供四个构造器和set相同,提供的Sorted函数也和set类型
**函数**
{%asset_img SortedMap.png%}
###### NavigableMap
**概述**
参考NavigableSet
**函数**
{%asset_img NavigableMap.png%}
#### 实现
##### 非并发
###### HashTable 哈希表
- HashMap
{%asset_img HashMap.png 类图:%}
**概述**
基于数组链表和红黑树构成的结构,除了允许*null key*和*null vlaue*并且不是*同步*的,与HashTable类似,并且该结构不保证顺序,也因此并不是SortedMap子类
若正确的在buckets中间分配元素,那么对于put和get函数的性能是O(x)常量时间;迭代视图的时间与bucket及key-value数量成正比;若迭代操作非常重要则不要将初始容量设置太大,或者将load factor设置太小
默认load factor为0.75,较高的值会减少空间占用但是会提高查找开销;设置初始容量时,应该考虑其大小和load factory,以便最小化重新散列的数量,当initial capacity >max number/load facotry,不会发生重散列
由于该结构不是同步的,若多线程访问,若存在线程修改map,则因该在外部使用同步(只有put/remove这样属于修改,修改某key对应的value不算),或者使用Collections.synchronizedMap返回同步Map
- 链表模式
```java
//field
transient Node<K,V>[] table;//链表数组
 transient int size;
 transient int modCount;
 int threshold;
 final float loadFactor;
//Constructor
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
//返回接近cap的2^x大小,该函数会返回比cap-1大一个幂的结果|cap|MAXIMUM_CAPACITY
//-1 === 1111111111111111 ,cap如果是2^x 2^x-1=1111111,该幂级比cap小1,获取0的数量, 并且将-1>>>,就会得到该数,最后+1就是cap|若cap!=2^x 则cap=1xxxxxxxxx,其中x必定有1,则cap-1=1xxxxxxxx,说明幂级不变
//该幂级和源cap相同,将-1>>>结果为11111111111,此时将+1会得到比cap高一个幂级的2^x
 static final int tableSizeFor(int cap) {
        int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);  
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }	
//补充:返回任意数i,按照补码排列,返回最高位右边到第一个1出现,0的数量,不包含最高位,如 -1则返回 0,1则返回31
//用了二分法进行判断	
 public static int numberOfLeadingZeros(int i) {
        // HD, Count leading 0's
        if (i <= 0)
            return i == 0 ? 32 : 0;
        int n = 31;
        if (i >= 1 << 16) { n -= 16; i >>>= 16; }
        if (i >= 1 <<  8) { n -=  8; i >>>=  8; }
        if (i >= 1 <<  4) { n -=  4; i >>>=  4; }
        if (i >= 1 <<  2) { n -=  2; i >>>=  2; }
        return n - (i >>> 1);
    }	
//链表节点
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }	
	
```
#### 迭代器
迭代器用来对外部隐藏细节以提供统一的接口,便于使用.
##### fast-fail机制
**概念**
1.在jcf中通过保持一个modCount int型变量,在数据结构本身任意的mod操作,都会将该值递增,任意数据结构的迭代器被构建时持有该值
2.这里将迭代器返回的视为view,无论是collection子类的iterator()返回,还是map中的三个视图,如果在获取view后,改变源导致modCout改变,那么在迭代器视图中,如next()函数会检查一致性,若不一致则抛出异常
3.这就是最大可能避免并发问题,记住并不是说非线程安全的结构不能用于并发,这里的要点是理解view的作用,同时迭代器提供romvoe()用于view对于源的修改,我思考了一下如果普通迭代器并发操作如remove操作,那么加锁就可以了
4.若进行带锁并发操作迭代器,效率和单线程差距?,锁的粒度最细也只能在如remove()函数上,这里如果并发处理就要用到splIterator
例如:
```java
//实现于AbstractList中的迭代器
private class Itr implements Iterator<E> {
int expectedModCount = modCount; //创建时获取当前modCount
 public E next() {
            checkForComodification(); //检查一致性
            try {
                int i = cursor;
                E next = get(i);
                lastRet = i;
                cursor = i + 1;
                return next;
            } catch (IndexOutOfBoundsException e) {
                checkForComodification();
                throw new NoSuchElementException();
            }
        }
    public void remove() { //这种操作迭代器本身维护了expectedModCount==新的modCount,如果并发操作,必定加锁
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                AbstractList.this.remove(lastRet); //此处源本身remove,并且修改了modCount
                if (lastRet < cursor)
                    cursor--;
                lastRet = -1;
                expectedModCount = modCount; //维护新的expectedModCount
            } catch (IndexOutOfBoundsException e) {
                throw new ConcurrentModificationException();
            }
        }	
    final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }		
}
```
##### SplIterator
**概念**
jdk8加入的SplIterator,顾名思义,该迭代器的逻辑是将视图分割,从而完成并发操作
{%codeblock lang:java Spliterator%}
public interface Spliterator<T> {
boolean tryAdvance(Consumer<? super T> action); //对当前元素进行action操作
default void forEachRemaining(Consumer<? super T> action) {
        do { } while (tryAdvance(action));
    }//以当前位置迭代执行action
}
Spliterator<T> trySplit();//将Spliterator分割,这是该迭代器完成并发的关键
{%endcodeblock%}
举例:
{%codeblock lang:java ArrayList中的实现%}
//ArrayList 接口
public Spliterator<E> spliterator() {
        return new ArrayListSpliterator(0, -1, 0);
    }

final class ArrayListSpliterator implements Spliterator<E> {
		private int index; //表示当前位置
        private int fence; // 该迭代器的边界
        private int expectedModCount; //fast-fail,逻辑还是和普通迭代器相同
	 ArrayListSpliterator(int origin, int fence, int expectedModCount) {
            this.index = origin;
            this.fence = fence;
            this.expectedModCount = expectedModCount;
        }

        private int getFence() { // initialize fence to size on first use
            int hi; // (a specialized variant appears in method forEach)
            if ((hi = fence) < 0) { //当第一次创建该迭代器时默认fence=-1,总之hi==边界fence
                expectedModCount = modCount;
                hi = fence = size;
            }
            return hi;
        }	
}		public ArrayListSpliterator trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1; //寻找当前位置----边界位置  >>>1
            return (lo >= mid) ? null : // divide range in half unless too small
                new ArrayListSpliterator(lo, index = mid, expectedModCount); //创建一个分割的前半部分迭代器[lo,mid),当前迭代器变成[mid,fence)
        }
{%endcodeblock%}
#### stream
- urml类图
{%asset_img stream.jpg %}
- 时序
{%asset_img stream2.jpg %}
- 关于stream源码调用实现
stream框架由中间操作生成 pipe(Head)<---->pipe(Stateless|Stateful)<----->pipe(Stateless|Stateful)这样的管道节点,当执行终止操作时,
产生sink---->sink---->sink--->sink这样的槽节点,然后遍历s迭代器,并逐层调用sink完成stream操作
补充:这里的逻辑和tomcat中Pipeline和其Valve相同,但是具体实现还是不同,很有趣, tomcat采取由组件持有pipeline,pipeline持有valve节点,逐层调用
{%codeblock lang:java PipelineHelper及其子类%}
//-----------------------------------------PipelineHelper---------------------------------------------------------------
abstract class PipelineHelper<P_OUT> {

abstract <P_IN> boolean copyIntoWithCancel(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator);
abstract<P_IN> Sink<P_IN> wrapSink(Sink<P_OUT> sink);//该函数表示每个节点如何将sink(槽)的包装方式,返回的结果是上一个sink
abstract<P_IN, S extends Sink<P_OUT>> S wrapAndCopyInto(S sink, Spliterator<P_IN> spliterator);//将节点向前推进封装sink,并从head开始调用
abstract<P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator);//调用pipe
}
//-----------------------------------------AbstractPipeline---------------------------------------------------------------
abstract class AbstractPipeline<E_IN, E_OUT, S extends BaseStream<E_OUT, S>>
        extends PipelineHelper<E_OUT> implements BaseStream<E_OUT, S> {
		private final AbstractPipeline sourceStage;

     /**
     * Backlink to the head of the pipeline chain (self if this is the source
     * stage).
     */
    @SuppressWarnings("rawtypes")
    private final AbstractPipeline sourceStage;
    /**
     * The "upstream" pipeline, or null if this is the source stage.
     */
    @SuppressWarnings("rawtypes")
    private final AbstractPipeline previousStage;

    /**
     * The operation flags for the intermediate operation represented by this
     * pipeline object.
     */
    protected final int sourceOrOpFlags;

    /**
     * The next stage in the pipeline, or null if this is the last stage.
     * Effectively final at the point of linking to the next pipeline.
     */
    @SuppressWarnings("rawtypes")
    private AbstractPipeline nextStage;

    /**
     * The number of intermediate operations between this pipeline object
     * and the stream source if sequential, or the previous stateful if parallel.
     * Valid at the point of pipeline preparation for evaluation.
     */
    private int depth;

    /**
     * The combined source and operation flags for the source and all operations
     * up to and including the operation represented by this pipeline object.
     * Valid at the point of pipeline preparation for evaluation.
     */
    private int combinedFlags;

    /**
     * The source spliterator. Only valid for the head pipeline.
     * Before the pipeline is consumed if non-null then {@code sourceSupplier}
     * must be null. After the pipeline is consumed if non-null then is set to
     * null.
     */
    private Spliterator<?> sourceSpliterator;

    /**
     * The source supplier. Only valid for the head pipeline. Before the
     * pipeline is consumed if non-null then {@code sourceSpliterator} must be
     * null. After the pipeline is consumed if non-null then is set to null.
     */
    private Supplier<? extends Spliterator<?>> sourceSupplier;

    /**
     * True if this pipeline has been linked or consumed
     */
    private boolean linkedOrConsumed;

    /**
     * True if there are any stateful ops in the pipeline; only valid for the
     * source stage.
     */
    private boolean sourceAnyStateful;

    private Runnable sourceCloseAction;

    /**
     * True if pipeline is parallel, otherwise the pipeline is sequential; only
     * valid for the source stage.
     */
    private boolean parallel;
		
	/**
     * Constructor for the head of a stream pipeline.
     *
     * @param source {@code Supplier<Spliterator>} describing the stream source
     * @param sourceFlags The source flags for the stream source, described in
     * {@link StreamOpFlag}
     * @param parallel True if the pipeline is parallel
     */
    AbstractPipeline(Supplier<? extends Spliterator<?>> source,
                     int sourceFlags, boolean parallel) { 
        this.previousStage = null;
        this.sourceSupplier = source; //不同的构造器区别在于源来自何处
        this.sourceStage = this;
        this.sourceOrOpFlags = sourceFlags & StreamOpFlag.STREAM_MASK;
        // The following is an optimization of:
        // StreamOpFlag.combineOpFlags(sourceOrOpFlags, StreamOpFlag.INITIAL_OPS_VALUE);
        this.combinedFlags = (~(sourceOrOpFlags << 1)) & StreamOpFlag.INITIAL_OPS_VALUE;
        this.depth = 0;
        this.parallel = parallel;
    }	
	AbstractPipeline(Spliterator<?> source,
                     int sourceFlags, boolean parallel) {
        this.previousStage = null;
        this.sourceSpliterator = source; //来自迭代器
        this.sourceStage = this; 
        this.sourceOrOpFlags = sourceFlags & StreamOpFlag.STREAM_MASK;
        // The following is an optimization of:
        // StreamOpFlag.combineOpFlags(sourceOrOpFlags, StreamOpFlag.INITIAL_OPS_VALUE);
        this.combinedFlags = (~(sourceOrOpFlags << 1)) & StreamOpFlag.INITIAL_OPS_VALUE;
        this.depth = 0;
        this.parallel = parallel;
    }
	AbstractPipeline(AbstractPipeline<?, E_IN, ?> previousStage, int opFlags) { 
        if (previousStage.linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
        previousStage.linkedOrConsumed = true;
        previousStage.nextStage = this; //和上一个pipe连接

        this.previousStage = previousStage;//构建双向链表
        this.sourceOrOpFlags = opFlags & StreamOpFlag.OP_MASK;
        this.combinedFlags = StreamOpFlag.combineOpFlags(opFlags, previousStage.combinedFlags);
        this.sourceStage = previousStage.sourceStage;
        if (opIsStateful())
            sourceStage.sourceAnyStateful = true;
        this.depth = previousStage.depth + 1;
    }
    //一般的执行最终操作
	 final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
        assert getOutputShape() == terminalOp.inputShape();
        if (linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
        linkedOrConsumed = true;

        return isParallel()
               ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags())) //具体如何执行可以取决于terminalOp实现
               : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
    }
	//一般执行时会调用的逻辑
	 @Override
    final <P_IN, S extends Sink<E_OUT>> S wrapAndCopyInto(S sink, Spliterator<P_IN> spliterator) {
        copyInto(wrapSink(Objects.requireNonNull(sink)), spliterator);
        return sink;
    }
	//
	@Override
    final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {如何调用sink
        Objects.requireNonNull(wrappedSink);

        if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
            wrappedSink.begin(spliterator.getExactSizeIfKnown()); //由当前sink调用begin(),一般会递归调用下去,唤醒下sink节点begin逻辑
            spliterator.forEachRemaining(wrappedSink); //遍历数据,并且每个数据都会途径sink.apceet()
            wrappedSink.end();//同begin()
        }
        else {
            copyIntoWithCancel(wrappedSink, spliterator);
        }
    }
	 @Override
    @SuppressWarnings("unchecked")
    final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) { //如何封装sink
        Objects.requireNonNull(sink);

        for ( @SuppressWarnings("rawtypes") AbstractPipeline p=AbstractPipeline.this; p.depth > 0; p=p.previousStage) { //获取当前pipe,实际就是调用最终操作的pipe,sink则是由最终操作构建的sink
            sink = p.opWrapSink(p.previousStage.combinedFlags, sink); //每次sink都表示上一个pipe的sink,第一次创建后 sink(最终操作调用者pipe)<-->sink(最终操作创建的)
        }
        return (Sink<P_IN>) sink; //返回为第一个中间操作sink
    }
	
	
	    final <P_IN> boolean copyIntoWithCancel(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
        @SuppressWarnings({"rawtypes","unchecked"})
        AbstractPipeline p = AbstractPipeline.this;
        while (p.depth > 0) {
            p = p.previousStage;
        }

        wrappedSink.begin(spliterator.getExactSizeIfKnown());
        boolean cancelled = p.forEachWithCancel(spliterator, wrappedSink);
        wrappedSink.end();
        return cancelled;
    }	
}
//-------------------------------------ReferencePipeline-------------------------------------------------------------------		
abstract class ReferencePipeline<P_IN, P_OUT> //OUT表示该pipe要给调用者返回的,也是Stream中实际元素的类型,in代表上一个pipe输入的
        extends AbstractPipeline<P_IN, P_OUT, Stream<P_OUT>>
        implements Stream<P_OUT>  {
		   ReferencePipeline(Supplier<? extends Spliterator<?>> source,
                      int sourceFlags, boolean parallel) {
        super(source, sourceFlags, parallel);
    }
	  // BaseStream

    @Override
    public final Iterator<P_OUT> iterator() {
        return Spliterators.iterator(spliterator());
    }
	//该类有两个三个内部静态类,这就是实际的节点,
//-------------------------------------Sink-------------------------------------------------------------------	
//该接口表示管道中的槽,继承Consumer	
interface Sink<T> extends Consumer<T> {
default void begin(long size) {}
default void end() {}
default void accept(int value) {
        throw new IllegalStateException("called wrong accept method");
    } //可以实现接受int
default void accept(long value) {
        throw new IllegalStateException("called wrong accept method");
    }	
default void accept(double value) {
        throw new IllegalStateException("called wrong accept method");
    }
//内部类	
    interface OfInt extends Sink<Integer>, IntConsumer { //int型
        @Override
        void accept(int value); //此处实现接受到int值后如何处理,用于intPipeline

        @Override
        default void accept(Integer i) {
            if (Tripwire.ENABLED)
                Tripwire.trip(getClass(), "{0} calling Sink.OfInt.accept(Integer)");
            accept(i.intValue());
        }
    }

    /**
     * {@code Sink} that implements {@code Sink<Long>}, re-abstracts
     * {@code accept(long)}, and wires {@code accept(Long)} to bridge to
     * {@code accept(long)}.
     */
    interface OfLong extends Sink<Long>, LongConsumer {
        @Override
        void accept(long value);

        @Override
        default void accept(Long i) {
            if (Tripwire.ENABLED)
                Tripwire.trip(getClass(), "{0} calling Sink.OfLong.accept(Long)");
            accept(i.longValue());
        }
    }

    /**
     * {@code Sink} that implements {@code Sink<Double>}, re-abstracts
     * {@code accept(double)}, and wires {@code accept(Double)} to bridge to
     * {@code accept(double)}.
     */
    interface OfDouble extends Sink<Double>, DoubleConsumer {
        @Override
        void accept(double value);

        @Override
        default void accept(Double i) {
            if (Tripwire.ENABLED)
                Tripwire.trip(getClass(), "{0} calling Sink.OfDouble.accept(Double)");
            accept(i.doubleValue());
        }
    }

	//中间操作应该产生的sink
	abstract static class ChainedReference<T, E_OUT> implements Sink<T> { //对应泛型种类
        protected final Sink<? super E_OUT> downstream;

        public ChainedReference(Sink<? super E_OUT> downstream) {
            this.downstream = Objects.requireNonNull(downstream);
        }

        @Override
        public void begin(long size) { //begin和end都和调用下层sink的begin和end将之传递
            downstream.begin(size);
        }

        @Override
        public void end() {
            downstream.end();
        }

        @Override
        public boolean cancellationRequested() {
            return downstream.cancellationRequested();
        }
    }
	abstract static class ChainedInt<E_OUT> implements Sink.OfInt {
        protected final Sink<? super E_OUT> downstream;

        public ChainedInt(Sink<? super E_OUT> downstream) {
            this.downstream = Objects.requireNonNull(downstream);
        }

        @Override
        public void begin(long size) {
            downstream.begin(size);
        }

        @Override
        public void end() {
            downstream.end();
        }

        @Override
        public boolean cancellationRequested() {
            return downstream.cancellationRequested();
        }
    }

    /**
     * Abstract {@code Sink} implementation designed for creating chains of
     * sinks.  The {@code begin}, {@code end}, and
     * {@code cancellationRequested} methods are wired to chain to the
     * downstream {@code Sink}.  This implementation takes a downstream
     * {@code Sink} of unknown input shape and produces a {@code Sink.OfLong}.
     * The implementation of the {@code accept()} method must call the correct
     * {@code accept()} method on the downstream {@code Sink}.
     */
    abstract static class ChainedLong<E_OUT> implements Sink.OfLong {
        protected final Sink<? super E_OUT> downstream;

        public ChainedLong(Sink<? super E_OUT> downstream) {
            this.downstream = Objects.requireNonNull(downstream);
        }

        @Override
        public void begin(long size) {
            downstream.begin(size);
        }

        @Override
        public void end() {
            downstream.end();
        }

        @Override
        public boolean cancellationRequested() {
            return downstream.cancellationRequested();
        }
    }

    /**
     * Abstract {@code Sink} implementation designed for creating chains of
     * sinks.  The {@code begin}, {@code end}, and
     * {@code cancellationRequested} methods are wired to chain to the
     * downstream {@code Sink}.  This implementation takes a downstream
     * {@code Sink} of unknown input shape and produces a {@code Sink.OfDouble}.
     * The implementation of the {@code accept()} method must call the correct
     * {@code accept()} method on the downstream {@code Sink}.
     */
    abstract static class ChainedDouble<E_OUT> implements Sink.OfDouble {
        protected final Sink<? super E_OUT> downstream;

        public ChainedDouble(Sink<? super E_OUT> downstream) {
            this.downstream = Objects.requireNonNull(downstream);
        }

        @Override
        public void begin(long size) {
            downstream.begin(size);
        }

        @Override
        public void end() {
            downstream.end();
        }

        @Override
        public boolean cancellationRequested() {
            return downstream.cancellationRequested();
        }
    }
}		
{%endcodeblock%}
{%codeblock lang:java ReferencePipeline%}
//总体而言大部分stream()操作返回的实体是该抽象类的子类
//该类定义两个子类stateless 和 stateful
abstract class ReferencePipeline<P_IN, P_OUT>
        extends AbstractPipeline<P_IN, P_OUT, Stream<P_OUT>>
        implements Stream<P_OUT>  {
		 static class Head<E_IN, E_OUT> extends ReferencePipeline<E_IN, E_OUT> {
        /**
         * Constructor for the source stage of a Stream.
         *
         * @param source {@code Spliterator} describing the stream source
         * @param sourceFlags the source flags for the stream source, described
         *                    in {@link StreamOpFlag}
         */
        Head(Spliterator<?> source,
             int sourceFlags, boolean parallel) {
            super(source, sourceFlags, parallel);
        }

        @Override
        final boolean opIsStateful() {
            throw new UnsupportedOperationException();
        }

        @Override
        final Sink<E_IN> opWrapSink(int flags, Sink<E_OUT> sink) {
            throw new UnsupportedOperationException(); //这个函数是主要关注点,此处决定了sink如何封装,明显如果head执行那么就异常,用户是无法如此执行的
        }

        // Optimized sequential terminal operations for the head of the pipeline

        @Override
        public void forEach(Consumer<? super E_OUT> action) {
            if (!isParallel()) {
                sourceStageSpliterator().forEachRemaining(action);
            }
            else {
                super.forEach(action);
            }
        }

        @Override
        public void forEachOrdered(Consumer<? super E_OUT> action) {
            if (!isParallel()) {
                sourceStageSpliterator().forEachRemaining(action);
            }
            else {
                super.forEachOrdered(action);
            }
        }
    }
		
	 abstract static class StatelessOp<E_IN, E_OUT>
            extends ReferencePipeline<E_IN, E_OUT> {
        /**
         * Construct a new Stream by appending a stateless intermediate
         * operation to an existing stream.
         *
         * @param upstream The upstream pipeline stage
         * @param inputShape The stream shape for the upstream pipeline stage
         * @param opFlags Operation flags for the new stage
         */
        StatelessOp(AbstractPipeline<?, E_IN, ?> upstream,
                    StreamShape inputShape,
                    int opFlags) {
            super(upstream, opFlags);
            assert upstream.getOutputShape() == inputShape;
        }

        @Override
        final boolean opIsStateful() { //无状态
            return false;
        }
    }

     abstract static class StatefulOp<E_IN, E_OUT>
            extends ReferencePipeline<E_IN, E_OUT> {
        /**
         * Construct a new Stream by appending a stateful intermediate operation
         * to an existing stream.
         * @param upstream The upstream pipeline stage
         * @param inputShape The stream shape for the upstream pipeline stage
         * @param opFlags Operation flags for the new stage
         */
        StatefulOp(AbstractPipeline<?, E_IN, ?> upstream,
                   StreamShape inputShape,
                   int opFlags) {
            super(upstream, opFlags);
            assert upstream.getOutputShape() == inputShape;
        }

        @Override
        final boolean opIsStateful() {
            return true;
        }

        @Override
        abstract <P_IN> Node<E_OUT> opEvaluateParallel(PipelineHelper<E_OUT> helper,
                                                       Spliterator<P_IN> spliterator,
                                                       IntFunction<E_OUT[]> generator);
    }	
//---------------------------------------------其他操作
//该类将操作分为状态操作|无状态操作|终止操作
//stateless
 public Stream<P_OUT> unordered() {
        if (!isOrdered())
            return this;
        return new StatelessOp<P_OUT, P_OUT>(this, StreamShape.REFERENCE, StreamOpFlag.NOT_ORDERED) { //该操作改变了StreamOpFlag.NOT_ORDERED
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<P_OUT> sink) {
                return sink;  //实际此处并不进行什么操作
            }
        };
    }

    @Override
    public final Stream<P_OUT> filter(Predicate<? super P_OUT> predicate) { //这是典型的中间操作逻辑
        Objects.requireNonNull(predicate);
        return new StatelessOp<P_OUT, P_OUT>(this, StreamShape.REFERENCE,
                                     StreamOpFlag.NOT_SIZED) {  //返回Stateless实现
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<P_OUT> sink) { //重写onWarapSink用于最终执行逻辑中的封装sink代码
                return new Sink.ChainedReference<P_OUT, P_OUT>(sink) {
                    @Override
                    public void begin(long size) {
                        downstream.begin(-1); 
                    }

                    @Override
                    public void accept(P_OUT u) {
                        if (predicate.test(u))  //u实际就是stream遍历源过程中的一个元素,此处捕获外部lambda表达式调用.test(u)
                            downstream.accept(u); //此处调用下一层sink
                    }
                };
            }
        };
    }	
}		
{%endcodeblock%}

- 特点
  - 不储存数据:并非是数据结构,通过数据结构 io 等pipeline获取数据
  - Functional in nature(功能性):返回数据,但是不修改源
  - 惰性求值|及早求值,当返回为Stream则为惰性求值
  - Possibly unbounded(无边界)
  - 考虑状态lambda
  - 无序性在并行操作有更好的性能
- 获取方式
  - Collection子类 stream()|parallelStream()  
  - Arrays.stream(Object[])
  - 静态工厂 Stream.of()| IntStream.range(int, int) | Stream.iterate(Object, UnaryOperator);
  - The lines of a file can be obtained from BufferedReader.lines(); 
  - Streams of file paths can be obtained from methods in Files; Files是1.7的一个类
  - Streams of random numbers can be obtained from Random.ints();
  - Numerous other stream-bearing methods in the JDK, including BitSet.stream(), Pattern.splitAsStream(java.lang.CharSequence), and JarFile.stream().
  - 第三方库
- stream操作和pipeline
  - stream 操作由中间操作和最终操作构成,组成pipeline;pipeline由源,中间操作如Stream.filter or Stream.map,最终操作Stream.forEach or Stream.reduce构成
  - 中间操作都是惰性求值
  - 当最终操作执行后,流被消耗,iterator() and spliterator()除外
- Function接口
  - Predicate(谓语):测试input是否符合条件
```java
public interface Predicate<T> {
    boolean test(T t); //判断是否符合
    
    default Predicate<T> and(Predicate<? super T> other) { //lambda1&lambda2
            Objects.requireNonNull(other);
            return (t) -> test(t) && other.test(t);
        }
    default Predicate<T> negate() {
            return (t) -> !test(t);
        }
        
    default Predicate<T> or(Predicate<? super T> other) {
                Objects.requireNonNull(other);
                return (t) -> test(t) || other.test(t);
            }       
    static <T> Predicate<T> isEqual(Object targetRef) { //静态函数用来创建一个predicate判断是否和targetRef相同
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }     
    static <T> Predicate<T> not(Predicate<? super T> target) { //创建一个和形参逻辑相反的predicate
        Objects.requireNonNull(target);
        return (Predicate<T>)target.negate();
    }    
}
```
- Function:接受T类型返回R类型
```java
public interface Function<T, R> {
    R apply(T t); //获取一个T输入,输出一个R类型
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) { 
            Objects.requireNonNull(before);
            return (V v) -> apply(before.apply(v));//新的lambda解释: 输入V->由before调用输出T->输入T由this(这里指代当前上下文,this被新的lambda捕获 )->输出R
        }
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) { 
            Objects.requireNonNull(after);
            return (T t) -> after.apply(apply(t));//新的lambda解释:输入T->this输出R->after接受输出? extend V
        }    
        
     static <T> Function<T, T> identity() {
            return t -> t;
        }    
}
```
- Consumer接受T类型,继续对该类型进行操作
```java
public interface Consumer<T> {
    void accept(T t);//接受T,进行操作
    default Consumer<T> andThen(Consumer<? super T> after) { //this操作,再进行after操作
            Objects.requireNonNull(after);
            return (T t) -> { accept(t); after.accept(t); };
        }
}
```
- BiFunction 接受T,U返回R
```java
public interface BiFunction<T, U, R> {
    R apply(T t, U u);
    default <V> BiFunction<T, U, V> andThen(Function<? super R, ? extends V> after) {
            Objects.requireNonNull(after);
            return (T t, U u) -> after.apply(apply(t, u)); //当前操作调用after
    }
}
//子类
public interface BinaryOperator<T> extends BiFunction<T,T,T> { //进行二元计算
     public static <T> BinaryOperator<T> minBy(Comparator<? super T> comparator) {
            Objects.requireNonNull(comparator);
            return (a, b) -> comparator.compare(a, b) <= 0 ? a : b;
        }
     public static <T> BinaryOperator<T> maxBy(Comparator<? super T> comparator) {
         Objects.requireNonNull(comparator);
         return (a, b) -> comparator.compare(a, b) >= 0 ? a : b;
     }       
}
```
#### 杂项
- 闭包
