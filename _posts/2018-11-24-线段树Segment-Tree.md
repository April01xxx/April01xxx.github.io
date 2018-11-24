---
layout: post
title: "线段树Segment Tree"
date: 2018-11-24 14:02
categories: [Algorithm]
tags: [algorithm]
---

LeetCode上刷题也有好几个月了,自我感觉对于一些常用的数据结构用法非常熟练,稍稍有些
自满,立马就折戟沉沙了.昨天刷到一题,Easy难度,题目如下:
> Given an integer array nums, find the sum of the elements between indices i
> and j (i ≤ j), inclusive.
>
> The update(i, val) function modifies nums by updating the element at index i to val.
>
> Example:
> Given nums = [1, 3, 5]
> sumRange(0, 2) -> 9
> update(1, 2)
> sumRange(0, 2) -> 8
> Note:
> + The array is only modifiable by the update function.
> + You may assume the number of calls to update and sumRange function is distributed evenly.

大意是说,给定一个数组,现有两种操作:
1. 修改指定下标处元素的值;
2. 给定一个下标范围[i,j],求这个区间的元素的和.
要求设计一种数据结构能够满足以上要求.

# 常规思维
---
题目看完,很容易想到若每次都重新计算不可取,故预先计算好结果,待要查询时直接返回即可.
令sum[i]表示下标小于i的元素的和.那么`sumRange(i,j)=sum[j+1]-sum[i]`.这个操作的时间
复杂度是O(1).但对于`update(i,val)`却稍微有点麻烦,只因每次update都要将受到影响的sum
的值进行调整,而为了知道调整的差值,必须保存一份原始数组的副本.思路有了,刷刷刷写下如下
代码:
```c
typedef struct {
    int size;
    int *copy;  // 原数组的副本,用于update时计算调整差值
    int *sum;   // sum[i]表示从[0,i)范围内元素的和
} NumArray;

NumArray* numArrayCreate(int* nums, int numsSize) {
    NumArray *na;
    int i;

    if (numsSize <= 0)
        return NULL;
    na = malloc(sizeof(*na));
    na->size = numsSize;
    na->copy = malloc(numsSize * sizeof(int));
    /* 为了编程方便,sum数组的大小多开一份空间. */
    na->sum = malloc((numsSize + 1) * sizeof(int));
    na->sum[0] = 0;
    for (i = 1; i <= numsSize; ++i) {
        na->copy[i - 1] = nums[i - 1];
        na->sum[i] = na->sum[i - 1] + nums[i - 1];
    }
    return na;
}

void numArrayUpdate(NumArray* obj, int i, int val) {
    int diff = val - obj->copy[i], j;

    obj->copy[i] = val;
    for (j = i + 1; j <= obj->size; ++j)
        obj->sum[j] += diff;
}

int numArraySumRange(NumArray* obj, int i, int j) {
    return obj->sum[j + 1] - obj->sum[i];
}

void numArrayFree(NumArray* obj) {
    if (obj) {
        free(obj->copy);
        free(obj->sum);
        free(obj);
    }
}
```
上述解法中`update`方法的平均时间复杂度是`O(n)`,`sumRange`方法的平均时间复杂度是
`O(1)`.在两种方法调用次数均等的情况下,对任意一种操作而言,其平均时间复杂度均是O(n).
其实在写完代码后就意识到这题应该还有更好的解法,其关键是找到一种数据结构能够方便快捷
维护某个区间的信息而不涉及不相关的区间.但苦想无果,未能想出好的方案.看了下这题的tag,
有个关键词`segment tree`.好吧,又遇到自己没见过的数据结构了.

# segment tree
---
[segment tree][segment tree]中文一般译作`线段树`,个人觉得还是非常形象的.简单来说
是一种基于二叉树的数据结构,只不过树中的每个节点维护的都是一段区间的信息:比如区间
的最大/小值,区间元素内的和等等.具体的介绍参见给出的维基百科链接.

[segment tree]: https://en.wikipedia.org/wiki/Segment_tree

当时看完维基百科的介绍顿时恍然大悟,这不就是自己想要的数据结构么!每一个节点维护的
都是一段区间的信息,又由于segment tree的性质:`它是一棵完全二叉树`,故相关操作的时间
复杂度是`O(logn)`.这比上面自己想出的方法的效率是要高很多的.那么接下来是如何实现这
样一棵树.

先来看看题目的需求:
+ **update(i,val):** 修改指定下标处元素的值.
+ **sumRange(i,j):** 获取指定区间[i,j]范围内元素的和.

