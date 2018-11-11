---
layout: post
title: "Redis源码dict结构初探"
date: 2018-11-11 17:27
categories: [redis]
tags: [redis, C]
---

字典(dictionary)又称为关联数组(associative array)或映射(map),是一种用于保存键值对
(key-value pair)的抽象数据结构.可以说是Redis底层最重要的数据结构了.这篇文章只是对
Redis中dict这种数据结构进行初步的介绍,关于具体实现机理留待下篇细说.

# dictEntry结构
---
```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```
dictEntry结构是哈希表中存储KV对的基本单元.`key`代表键,`v`是一个联合(union),代表值.
`next`是指向下一个dictEntry的指针,其目的是为了解决[哈希冲突][collision],hash冲突的解决方法
有很多,这里选择的是链地址法(separate chaining).

[collision]: https://en.wikipedia.org/wiki/Hash_table#Collision_resolution

# dictht结构(哈希表)
---
```c
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```
哈希表的结构也比较简单,`table`指向一个dictEntry结构的数组,`size`表示哈希表的大小,
`sizemask`是一个掩码,用作计算某个key映射到table中的索引,其大小为`size-1`.`used`
表示哈希表中已有的元素个数.

# dict结构(字典)
---
```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);  // 根据key计算hash值.
    void *(*keyDup)(void *privdata, const void *key); // 复制键的函数.
    void *(*valDup)(void *privdata, const void *obj); // 复制值的函数.
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);  // 对比键的函数.
    void (*keyDestructor)(void *privdata, void *key); // 销毁键的函数.
    void (*valDestructor)(void *privdata, void *obj); // 销毁值的函数.
} dictType;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
对于每个dict结构,有如下成员:
+ <strong>type</strong>: 字典的类型.简单来说,就是字典中存储的数据的类型,为什么需要
这个类型呢?我的理解是,对于不同的类型,比如说整数或者字符串,更复杂的结构体等等,需要不
同的函数,从`struct dictType`的定义也可窥见一二.里面提供了用于计算hash值的函数,键值的
复制/销毁函数,键的比较函数.
+ <strong>privdata</strong>: 保存了需要传给dictType中特定函数的可选参数.
+ <strong>ht[2]</strong>: 两张哈希表.为什么是两张,存储数据的话一张就够了啊?这涉及到
一个概念:[负载因子(load factor)][load factor].我们知道哈希表根据key访问元素的时间复杂度是O(1),
但当哈希表中元素的数量非常多时,哈希冲突的可能性会越来越高,这会使得访问效率下降,为了
解决这个问题,当负载因子达到一定值时需要重建哈希表(rehash).在Redis中,rehash是渐进式的,
并非一次性将所有键值对从ht[0]中rehash到ht[1],而是一次处理一部分,有个中间过程,倘若在
这个过程中,需要查询某个key的value,可能需要查找ht[0]和ht[1].
+ <strong>rehashidx</strong>: rehash标志,若未发生rehash,则取值为-1,开始rehash则将此值
置为0,每发生一次rehash,值就加1,rehash完成后值又置为-1.
+ <strong>iterators</strong>: 当前正在运行的迭代器数量.为了遍历字典而创建迭代器,这个
值表明当前有多少个迭代器在运行.

[load factor]: https://en.wikipedia.org/wiki/Hash_table#Key_statistics

# 渐进式rehash
---
下面简要的描述下rehash的算法,具体的细节留待后面的篇章中描述.
1. 为ht[1]分配空间,针对哈希表的扩展(负载因子过大)或者收缩(负载因子过小)两种情况,
分配空间的策略有所不同:
    + *扩展*: ht[1]的大小为第一个大于等于`2 * ht[0].used`的$2^{n}$(2的n次幂).
    + *收缩*: ht[1]的大小为第一个大于等于`ht[0].used`的$2^{n}$(2的n次幂).
2. 将`rehashidx`的值置为0,表示rehash开始;
3. 在rehash期间,对字典进行删/改/查操作时,程序除了执行指定操作外,还会将对应的key
在ht[0]中的记录rehash到ht[1]中,同时删除ht[0]中的记录,若是新增操作,则直接插入ht[1]中.
每次rehash完成,`rehashidx`的值加1.
4. 当ht[0]中没有元素时,rehash完成,将ht[0]的空间释放,ht[1]置为ht[0],ht[1]指向空哈希表,
`rehashidx`的值置为-1.

上述算法避免了集中式rehash可能造成的性能影响,但会占用额外的空间,而且在rehash的过程中,
删/改/查操作可能要查询ht[0]和ht[1]两张哈希表.此时遍历哈希表也会变得复杂.
