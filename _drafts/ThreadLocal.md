---
layout: post
title: ThreadLocal
date: 2019-03-24 22:12:30
categories: 开发
tags: 
- 并发
- Java
---

ThreadLocal的作用并不是解决多线程共享变量的问题，而是存储那些线程间隔离，但在不同方法间共享的变量。这是线程安全的一种无同步方案，另一种是无同步方案是幂等的可重入代码。

下面先模拟一个基本的ThreadLocal存储User id的模型，然后解析原理。

<!-- more -->

---
#### 示例
```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadLocalTest {

  //工作线程
  class Worker implements Runnable {

    ThreadLocal<Integer> userId = ThreadLocal.withInitial(() -> 1);

    UserRepo userRepo;

    Worker(UserRepo userRepo) {
      this.userRepo = userRepo;
    }

    @Override
    public void run() {
      for (int i = 0; i < 10; i++) {
        handler();
        try {
          Thread.sleep(30);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    }

    private void handler() {
      userId.set(userRepo.getUserId());
      System.out.println(Thread.currentThread().getName() + " userId: " + userId.get());
    }

  }

  //模拟拿自增user id
  class UserRepo {

    private AtomicInteger incrUserId = new AtomicInteger(1);

    private Integer getUserId() {
      return incrUserId.getAndIncrement();
    }

  }

  private void test() {
    UserRepo userRepo = new UserRepo();
    for (int i = 0; i < 15; i++) {
      new Thread(new Worker(userRepo)).start();
    }
  }

  public static void main(String[] args) {
    ThreadLocalTest test = new ThreadLocalTest();
    test.test();
  }
  
}
```

结果如下
```
........(上略)
Thread-13 userId: 135
Thread-0 userId: 136
Thread-2 userId: 137
Thread-1 userId: 138
Thread-4 userId: 139
Thread-5 userId: 140
Thread-3 userId: 141
Thread-6 userId: 142
Thread-7 userId: 143
Thread-9 userId: 144
Thread-10 userId: 145
Thread-11 userId: 146
Thread-8 userId: 147
Thread-12 userId: 149
Thread-14 userId: 148
Thread-13 userId: 150
```

---
#### 原理
核心是ThreadLocal的内部静态类ThreadLocalMap。map的key是ThreadLocal对象，value是和ThreadLocal对象有关联的值。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
注意内部Entry是WeakReference的，原因是出于性能考虑。由于不是强联系，所以其他正在使用ThreadLocal的线程，不会妨碍gc那些来自同一个ThreadLocal的终止后的线程的变量，简单来讲就是待gc的变量会被正确gc。

在ThreadLocalMap 的 remove 方法中，除了讲entry的引用设为null以外，还调用了一个expungeStaleEntry方法：
```java
if (e.get() == key) {
    e.clear();
    expungeStaleEntry(i);
    return;
}
```

其中会将所有键为 null 的 Entry 的值设置为 null，这样可以防止内存泄露，已经不再被使用且已被回收的 ThreadLocal 对象对应的Entry也会被gc清除：
```java
if (k == null) {
    e.value = null;
    tab[i] = null;
    size--;
}
```
在同样的还有rehash, resize方法方法中，也有类似的设置value为null的操作。


在创建线程时，该线程持有threadLocals。这个引用是在ThreadLocal的createMap方法中设定的，否则为null。
```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

调用ThreadLocalMap的构造方法：
```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```



再返回来看ThreadLocal就很好理解了
get方法：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t); //获取当前线程的ThreadLocalMap
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this); //从map中取值，key就是当前ThreadLocal对象
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

set方法：
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t); //获取当前线程的ThreadLocalMap
    if (map != null)
        map.set(this, value); //向map中存值，key就是当前ThreadLocal对象
    else
        createMap(t, value); 
}
```
---

#### 应用
很常见的应用在Session中存储数据。一个Session对应一个线程，对应一组线程内方法间的共享变量，这些变量都可以由ThreadLocal存储。

参考下[结合ThreadLocal来看spring事务源码，感受下清泉般的洗涤！](https://www.cnblogs.com/youzhibing/p/6690341.html)，可以看到在Spring事务中，也有类似ThreadLocal的操作，将数据库connection绑定到当前线程，使用的也是一个map。


