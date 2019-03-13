---
title: 二分查找
tags:
  - 算法
categories:
  - algorithm
date: 2015-03-10 16:59:19
---
提到查找算法时，我们一般都会想到二分查找算法。这个算法非常有用，值得研习。

## 算法描述

在二分查找中，要在有序数组里查找元素x，我们会先去数组中间元素与x作比较。若x小于中间元素，则搜索数组的左半部。若x大于中间元素，则搜索数组的右半部。然后重复这个过程，将左半部和右半部视为子数组继续搜索。我们再次取这个子数组的中间元素与x作比较,然后搜索左半部或右半部。我们会重复这一过程，直至找到x或子数组大小为0。

## 算法实现

概论上似乎很简单，但是要真正掌握全部细节，却比想象的要困难。以下是算法实现。

``` c
int binarySearch(int* a, int length, int x)
{
    int low = 0;
    int high = length - 1;
    int middle = 0;
    while(low <= high)
    {
        middle = (low + high) / 2;
        if (a[middle] > x)
        {
            high = middle - 1;
        } 
        else if (a[middle] < x)
        {
            low = middle + 1;
        }
        else
        {
            return middle;
        }
    }
    return -1;
}
```