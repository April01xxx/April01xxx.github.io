---
layout: post
title: "二分查找(binary search)的一些注意事项"
date: 2018-11-14 21:00
categories: [algorithm]
tags: [algorithm]
---

[二分查找(binary search)][binary search]可以说是一种非常简单却又实用的算法,因为简
单,所以一直没对它有什么系统性的总结,每次遇到需要使用该算法解决的问题,对于
[边界情况][boundary case]每次都要花点时间思考,恰好这次在LeetCode刷题,又遇到了一点问题,索性把
自己的一些经验记录下来.

[binary search]: https://en.wikipedia.org/wiki/Binary_search_algorithm
[boundary case]: https://en.wikipedia.org/wiki/Boundary_case

# 缘由
---
在LeetCode上刷题也有几个月了,刷了200多道,也不知道算快还是慢(ps:应该算慢吧...).早上
遇到一题如下:
> You are a product manager and currently leading a team to develop a new product.
> Unfortunately, the latest version of your product fails the quality check. Since
> each version is developed based on the previous version, all the versions after
> a bad version are also bad.
>
> Suppose you have n versions [1, 2, ..., n] and you want to find out the first
> bad one, which causes all the following ones to be bad.
>
> You are given an API bool isBadVersion(version) which will return whether version
> is bad. Implement a function to find the first bad version. You should minimize
> the number of calls to the API.
>

大意是说有n个连续的数字,每个数字表示一个版本号,从某个数字开始后面的版本号都是错的,
有一个API函数`isBadVersion(version)`可以判断一个版本号是否错误.要求用最少的次数找到
第一个错误的版本号.

# 问题在哪呢?
---
要求最少的次数,要查找的区间是有序的,自然就想到二分查找了.刷刷刷写下了下面的代码:
```c++
bool isBadVersion(int version);

class Solution {
public:
    int firstBadVersion(int n) {
        int lo = 1, hi = n;

        while (lo < hi) {
            int mid = (lo + hi) >> 1;
            if (isBadVersion(mid))
                hi = mid;
            else
                lo = mid + 1;
        }
        return hi;
    }
};
```
提交,自信满满等着弹出那个漂亮的绿色图标:Accept!嗯?Judging的时间有点长,是不是又有
什么变态的testcase?嗯?弹出了红色的TLE!看了半天没看出哪里有BUG(其实有经验的人应该
很容易就发现问题了,怪自己平时总结的太少),一脸懵逼.

# 细心细心再细心
---
最后找到原因是`int mid = (lo + hi) >> 1`这行代码.当n非常大,例如`INT_MAX`,`lo+hi`
这个表达式的结果可能会溢出,这就导致上述代码死循环,结果就是TLE了.原因找到了,修改
起来自然很简单,既然输入是`int`型,那把lo,hi的定义修改为`long`类型就可以了.那假如
输入是`long`类型,lo,hi的定义就得是`long long`,很显然,这种方法能解决特定的问题,但
不具备普适性.一种比较好的做法是这样写:`mid = lo + (hi - lo) / 2`.虽然在数学上两者
是等价的,但是后面这种写法,避免了溢出的可能性.写到这里的时候突然想起来,看过的算法
相关的教材上写二分查找的代码时,都是用的后一种方法,看书的时候也未深究,果然这下入坑了.

# 一些心得
---
二分查找的算法虽然很简单,但我觉得有几点可能是初学者容易纠结的地方:
1. **lo, hi的初始化:** 假设数组有N个元素,`lo`的初始化一般都是`0`,没什么问题.但是`hi`
的初始化,有见到过`hi = N`的,也有`hi = N - 1`的,这两个有啥区别?
2. **while循环的条件:** 有见到过`lo < hi`的,也有`lo <= hi`的,应该用哪个?
3. **循环体内lo,hi的赋值:** 到底是`lo = mid`还是`hi = mid`?
4. **循环结束后的返回值:** 有返回`lo`的,有返回`hi`的,哪个是对的?

针对上面4个问题一一说明:
1. **hi的初始化:** 若初始化为数组元素个数`N`,意味着`hi`是一个哨兵,标识了查找的右边界,
查找的时候必须小于这个边界;若初始化为`N-1`,则查找的时候可以等于这个边界;
2. **lo < hi 还是 lo <= hi:** 我觉得要弄清楚这里怎么写,必须得先弄清楚`mid = lo + (hi - lo) / 2`
的含义.若`lo < hi`那么一定有`lo <= mid < hi`;也就是说`mid`的取值取不到`hi`.如果
`lo <= hi`,那么`mid`是可以取值到`hi`的.根据第一点中`hi`的初值,此处决定如何限定循环
条件.
3. **lo = mid 还是 hi = mid:** 若`mid`的值不满足条件,假设此时应该向左搜索,如果你的循环
条件是`lo < hi`那说明`mid`要能取到所有小于`hi`的值,故`hi = mid`,若循环条件是`lo <= hi`,
那说明`mid`能取到`hi`,此时`hi = mid - 1`,因为`mid`处的值肯定不满足条件,故`hi`取mid前一
位.`lo`的赋值同理.
4. **循环结束后的返回值:** 应该返回`lo`还是`hi`是依据前面三点得到的.严格来说,从第一点
确认后,后面的也跟着确定了.

总的来说首先要确定查找的范围区间,是一个左闭右闭的区间还是一个左闭右开的区间;基于这
一点得出循环的限定条件,根据mid处的值是否满足要求决定lo,hi的赋值和最终返回值.
