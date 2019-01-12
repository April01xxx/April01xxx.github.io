---
layout: post
title: "Unix环境编程之self pipe"
date: 2019-01-12 15:50
categories: [postgres]
tags: [C]
---

最近在看PostgreSQL的源码,充斥各种细节,有些是系统运作机制方面的,有些是实现技巧
方面的.直观的感受就是不要积攒这些弄不懂的细节,否则一个一个的"小不懂"积攒起来
就算读完了整个代码,还是一种似似而非,如坠云雾的感觉,没个头绪.今天要说的是一个
实现技巧方面的细节.

# 信号丢失?
信号是一种软中断,可用作进程间通信的一种手段.因为信号可能在`任何时候`到达,故编
写可靠的程序需要时刻注意潜在的`race condition`.只考虑可能会发生信号的情况下,
程序中要做到可靠的信号处理是不难的,但往往我们的程序逻辑比较复杂,最典型的莫过
于:既要及时响应用户输入,同时可能还要监控其他程序发送的消息.解决这类场景的方法
一般是多线程或者IO复用.我们今天要谈论的问题就是:`信号处理+IO复用`.

不同平台实现了自身的IO复用组件,典型的有`select/epoll/kqueue`,由于经典的APUE是
基于select来描述的,接下来的例子也以select为例来说明.

考虑如下的一种应用场景:
1. 程序需要监控某个信号,例如发生了SIGUSR1信号后设置一个全局变量的值表示某种准
备条件已经就绪接下来可以做相应处理了.
2. 程序还要监控一些文件描述符的可读写事件,例如socket描述符的读写;

以上两个问题单独来看都很简单,分别做如下处理即可:
1. 为SIGUSR1信号安装信号处理函数;
2. 利用IO复用机制,将待监听的文件描述符及其相关事件放到一个集合中由select函数
实现.

典型的代码结构可能如下:
```c
void signal_handler(int signo)
{
    some_condition = true;
}

void process()
{
    fd_set rset, wset;

    FD_ZERO(&rset);
    FD_ZERO(&wset);

    FD_SET(read_socket, &rset);
    FD_SET(write_socket, &wset);

    install_signal_handler(SIGUSR1, signal_handler);

    nready = select(maxfd + 1, &rset, &wset, NULL, NULL);

    // woke up
    if (some_condition)
        // do something

    while (nready-- > 0)
        // do something
}
```
乍一看这段代码好像没问题,但我们说过:信号可能在`任何时刻`到达,无法预测.就像你
没法预测你女朋友啥时候会生气一样.上面这段代码的意图是说:无论是发生了信号还是
有描述符准备好读写了就从select中"醒来"处理.倘若在`install_signal_handler`和
`select`之间发生了信号怎么办?答案是直到有某个描述符准备好可读写了才会从select
中醒来,看起来就像`信号丢失`了一样.有人可能会说我们可以在安装信号处理函数之前
阻塞相关信号,开始调用select之前再放开阻塞.问题依旧,只是这样可以将race condition
的空窗期缩小,某些极端情况下还是会出问题.

# 如何解决
记得当初看APUE的时候,书上只说如果你接下来是要陷入睡眠比如pause,可以使用系统
提供的`sigsuspend`函数:
```c
#include <signal.h>

int sigsuspend(const sigset_t *sigmask);
```
该函数接收一个信号掩码作为参数,调用该函数会`原子`地将进程的信号掩码设置为输入
参数指定的值,`然后陷入休眠状态直到发生了某个信号`.但我们的需求是要调用select
处理文件描述符的可读写事件,如何解决这个问题书上没说(我看的是第二版,后续版本中
不知是否有提出如何解决该问题),后来也没太关注,直到最近看PostgreSQL的源码,里面
在每个子进程启动时会创建一个管道,奇怪的是该管道的读端和写端都由该进程本身持有,
一般来说管道是用作进程间通信的,非常不理解PG中这样做的意图.遂Google了一番,小有
所得.机智的小伙伴看到这里可能已经知道利用管道可以解决APUE书上提到的那个问题.

