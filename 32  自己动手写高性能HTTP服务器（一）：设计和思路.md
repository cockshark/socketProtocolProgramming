# 32 | 自己动手写高性能HTTP服务器（一）：设计和思路

**在开始编写高性能 HTTP 服务器之前，我们先要构建一个支持 TCP 的高性能网络编程框架，完成这个 TCP 高性能网络框架之后，再增加 HTTP 特性的支持就比较容易了，这样就可以很快开发出一个高性能的 HTTP 服务器程序。**



### 设计需求

在第三个模块性能篇中，我们已经使用这个网络编程框架完成了多个应用程序的开发，这也等于对这个网络编程框架提出了编程接口方面的需求，综合之前的使用经验，这个 TCP 高性能网络框架需要满足的需求有以下三点。

**第一，采用 reactor 模型，可以灵活使用 poll/epoll 作为事件分发实现。**

**第二，必须支持多线程，从而可以支持单线程单 reactor 模式，也可以支持多线程主 - 从 reactor 模式。可以将套接字上的 I/O 事件分离到多个线程上。**

**第三，封装读写操作到 Buffer 对象中。**

**按照这三个需求，正好可以把整体设计思路分成三块来讲解，分别包括反应堆模式设计、I/O 模型和多线程模型设计、数据读写封装和 buffer。今天我们主要讲一下主要的设计思路和数据结构，以及反应堆模式设计。**

### 主要设计思路

#### 反应堆模式设计

反应堆模式，按照性能篇的讲解，主要是设计一个基于事件分发和回调的反应堆框架。这个框架里面的主要对象包括：

- event_loop

你可以把 event_loop 这个对象理解成和一个线程绑定的无限事件循环，你会在各种语言里看到 event_loop 这个抽象。这是什么意思呢？简单来说，它就是一个无限循环着的事件分发器，一旦有事件发生，它就会回调预先定义好的回调函数，完成事件的处理。

**具体来说，event_loop 使用 poll 或者 epoll 方法将一个线程阻塞，等待各种 I/O 事件的发生。**

- channel

**对各种注册到 event_loop 上的对象，我们抽象成 channel 来表示，例如注册到 event_loop 上的监听事件，注册到 event_loop 上的套接字读写事件等。在各种语言的 API 里，你都会看到 channel 这个对象，大体上它们表达的意思跟我们这里的设计思路是比较一致的。**

- acceptor

acceptor 对象表示的是服务器端监听器，acceptor 对象最终会作为一个 channel 对象，注册到 event_loop 上，以便进行连接完成的事件分发和检测。

- event_dispatcher

event_dispatcher 是对事件分发机制的一种抽象，也就是说，可以实现一个基于 poll 的 poll_dispatcher，也可以实现一个基于 epoll 的 epoll_dispatcher。在这里，我们统一设计一个 event_dispatcher 结构体，来抽象这些行为。

- channel_map

channel_map 保存了描述字到 channel 的映射，这样就可以在事件发生时，根据事件类型对应的套接字快速找到 chanel 对象里的事件处理函数。



#### I/O 模型和多线程模型设计

I/O 线程和多线程模型，主要解决 event_loop 的线程运行问题，以及事件分发和回调的线程执行问题。

- thread_pool

**thread_pool 维护了一个 sub-reactor 的线程列表，它可以提供给主 reactor 线程使用，每次当有新的连接建立时，可以从 thread_pool 里获取一个线程，以便用它来完成对新连接套接字的 read/write 事件注册，将 I/O 线程和主 reactor 线程分离。**

- event_loop_thread

event_loop_thread 是 reactor 的线程实现，连接套接字的 read/write 事件检测都是在这个线程里完成的。

#### Buffer 和数据读写

- buffer

**buffer 对象屏蔽了对套接字进行的写和读的操作，如果没有 buffer 对象，连接套接字的 read/write 事件都需要和字节流直接打交道，这显然是不友好的。所以，我们也提供了一个基本的 buffer 对象，用来表示从连接套接字收取的数据，以及应用程序即将需要发送出去的数据。**

- tcp_connection

tcp_connection 这个对象描述的是已建立的 TCP 连接。它的属性包括接收缓冲区、发送缓冲区、channel 对象等。这些都是一个 TCP 连接的天然属性。

**tcp_connection 是大部分应用程序和我们的高性能框架直接打交道的数据结构。我们不想把最下层的 channel 对象暴露给应用程序，因为抽象的 channel 对象不仅仅可以表示 tcp_connection，前面提到的监听套接字也是一个 channel 对象，后面提到的唤醒 socketpair 也是一个 channel 对象。所以，我们设计了 tcp_connection 这个对象，希望可以提供给用户比较清晰的编程入口。**



