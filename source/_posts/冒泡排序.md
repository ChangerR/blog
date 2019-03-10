---
title: 排序算法-冒泡排序
p: 排序算法
date: 2015-03-11 21:15:43
tags:
  - 算法
  - 排序
---
## 执行时间
平均情况与最差情况为O(n^2), 存储空间O(1)

## 算法描述
冒泡排序是先从数组第一个元素开始，依次比较相邻两个数，若前者比后者大，就将两者交换位置，然后处理下一对，依次类推，不断扫描数组，直至完成排序

## 算法实现

``` c

void bubbleSort(int* a, int length)
{
    bool run = true;
    while (run)
    {
        run = false;
        for (int i = 0; i < length - 1; i++)
        {
            if (a[i] > a[i + 1])
            {
                run = true;
                int temp = a[i];
                a[i] = a[i + 1];
                a[i + 1] = temp;
            }
        }
    }
}
```