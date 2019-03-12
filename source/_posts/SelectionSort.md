---
title: 排序算法-选择排序
p: 排序算法
date: 2015-03-12 21:47:33
tags:
  - 算法
  - 排序
---

## 执行时间
平均情况与最差情况为O(n^2), 存储空间O(1)

## 算法描述
选择排序简单而低效。我们会线性逐一扫描数组元素，从中挑出最小的元素，将他移到最前面。然后，再次线性扫描数组，找到第二小的元素，并移到前面。如此反复知道全部元素各归其位。

## 算法实现
``` c
void selectionSort(int* a, int length)
{
    for (int i = 0; i < length - 1; i++)
    {
        for (int j = i + 1; j < length; j++)
        {
            if (a[i] > a[j])
            {
                int temp = a[i];
                a[i] = a[j];
                a[j] = temp;
            }
        }
    }
}
```
