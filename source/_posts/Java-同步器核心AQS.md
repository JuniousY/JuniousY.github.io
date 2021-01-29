---
title: 【本科时期文章】Java 同步器核心AQS
date: 2019-06-14 11:27:12
categories: 开发
tags: 
- 并发
- Java
---

juc(java.util.concurrent) 基于 AQS （ AbstractQueuedSynchronizer ）框架构建锁机制。本文将介绍AQS是如何实现共享状态同步功能，并在此基础上如何实现同步锁机制。

<!-- more -->

## AbstractQueuedSynchronizer
### CLH同步队列
AQS如其名所示，使用了队列。当共享资源（即多个线程竞争的资源）被某个线程占有时，其他请求该资源的线程将会阻塞，进入CLH同步队列。

队列的节点为AQS内部类Node。Node持有前驱和后继，因此队列为双向队列。有如下状态：
- SIGNAL 后继节点阻塞(park)或即将阻塞。当前节点完成任务后要唤醒(unpark)后继节点。
- CANCELLED 节点从同步队列中取消
- CONDITION 当前节点进入等待队列中
- PROPAGATE 表示下一次共享式同步状态获取将会无条件传播下去
- 0 其他

AQS通过头尾指针来管理同步队列，同时实现包括获取锁失败的线程进行入队，释放锁时唤醒对同步队列中的线程。未获取到锁的线程会创建节点线程安全（compareAndSetTail）的加入队列尾部。同步队列遵循FIFO，首节点是获取同步状态成功的节点。

### 获取锁
未获取到锁（tryAcquire失败）的线程将创建一个节点，设置到尾节点。
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    	selfInterrupt();
}

//创建节点至尾节点
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 如果compareAndSetTail失败或者队列里没有节点
    enq(node);
    return node;
}
```

enq是一个CAS的入队方法：
```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
```

acquireQueued方法的作用是获取锁。
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 获取锁成功
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 获取失败则阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 释放锁
首节点的线程在释放锁时，将会唤醒后继节点。而后继节点将会在获取锁成功时将自己设置为首节点。
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒后继节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

### 响应中断式获取锁
可响应中断式锁可调用方法lock.lockInterruptibly();而该方法其底层会调用AQS的acquireInterruptibly方法。
```java
public final void acquireInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 唯一的区别是当parkAndCheckInterrupt返回true时即线程阻塞时该线程被中断，代码抛出被中断异常。
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



### 超时等待获取锁
通过调用lock.tryLock(timeout,TimeUnit)方式达到超时等待获取锁的效果，调用AQS的方法tryAcquireNanos()。
```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
tongbuqi
private boolean doAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 计算等待时间
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
    }
```

### 共享锁的获取
最后看下共享锁的获取。
```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
    	//获取锁失败时调用
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                // 当tryAcquireShared返回值>=0时取得锁
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```


### 队列外成员变量
AQ还有`state`成员变量，volatile int类型，用于同步线程之间的共享状态。当state>0时表示已经获取了锁，对于重入锁来说state值即重入数，当state = 0时表示释放了锁。具体说明见下面各同步器的实现。

## 实现同步器
每一种同步器都通过实现`tryacquire`（包括如`tryAcquireShared`之类的方法）、`tryRelease`来实现同步功能。

### ReentrantLock
主要看获取锁的过程
非公平锁获取锁：
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //如果当前重进入数为0,说明有机会取得锁
    if (c == 0) {
    	//抢占式获取锁 compareAndSetState是原子方法
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果当前线程本身就持有锁，那么叠加重进入数，并且继续获得锁
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    //以上条件都不满足，那么线程进入等待队列。
    return false;
}
```
公平锁获取锁类似：
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
    // 区别之处，非抢占式
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



### Semaphore
以`state`作为信号量使用，例子：
```java
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires; //剩下多少许可资源
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

### CountDownLatch
以`state`作为计数器，`state`为0时等待结束：
```java
public void await() throws InterruptedException {
	//阻塞直到state为0
	sync.acquireSharedInterruptibly(1);
}
```
用同步器方法减少state
```java
public void countDown() {
	sync.releaseShared(1);
}
```
```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```



