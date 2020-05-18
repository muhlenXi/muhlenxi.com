---
title: 聊聊 GCD
toc: true
donate: false
tags: []
date: 2020-04-16 10:34:18
categories: iOS
description: Grand Central Dispatch 是 Apple 开发用来执行任务的一个强大工具。
---


Grand Central Dispatch 是 Apple 开发用来执行任务的一个强大工具，这里的任务一般指  block 中的代码片段。Dispatch queue 能指定将不同的任务分发到不同的 queue 中同步或异步的执行。

Dispatch queue 是一个先进先出的数据结构，你添加到 queue 中的任务是按照添加顺序依次执行的，GCD 主要有以下三种不同类型的 queue。

- Serial
- Concurrent
- Main dispatch queue

##  Serial Queue

### 串行执行任务

Keyword: `DispatchQueue` `dispatchPrecondition`

```swift
let serialQueue = DispatchQueue(label: "com.muhlenxi.queue")
serialQueue.async {
    print("Do work 1 ")
}
serialQueue.async {
    for _ in 0..<2 {
        print("Do work 2")
    }
}
serialQueue.async {
    for _ in 0..<3{
        print("Do work 3")
    }
}
    
dispatchPrecondition(condition: .onQueue(DispatchQueue.main))
print("finish work in main queue")
```

### 指定 target queue

Keyword：`target`

```swift
let targetQueue = DispatchQueue(label: "com.muhlenxi.target")
let q1 = DispatchQueue(label: "q1", target: targetQueue)
let q2 = DispatchQueue(label: "q2", target: targetQueue)
q1.async {
    print("Do work in q1")
    dispatchPrecondition(condition: .onQueue(targetQueue))
    dispatchPrecondition(condition: .onQueue(q1))
    dispatchPrecondition(condition: .notOnQueue(q2))
}
q1.suspend()
q2.async {
    print("Do work in q2")
}
```

##  Concurrent Queue

### 并行执行任务

Keyword: `DispatchQueue`

```swift
let concurrentQueue = DispatchQueue(label: "com.muhlenxi.concurrent", attributes: .concurrent)
concurrentQueue.async {
    print("Do work 1 ")
}
let workItem2 = DispatchWorkItem {
    for _ in 0..<2 {
        print("Do work 2")
    }
}
concurrentQueue.async(execute: workItem2)
concurrentQueue.async {
    for _ in 0..<3{
        print("Do work 3")
    }
}
concurrentQueue.async {
    for _ in 0..<4{
        print("Do work 4")
    }
}
    
print("Finish work 1 2 3 4")
```

### 任务取消 或 阻塞线程直到完成

Keyword: `DispatchWorkItem`

```swift
let queue = DispatchQueue(label: "com.muhlenxi.concurrent", attributes: .concurrent)
    
let workItem1 = DispatchWorkItem {
    print("Do work 1 ")
}
queue.async(execute: workItem1)
    
let workItem2 = DispatchWorkItem {
    for _ in 0..<2 {
        print("Do work 2")
    }
}
queue.async(execute: workItem2)
workItem2.wait()
    
let workItem3 = DispatchWorkItem {
    for _ in 0..<3{
        print("Do work 3")
    }
}
queue.async(execute: workItem3)
workItem3.wait()
    
let workItem4 = DispatchWorkItem {
    for _ in 0..<4{
        print("Do work 4")
    }
}
queue.asyncAfter(deadline: DispatchTime.now() + 1, execute: workItem4)
workItem4.cancel()
```

### 多队列任务执行完毕统一处理

Keyword: `DispatchGroup`

```swift
let queue = DispatchQueue(label: "com.muhlenxi.concurrent", attributes: .concurrent)
        let queue2 = DispatchQueue(label: "com.muhlenxi.concurrent2", attributes: .concurrent)
    let queue3 = DispatchQueue(label: "com.muhlenxi.concurrent3", attributes: .concurrent)
let group = DispatchGroup()
    
queue.async(group: group) {
    print("Do work 1 ")
}
let workItem2 = DispatchWorkItem {
    for _ in 0..<2 {
        print("Do work 2")
    }
}
queue2.async(group: group, execute: workItem2)
queue3.async(group: group) {
    for _ in 0..<3{
        print("Do work 3")
    }
}
queue3.async(group: group) {
    for _ in 0..<4{
        print("Do work 4")
    }
}
    
group.notify(queue: DispatchQueue.main) {
    dispatchPrecondition(condition: .onQueue(DispatchQueue.main))
    print("Finish work 1 2 3 4")
}
```

### 指定任务的执行顺序

Keyword: `barrier`

```swift
let concurrentQueue = DispatchQueue(label: "com.muhlenxi.concurrent", attributes: .concurrent)
concurrentQueue.async(flags: .barrier) {
    print("Do work 1 ")
}
concurrentQueue.async {
    for _ in 0..<2 {
        print("Do work 2")
    }
}
concurrentQueue.async(flags: .barrier){
    for _ in 0..<3 {
        print("Do work 3")
    }
}
concurrentQueue.async {
    for _ in 0..<4{
        print("Do work 4")
    }
}
```

Keyword: `DispatchSemaphore`

```swift
let concurrentQueue = DispatchQueue(label: "com.muhlenxi.concurrent", attributes: .concurrent)
let semaphore = DispatchSemaphore(value: 1)
concurrentQueue.async(){
    semaphore.wait()
    for _ in 0..<2 {
        print("Do work 2")
    }
    semaphore.signal()
}
concurrentQueue.async(){
    semaphore.wait()
    for _ in 0..<3 {
        print("Do work 3")
    }
    semaphore.signal()
}
concurrentQueue.async {
    semaphore.wait()
    for _ in 0..<4{
        print("Do work 4")
    }
    semaphore.signal()
}
```

### 指定任务的优先级 Quality of Service

Keyword: `background` `utility` `userInitiated` `userInteractive`

```swift
let queue = DispatchQueue(label: "com.muhlenxi.target", attributes: .concurrent)
queue.async(qos: .background) {
    print("Do work 1")
}
queue.async(qos: .utility) {
    print("Do work 2")
}
queue.async(qos: .userInitiated) {
    print("Do work 3")
}
queue.async(qos: .userInteractive) {
    print("Do work 4")
}
```

##  Lock

- C Lock: pthread_mutex_t
- Foundation: NSLock
- Darwin.os.lock: os_unfair_lock

```swift
class UnfairLock {
    private var osLock = os_unfair_lock()
    
    func lock() {
        os_unfair_lock_lock(&osLock)
    }
    
    func unlock() {
        os_unfair_lock_unlock(&osLock)
    }
}
```

DispatchQueue: DispatchQueue.sync(execute:)

```swift
class MyObject {
    private var internalState: Int
    private let internalQueue: DispatchQueue
    
    init(internalState: Int) {
        self.internalState = internalState
        self.internalQueue = DispatchQueue(label: "com.muhlenxi.queue")
    }
    
    var state: Int {
        get {
            return internalQueue.sync { internalState }
        }
        set (newState) {
            internalQueue.sync { internalState = newState }
        }
    }
}
```