# 私有管道(self pipe)
由于这个管道的读写两端都由该进程本身持有,故称之为私有管道(self pipe).所谓管道,
本质上还是文件描述符(Unix下一切皆文件),既然是文件描述符,那就可以用IO复用组件
来处理.

为了解决`信号处理+IO复用`应用场景下存在的race condition问题,我们需要在进程启动
时创建一个私有管道,具体做法如下:
1. 调用pipe创建一个管道.
2. 将管道的读写文件描述符都设置为非阻塞.这一步很关键.
3. 将管道的读描述符添加到select监听的读描述符集中.
4. 当发生了相关的信号,在信号处理函数中向管道的写端写入相关的信息,表明信号发生了.

接下来的事情就很普通了,select发现管道可读,于是"醒来".完美解决了信号"丢失"问题.

# demo
光说不练假把式,show me your code.
```c
/**
 * test for signal.
 */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/uio.h>

#define UNUSED(x) ((x) = (x))

/* count the captured signal. */
static sig_atomic_t count = 0;

/* self pipe used to solve the race condition between signal and select. */
static int selfpipe[2] = { -1, -1 };

/**
 * signal handler for SIGUSR1 and SIGINT.
 */
void
sig_handler(int signo)
{
    if (signo == SIGUSR1)
        ++count;
    UNUSED(signo);
#ifdef USE_SELFPIPE
    if (write(selfpipe[1], &signo, sizeof(signo)) != sizeof(signo)) {
        printf("write error.\n");
        exit(-1);
    }
#endif
}

/**
 * install signal handler.
 */
typedef void (*sig_t)(int);

int
install_sig_handler(int signo, sig_t func)
{
    struct sigaction act, oact;

    act.sa_handler = func;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;

    return sigaction(signo, &act, &oact);
}

/**
 * create self pipe.
 */
void
create_selfpipe(int selfpipe[2])
{
    if (pipe(selfpipe) < 0) {
        printf("create self pipe error.\n");
        exit(-1);
    }

    /* set pipe unblock */
    if (fcntl(selfpipe[0], F_SETFL, O_NONBLOCK) < 0 ||
        fcntl(selfpipe[1], F_SETFL, O_NONBLOCK) < 0) {
        printf("fcntl selfpipe unblock error.\n");
        exit(-1);
    }
}

int
main(int argc, char *argv[])
{
    pid_t pid;
    fd_set rset;
    int nready;

    UNUSED(argc);
    UNUSED(argv);

    create_selfpipe(selfpipe);

    install_sig_handler(SIGUSR1, sig_handler);
    install_sig_handler(SIGINT, sig_handler);

    pid = getpid();
    printf("My process ID is: %d\n", pid);

    printf("Sleeping...\n");
    sleep(5);
    printf("Woke up...\n");

    for ( ; ; ) {
        FD_ZERO(&rset);
        FD_SET(selfpipe[0], &rset);

        nready = select(selfpipe[0] + 1, &rset, NULL, NULL, NULL);
        if (nready < 0) {
            /* error occurs */
            if (errno != EINTR && errno != EAGAIN) {
                printf("select error.\n");
                exit(-1);
            }
        }

        while (nready-- > 0) {
            if (FD_ISSET(selfpipe[0], &rset)) {
                int signo;

                if (read(selfpipe[0], &signo, sizeof(signo)) != sizeof(signo)) {
                    printf("read error.\n");
                    exit(-1);
                }
                if (signo == SIGUSR1)
                    printf("signal SIGUSR1 captured. count = %d\n", count);
                else {
                    printf("other signals captured, exit with errcode 0.\n");
                    exit(0);
                }
            }
        }
    }
    return 0; /* nerver reach here */
}
```
编译时若不添加`-DUSE_SELFPIPE`则在信号处理函数中不会写管道,在main函数中sleep
期间发送的信号就会"丢失",若增加该编译选项,则信号不会"丢失".