从以上需求出发可知,对于每个节点我们要维护以下信息:
1. **区间范围信息:** 对于线段树中的任意一个节点,需要知道它表示的哪一段区间.特别地,
`根节点表示整个区间[0,LEN)的信息,叶子节点表示数组中某个单独元素的信息`.
2. **区间元素的和:** 为了快速计算sumRange(i,j),在每个节点中还要保存这段区间内所有元
素的和.

以数组`[9,0,7,5,3]`为例形成的`segment tree`示意图如下:[图没画好,待画好后补充]

# segment tree实现
---
通过以上分析可知,线段树中每个节点可用如下数据结构描述:
```c
struct segmentTreeNode {
    int lo;     // 该节点所表示区间的左边界.
    int hi;     // 该节点所表示区间的右边界.
    int sum;    // 该节点所表示区间的元素的和.
    struct segmentTreeNode *left, *right;   // 二叉树左右孩子指针
} segmentTreeNode;
```
所谓数据结构和算法,接下来自然是基于给定的数据结构寻找合适的算法来满足题目的要求.

## segmentTreeCreate创建线段树
---
线段树是本质上还是一棵二叉树,所以在创建树的过程中`采用递归的方法自顶向下建树`.
```c
typedef struct segmentTreeNode *segmentTree;

segmentTree segmentTreeCreate(int *array, int lo, int hi) {
    segmentTree st;
    int mid;

    if (hi < lo)
        return NULL;
    st = malloc(sizeof(*st));
    st->lo = lo;
    st->hi = hi;
    if (lo == hi) {
        /* 区间大小为1,到达叶子节点. */
        st->sum = array[lo];
        st->left = st->right = NULL;
        return st;
    }

    /* 二叉树,这里的二分实际上是对整个区间进行二分. */
    mid = lo + (hi - lo) / 2;
    st->left = segmentTreeCreate(array, lo, mid);
    st->right = segmentTreeCreate(array, mid + 1, hi);
    /* 自顶向下建树,父节点区间的元素和等于左右孩子区间的和. */
    st->sum = st->left->sum + st->right->sum;
    return st;
}
```

## segmentTreeUpdate线段树更新
---
线段树可以满足多种更新操作,例如整个区间的所有元素累加一个固定值,减去一个固定值等等.
这里要求的是更新某个指定下标的元素的值,实际就是一个查找操作,需要注意的是叶子节点的
值做了更新,沿途向上的所有节点的`sum`属性(区间内元素的和)也要同步更新.
```c
void segmentTreeUpdate(segmentTree st, int i, int val) {
    int mid;

    if (st->lo == i && st->hi == i) {
        // 找到要更新的节点.
        st->sum = val;
        return;
    }

    mid = st->lo + (st->hi - st->lo) / 2;
    if (i > mid)
        segmentTreeUpdate(st->right, i, val);
    else
        segmentTreeUpdate(st->left, i, val);
    st->sum = st->left->sum + st->right->sum;
}
```

## segmentTreeQuery线段树查找
这里的需求是查询某一范围区间内所有元素的和.从根节点开始查找,假定查找的范围区间是
[i,j],在查找的过程中,会面临以下几种情况:
1. 该节点所表示的区间恰好就是要查找的区间:`node->lo == i && node->hi == j`.此时直
接返回`node->sum`即可.
2. 该节点所表示的区间范围大于(包含)[i,j],此时又细分为以下三种情况,
   令`mid = node->lo + (node->hi - node->lo) / 2`:
    + [i,j]落在区间的左半部:`j <= mid`.此时应沿着该节点的左子树继续查找.
    + [i,j]落在该区间的右半部:`i > mid`.此时应沿着该节点的右子树继续查找.
    + [i,j]包含该区间的中间值:`i <= mid && j > mid`,此时应将[i,j]拆分为`[i,mid]`和
      `[mid+1,j]`两个部分分别沿该节点左右子树查找.

具体实现如下:
```c
int segmentTreeQuery(segmentTree st, int i, int j) {
    int mid;

    if (st->lo == i && st->hi == j)
        return st->sum;
    mid = st->lo + (st->hi - st->lo) / 2;
    if (j <= mid)
        return segmentTreeQuery(st->left, i, j);
    else if (i > mid)
        return segmentTreeQuery(st->right, i, j);
    else
        return segmentTreeQuery(st->left, i, mid)
               + segmentTreeQuery(st->right, mid + 1, j);
}
```

上述方法是用指针实现的线段树,因为线段树是一棵完全二叉树(和堆一样),故也可以用数组
实现,具体方法这里就不赘述了,总的思路是用数组下标替代指针来表示左右孩子的位置.最后
不禁感叹,虽然堆栈,队列,二叉树这些结构很简单,但通过给其附加一些额外信息却能妙用无穷,
诸如`单调栈`,`单调队列`,`平衡二叉查找树`等等.路漫漫其修远兮...
