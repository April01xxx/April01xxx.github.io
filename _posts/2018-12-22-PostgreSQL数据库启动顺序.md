---
layout: post
title: "PostgreSQL数据库启动进程顺序"
date: 2018-12-22 20:18
categories: [postgres]
tags: [postgres]
---

读源码只读不行,结合GDB调试往往能起到事半功倍的效果,就是有点花时间.加上PG是多进程架构的,用GDB调试起来也有点麻烦,
这里记录下调试过程中的关键点.

# 预备知识
---
用GDB调试多进程结构的程序有以下几个命令要了解:
+ *follow-fork-mode:* 该参数用来设置在fork返回后GDB调试跟随的进程,有两个选项:`child`和`parent`.默认值是`parent`.
+ *detach-on-fork:* 该参数用来设置fork出的子进程是否分离(detach),有两个选项:`on`和`off`.默认值是`on`.
+ *info inferiors:* 该命令用来查看当前运行的进程信息,每个进程都会有一个编号.
+ *inferior Number:* `Number`替换为对应的编号,该命令用来在进程间进行切换.

另外在调试多进程结构的程序时,断点的设置也很关键.就拿PG来说,子进程往往都是服务进程(一般都是死循环),如果断点设置不合理,调试
的时候很容易进入死循环里面.再加上daemon进程往往都会屏蔽SIGINT等信号量,一旦进入死循环,CTRL+C也无法退出,这个时候只能杀
死GDB进程重头来过.一般的做法是在`fork`附近的子进程和父进程入口处分别设置断点,跟随调试时注意不进入死循环就好.

# 关键点
---
这里以`PG 11.1`源码为例进行说明.有以下关键点:
1. *main:* PG的源码是用C语言编写的,当然少不了main函数啦,入口在`src/backend/main/main.c:60`行.
2. *PostmasterMain:* 多用户模式下主要逻辑处理函数入口,在`src/backend/main/main.c:228`行.该函数里面会阻塞一些信号,
包括`SIGCHLD`信号.
3. *reaper:* 这个点非常关键...在`src/backend/postmaster/postmaster.c:657`行.最开始读代码的时候,我的习惯一般是先
大概扫一遍,看下注释,了解流程.这一行代码如下`pgsignal_no_restart(SIGCHLD, reaper);`.非常简单,就是注册了一个`SIGCHLD`
信号的处理函数`reaper`.当时看到这里也没深究,就跳过了,结果导致后面调试设置断点的时候一头雾水.这也算是一个经验:`如果程序结构
是多进程的,而且还专门写了一个函数来处理子进程结束时的清理工作,那么这个子进程的流程一定要过一遍!!!`
4. *SysLoggerPid:* 这个入口在`src/backend/postmaster/postmaster.c:1281`行,这行会`fork`第一个子进程用来记录系统
日志.
5. *StartupDataBase:* 入口在`src/backend/postmaster/postmaster.c:1371`行,嗯,这行前面的注释很带感.相信读过这个
注释的都会会心一笑.
6. *fork_process:* 入口在`src/backend/postmaster/postmaster.c:5317`行.实际上这是`StartupDataBase`函数里面的
处理流程.`这里会fork一个非常关键的子进程,该子进程做完相关处理后会退出.`嗯,描述上看来很平常.想想APUE里面的话,子进程退出后
会给父进程发送一个`SIGCHLD`信号,但是`PostmasterMain`函数里面把这个信号阻塞了,于是主进程这个时候是不知道子进程咋样了.
7. *ServerLoop:* 入口在`src/backend/postmaster/postmaster.c:1379`行.看名字很好理解,服务主循环(死循环).
8. *UnBlockSig:* 入口在`src/backend/postmaster/postmaster.c:1668`行.嗯,这也是一行不起眼的代码:`PG_SETMASK(&UnBlockSig);`.
最开始看源码的时候也就扫了眼,不就是将阻塞的信号放开嘛.等等!刚才说子进程的SIGCHLD信号被阻塞了,这里放开会怎样?!是的没错,
父进程会收到这个信号,`第3点`中说了父进程注册了一个`reaper`函数,关键就在这个函数里面.可能是看APUE上的例子太简单,往往sig_handler
中都只是简单的获取进程的退出状态,但是这个reaper(中文译为收割机)很不一般!

# reaper
---
要说这个reaper为啥不一般,还有些细节要交代.上面第5个关键点,调用`StartupDataBase`函数时,会将子进程的pid保存在一个全局变量
`StartupPID`中,这个pid就是第6点中fork出来的那个不一般的子进程的pid.接下来我们看看reaper中做了啥,入口在`postmaster.c`
文件的`2771`行:
```c
static void
reaper(SIGNAL_ARGS)
{
    ...
    while ((pid = waitpid(-1, &exitstatus, WNOHANG)) > 0)
    {
        if (pid == StartupPID)
        {
            StartupPID = 0;
            ...
            if (CheckpointerPID == 0)
                CheckpointerPID = StartCheckpointer();
            if (BgWriterPID == 0)
                BgWriterPID = StartBackgroundWriter();
            if (WalWriterPID == 0)
                WalWriterPID = StartWalWriter();
            ...
        }
    }
    ... 
}
```
代码很清晰,如果这个结束的子进程是刚刚`StartupDataBase`时的那个子进程,那我们就启动额外的子进程,上面代码中只列举了3个:检查点进程,
后台写日志进程,WAL(write ahead log)进程,还有自动清理进程autovac,归档进程arch,统计进程stat都在这里处理.至此整个PG数据库的启动
顺序就非常清晰了.当这些进程都启动后,父进程就会进入`select`循环中监听描述符上的事件,每当有连接到数据库时,fork一个子进程来处理请求.
