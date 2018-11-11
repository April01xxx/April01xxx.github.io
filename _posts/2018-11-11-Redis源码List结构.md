---
layout: post
title: "Redis源码List结构"
date: 2018-11-11 16:06
categories: [redis]
tags: [redis, C]
---

Redis底层的链表实现,相关的文件是`adlist.h`和`adlist.c`,实现了双端链表,并做了一些
封装.对于链表这种数据结构的运用,在平常的应用开发中比较少,更多的是数组.直到最近在
LeetCode上刷题,遇到一道LRU Cache的设计题,深刻的体会到这种数据结构的魅力.

# ListNode定义
```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```
这是链表中节点的定义,没啥好说的,`prev`和`next`两个指针分别指向当前节点的前一个节点
和后一个节点,`value`中存储的是节点的元素,这里用的是`void *`类型,可以方便存储任何类
型的元素.

# List定义
```c
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```
双端链表的数据抽象,`head`和`tail`指针分别指向链表的头和尾节点,这种设计的初衷是为了
更灵活的访问.`len`则记录了链表中元素的个数.`dup`,`free`和`match`是三个函数指针,功能
如下:
+ <strong>dup</strong>: 复制节点,注意函数的入参是`void *`,说明入参是`value`.
+ <strong>free</strong>: 释放`value`占用的空间.
+ <strong>match</strong>: 比较两个节点的值是否相等.

# ListIter定义
```c
typedef struct listIter {
    listNode *next;
    int direction;
} listIter;
```
链表的迭代器,如果熟悉C++中的迭代概念的话应该很容易理解,这里就是一个简单的封装,使得
对链表元素的访问的语义更加清晰.增加了`direction`变量用来标识迭代器的方向.

