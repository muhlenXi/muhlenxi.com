---
title: 『底层探索』15 - NSThread 详解
toc: true
donate: false
tags: []
date: 2020-12-12 16:55:04
categories: [底层探索]
---

本文主要是 `NSThread` 的详细使用方法和注意事项。

<!-- more -->

iOS 中的每个进程都是由一个或多个线程组成的。每个 App 默认开启一个主线程（main thread）来处理各种事件和任务。除此之外，我们也可以创建额外的线程来处理我们自己的任务。

线程的开启会消耗相应的系统资源。比如在 iOS 中，每个线程会占用 1 kB 的空间存储自身信息，除此之外还开辟 512 KB 的栈空间，其中主线程是 1 MB 栈空间。因此我们要控制线程的数量。

下面我们讲一下 iOS 中的线程是如何创建、执行任务、和线程间通讯的。

本文用到的完整代码在 [NSThreadDemo](https://github.com/muhlenXi-Team/NSThreadDemo.git) 。

## 生命周期

```objc
NSThread *new = [[NSThread alloc] initWithTarget:self selector:@selector(doWork) object:nil];
new.name = @"mx";
[new start];
```

```objc
[NSThread detachNewThreadWithBlock:^{
    [self doWork];
}];
```

```objc
[NSThread detachNewThreadSelector:@selector(doWork) toTarget:self withObject:nil];
```

```objc
[NSThread currentThread];
```

## 线程通讯

```objc
[self performSelectorOnMainThread:@selector(doWork) withObject:nil waitUntilDone:NO];
```

```objc
[self performSelectorInBackground:@selector(doWork) withObject:nil];
```

```objc
[self performSelector:@selector(doWork) onThread:self.myThread withObject:nil waitUntilDone:NO];
```

## 线程常驻

```objc
// 创建线程
self.myThread = [[NSThread alloc] initWithBlock:^{
    // 线程常驻
    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
    [runloop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
    [runloop run];
}];
self.myThread.name = @"mx thread";
[self.myThread start];

// 执行任务    
[self performSelector:@selector(doWork) onThread:self.myThread withObject:nil waitUntilDone:NO];
```

## 参考资料

- [](https://developer.apple.com/documentation/foundation/nsthread)
- [Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html#//apple_ref/doc/uid/10000057i-CH1-SW1)
- [Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1)