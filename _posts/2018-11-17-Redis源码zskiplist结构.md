---
layout: post
title: "Redis源码zskiplist结构"
date: 2018-11-17 10:04
categories: [redis]
tags: [redis, C]
image: skiplist1.jpg
---

这篇主要是来说一下Redis中的[跳跃表][skip list]结构,印象中在数据结构相关的教材中跳跃表
好像都不是重点,更多的都在讲述堆栈,队列和树.关于跳跃表的介绍,请参见上述链接,这里不多赘述,
这里主要是记录我对这种数据结构的理解.

[skip list]: https://en.wikipedia.org/wiki/Skip_list

# 链表(linked list)
---
要理解跳跃表,我觉得先要清楚链表这种简单数据结构的优缺点.相比于数组而言,从链表中插入或者
删除一个节点都非常方便,存储空间可以是分散的,但随即访问性能很差.倘若存储的元素都是有序的,
在数组中可以利用[二分查找][binary search]来解决,在链表中则只能顺序遍历.例如有如下有序序列:
`[30, 40, 50, 60, 70, 90]`,要找出元素`70`是否在其中,利用如下二分查找代码只需要3次即可完成:

[binary search]: https://en.wikipedia.org/wiki/Binary_search_algorithm

```c
bool bsearch(int *seq, int n, int target)
{
    int lo = 0, hi = n;

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;

        if (seq[mid] == target)
            return true;
        else if (seq[mid] < target)
            lo = mid + 1;
        else
            hi = mid;
    }
    return false;
}
```
对于链表则只能顺序遍历:
```c
typedef struct list {
    int val;
    struct list *next;
} list;

bool search(list *head, int target)
{
    while (head) {
        if (head->val == target)
            return true;
        head = head->next;
    }
    return false;
}
```
很显然,在链表的遍历过程中,元素有序这个条件我们没利用,而二分查找正是利用了这个条件,所以
才提高了查找效率.但链表这种结构本身是`线性而且离散`的,没法存储元素有序这个信息.那怎么办
呢?我们只能在链表的每个节点中添加一些附加信息来记录.

# 高速公路指示牌
---
在高速上行驶时往往会看到很多指示牌,有一种会提前告知你距离下个高速路出口的距离,如果错过了
就只能等下个出口(严禁逆向行驶!).那在有序链表的结构中可不可以设置类似这样的指示牌呢?答案是
可以,但有几个问题要先解决:
1. 指示牌中应该存储哪些信息?
2. 每隔多远(多少个节点)设置一个指示牌?

针对两个问题,分别说明:
1. 我们设置指示牌是为了提高查找的效率,在这个例子中,我们只需要记录元素的有序性,所以指示牌
中简单的存储下个节点的信息即可.
2. 指示牌间距若设置的太远,提供的信息不足;若设置的太近,浪费空间不说并不能有效的提高查找效率.
其实数据结构和算法的教材中已经给了我们提示:`二分法`.但链表是离散的,怎么二分呢?有一种做法
可以达到类似的效果:
    + 在原链表的节点中,每隔一个节点抽取一个节点,这样组成一个新的链表;假设原链表称为L0,这个链表
    我们称为L1;
    + 在链表L1中每隔一个节点抽取一个节点组成新的链表L2;
    + 重复以上步骤,直至链表中只有两个节点;

示意图如下:
![skiplist1]({{ site.url }}/static/img/_posts/skiplist1.jpg)

经过以上步骤,我们得到了一个分层的链表
