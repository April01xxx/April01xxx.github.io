---
layout: post
title: "PostgreSQL之CHECKPOINT"
date: 2019-05-29 10:45
categories: [postgres]
tags: [postgres]
---

最近开始接触PG测试相关的工作，既然是测试嘛，少不了生成测试数据，今天在生成数据时发现PG的系统日志里面频繁的输出以下信息：
>LOG:  checkpoints are occurring too frequently (28 seconds apart)  
>HINT:  Consider increasing the configuration parameter "max_wal_size".

提示的非常明显：`checkpoint发生的太频繁了`。

由于对PG数据库调优没怎么了解，在initdb初始化实例后未修改配置信息，保留默认配置（好吧其实这是我故意的，就想看看默认参数会不会出问题，这样一旦出了问题就知道是哪些参数有影响了）。测试机器一般，32G内存，16核CPU。

# 相关的默认配置参数
-------------------
这里先贴出来与checkpoint过于频繁相关的默认配置参数：
>shared_buffers = 128MB                 # min 128kB  
>checkpoint_timeout = 5min              # range 30s-1d  
>max_wal_size = 1GB  
>min_wal_size = 80MB  
>checkpoint_completion_target = 0.5     # checkpoint target duration, 0.0 - 1.0  
>checkpoint_warning = 30s               # 0 disables  
>full_page_writes = on                  # recover from partial page writes  

各参数的含义就不展开了，有疑问可以查[官方手册][checkpoint]。

[checkpoint]: https://www.postgresql.org/docs/11/runtime-config-wal.html#RUNTIME-CONFIG-WAL-CHECKPOINTS

# 数据库何时执行checkpoint
-------------------------
既然问题是checkpoint太频繁，那先要知道何时会发生checkpoint，关于这点，[The internals of PG][internal PG]有总结：
>1. 时间到了。系统每隔一段时间（`checkpoint_timeout`参数）会执行一次checkpoint。  
>2. WAL日志的大小超过了预设值（`max_wal_size`参数）。  
>3. 以smart或fast模式关闭数据库。  

[internal PG]: http://www.interdb.jp/pg/pgsql09.html#_9.7.

# 问题分析
从测试数据库的系统日志可知，checkpoint太频繁的原因是WAL日志大小超过了预设值（因为上一次checkpoint在28s之前，没超过系统默认值5min），这样看来日志里面给出的提示还是非常准确的：增加max_wal_size的大小。接下来就简单了，修改max_wal_size的值，重启数据库，问题解决。

# 问题解决了吗？
--------------
确实，修改max_wal_size参数可以解决这个问题，但这个问题后面其实还隐藏另一个问题：`为什么WAL日志增长的这么快`？正是因为WAL增长的过快才会导致没到系统默认执行checkpoint的时间（5min）就因日志过大触发checkpoint。

这需要结合实际业务场景来分析。在生成测试数据时（脚本开了10个进程往数据库中插入数据），数据库发生了大量的写入事件，PG采用WAL（日志先于数据落盘）机制保障数据不会丢失，故此时会频繁的写入WAL日志，加上[full_page_writes][fpw]参数默认开启，进一步加速了WAL的增长。

[fpw]: https://www.postgresql.org/docs/11/runtime-config-wal.html#GUC-FULL-PAGE-WRITES


