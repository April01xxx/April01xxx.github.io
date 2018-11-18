---
layout: post
title: "Redis源码zskiplist跳跃表实现"
date: 2018-11-18 11:56
categories: [redis]
tags: [redis, C]
---

这篇我们来看下Redis中跳跃表zskiplist的实现,为了支持上层应用,定义了较多的API,这里只关注
跳跃表的`创建`,`插入`和`删除`.没有`查找`是因为在插入或者删除的过程中已经包含了.理解了这
些基本的接口,剩下的API应该都能迎刃而解.

# 创建跳跃表
---
在前面的文章中,我们已经梳理了跳跃表的数据结构,这里我们先来看看跳跃表是如何创建的.整个逻辑
由`zslCreate`函数实现,相关代码在`t_zset.c`文件中:
```c
/* 创建一个跳跃表节点,参数level表示这个节点的初始层数,由前面文章中提到的`zslRandomLevel`
 * 函数生成. */
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}

zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```
这里先调用`zmalloc`函数为跳跃表分配内存空间,然后是成员变量的初始化.这里有两点要注意的:
1. `level`成员的初值为`1`,也就是说默认是一层.(这有点废话,如果是0层岂不是没法存储元素).
2. `header`头结点是一个`哑结点(dummy node)`,其中并不存储任何元素,且该节点初始化时是有
`ZSKIPLIST_MAXLEVEL`层,每层的`跨度span`初值为0.

# 跳跃表插入
---
要理解跳跃表的插入,先要理解跳跃表的数据结构,可以在脑海中想象一下一串有序的元素是如何存
储在跳跃表中的.先来看下插入的逻辑,由`zslInsert`函数实现,源码在`t_zset.c`中:
```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    /* 这个两层循环的目的是找出元素适合插入的位置,从最上层的链表开始查找,逐步向下,
     * 直到找到在最底层链表中合适的位置. */
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    level = zslRandomLevel();   /* 计算新插入节点的层数. */
    if (level > zsl->level) {
        /* 如果新插入的节点的层数比当前跳跃表的最大层数要大,
         * 需要将rank和update中维护的信息做初始化.rank和update
         * 的含义见下面分析. */
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,ele);
    /* 节点的插入.实际上就是一个普通链表的插入操作,只不过这里有level层链表需要
     * 处理,另外,还要维护每个节点的跨度span信息. */
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```
要理解插入的逻辑,我觉得理解代码中两个关键变量的含义即可:
1. **update[ZSKIPLIST_MAXLEVEL]:** 这是一个指针数组,`update[i]表示第i层链表中待插入
节点的前一个节点的指针`.维护这个信息是为了在链表中插入新的节点更方便.
2. **rank[ZSKIPLIST_MAXLEVEL]:** 一个无符号整型数组,`rank[i]表示待插入节点在第i层链表
中的排名,或者说是距离header节点的跨度`.维护这个信息是为了计算新插入节点的`跨度span`.

# 跳跃表删除
---
其实理解了跳跃表插入的过程中`查找合适插入位置`的那段代码逻辑,跳跃表的删除也就没
什么难点了.删除的逻辑由`zslDelete`函数实现,源码也在`t_zset.c`文件中.
```c
/* Internal function used by zslDelete, zslDeleteByScore and zslDeleteByRank */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    /* 若删除的是跳跃表中拥有最大层数的节点,只能逐层判断是否下一个最大层数是
     * 多少.这里有一点要注意的是,拥有最大层数的节点可能不止一个,这通过判断头
     * 节点header中对应层的下一个节点是否为NULL可知. */
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}

/* Delete an element with matching score/element from the skiplist.
 * The function returns 1 if the node was found and deleted, otherwise
 * 0 is returned.
 *
 * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise
 * it is not freed (but just unlinked) and *node is set to the node pointer,
 * so that it is possible for the caller to reuse the node (including the
 * referenced SDS string at node->ele). */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```
相比于插入的逻辑,删除节点的逻辑没有维护`rank`信息,那是因为删除一个节点,自然只需要
将前面一个节点的`跨度span`值减一即可.另外由调用者来决定是否释放被删除节点所占用的
空间,`若不释放,则函数入参node不能为NULL,该函数会将找到的节点的指针存入node中返回`.
