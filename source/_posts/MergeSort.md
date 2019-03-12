---
title: 排序算法-归并排序
p: 排序算法
date: 2015-03-13 21:15:43
tags:
  - 算法
  - 排序
---
## 执行时间
平均情况与最差情况为O(nlog(n)), 存储空间: 视情况而定

## 算法描述
归并排序是将数组分成两半，这两半分别排序后，再归并在一起。排序某一半时，继续沿用同样的排序算法，最终，将归并两个只含一个元素的数组。这个算法的重担都落在“归并”的部分上。

## 算法实现

``` c

void mergeSort(int* a, int low, int high)
{
    if (low < high)
    {
        int middle = (low + high) / 2;
        mergeSort(a, low, middle);
        mergeSort(a, middle + 1, high);
        merge(a, low, middle, high);
    }
}

void merge(int* a, int low, int middle, int high)
{
    int* helper = new int[high + 1 - low];
    for (int index = 0; index < high + 1 - low; index++)
    {
        helper[index] = a[index + low];
    }
    int helperLeft = 0;
    int helperRight = middle + 1 - low;
    int current = low;

    while(helperLeft <= middle - low && helperRight <= high - low)
    {
        if (helper[helperLeft] <= helper[helpRight])
        {
            a[current] = helper[helperLeft];
            helperLeft++;
        }
        else
        {
            a[current] = helper[helpRight];
            helperRight++;
        }
        current++;
    }

    int remaining = middle - low - helperLeft;
    for (int i = 0; i < remaining; i++)
    {
        a[current + i] = helper[helperLeft + i];
    }
    delete[] helper;
}
```