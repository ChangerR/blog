---
title: 排序算法-快速排序
p: 排序算法
tags:
  - 算法
  - 排序
categories:
  - algorithm
date: 2015-03-20 21:15:43
---
## 执行时间
平均情况为O(nlog(n)), 最差情况为O(n^2), 存储空间O(log(n))

## 算法描述
快速排序是随机挑选一个元素，对数组进行分割，以将所有比它小的元素排在前面，比它大的元素排在后面。这里的分割经由一系列元素交换的动作完成。
如果我们根据某元素在对数组进行分割，并反复执行，最后数组变成有序，然而，因为无法确保分割元素就是数组的中位数或者接近中位数，快速排序的效率可能会非常低下，这也是为什么最差情况时间复杂度为O(n^2)。

## 算法实现

``` c

void quickSort(int* a, int left, int right)
{
    int index = partition(a, left, right);
    if (left < index - 1)
    {
        quickSort(a, left, index - 1);
    }
    if (index < right)
    {
        quickSort(a, index, right);
    }
}

int partition(int* a, int left, int right)
{
    int n = a[(left + right) / 2];
    while(left < right)
    {
        while(a[left] > n) left++;

        while(a[right] < n) right--;

        if (left < right)
        {
            int temp = a[left];
            a[left] = a[right];
            a[right] = temp;
            left++;
            right--;
        }
    }
    return left;
}
```
