---
title: 【本科时期文章】从Redis I/O多路复用到Java NIO Selector
date: 2019-06-04 15:19:21
categories: 开发
tags: 
- Redis
- Java
---

### Redis的I/O多路复用架构
Redis的一大特点就是单线程架构。单线程架构既避免了多线程可能产生的竞争问题，又避免了多线程的频繁上下文切换问题，是Redis高效率的保证。


<!-- more -->

对于网络I/O操作，Redis基于 Reactor 模式可以用单个线程处理多个Socket。内部实现为使用文件事件处理器(file event handler)进行网络事件处理器，这个文件事件处理器是单线程的。文件事件处理器采用` I/O 多路复用机制(multiplexing)`同时监听多个 socket。产生事件的 socket 压入内存队列中，事件分派器根据 socket 上的事件类型来选择对应的事件处理器进行处理。操作包括应答（accept）、读取（read）、写入（write）、关闭（close）等。文件事件处理器的结构包含 4 个部分：
- 多个 socket
- I/O 多路复用程序
- 文件事件分派器
- 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）
连接应答处理器会创建一个能与客户端通信的 socket01，通过这个返回结果给客户端。Redis单线程的核心就是I/O 多路复用程序。

I/O多路复用（IO Multiplexing）有时也称为异步阻塞IO，是一种事件驱动的I/O模型。单个I/O操作在一般情况下往往不能直接返回，传统的阻塞 I/O 模型会阻塞直到系统内核返回数据。而在 I/O 多路复用模型中，系统调用select/poll/epoll 函数会不断的查询所监测的 socket 文件描述符，查看其中是否有 socket 准备好读写了，如果有，那么系统就会通知用户进程。

Redis 的 I/O 多路复用程序的所有功能都是通过包装常见的 select 、 epoll 、 evport 和 kqueue 这些 I/O 多路复用函数库来实现的， 每个 I/O 多路复用函数库在 Redis 源码中都对应一个单独的文件。

以ae_select.c实现的封装select方法为例。`select`方法定义如下所示，检测是否可读、可写、异常，返回准备完毕的descriptors个数。
```c
extern int select (int __nfds, fd_set *__restrict __readfds,
		   fd_set *__restrict __writefds,
		   fd_set *__restrict __exceptfds,
		   struct timeval *__restrict __timeout);
```
Redis封装首先通过`aeApiCreate`初始化 rfds 和 wfds，注册到aeEventLoop中去。
```c
static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    FD_ZERO(&state->rfds);
    FD_ZERO(&state->wfds);
    eventLoop->apidata = state;
    return 0;
}
```

而 `aeApiAddEvent` 和 `aeApiDelEvent` 会通过 FD_SET 和 FD_CLR 修改 fd_set 中对应 FD 的标志位。
```c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;

    if (mask & AE_READABLE) FD_SET(fd,&state->rfds);
    if (mask & AE_WRITABLE) FD_SET(fd,&state->wfds);
    return 0;
}

static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;

    if (mask & AE_READABLE) FD_CLR(fd,&state->rfds);
    if (mask & AE_WRITABLE) FD_CLR(fd,&state->wfds);
}
```

`aeApiPoll`是实际调用 select 函数的部分，其作用就是在 I/O 多路复用函数返回时，将对应的 FD 加入 aeEventLoop 的 fired 数组中，并返回事件的个数：
```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, j, numevents = 0;

    memcpy(&state->_rfds,&state->rfds,sizeof(fd_set));
    memcpy(&state->_wfds,&state->wfds,sizeof(fd_set));

    retval = select(eventLoop->maxfd+1,
                &state->_rfds,&state->_wfds,NULL,tvp);
    if (retval > 0) {
        for (j = 0; j <= eventLoop->maxfd; j++) {
            int mask = 0;
            aeFileEvent *fe = &eventLoop->events[j];

            if (fe->mask == AE_NONE) continue;
            if (fe->mask & AE_READABLE && FD_ISSET(j,&state->_rfds))
                mask |= AE_READABLE;
            if (fe->mask & AE_WRITABLE && FD_ISSET(j,&state->_wfds))
                mask |= AE_WRITABLE;
            eventLoop->fired[numevents].fd = j;
            eventLoop->fired[numevents].mask = mask;
            numevents++;
        }
    }
    return numevents;
}
```
epoll函数的封装类似。区别在于 epoll_wait 函数返回时并不需要遍历所有的 FD 查看读写情况；在  epoll_wait 函数返回时会提供一个 epoll_event 数组，其中保存了发生的 epoll 事件（EPOLLIN、EPOLLOUT、EPOLLERR 和 EPOLLHUP）以及发生该事件的 FD。Redis封装的调用只需要将`epoll_event`数组中存储的信息加入eventLoop的 fired 数组中，将信息传递给上层模块：
```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}
```

