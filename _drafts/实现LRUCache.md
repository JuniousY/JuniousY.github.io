---
layout: post
title: 实现LRUCache
date: 2019-03-11 22:57:49
categories: 开发
tags: 算法
---

LRU Cache 算法是操作系统在进行内存管理时可以采用的一种页面置换算法。LRU，就是Least Recently Used的简称，这个算法叫做最近最少使用算法。除了在页面置换中可以使用这一算法，其他需要缓存的场景也可以运用这一算法。这一算法的核心目的就是依照程序的就近原则，尽可能在有限的空间内缓存最多以后会使用到的内容。另外，实现这一算法也是一道[LeetCode题目](https://leetcode.com/problems/lru-cache/)。本文就是演示如何使用java语言实现这一算法。

<!-- more -->

---
### LinkedHashMap实现
LinkedHashMap是最容易的实现方式，因为它内部的实现方式很贴合这一应用，至于为什么下面会有介绍。
LinkedHashMap和普通的HashMap不同的地方在于，它保存了迭代顺序，该迭代顺序可以是插入顺序或者是访问顺序。而LRU要求最近读取过得内容有最高的缓存优先度，也就是按照访问顺序来进行迭代。而通过重写removeEldestEntry方法可以让LinkedHashMap保留有限多的数据，删除缓存中不需要的数据。

```java
//简易实现
class LRUCache<K, V> {

  private static final float loadFactor = 0.75f;

  private int capacity;

  private final LinkedHashMap<K, V> map;

  public LRUCache(int capacity) {
    if (capacity < 0) {
      capacity = 0;
    }
    this.capacity = capacity;
    //构造函数参数分别是initialCapacity、loadFactor、accessOrder，accessOrder为true即按访问顺序迭代
    map = new LinkedHashMap<K, V>(0, loadFactor, true){
      @Override
      protected boolean removeEldestEntry(Entry eldest) {
        return size() > LRUCache.this.capacity;
      }
    };

  }

  public final V get(K key) {
    return map.get(key);
  }

  public final void put(K key, V value) {
    map.put(key, value);
  }
  
}
```

---
### HashMap + 双向链表实现
之所以LinkedHashMap能保有这样的性质，是因为它内部的实现是依托了HashMap和双向链表，因此不用LinkedHashMap我们也能实现LRUCache算法。

基本框架
```java
public class LRUCache<K, V> {

  private int capacity;

  private HashMap<K, Node<K, V>> map;

  private Node<K, V> head;
  private Node<K, V> tail;
    
  public LRUCache(int capacity) {
    this.capacity = capacity;
    map = new HashMap<>(capacity);
    head = new Node<>();
    tail = new Node<>();
    head.next = tail;
    tail.pre = head;
  } 

  class Node<K, V> {

    K key;
    V value;
    Node<K, V> pre;
    Node<K, V> next;

  }

}
```

公用方法
```java
private void raiseNode(Node<K, V> node) {
    if (node.pre == head) {
        return;
    }

    Node<K, V> pre = node.pre;
    Node<K, V> next = node.next;
    pre.next = next;
    next.pre = pre;

    setFirst(node);
}

private void setFirst(Node<K, V> node) {
    Node<K, V> first = head.next;
    head.next = node;
    node.pre = head;
    first.pre = node;
    node.next = first;
}
```

get方法，从map里拿Value，同时将它置为链表头

```java
public V get(K key) {
    if (!map.containsKey(key)) {
        return null;
    }
    Node<K, V> node = map.get(key);
    raiseNode(node);
    return node.value;
}
```


save方法，如果缓存已满，删除链表尾的值，再添加新的值到链表头
```java
public void save(K key, V value) {
    if (map.containsKey(key)) {
        updateNode(key, value);
    } else {
        insertNode(key, value);
    }
}

private void updateNode(K key, V value) {
    Node node = map.get(key);
    node.value = value;
    raiseNode(node);
}

private void insertNode(K key, V value) {
    if (isFull()) {
        removeLast();
    }
    Node node = new Node();
    node.key = key;
    node.value = value;
    setFirst(node);
    map.put(key, node);
}

private boolean isFull() {
    return map.size() >= capacity;
}
```



测试
```java
import org.junit.Test;

public class LRUCacheTest {

  LRUCache<Integer, Integer> cache = new LRUCache(3);

  @Test
  public void test() {
    cache.save(1, 7);
    cache.save(2, 0);
    cache.save(3, 1);
    cache.save(4, 2);
    assert 0 == cache.get(2);
    assert null == cache.get(7);
    cache.save(5, 3);
    assert 0 == cache.get(2);
    cache.save(6, 4);
    assert null == cache.get(4);
  }

}

//head -> 7 -> tail
//head -> 0 -> 7 -> tail
//head -> 1 -> 0 -> 7 -> tail
//head -> 2 -> 1 -> 0 -> tail
//head -> 3 -> 2 -> 1 -> tail
//head -> 2 -> 3 -> 1 -> tail
//head -> 4 -> 2 -> 3 -> tail
```