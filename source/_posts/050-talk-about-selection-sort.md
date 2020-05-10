---
title: 选择排序 (Selection Sort)
date: 2019-06-25 22:47:57
categories: 算法
tags: [选择排序]
donate: false
---

选择排序是一种简单直观的排序算法，总的原则就是依次找到想要的元素然后交换。

<!--more-->

排序就是将一组对象按照某种逻辑顺序重新排列的过程。比如根据数值的大小升序或降序排序。


# 选择排序

[选择排序](https://zh.wikipedia.org/wiki/%E9%80%89%E6%8B%A9%E6%8E%92%E5%BA%8F)（Selection Sort） 是一种简单直观的排序算法。

排序算法（默认是升序）的原理是这样的。首先，找到数组中最小的那个元素，将它与数组中的第一个元素交换位置。（如果第一个元素就是最小元素，则跳过）然后依次在剩下的元素中找到最小的元素，将它与数组中的第二个元素交换位置。如此循环，直到将整个数组排序。

对于一个长度为 **n** 的数组，选择排序在最坏的情况下需要大约  **n^2/2** 次比较和 **n** 次交换。

## 思路分析

假设我们做的是升序排序，主要思路是依次遍历数组中的每个元素，**当遍历到当前元素时，从当前元素开始到最后一个元素中寻找值最小的元素，如果能找到值最小元素，则与当前的元素做交换**。当遍历到最后一个元素时，排序也就完成了。

## 算法实现

### Swift

用 Swift 实现的选择排序代码如下所示：

```swift
/// 选择排序
func selectionSort(array: inout [Int]){
    guard array.count > 1 else {
        return
    }

    for i in 0..<array.count {
        var minValueIndex = i 
        for j in i+1..<array.count {
            if array[j] < array[minValueIndex] {
                minValueIndex = j
            }
        }
        if minValueIndex != i {
            let temp = array[i]
            array[i] = array[minValueIndex]
            array[minValueIndex] = temp
        }
    }
}
```

### Objective-C


用 Objective-C 语言实现的算法如下：

```objc
- (NSArray*) selectionSort: (NSArray *) unsortedArray {
    if (unsortedArray.count <= 1) {
        return unsortedArray;
    }
    
    NSMutableArray *sortedArray = [unsortedArray mutableCopy];
    NSInteger minValueIndex = 0;
    
    for (NSInteger i = 0; i < sortedArray.count-1; i++) {
        minValueIndex = i;
        for (NSInteger j = i + 1; j < sortedArray.count; j++) {
            if ([sortedArray[j] integerValue] < [sortedArray[minValueIndex] integerValue]) {
                minValueIndex = j;
            }
        }
        if (minValueIndex != i) {
            [sortedArray exchangeObjectAtIndex:minValueIndex withObjectAtIndex:i];
        }
    }
    return [sortedArray copy];
}
```

### Java

```java
public static int[] selectionSort(int[] array) {
    if(array.length < 2) {
        return array;
    }

    int[] result = array;
    for(int i = 0; i < result.length; i ++) {
        int minValueIndex = i;
        for(int j = i + 1; j < result.length; j++) {
            if (result[j] < result[minValueIndex]) {
                minValueIndex = j;
            }
        }
        if(minValueIndex != i) {
            int temp = result[minValueIndex];
            result[minValueIndex] = result[i];
            result[i] = temp;
        }
    }
    return result;
}
```

## 验证算法

```swift
var list = [2, 3, 5, 7, 4, 8, 6 ,10 ,1, 9]
// 将会打印 [2, 3, 5, 7, 4, 8, 6 ,10 ,1, 9]
print(list) 
// 进行选择排序
selectionSort(unsortedArray: &list)
// 将会打印 [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
print(list) 
```

## 算法总结

`稳定性` ：是非稳定算法，因为排序过程中会打乱相同元素的相对位置。

`空间复杂度`：整个排序过程是在原数组上进行排序的，所以是 O(1)。

`时间复杂度`：排序算法包含双层嵌套循环，故为 O（n^2）。