当Socket变得可读时（客户端对Socket执行 write 操作，或者执行 close 操作）， 或者有新的可应答（acceptable）Socket出现时（客户端对服务器的监听Socket执行 connect 操作），Socket产生 AE_READABLE 事件。而当Socket变得可写时（客户端对Socket执行 read 操作）， Socket产生 AE_WRITABLE 事件。
I/O 多路复用程序允许服务器同时监听Socket的 AE_READABLE 事件和 AE_WRITABLE 事件， 如果一个Socket同时产生了这两种事件， 那么文件事件分派器会优先处理 AE_READABLE 事件， 等到 AE_READABLE 事件处理完之后， 才处理 AE_WRITABLE 事件。换句话说， 如果一个Socket又可读又可写的话， 那么服务器将先读Socket， 后写Socket。

### Java NIO Selector
Java中也有I/O多路复用的方式，例子为NIO的`Selector`。
`selector`的创建方式为调用`Selector`类的静态方法，由`SelectorProvider`提供：`Selector selector = Selector.open();`
```java
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
```
`SelectorProvider`是单例模式，Linux默认提供`EPollSelectorProvider`，即提供的Selector为`EPollSelectorImpl`。
```java
public static SelectorProvider provider() {
    synchronized (lock) {
        if (provider != null)
            return provider;
        return AccessController.doPrivileged(
            new PrivilegedAction<SelectorProvider>() {
                public SelectorProvider run() {
                    if (loadProviderFromProperty())
                        return provider;
                    if (loadProviderAsService())
                        return provider;
                    provider = sun.nio.ch.DefaultSelectorProvider.create();
                    return provider;
                }
            });
    }
}

//.....

/**
     * Returns the default SelectorProvider.
     */
public static SelectorProvider create() {
    String osname = AccessController
        .doPrivileged(new GetPropertyAction("os.name"));
    if (osname.equals("SunOS"))
        return createProvider("sun.nio.ch.DevPollSelectorProvider");
    if (osname.equals("Linux"))
        return createProvider("sun.nio.ch.EPollSelectorProvider");
    return new sun.nio.ch.PollSelectorProvider();
}
```

调用系统Epoll方法的地方在`EPollArrayWrapper`类的`poll`方法中，该类由`EPollSelectorImpl`持有：
```java
int poll(long timeout) throws IOException {
    updateRegistrations();
    updated = epollWait(pollArrayAddress, NUM_EPOLLEVENTS, timeout, epfd);
    for (int i=0; i<updated; i++) {
        if (getDescriptor(i) == incomingInterruptFD) {
            interruptedIndex = i;
            interrupted = true;
            break;
        }
    }
    return updated;
}
```

`Selector`使用中需要绑定`Channel`。以`ServerSocketChannel`为例：
```java
ServerSocketChannel serverSocket = ServerSocketChannel.open();
serverSocket.bind(new InetSocketAddress("localhost", 5454));
serverSocket.configureBlocking(false);
serverSocket.register(selector, SelectionKey.OP_ACCEPT);
```
注册时会调用`Selector`的回调方法`register`，生成`SelectionKey`。
```java
protected final SelectionKey register(AbstractSelectableChannel ch,
                                      int ops,
                                      Object attachment)
{
    if (!(ch instanceof SelChImpl))
        throw new IllegalSelectorException();
    SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
    k.attach(attachment);
    synchronized (publicKeys) {
        implRegister(k);
    }
    k.interestOps(ops);
    return k;
}
```

最后在使用时根据`SelectionKeys`遍历查看状态。可以通过监听的事件有：
- Connect – OP_CONNECT client尝试连接
- Accept – OP_ACCEPT server端接受连接
- Read – OP_READ server端可以开始从channel里读取
- Write – OP_WRITE server端可以向channel里写

使用方式类似：
```java
while (true) {
    selector.select();
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> iter = selectedKeys.iterator();
    while (iter.hasNext()) {

        SelectionKey key = iter.next();

        if (key.isAcceptable()) {
            register(selector, serverSocket);
        }

        if (key.isReadable()) {
            answerWithEcho(buffer, key);
        }
        iter.remove();
    }
}
```

`Selector`的wakeup()方法主要作用是解除阻塞在Selector.select()/select(long)上的线程，立即返回，调用了本地的中断方法。可以在注册了新的channel或者事件、channel关闭，取消注册时使用，或者优先级更高的事件触发（如定时器事件），希望及时处理。

通过NIO的I/O多路复用方式可以节约线程资源，提高网络I/O效率。


#### 参考
- [Redis 设计与实现-文件事件](http://redisbook.com/preview/event/file_event.html)
- [Redis 和 I/O 多路复用](https://draveness.me/redis-io-multiplexing)
- [Introduction to the Java NIO Selector](https://www.baeldung.com/java-nio-selector)