---
layout: post
title: "Redis源码zskiplist结构"
date: 2018-11-17 10:04
categories: [redis]
tags: [redis, C]
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
![skiplist1](https://github.com/April01xxx/April01xxx.github.io/raw/master/static/img/_posts/skiplist1.jpg)

经过以上步骤,我们得到了一个分层的链表,如果要在这个链表中查找元素`80`是否存在,从链表`L2`中知道
元素`70`所在的位置,于是前面的节点都可以忽略,从`70`后面开始查找,而`70`的后面是`NULL`节点,故接下
来在链表`L1`中查找,发现后面还是`NULL`,只能再向下到链表`L0`中查找,结果发现后面是`90`,故元素`80`
不在链表中.

至此通过在链表中添加附加信息形成多层链表有效的提高了查找的效率,但还有些问题上面没有提及.我们在
设置附加信息时是采取了`等间距抽样`的方式,只有每段的元素个数均匀的情况下,查找的效率才会有效提升.
这也是为什么快速排序算法中非常注重枢纽元选取的原因:选取合适的枢纽元,分割后的区间中元素个数尽可能
的均匀,快排的效率才能达到O(logN).对于跳跃表而言,如果只是查询,没有插入或者删除操作,这样等间距设置的
方法没问题,但若要往链表中插入元素,又或者删除元素,为了保持元素的均匀分布,在插入或者删除后需要调整
我们设置的`"指示牌"`位置,这是一种级联操作,会影响到整个链表,虽然查询的效率提高了,但插入和删除的效率
却还是O(N).那怎么解决这个问题呢?让我们来看看Redis中是如何处理的.

# 随机化 && 幂法则(Power Law)
---
Redis中跳跃表基本是依据William Pugh发表的关于跳跃表的论文实现的:[Skip Lists: A Probabilistic Alternative to Balanced Trees][pdf],
链表中的一个节点能否被提升到上层链表中是由概率决定的:随机化.在提升节点的过程中,又必须依据
[幂法则][power law],通俗点来讲就是`节点本身的层数越高,被提升的概率就越低`.这个逻辑是由`zslRandomLevel`
函数实现的,源码在`t_zset.c`中:
```c
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^32 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
有几点要注意:
1. 每个节点的层数有个上限:`ZSKIPLIST_MAXLEVEL`,在我阅读的redis 4.0版本源码中,取值为`32`,
最新的版本里面这个值改为`64`了.每个节点被提升的概率是由`ZSKIPLIST_P`决定的,取值为`0.25`.
这两个宏定义均在`server.h`文件中.
2. 通过`random()`函数来随机选择被提升的节点.这里有一点很关键,`random()`函数的结果是`均匀
分布`的,那么`random() & 0xFFFF`得到的就是一系列均匀落在`[0, 0xFFFF]`区间的数.所以`while`
循环的逻辑就是判断随机产生的这个数是否落在`[0, ZSKIPLIST_P * 0xFFFF]`区间中.假设`第一次`
落在这个区间的概率是p,因为`random()的输出是均匀分布的`,所以`p=ZSKIPLIST_P`.`第二次`还落
在这个区间的概率就是`p*p`,依此类推,`第k次`还落在这个区间的概率就是`p^k`.这就是所谓的`幂
法则`.
3. 通过数学推导可以得出每个节点评价拥有的指针数量是`1/(1-P)`,跳跃表的查找,插入和删除操作
的时间复杂度都是`O(logN)`.

[pdf]: https://www.epaperpress.com/sortsearch/download/skiplist.pdf
[power law]: https://en.wikipedia.org/wiki/Power_law

# skip list结构
---
最后来看看Redis内部是如何实现跳跃表的.主要涉及两个结构体:`zskiplistNode`和`zskiplist`.
相关定义都在`server.h`文件中.
```c
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```
先来看看`zskiplistNode`中各成员的含义:
+ **ele:** 一个SDS结构,在前面的文章中已经提过,简单来说就是用来存放用户输入的信息.
+ **score:** 分值,每个`ele`都有一个分值,由于跳跃表是有序的,在元素插入时会优先
根据`score`分值进行排序,若分值相同,则根据`ele`排序.
+ **backward:** 后向指针,指向当前节点的前面(prev)一个节点,这就形成了一个双向链表.
+ **level[]:** 这是一个柔性数组,在讲SDS结构时已经提过.用来记录当前节点有多少层,
每层的下一个节点是谁(`forward`指针指向的节点),距离下一个节点的`距离(span)`是多少.
`span`的准确含义是当前节点和forward指向的节点中间跨越的节点个数.之所以引入`span`
记录距离信息,是为了方便计算每个节点在整个链表中的排名.比如我想要找出链表中排名第
10的元素,这时`span`就派上用场了.

`zskiplist`的结构就简单很多:
+ **header:** 整个跳跃表的头结点,需要注意的是这是一个`哑结点(dummy node)`,创建时`level`
有`ZSKIPLIST_MAXLEVEL`层,不存储实际的用户数据(ele和score).
+ **tail:** 跳跃表的尾节点,指向实际的链表节点,若链表为空则指向`NULL`.
+ **length:** 整个链表的长度.
+ **level:** 链表中层数大的节点的层数,不包括`header`节点.

通过这两个结构,最终形成了如下所示的跳跃表:[图没画好,画好后补充]

关于跳跃表的插入和删除,留待下篇文章细说.
