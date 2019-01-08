---
layout: post
title: "PostgreSQL源码中的EXEC_BACKEND"
date: 2019-01-08 17:51
categories: [postgres]
tags: [postgres]
---

断断续续看PG的源码也有一段时间了,关于这个`EXEC_BACKEND`的含义一直没去关注,只知道代码里面很多ifdef的宏定义,用GDB调试的时候也没进去相应的分支,
也就没管它到底是啥意思.不过内心积攒的疑问终于爆发了,决定弄清楚它的意思.

# EXEC_BACKEND
---
在PG源码的注释里面后台进程一般称为`backend`,最开始看到这个宏的时候望文生义,理解为`execute backend`,猜不出个啥意思.但为啥要有些地方的逻辑要加上
这个判断呢?查了些资料终于理解了.PG服务器采用一种多进程架构,核心代码中可以看到`fork`生成相关子进程,而这个EXEC_BACKEND宏是为另外一种方式`fork-exec`
定义的.要理解为什么需要特别定义这个宏,需要理解fork的特性.

# fork
---
虽然APUE断断续续也看过几遍,但细节还是记不住.只知道fork生成的子进程是父进程的拷贝,操作系统一般采用copy-on-write技术来减少开销.一些细节信息比如
子进程是否会继承父进程的信号之类的就记不清了。在CentOS上查看系统手册,这里只摘录PG源码阅读中比较关注的几点：
1. The child does not inherit its parent's memory locks.子进程不会继承父进程的内存锁,这里的内存锁指的是将进程空间中的部分内容保留在内存中避免被
页调度策略刷到交换区（swap area）.
2. The child's set of pending signals is initially empty.如果父进程中有pending的信号,子进程中不会继承.
3. The child does not inherit timers from its parent.子进程不会继承父进程的时钟,例如alarm.

一般复杂的系统例如PG,都会实现自己的内存管理,这种一般在堆上面分配的空间是会被子进程继承的,例如`MemoryContextInit()`函数创建的memory-context,而
通过fork-exec方式创建的进程则不会继承,故在PG的源码中有些地方需要通过EXEC_BACKEND宏来判断是否需要做一些资源的初始化操作.
