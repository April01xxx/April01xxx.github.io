---
layout: post
title: "Redis源码intset结构"
date: 2018-11-19 21:58
categories: [redis]
tags: [redis, C]
---

这篇主要描述Redis中`整数集合(intset)`的实现.顾名思义,这种数据结构是用来存储整数
类型的.当需要存储的数据都是整数且个数较少时,内部会采用这种结构.此外这种数据结构
还有个性质是:`内部元素都是有序的`.

# intset结构
---
`intset`结构的定义在头文件`intset.h`中:
```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```
整个结构的定义比较简单:
+ **encoding:** 编码方式,用来表明该集合中存储的数据的类型,目前有三种类型:`INTSET_ENC_INT16`,
`INTSET_ENC_INT32`,`INTSET_ENC_INT64`.分别表示存储的整数占用的`bit`位数:`16bit`,`32bit`,
`64bit`.
+ **length:** 集合中元素的个数.
+ **contents[]:** 集合中存放数据的地方,注意这是一个`柔性数组`,且类型为`int8_t`,也就是说
数组中的每个元素占用`8bit`空间.这样做的目的是为了节省内存,倘若集合中的元素取值范围都在
`[-32768, 32767]`之间,那么每个元素只需要`16bit`即可.凡事有利有弊,这样做虽然节省了内存,但
增加了编程的复杂性:例如一个整数占用了16bit,那么在contents中获取该元素的值时,就需要注意
[字节序][byte order]的问题.在Redis内部根据机器的配置(`config.h`文件中的配置信息)判断本机
属于`大端`还是`小端`,若为`大端`则会转换为`小端`存储.**也就是说整个intset结构都是以小端字
节序方式存储的.**

[byte order]: https://en.wikipedia.org/wiki/Endianness

# intset集合插入
---
关于intset集合的操作,这里只分析下元素的插入,整个逻辑由`intsetAdd`函数实现,相关源码在
`intset.c`文件中:
```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```
由于整个集合底层是采用数组形式保存元素,故插入元素的平均时间复杂度是O(N).代码中出现类似
`intrev32ifbe`函数实际上是一个宏定义,功能是当机器字节序是大端时将其反转,也就是转换为
小端(int reverse if big endian).大致看下整个插入的逻辑:
1. 计算待插入元素`value`的编码方式`valenc`,说白了就是计算需要多少bit位来存储.
2. `if (valenc > intrev32ifbe(is->encoding))`这条语句的含义是比较当前集合所采用的编码
方式和待插入元素的编码方式的大小,**如果待插入元素的编码方式大,那么需要增加原集合中每个
元素占用的bit位.**这个操作是通过`intsetUpgradeAndAdd`函数完成的,接下来会简单剖析下这个
函数.
3. 如果待插入元素的编码方式小于等于集合目前采用的方式,那么待插入元素按照集合目前编码
方式存储.
4. 编码的问题搞定了,接下来就是找到元素的插入位置.由于整个集合中的元素是有序的,故调用
`intsetSearch`函数通过二分查找来寻找合适的插入位置.`intset`集合中不允许出现重复元素,
故若已经存在则直接返回.
5. 在一个数组中插入元素,代价是非常高的.这里先调用`intsetResize`函数重新分配空间,若待插入
的元素不在最后,则会调用`intsetMoveTail`函数将要插入位置后面的元素向后移动一格.函数内部是
采用[memmove][memmove]实现批量拷贝,相比于`memcpy`函数,`memmove`在拷贝区域发生重叠时仍能
正常工作.

[memmove]: https://en.cppreference.com/w/c/string/byte/memmove

# intsetUpgradeAndAdd集合元素空间提升
---
上面提到当待插入元素占用的bit数超过集合目前所采用的bit数时会调用`intsetUpgradeAndAdd`
函数提升集合中所有元素的占用空间.其内部实现这一逻辑的核心函数是`_intsetSet`,源码也在
`intset.c`文件中:
```c
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```
之所以单独把这个函数拿出来讲,是因为这里涉及到语言底层的一些细节.我们以集合元素都要
提升至64个bit位为例来说明.整个逻辑由`((int64_t*)is->contents)[pos] = value`这行代码
完成.`memrev64ifbe(((int64_t*)is->contents)+pos)`这行代码的作用是:如果机器是大端字节
序,则将整数转换为小端字节序(memory reverse if big endian).

要理解为何这一行代码就完成了空间的提升,需要理解C语言中指针运算的逻辑:
1. `(int64_t *)is->contents`:这是一个强制类型转换,将`int8_t *`转换为`int64_t *`.
2. `ptr[pos]`的含义等同于`ptr+pos`,实际位移的字节数和指针`ptr`所指向的类型有关.这里
`ptr`是指向`占用8字节`的整型数组的指针,故实际位移字节数是`pos*8`.
理解了这两点,这段代码的逻辑自然就清楚了.


