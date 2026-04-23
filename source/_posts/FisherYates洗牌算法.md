---
title: Fisher-Yates洗牌算法
date: 2026-03-24
tag: [Unity]
categories: [Tech]
toc: true
comments: true
---
# Fisher-Yates算法

## 介绍

Fisher-Yates随机排序算法常用于卡牌游戏的发牌操作中，为了保证公平性，需要将卡牌序列公平打乱随机化。

## 算法思想

Fisher-Yates算法的核心思想是用一个指针维护序列。

* 初始指针指向末端数据
* 将数据和前面序列内的随机数据（包括自身）交换
* 向前移动指针表示已经洗过牌

在O(n)时间内完成洗牌操作，并且只是用O(1)空间复杂度

## 算法实现

```python
def shuffle(arr):
  last_index = len(arr) - 1
  while last_index > 0:
    rand_index = random.randint(0, last_index)
    temp = arr[last_index]
    arr[last_index] = arr[rand_index]
    arr[rand_index] = temp
    last_index -= 1
```

## 参考

[【游戏开发秘籍】最公平高效的发牌员？深度解析游戏中的洗牌算法！\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1tNKfz1Eqc?spm_id_from=333.788.videopod.sections&vd_source=0c4720590a80024b8323b2bb6910d392)