#### **反应堆模式设计**

#### 概述

下面，我们详细讲解一下以 **event_loop 为核心的反应堆模式设计**。我在文稿里放置了一张 event_loop 的运行详图，你可以对照这张图来理解。

![](image\event_loop.png)

当 event_loop_run 完成之后，线程进入循环，首先执行 dispatch 事件分发，一旦有事件发生，就会调用 channel_event_activate 函数，在这个函数中完成事件回调函数 eventReadcallback 和 eventWritecallback 的调用，最后再进行 event_loop_handle_pending_channel，用来修改当前监听的事件列表，完成这个部分之后，又进入了事件分发循环。

#### event_loop 分析

说 event_loop 是整个反应堆模式设计的核心，一点也不为过。先看一下 event_loop 的数据结构。

**在这个数据结构中，最重要的莫过于 event_dispatcher 对象了。你可以简单地把 event_dispatcher 理解为 poll 或者 epoll，它可以让我们的线程挂起，等待事件的发生**

这里有一个小技巧，就是 event_dispatcher_data，它被定义为一个 void * 类型，可以按照我们的需求，任意放置一个我们需要的对象指针。这样，针对不同的实现，例如 poll 或者 epoll，都可以根据需求，放置不同的数据对象。

event_loop 中还保留了几个跟多线程有关的对象，如 owner_thread_id 是保留了每个 event loop 的线程 ID，mutex 和 con 是用来进行线程同步的。

socketPair 是父线程用来通知子线程有新的事件需要处理。pending_head 和 pending_tail 是保留在子线程内的需要处理的新的事件。

~~~linux

struct event_loop {
    int quit;
    const struct event_dispatcher *eventDispatcher;

    /** 对应的event_dispatcher的数据. */
    void *event_dispatcher_data;
    struct channel_map *channelMap;

    int is_handle_pending;
    struct channel_element *pending_head;
    struct channel_element *pending_tail;

    pthread_t owner_thread_id;
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    int socketPair[2];
    char *thread_name;
};
~~~

下面我们看一下 event_loop 最主要的方法 event_loop_run 方法，前面提到过，event_loop 就是一个无限 while 循环，不断地在分发事件。

~~~linux

/**
 *
 * 1.参数验证
 * 2.调用dispatcher来进行事件分发,分发完回调事件处理函数
 */
int event_loop_run(struct event_loop *eventLoop) {
    assert(eventLoop != NULL);

    struct event_dispatcher *dispatcher = eventLoop->eventDispatcher;

    if (eventLoop->owner_thread_id != pthread_self()) {
        exit(1);
    }

    yolanda_msgx("event loop run, %s", eventLoop->thread_name);
    struct timeval timeval;
    timeval.tv_sec = 1;

    while (!eventLoop->quit) {
        //block here to wait I/O event, and get active channels
        dispatcher->dispatch(eventLoop, &timeval);

        //handle the pending channel
        event_loop_handle_pending_channel(eventLoop);
    }

    yolanda_msgx("event loop end, %s", eventLoop->thread_name);
    return 0;
}
~~~

代码很明显地反映了这一点，这里我们在 event_loop 不退出的情况下，一直在循环，循环体中调用了 dispatcher 对象的 dispatch 方法来等待事件的发生。

#### event_dispacher 分析

为了实现不同的事件分发机制，这里把 poll、epoll 等抽象成了一个 event_dispatcher 结构。event_dispatcher 的具体实现有 poll_dispatcher 和 epoll_dispatcher 两种，实现的方法和性能篇21讲和22 讲类似，这里就不再赘述，你如果有兴趣的话，可以直接研读代码

~~~linux

/** 抽象的event_dispatcher结构体，对应的实现如select,poll,epoll等I/O复用. */
struct event_dispatcher {
    /**  对应实现 */
    const char *name;

    /**  初始化函数 */
    void *(*init)(struct event_loop * eventLoop);

    /** 通知dispatcher新增一个channel事件*/
    int (*add)(struct event_loop * eventLoop, struct channel * channel);

    /** 通知dispatcher删除一个channel事件*/
    int (*del)(struct event_loop * eventLoop, struct channel * channel);

    /** 通知dispatcher更新channel对应的事件*/
    int (*update)(struct event_loop * eventLoop, struct channel * channel);

    /** 实现事件分发，然后调用event_loop的event_activate方法执行callback*/
    int (*dispatch)(struct event_loop * eventLoop, struct timeval *);

    /** 清除数据 */
    void (*clear)(struct event_loop * eventLoop);
};
~~~

#### channel 对象分析

