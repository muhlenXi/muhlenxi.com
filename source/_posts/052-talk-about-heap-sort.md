---
title: 聊聊 堆排序 (Heap Sort)
date: 2019-10-06 21:22:21
categories: 算法
tags: [堆排序]
description: 想要了解堆排序，首先要了解堆，堆是一种特殊的完全二叉树。其次，我们要了解二叉树的相关知识。
---


## 前言

想要了解堆排序，首先要了解堆，堆是一种特殊的完全二叉树。其次，我们要了解二叉树的相关知识。

<!-- more -->

## 树

[树](https://zh.wikipedia.org/wiki/%E6%A0%91_(%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)) 是一种数据结构，用来模拟具有树状结构性质的数据集合。一颗树由 N 个节点组成，其中每个节点有且只有有限个子节点。没有子节点的节点称为叶节点。

[二叉树](https://zh.wikipedia.org/wiki/%E4%BA%8C%E5%8F%89%E6%A0%91) 是每个节点有且仅有最多两个子节点的一种树。节点左边的子树称为左子树，节点右边的子树称为右子树。

[满二叉树](https://zh.wikipedia.org/wiki/%E4%BA%8C%E5%8F%89%E6%A0%91) 是指的具有以下特征的二叉树：

- 树的每一层上的节点数都是最大节点数。即第一层有 1 个节点，第 2 层有 2 个节点，第 3 层有 4 个节点，第 4 层有 8 个节点，第 n 层有 2 ^ (n-1) 个节点。

[完全二叉树](https://zh.wikipedia.org/wiki/%E4%BA%8C%E5%8F%89%E6%A0%91) 是指的具有以下特征的二叉树：

- 1、除了最后一层，其余层都是满的二叉树。
- 2、如果最后一层不满，则只在右边缺少连续若干节点。


[堆](https://zh.wikipedia.org/zh-hans/%E5%A0%86%E7%A9%8D) 是一种特殊的完全二叉树，堆中的父节点永远大于或小于它的子节点。我们分别称之为最大堆或最小堆。

- 在最大堆中，每个父节点的值都大于它的子节点。
- 在最小堆中，每个父节点的值都小于它的子节点。


## 堆排序

本文使用的是最大堆。

[堆排序](https://zh.wikipedia.org/wiki/%E5%A0%86%E6%8E%92%E5%BA%8F) 的原理主要是利用了最大堆的特性，在最大堆中，堆顶的元素值最大。我们依次拿走最大堆的堆顶元素，将剩余的元素再次组成最大堆。这样得到的堆顶元素的集合就是一个有序集合。

在用数组表示的堆中，第一个元素就是该数组中值最大的元素。堆排序，主要分为以下三个步骤：

- 1、将要排序的数组构建成堆。
- 2、从数组的最后一个元素 n 开始，依次将第 0 个元素与该元素交换，然后再继续构建最大堆。
- 3、重复步骤 2，直到第 0 个元素。


## 算法分析

堆排序的关键步骤是要实现 heapify 和 buildHeap 函数。

对于一个下标为 **i** 的节点来说：

- **左子节点**的下标为 **2 * i + 1**
- **右子节点**的下标为 **2 * i + 2**
- **父节点**的下标为 **(i - 1) / 2**

## 算法实现

用 Swift 实现的堆排序如下：

```swift
/// 对堆中的元素执行调整最大堆的操作
func heapify(tree: inout [Int], n: Int, i: Int) {
    let left = 2*i+1
    let right = 2*i+2
    var max = i
    
    if left < n && tree[left] > tree[max] {
        max = left
    }
    if right < n && tree[right] > tree[max] {
        max = right
    }
    if max != i {
        tree.swapAt(i, max)
        heapify(tree: &tree, n: n, i: max)
    }
}

/// 将树中的元素构建成最大堆
func buildHeap(tree: inout [Int], n: Int) {
    let lastNode = n - 1
    var p = (lastNode-1)/2
    while p >= 0 {
        heapify(tree: &tree, n: n, i: p)
        p -= 1
    }
}

/// 堆排序
func heapSort(tree: inout [Int]) {
    buildHeap(tree: &tree, n: tree.count)
    
    var i = tree.count-1
    while i >= 0 {
        tree.swapAt(0, i)
        heapify(tree: &tree, n: i, i: 0)
        i -= 1
    }
}
```

用 Objective-C 实现的算法如下：

```objc
- (void)heapSort:(NSMutableArray *) tree {
    [self buidMaxHeap:tree];
    NSInteger last = tree.count-1;
    while (last >= 0) {
        [self swap:tree i:0 j:last];
        [self heapfy:tree n:last i:0];
        last--;
    }
}

- (void)buidMaxHeap:(NSMutableArray *) tree {
    NSInteger parent = (tree.count-1-1)/2;
    while (parent >= 0) {
        [self heapfy:tree n:tree.count i:parent];
        parent--;
    }
}

- (void)heapfy:(NSMutableArray *)tree n:(NSInteger)n i:(NSInteger)i {
    if (i >= n) {
        return;
    }
    
    NSInteger max = i;
    NSInteger c1 = i * 2 + 1;
    NSInteger c2 = i * 2 + 2;
    
    if (c1 < n && [tree[c1] integerValue] > [tree[max] integerValue]) {
        max = c1;
    }
    if (c2 < n && [tree[c2] integerValue] > [tree[max] integerValue]) {
        max = c2;
    }
    if (max != i) {
        [self swap:tree i:max j:i];
        [self heapfy:tree n:n i:max];
    }
}

- (void)swap:(NSMutableArray *)array i:(NSInteger)i j:(NSInteger)j {
    NSNumber *temp = array[i];
    array[i] = array[j];
    array[j] = temp;
}
```

## 算法验证

```swift
var list = [2, 3, 5, 7, 4, 8, 6 ,10 ,1, 9]
heapSort(tree: &list)
// will print [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
print(list)
```

## 算法总结

- 堆排序的平均时间复杂度为 O(nlogn)，空间复杂度为 O(1)。

- 堆排序每次都将堆顶元素移到最后，因此是不稳定排序。