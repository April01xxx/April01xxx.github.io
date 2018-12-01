---
layout: post
title: "Redis中的事件-file event"
date: 2018-12-01 12:53
categories: [redis]
tags: [redis, C]
---

关于Redis的底层数据结构基本都过了一遍,小有所得.基于这些基础的数据结构,redis封装了
自己的对象类型.很早有个想法是看看每个类型是如何创建,如何存储,如何消亡的.最开始打算
用gdb跟踪调试,结果发现不得其门而入.原因在于缺少对redis服务器结构体系的宏观认识.于是
打算研究下服务器的各个模块.这一篇主要介绍redis中的文件事件.redis的文件事件处理器
(`file event handler`)是基于了[reactor design pattern][reactor]实现的,相关信息可以
参考给出的Wikipedia链接说明,我基于自身的理解也对此做了一个总结:
![reactor pattern][reactor pattern]

[reactor]: https://en.wikipedia.org/wiki/Reactor_pattern
[reactor pattern]: https://github.com/April01xxx/April01xxx.github.io/raw/master/static/img/_posts/reactor_pattern.jpg


# Redis中的文件事件
---
所谓文件事件(file event)我的理解就是redis客户端与服务端,redis主从服务器之间的一系列
网络通信事件的抽象.例如客户端与服务器之间建立连接,读/写事件等.

我们先来看看redis是如何对文件事件进行抽象的.相关的数据结构定义在`ae.h`头文件中:
```c
typedef void aeFileProc(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask);

typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;
```
+ **mask:** 标识文件事件的类型.
+ **rfileProc:** 读事件处理器.
+ **wfileProc:** 写事件处理器.
+ **clientData:** 待处理的数据.

每一个(可读/写)事件都会给它注册相关的`回调函数`,实际上就是对应的事件处理器.当事件
发生时回调相关函数处理数据.这是一种reactor模式.

# 事件状态
---
在redis内部会维护一个所有事件的状态机:`aeEventLoop`:
```c
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```
其中有些成员是与`时间事件`相关的,这里暂不展开.我们先关注`文件事件`涉及的成员:
+ **maxfd:** 当前已经注册了的最大文件描述符.
+ **setsize:** 最大需要处理的文件描述符数量.
+ **events:** 已经注册的事件.
+ **apidata:** 与API相关的特有数据.这是因为对于不同的平台,redis会选择当前性能
最高的`demultiplexer`解复用器处理请求,而它们所需的数据结构是不一样的.接下来会
以`select`实现的解复用器为例进行说明.

在redis服务器启动时会创建一个事件状态机,相关源码在`ae.c`中:
```c
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;

    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    if (eventLoop->events == NULL || eventLoop->fired == NULL) goto err;
    eventLoop->setsize = setsize;
    eventLoop->lastTime = time(NULL);
    eventLoop->timeEventHead = NULL;
    eventLoop->timeEventNextId = 0;
    eventLoop->stop = 0;
    eventLoop->maxfd = -1;
    eventLoop->beforesleep = NULL;
    eventLoop->aftersleep = NULL;
    if (aeApiCreate(eventLoop) == -1) goto err;
    /* Events with mask == AE_NONE are not set. So let's initialize the
     * vector with it. */
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;
    return eventLoop;

err:
    if (eventLoop) {
        zfree(eventLoop->events);
        zfree(eventLoop->fired);
        zfree(eventLoop);
    }
    return NULL;
}
```
`aeApiCreate`函数封装了底层采用的解复用器实现细节,这里以`select`函数实现为例,看看
它到底封装了哪些数据:
```
typedef struct aeApiState {
    fd_set rfds, wfds;
    /* We need to have a copy of the fd sets as it's not safe to reuse
     * FD sets after select(). */
    fd_set _rfds, _wfds;
} aeApiState;

static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    FD_ZERO(&state->rfds);
    FD_ZERO(&state->wfds);
    eventLoop->apidata = state;
    return 0;
}
```
首先它维护了一个应用接口状态`aeApiState`,其中封装了这个接口操作的数据:`rfds`,
`wfds`,`_rfds`,`_wfds`.可以看出这实际上就是`select`函数操作的`文件描述符集fd_set`.
`_rfds`,`_wfds`保存了要扫描的文件描述符集的副本,这是因为`select`函数返回时会修改
传入的参数,每次select调用开始前都要重新设置要扫描的集合.

事件状态机创建成功后,服务器每次接收到新的文件事件就会调用`aeCreateFileEvent`创建
相应的事件`aeFileEvent`,绑定事件处理器`aeFileProc`,并注册到`aeEventLoop->events`中.
接下来的文章中会针对典型的文件事件:客户端连接,命令执行等,探究背后的实现逻辑.
