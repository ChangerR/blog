---
title: 排序算法-堆排序
p: 排序算法
date: 2015-03-21 21:15:43
tags:
  - 算法
  - 排序
---
## 执行时间
平均情况和最差情况为O(nlog(n)), 存储空间O(1)

## 算法描述
堆排序其实是一种树形排序算法，其本质就是通过构造最大或者最小堆，不断弹出堆顶数据对数据进行排序。

## 算法实现

``` c

void heapSink(int* a, int min, int max)
{
    while( (min << 1) < max)
    {
        int j = (min << 1);

        if (j + 1 < max && a[j] < a[j+1])
        {
            j = j + 1;
        }
        if (a[j] > a[min])
        {
            int temp = a[j];
            a[j] = a[min];
            a[min] = temp;
            min = j;
        }
        else
        {
            break;
        }
    }
}

void heapSort(int* a, int length)
{
    int* virtualArray = a - 1;

    //构造最大堆
    for (int i = (length - 1) / 2; i >= 0; --i)
    {
        heapSink(virtualArray, i + 1, length + 1);
    }

    //根据最大堆，循环排序
    for (int i = length - 1; i > 0; --i)
    {
        int temp = a[i];
        a[i] = a[0];
        a[0] = temp;
        heapSink(virtualArray, 1, i + 1);
    }
}
```