**channel 对象是用来和 event_dispather 进行交互的最主要的结构体，它抽象了事件分发。一个 channel 对应一个描述字，描述字上可以有 READ 可读事件，也可以有 WRITE 可写事件。channel 对象绑定了事件处理函数 event_read_callback 和 event_write_callback。**

~~~linux

typedef int (*event_read_callback)(void *data);

typedef int (*event_write_callback)(void *data);

struct channel {
    int fd;
    int events;   //表示event类型

    event_read_callback eventReadCallback;
    event_write_callback eventWriteCallback;
    void *data; //callback data, 可能是event_loop，也可能是tcp_server或者tcp_connection
};
~~~

#### channel_map 对象分析

event_dispatcher 在获得活动事件列表之后，需要通过文件描述字找到对应的 channel，从而回调 channel 上的事件处理函数 event_read_callback 和 event_write_callback，为此，设计了 channel_map 对象。

~~~linux

/**
 * channel映射表, key为对应的socket描述字
 */
struct channel_map {
    void **entries;

    /* The number of entries available in entries */
    int nentries;
};
~~~

channel_map 对象是一个数组，数组的下标即为描述字，数组的元素为 channel 对象的地址。

比如描述字 3 对应的 channel，就可以这样直接得到。

~~~linux

struct chanenl * channel = map->entries[3];
~~~

这样，当 event_dispatcher 需要回调 chanel 上的读、写函数时，调用 channel_event_activate 就可以，下面是 channel_event_activate 的实现，在找到了对应的 channel 对象之后，根据事件类型，回调了读函数或者写函数。注意，这里使用了 EVENT_READ 和 EVENT_WRITE 来抽象了 poll 和 epoll 的所有读写事件类型。

~~~linux

int channel_event_activate(struct event_loop *eventLoop, int fd, int revents) {
    struct channel_map *map = eventLoop->channelMap;
    yolanda_msgx("activate channel fd == %d, revents=%d, %s", fd, revents, eventLoop->thread_name);

    if (fd < 0)
        return 0;

    if (fd >= map->nentries)return (-1);

    struct channel *channel = map->entries[fd];
    assert(fd == channel->fd);

    if (revents & (EVENT_READ)) {
        if (channel->eventReadCallback) channel->eventReadCallback(channel->data);
    }
    if (revents & (EVENT_WRITE)) {
        if (channel->eventWriteCallback) channel->eventWriteCallback(channel->data);
    }

    return 0;
}
~~~

#### 增加、删除、修改 channel event

那么如何增加新的 channel event 事件呢？这几个函数是用来增加、删除和修改 channel event 事件的。

~~~linux

int event_loop_add_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);

int event_loop_remove_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);

int event_loop_update_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);
~~~

前面三个函数提供了入口能力，而真正的实现则落在这三个函数上：

~~~linux

int event_loop_handle_pending_add(struct event_loop *eventLoop, int fd, struct channel *channel);

int event_loop_handle_pending_remove(struct event_loop *eventLoop, int fd, struct channel *channel);

int event_loop_handle_pending_update(struct event_loop *eventLoop, int fd, struct channel *channel);
~~~

我们看一下其中的一个实现，event_loop_handle_pendign_add 在当前 event_loop 的 channel_map 里增加一个新的 key-value 对，key 是文件描述字，value 是 channel 对象的地址。之后调用 event_dispatcher 对象的 add 方法增加 channel event 事件。注意这个方法总在当前的 I/O 线程中执行。

~~~linux

// in the i/o thread
int event_loop_handle_pending_add(struct event_loop *eventLoop, int fd, struct channel *channel) {
    yolanda_msgx("add channel fd == %d, %s", fd, eventLoop->thread_name);
    struct channel_map *map = eventLoop->channelMap;

    if (fd < 0)
        return 0;

    if (fd >= map->nentries) {
        if (map_make_space(map, fd, sizeof(struct channel *)) == -1)
            return (-1);
    }

    //第一次创建，增加
    if ((map)->entries[fd] == NULL) {
        map->entries[fd] = channel;
        //add channel
        struct event_dispatcher *eventDispatcher = eventLoop->eventDispatcher;
        eventDispatcher->add(eventLoop, channel);
        return 1;
    }

    return 0;
}
~~~



### 总结

在这一讲里，我们介绍了高性能网络编程框架的主要设计思路和基本数据结构，以及反应堆设计相关的具体做法。在接下来的章节中，我们将继续编写高性能网络编程框架的线程模型以及读写 Buffer 部分



### 思考题

第一道，如果你有兴趣，不妨实现一个 select_dispatcher 对象，用 select 方法实现定义好的 event_dispatcher 接口；

第二道，仔细研读 channel_map 实现中的 map_make_space 部分，说说你的理解。