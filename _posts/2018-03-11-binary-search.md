---
title: 数据结构与算法（二）：二分搜索
updated: 2018-03-11 13：21
tag: binary search, data structures, algorithms
---

## 概述
以前对二分查找的认识只停留在有序数组查找给定整数上，后来发现一类问题都可以用二分的思想来做，概括来说就是：如果要求的结果所在的集合（值域）和要搜索的数的集合（定义域）存在单调（映射）关系，就可以通过二分思想来解决，说起来有点抽象，后面将用几个例子来说明。

二分思想以其每次迭代将规模砍一半的效率，有着极其广阔的应用。

本文分两大部分，第一部分对二分查找的各个细节探讨；第二部分拓展二分查找为一般的二分思想。

## 二分查找--边边角角
### 基本代码（C++）

```C++
int find(vector<int> &arr, int target) {
    int left = 0, right = arr.size()-1;

    while (left <= right) {
        int mid = (left + right) / 2;

        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target){
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }

    return -1;
}
```
像上面这一段平平无奇的代码，在实际运用的时候却又诸多变化。二分查找的基本思路不再多说，现在只分析边边角角（Corner Case）。这就涉及到我们一个基本原则，既要让所有元素有可以搜索到的机会，又不至于陷入死循环。具体来说有以下几个问题：

1. 循环条件。什么时候用`left <= right`，什么时候用`left < right`呢？
2. 取中方式。`mid`常用的有几种计算方式：`mid = (left+right)/2`， `mid = left+(right-left)/2` 和 `mid = (left+right+1)/2`。
3. 边界移动。`left`和`right`都有两种移动方式，拿`left`来说：`left=mid+1`和`left=mid`。
4. 最终状态。如果没有查找到指定值，比如说[1,2,3,5]中查找数字4，那么最后跳出循环后，left和right分别指向5和3的位置。也就是说，在4应该插入的位置两侧，并且left>right。

上面的几个问题其实是相互勾连的，视遇到的问题来适当组合。比如说，查找一个有序数组{1, 2, 3, 3, 3, 5}某个数字(3)的左右边界:{2, 4}

```
int find_range(vector<int> &arr, int target, bool left_range) {
    int left = 0, right = arr.size() - 1;
    
    while (left < right) {
        int mid = left_range ? (left + right) / 2 : (left + right + 1) / 2;
        if (arr[mid] == target) {
        // Find left boundary: if we round right when calculate mid, then
        // we will trap into infinite loop here(imagine
        // we only have two elements in the array).
        // Find right boundary: the similar reason.
            left_range ? right = mid : left = mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }

    return left;
}
```
上面的代码，在要查找的数字存在的时候是对的，如果不存在需要加额外判断。需要加什么判断呢？这也就是需要说明的另一个地方：当我们进行二分查找的时候，是在中途找到结果就退出(return)呢，还是一直到循环条件被打破退出呢？前者用来最简单的查找指定值，而后者一般是查找某个边界，或者符合条件的最值。该问题因为是由于破坏了循环条件`left < right`退出的，所以得判断下`arr[left]`是否和`target`相等。




## 小结
有些点还没想清楚，以后再补充。









     









