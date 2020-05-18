---
title: 聊聊 常见十大排序
toc: true
donate: false
tags: []
date: 2020-04-20 10:33:48
categories: 算法
---

排序就是把一组对象按照指定的条件（升序或倒序）重新排列的过程。

<!-- more -->

## 01 - 思考

开始前不妨思考以下几个问题：

- 想一想什么是排序呢？
- 想一想怎么描述排序的效率？时间复杂度，空间复杂度

## 02 - 常见排序

本文中的代码使用的 Swift，默认排序是升序排序，其他的大同小异，这里不一一赘述了。

### 选择排序

排序方法：

1. 遍历过程为从一个元素到倒数第二个元素。
2. 在遍历中，要执行的操作是每次从当前元素之后的序列中，寻找最小元素，然后与当前元素交换，然后再处理下一个元素。

```swift
func selection(_ numbers: inout [Int]) {
    guard numbers.count > 1 else { return }
    
    for indexI in 0..<numbers.count-1 {
        var minIndex = indexI
        for indexJ in minIndex+1..<numbers.count {
            minIndex =  numbers[indexJ] < numbers[minIndex] ? indexJ : minIndex
        }
        numbers.swapAt(indexI, minIndex)
    }
}
```

### 冒泡排序

排序方法：

1. 遍历过程为多趟逐次递减遍历，即第一次遍历到最后一个元素，第二次为倒数第二个元素，直到第一个元素为止。
2. 在每次遍历中，要执行的操作是，依次比较当前元素和下一个元素的值，不满足条件则交换。

```swift
func bubble(_ numbers: inout [Int]) {
    guard numbers.count > 1 else {
        return
    }
    
    var total = numbers.count
    while total > 0 {
        
        var changed = false
        var index = 0
        while index < numbers.count-1 {
            if numbers[index] > numbers[index+1] {
                numbers.swapAt(index, index+1)
                changed = true
            }
            index += 1
        }
        if !changed {
            break
        }
        
        total -= 1
    }
}
```

### 插入排序

排序方法：

1. 遍历过程为从第二个元素遍历到最后一个元素。
2. 遍历中执行的操作是从当前元素之前的队列中，找到要插入的位置后，将当前元素插入到序列中。

```swift
func insert(_ numbers: inout [Int], gap: Int = 1) {
    guard numbers.count > 1 else { return }
    
    for g in 0..<gap {
        var i = g
        while i < numbers.count {
            var current = i
            while current-gap >= 0 && numbers[current-gap] > numbers[current] {
                numbers.swapAt(current, current-gap)
                current -= gap
            }
            
            i += gap
        }
    }
}
```

### 希尔排序

希尔排序是对插入排序的改进，是一种增量递减排序。

排序方法：

1. 首先生成一个增量递减序列，一般是 n /  2。
2. 遍历增量递减序列种，对于每一个增量 gap，要执行的操作是，对数组中依次相距 gap 的序列进行插入排序。增量序列遍历完得到的数组就是排序后数组。

```swift
func shell(_ numbers: inout [Int]) {
    guard numbers.count > 1 else { return }
    
    var gap = numbers.count / 2
    while gap != 0 {
        insert(&numbers, gap: gap)
        gap = gap / 2
    }
}
```

### 归并排序

排序方法：

1. 对数组中的元素进行递归分割，直到每个序列只包含一个元素为止。
2. 有序合并分割后的每个序列，合并后数组就是排序后的数组。

```swift
func merge(_ numbers: [Int]) -> [Int] {
    return divide(nums)
}

/// 分割
func divide(_ numbers: [Int]) -> [Int] {
    guard numbers.count > 1 else { return numbers }
    
    let mid = (0+numbers.count-1)/2
    let leftNumbers = divide(Array(numbers[0...mid]))
    let rightNumbers = divide(Array(numbers[mid+1..<numbers.count]))
    return join(leftNumbers, rightNumbers)
}

/// 合并
func join(_ leftNumbers: [Int], _ rightNumbers: [Int]) -> [Int] {
    var result = [Int]()
    
    var leftIndex = 0
    var rightIndex = 0
    while leftIndex < leftNumbers.count && rightIndex < rightNumbers.count {
        if leftNumbers[leftIndex] <= rightNumbers[rightIndex] {
            result.append(leftNumbers[leftIndex])
            leftIndex += 1
        } else {
            result.append(rightNumbers[rightIndex])
            rightIndex += 1
        }
    }
    
    if leftIndex < leftNumbers.count {
        result.append(contentsOf: Array(leftNumbers[leftIndex..<leftNumbers.count]))
    }
    if rightIndex < rightNumbers.count {
        result.append(contentsOf: Array(rightNumbers[rightIndex..<rightNumbers.count]))
    }
    
    return result
}
```

### 快速排序

排序方法：

1. 以序列中间元素为基准，将序列重组为小于中间元素，等于中间元素，大于中间元素的三个部分。
2. 递归地对小于中间元素序列、大于中间元素序列执行操作1，直到子序列的个数等于1为止，此时得到的数组就是排序后的数组。

```swift
func quick(_ numbers: [Int]) -> [Int] {
    guard numbers.count > 1 else {
        return numbers
    }
    
    let pivot = numbers[numbers.count/2]
    let less = numbers.filter { $0 < pivot }
    let equal = numbers.filter { $0 == pivot }
    let greater = numbers.filter { $0 > pivot }
    
    return quick(less) + equal + quick(greater)
}
```

### 计数排序

排序方法：

