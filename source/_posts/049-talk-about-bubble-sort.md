---
title: 聊聊 冒泡排序 （Bubble Sort）
date: 2019-06-19 22:22:53
categories: 算法
tags: [冒泡排序]
description: 冒泡排序是一种看起来很形象的排序算法。
---

版权声明：本文为 [muhlenXi](http://www.muhlenxi.com) 原创文章，未经博主允许不得转载，如有问题，欢迎指正。

## Foreword

[冒泡排序](https://zh.wikipedia.org/wiki/%E5%86%92%E6%B3%A1%E6%8E%92%E5%BA%8F)（Bubble Sort）的排序方法是在每一轮排序过程中，依次比较相邻元素的大小，如果顺序不满足排序的要求，则交换这两个元素。这样第一轮排序结束后，则值最大的元素放到最后。

<!-- more -->

一个 包含 **n** 个元素的数组在最坏的情况下，需要进行 **n - 1** 次排序过程，方能使数组中的元素变得有序。

## 算法分析

整个算法由双层嵌套循环组成。

外面的循环控制冒泡的趟数，从 下标 0 开始跑趟，**n** 个元素总共需要 **n  -  1** 趟。

内层循环控制单次冒泡过程。每趟中，从 下标 0 开始冒泡， 第 **i** 趟需要 **n - 1 - i** 次冒泡。

注意事项：

- 当数组中元素小于 2 时，则不需要排序。
- 当内层冒泡时，不再有元素交换时，则说明冒泡排序已经完成，此时应该跳出外层循环，终止排序。

## 算法实现

1、用 Swift 实现的冒泡排序代码如下所示：

```swift
/// 冒泡排序 升序
func bubbleSort(unsortedArray: inout [Int]){
    guard unsortedArray.count > 1 else{
        return 
    }
    
    for i in 0 ..< unsortedArray.count-1 {
        var exchanged = false
        for j in 0 ..< unsortedArray.count-1-i {
            if unsortedArray[j] > unsortedArray[j+1] {
                unsortedArray.swapAt(j, j+1)
                exchanged = true
            }
        }
        if !exchanged {
            break
        }
    }
}
```

2、用 Objective-C 实现的算法如下：

```objc
- (NSArray*) bubbleSort: (NSArray *) unsortedArray {
    if (unsortedArray.count <= 1) {
        return unsortedArray;
    }
    
    NSMutableArray *sortedArray = [unsortedArray mutableCopy];
    
    for (int i = 0; i < sortedArray.count-1; i++) {
        BOOL exchanged = NO;
        for (int j = 0; j< sortedArray.count-1-i; j++) {
            if ([sortedArray[j] integerValue] > [sortedArray[j+1] integerValue]) {
                [sortedArray exchangeObjectAtIndex:j withObjectAtIndex:j+1];
                exchanged = YES;
            }
        }
        if (!exchanged) {
            break;
        }
    }
    
    return [sortedArray copy];
}
```

3、用 Java 实现的算法如下：

```java
static int[] bubbleSort(int[] array) {
    if(array.length < 2) {
        return array;
    }

    int[] result = array;

    for(int i = 0; i < result.length-1; i++) {
        boolean changed = false;
        for(int j = 0; j < result.length-1-i; j++) {
            if(result[j] > result[j+1]) {
                int temp = result[j];
                result[j] = result[j+1];
                result[j+1] = temp;
                changed = true;
            }
        }
        if(changed == false) {
            break;
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
// 进行冒泡排序
bubbleSort(unsortedArray: &list)
// 将会打印 [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
print(list) 
```

## 算法总结

`稳定性` ：是稳定算法，因为排序过程中相邻会依次比较，不会打乱相同元素的相对位置。

`空间复杂度`：整个排序过程是在原数组上进行排序的，所以是 O(1)。

`时间复杂度`：排序算法包含双层嵌套循环，故为 O(n^2)。

## 参考资料

- 1、[Backup & Demo](https://github.com/muhlenXi/algorithm)

