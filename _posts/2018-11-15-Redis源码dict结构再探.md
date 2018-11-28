---
layout: post
title: "Redis源码dict结构再探"
date: 2018-11-15 19:56
categories: [redis]
tags: [redis, C]
---

之前大致描述了Redis底层dict结构的定义,也提到了在负载因子过高或者过低时会发生rehash.
这次就来仔细看看具体是怎么实现的.先附上一张相关数据结构的关系图:
![dict][dict]

[dict]: https://github.com/April01xxx/April01xxx.github.io/raw/master/static/img/_posts/dict.jpg

# 创建字典dict
---
字典dict相关的实现函数都在`dict.c`文件中,先来看看字典是如何创建的:
```c
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    dict *d = zmalloc(sizeof(*d));

    _dictInit(d, type, privDataPtr);
    return d;
}
```
字典的创建很简单,先调用`zmalloc`(关于这个函数以后会细说,这里先不深入,只需要知道它
的作用类似malloc,是用于分配内存即可)函数分配字典dict结构体所需的内存空间,然后调用
`_dictInit`函数,从函数名大概能猜到是初始化,最后返回指向字典的指针`d`.整个代码非常
紧凑,干练,函数命名也很规范,用下划线`_`打头的一般是内部函数,不对外暴露接口,函数名
很好的表达了函数的功能.

接下来看看`_dictInit`函数做了哪些初始化操作:
```c
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}
```
首先是调用`_dictReset`函数对字典的两张哈希表做处理,接着对字典的成员变量进行初始化.

再来看看`_dictReset`函数的具体内容:
```c
static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```
逻辑也很简单,就是对哈希结构dictht的成员变量做初始化.

通过对这三个函数的分析,字典dict的创建过程应该很清楚了,这里有一点要注意的是:**创建
的字典只分配了相应的字典dict结构体和哈希表dictht结构体所需的内存空间,并未分配实际
用于存储数据的dictEntry结构所需的空间.**说白了就是一个空的字典,那何时真正分配空间呢?
自然是应用需要存储数据的时候,从这里也可以看出来,字典dict结构的大小是动态分配的.

# 字典dict动态调整
---
之前有提到过,字典dict的空间是动态分配的:根据哈希表的负载因子动态调整哈希表的大小.
接下来看看具体是如何调整的.核心逻辑是在`dictRehash`中实现的:
```c
/* 参数n的意义: 控制一次rehash操作的数量.通俗点来讲,就是调用dictRehash一次可以
 * rehash多少个桶.这个参数的目的是为了控制rehash的时间,若n取值很大,则会造成此
 * 函数耗时很久.如果字典正在rehash的过程中发生了增/删/改/查操作,除了完成指定
 * 操作外,也会调用此函数进行rehash,这时传入的n取值为1.
 */
int dictRehash(dict *d, int n) {
    /* 变量empty_visits用来限定在rehash的过程中,最多允许访问的空桶的数量.
     * 之所以进行限制,也是为了保证一次rehash的时间不会太长.假如n取值为10,
     * 同时哈希表的负载因子非常小,这就导致哈希表中很多空桶,如果不计算rehash
     * 过程中访问的空桶数量,那可能会花费非常多的时间才能完成10个桶的rehash
     * 操作.
     */
    int empty_visits = n * 10;
    if (!dictIsRehashing(d)) return 0;

    while (n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* 变量rehashidx表示哈希表ht[0]的rehash进度:索引值小于rehashidx的
         * 桶都已经完成了rehash操作.
         */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while (d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        while (de) {
            uint64_t h;

            nextde = de->next;
            /* 计算rehash后在ht[1]中的索引值.这里要注意的是,rehash后桶里面
             * 元素的顺序会反转.因为在插入ht[1]时,每次都是在链表头部插入.
             */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        /* 每rehash一个桶,rehashidx的值就加1. */
        d->rehashidx++;
    }

    /* 判断是否还需要rehash. */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    return 1;
}
```
之前有提到过,rehash的过程是分步完成的(我们称为渐进式rehash),我们先来看一下rehash
的示意图:[还没画好,画好了补充]

在rehash的过程中,是以`桶(bucket)`为单位进行的,由于Redis中采用链地址法解决hash冲突,
所以一个桶实际上就是一个链表.在rehash的过程中,通过变量`rehashidx`来记录rehash的进度:
`在哈希表ht[0]中索引值小于rehashidx的桶都已经rehash到ht[1]中了.`具体的细节参见上述
代码注释.