1. 生成一个下标最小为0，最大为数组中最大值，值为0的一个统计序列。
2. 遍历数组中的元素，以元素的值为 index，增加统计序列的出现次数。
3. 遍历统计序列，根据出现次数，生成排序后的数组。

```swift
func count(_ numbers: [Int]) -> [Int]{
    guard numbers.count > 1 else {
        return numbers
    }
    
    // 计数
    let maxVal = numbers.max()!
    var counts = Array(repeating: 0, count: maxVal+1)
    for element in numbers {
        counts[element] += 1
    }
    
    // 重组
    var result = numbers
    var resultIndex = 0
    for (index, val) in counts.enumerated() {
        for _ in 0..<val {
            result[resultIndex] = index
            resultIndex += 1
        }
    }
    return result
}
```

### 基数排序

排序方法：

1. 生成一个下标为0，最大值为9的暂存序列。
2. 寻找数组中值最大的元素，统计其位数。
3. 从最低位开始，循环的执行操作4，循环完成后得到的数组就是排序后的数组。
4. 将数组中的元素，按照相应位上数值分到暂存系列中，分完再将暂存序列合并后的数组赋值给原数组。

```swift
func radix(_ numbers: [Int]) -> [Int] {
    guard numbers.count > 1 else {
        return numbers
    }
    
    let totalDigit = getTotalDigit(numbers.max()!)
    var tempts = Array(repeating: [Int](), count: 10)
    var result = numbers
    
    for digit in 0..<totalDigit {
        // 按照指定 digit 进行分类
        tempts = Array(repeating: [Int](), count: 10)
        for element in result {
            let index = getDigit(element, digit)
            tempts[index].append(element)
        }
        // 合并
        result = tempts.flatMap { $0 }
    }
    return result
}

/// 获取数值的总位数
func getTotalDigit(_ number: Int) -> Int {
    guard number > 0 else {
        return 1
    }
    
    var length = 0
    var target = 1
    while number >= target {
        length += 1
        target *= 10
    }
    return length
}

/// 获取数值相应位上的数字
func getDigit(_ number: Int, _ Index: Int) -> Int{
    let num1 = pow(10, Index+1)
    let num2 = pow(10, Index)
    return number % num1 / num2
}

/// 计算 x 的 y 次方
func pow(_ x: Int, _ y: Int) -> Int {
    var result = 1
    for _ in 0..<y {
        result *= x
    }
    return result
}
```

### 桶排序

排序方法：

1. 生成一个元素类型是数组，总个数为 n 的的空序列，序列中的每个数组可以理解为一个桶。
2. 找出数组最大值，最小值，将它们的差除以桶的总个数得到值就是每个桶的容量。
3. 确定每个桶允许放入元素的数值范围后，遍历数组中的元素，将元素放入到每个桶中，使桶中的元素也是有序排列。
4. 将包含桶的序列合并成一个新的数组就是排序后的数组。

```swift
func bucket(_ numbers: [Int]) -> [Int] {
    guard numbers.count > 0 else { return numbers}
    
    let totalBucket = 10
    var buckets = Array(repeating: [Int](), count: totalBucket)
    let minVal = numbers.min()!
    let maxVal = numbers.max()!

    for element in numbers {
        let bucketIndex = getBucketIndex(element, minVal, maxVal, totalBucket)
        addElementToBucket(&buckets[bucketIndex], element)
    }
    
    return buckets.flatMap { $0 }
}

/// 获取要加入桶的 index
func getBucketIndex(_ element: Int, _ minVal: Int, _ maxVal: Int, _ totalBucket: Int) -> Int {
    var index = 0
    let capacity = Double(maxVal-minVal) / Double(totalBucket)
    var target = capacity
    while Double(element-minVal) > target {
        target += capacity
        index += 1
    }
    return index
}

/// 桶内新增元素并排序
func addElementToBucket(_ bucket: inout [Int], _ element: Int) {
    bucket.append(element)
    // sort
    var index = bucket.count - 1
    while index - 1 >= 0 && bucket[index-1] > bucket[index] {
        bucket.swapAt(index-1, index)
        index -= 1
    }
}
```

### 堆排序

排序方法：

1. 将数组中的元素构建成最大堆结构后，第一个元素与最后一个元素交换。
2. 对去掉最后一个元素的数组继续执行操作1，直到数组中元素的个数为0.

```swift
// 堆排序
func heap(_ numbers: inout [Int]) {
    var lastIndex = numbers.count - 1
    while lastIndex >= 0 {
        buildHeap(&numbers, lastIndex)
        numbers.swapAt(0, lastIndex)
        lastIndex -= 1
    }
}

// 构建最大堆
func buildHeap(_ numbers: inout [Int], _ lastIndex: Int) {
    var parent = (lastIndex - 1) / 2
    while parent >= 0 {
        heapify(&numbers, parent, lastIndex)
        parent -= 1
    }
}

// 对指定 index 元素进行 heapify, lastIndex 数组的最后一个元素的 index
func heapify(_ numbers: inout [Int], _ index: Int, _ lastIndex: Int) {
    guard index < lastIndex else {
        return
    }
    
    let left = index * 2 + 1
    let right = index * 2 + 2
    var max = index
    if left <= lastIndex && numbers[left] > numbers[max] {
        max = left
    }
    if right <= lastIndex && numbers[right] > numbers[max] {
        max = right
    }
    if max != index {
        numbers.swapAt(index, max)
        heapify(&numbers, max, lastIndex)
    }
}
```
