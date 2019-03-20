---
title: Java Collection笔记
date: 2019-03-20 20:19:10
categories: 开发
tags: Java

---

这是自己整理的一些Collection的要点笔记，比较零碎，可能可读性不是很强。有新内容时会进行补充。
Java Collection框架：
- Set  , HashSet TreeSet(实现SortedSet)
  - SortedSet
- List , LinkedList ArrayList
- Queue,  PriorityQueue
- Dequeue
- Map , HashMap TreeMap(实现SortedMap)
  - SortedMap

<!-- more -->

基本方法 add(), remove(), contains(), isEmpty(), addAll()

---
##### hashmap
线程不安全，允许存null。
实现：
1. 内部有一个静态类`Node<K,V>` ， 实现` Map.Entry<K,V>`，是“ Basic hash bin node”（文档原文）。而`TreeNode`也是节点的实现，适用于有冲突的情况，冲突后形成的是红黑树。
2. 计算hash值方法：高16位和低16位hashcode异或，降低hash值范围小时的冲突：
```java
static final int hash(Object key) {
	int h;
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
3. 用数组存Node，数组长度必须是2的幂
```java
transient Node<K,V>[] table;
```

4. 缓存entrySet
```java
transient Set<Map.Entry<K,V>> entrySet;
```

5. 取：按hash值作为数组下标去取Node。下标是`(tab.length - 1) & hash`。 由于桶的长度是2的n次方，这么做其实是等于 一个模运算。比如hash是31(11111)，length是4(100)，-1后是11，与运算后是3(11)，就是取模。
如果有冲突了，则有多个Node放在一个桶里，要么顺序查找（链表），要么按TreeNode去取（红黑树）。
```java
public V get(Object key) {
	Node<K,V> e;
	return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

6. 存，往数组的`(tab.length - 1) & hash`处放。桶里没有的话则直接放，有的话，找有没有相同的值，有的话替换。加了后如果容量达到threshold就resize();
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

```

7. **`resize()` 方法**，初始化数组或扩容。扩容时数组容量扩大到2倍然后ReHash，遍历原Entry数组，把所有的Entry重新Hash到新数组。通过`e.hash & (newCap - 1)`算出新的数组下标，原因是因为数组全是2的幂，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。然后链表和treenode重新放

HashMap 在第一次 put 时初始化，类似 ArrayList 在第一次 add 时分配空间。
在哈希碰撞的链表长度达到TREEIFY_THRESHOLD（默认8)后，会把该链表转变成树结构

---
##### concurrenthashmap
HashMap允许一个key和value为null，而ConcurrentHashMap不允许key和value为null，如果发现key或者value为null，则会抛出NPE。

和hashmap一样有Node<K,V>

sizeCtl：`private transient volatile int sizeCtl;`这是一个用于同步多个线程的共享变量，如果值为负数，则说明table正在被某个线程初始化或者扩容。如果某个线程想要初始化table或者对table扩容，需要去竞争sizeCtl这个共享变量，获得变量的线程才有许可去进行接下来的操作，没能获得的线程将会一直自旋来尝试获得这个共享变量。获得sizeCtl这个变量的线程在完成工作之后再设置回来，使其他的线程可以走出自旋进行接下来的操作

查询和hashmap差不多，(hashCode & (length - 1))取下标。table数组是被volatile关键字修饰，解决了可见性问题

存要复杂一点。首先计算table下标，下标没数据就通过调用casTabAt方法插入数据。有的话，那么就给该下标处的Node（不管是链表的头还是树的根）加锁插入。 	

扩容操作比较复杂。扩容操作的条件是如果table过小，并且没有被扩容，那么就需要进行扩容，需要使用transfer方法来将久的记录迁移到新的table中去。整个扩容操作分为两个部分，要用到内部类forwardNode。第一部分是构建一个nextTable,它的容量是原来的两倍，这个操作是单线程完成的。
第二个部分就是将原来table中的元素复制到nextTable中，这里允许多线程进行操作。

size()方法，结合baseCount和counterCells数组来得到，通过累计两者的数量即可获得当前ConcurrentHashMap中的记录总量。

---
##### HashSet
用HashMap实现。(内部：`private transient HashMap<E,Object> map;`)

---
##### ArrayList  
fail fast机制：checkForComodification()方法检查modCount，检查有无结构性的改变，变了抛`ConcurrentModificationException`。

扩容调`Arrays.copyOf(elementData, newCapacity);`

内部有迭代器类 Iterator。

----

##### LinkedList
实现List和Deque（即可以当栈、队列、双向队列使用）

内部是一个双向链表

字段存有 size、 first Node（头节点）、last Node。通过头结点、尾节点可以很快地进行双向入队出队操作。
随机存储效率不如ArrayList，要遍历节点。按下标读取时，会按照size，判断是链表前半段还是后半段，根据这个从头或尾节点开始遍历。

和ArrayDeque的区别之一：LinkedList可以存null，而ArrayDeque不能存null。这点在写算法题的时候可以注意一下。

---
##### ArrayDeque
转一张表整理方法。一套接口遇到失败就会抛出异常，另一套遇到失败会返回特殊值。
| Queue Method | Equivalent Deque Method | 说明                                   |
| ------------ | ----------------------- | -------------------------------------- |
| `add(e)`     | `addLast(e)`            | 向队尾插入元素，失败则抛出异常         |
| `offer(e)`   | `offerLast(e)`          | 向队尾插入元素，失败则返回`false`      |
| `remove()`   | `removeFirst()`         | 获取并删除队首元素，失败则抛出异常     |
| `poll()`     | `pollFirst()`           | 获取并删除队首元素，失败则返回`null`   |
| `element()`  | `getFirst()`            | 获取但不删除队首元素，失败则抛出异常   |
| `peek()`     | `peekFirst()`           | 获取但不删除队首元素，失败则返回`null` |

内部elements数组的容量一定是2的倍数，并且不会满。存数组的head和tail下标，形成一个循环数组，当这两个下标相等时，数组为空。而在添加元素时，如果这两个下标相等，说明数组已满，将容量翻倍。扩容时重置头索引和尾索引，头索引置为0，尾索引置为原容量的值。

----
##### CopyOnWriteArrayList
线程安全
add set之类的操作都是新建一个复制arraylist
适用于 读多些少, 并且数据内容变化比较少的场景