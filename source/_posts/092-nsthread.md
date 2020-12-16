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

## NSThread 简介

`NSThread` 是 Apple 用于执行任务的线程对象。当你想要你的任务在指定的线程上执行时，你可以使用这个对象。

当你执行一个耗时的任务时，你不想要阻塞 App 的运行，也就是阻塞用户交互和事件处理的主线程。那么你可以将你的任务放到一个子线程中去执行。

同样，你可以可以讲一个大任务分解为若干个小任务，然后在多个线程中执行这些小任务，从而提高执行任务的效率。


## 生命周期

1、线程的初始化，也就是显式地创建线程。

- 通过 `init` 方法
- 通过 `initWithTarget:selector:object:` 方法
- 通过 `initWithBlock` 方法

示例代码：

```objc
self.mxThread = [[MXThread alloc] init];
[self.mxThread start];
```

```objc
NSThread *new = [[NSThread alloc] initWithTarget:self selector:@selector(doWork) object:nil];
[new start];
```

```objc
self.myThread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"my thread block 调用了");
}];
```

当创建线程对象后，我们通过调用线程的 `start` 或 `main` 方法来启动一个线程。

2、隐式地创建并开启一个线程。

-  调用 `detachNewThreadWithBlock` 方法
-  调用 `detachNewThreadSelector:toTarget:withObject` 方法

示例代码：

```objc
[NSThread detachNewThreadWithBlock:^{
    [self doWork];
}];
```

```objc
[NSThread detachNewThreadSelector:@selector(doWork) toTarget:self withObject:nil];
```

3、线程休眠、线程终止、线程取消

- `sleepUntilDate:`  在 date 时刻前一直阻塞当前线程
- `sleepForTimeInterval:` 当前线程休眠指定时间
- `exit` 终止当前线程
- `cancel` 更新线程为取消状态

4、获取当前线程的状态。

- `isExecuting` 是否正在执行任务
- `isCancelled` 是否取消任务执行
- `isFinished`  任务是否执行完毕

示例代码：

```objc
BOOL isExecuting = [self.myThread isExecuting];
BOOL isCancelled = [self.myThread isCancelled];
BOOL isFinished = [self.myThread isFinished];
```

5、主线程

- `isMainThread` 当前线程是否是主线程
- `mainThread` 获取主线程

示例代码：

```objc
BOOL isMainThread = [self.myThread isMainThread];
// 获取主线程
NSThread *mainThread = [NSThread mainThread];
```

## 线程参数设置

每个线程允许我们设置的属性如下：

- `name` 线程的名字，调试方便。
- `stackSize` 线程栈空间大小
- `threadDictionary` 线程对象字典。你可以在字典中存储你想要的数据。
- `threadPriority` 线程优先级

获取当前线程（常用方法之一）

```objc
[NSThread currentThread];
```

## 线程通讯

我们可以让任务在指定的线程上执行。

- `performSelectorOnMainThread:withObject:waitUntilDone` 在主线程执行
- `performSelectorInBackground:withObject` 在后台线程执行
- `performSelector:onThread:withObject:waitUntilDone` 在自定义线程执行


示例代码：

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

当线程中的任务执行完毕后，这个线程就被销毁了。有的时候，我们想一直保留这个线程，也就是线程保活。

实现思路：开启所在线程的 `runloop`，然后 `addPort`，在控制变量下循环调用 `runMode` 方法。

开启 `runloop` 的方式有三种：

- `run` runloop 会一直运行下去。
- `runUntilDate` 在截止期前会一直运行下去。
- `runMode` 在截止期前运行一次。

开启 runloop 示例代码：

```objc
[runloop run];

[runloop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:3600]];

[runloop runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
```

线程常驻示例代码：

当你想要退出 runloop 的时候，只要将控制变量 `isQuiting` 设置为 `YEZ` 即可。

```objc
// 创建线程
self.myThread = [[NSThread alloc] initWithBlock:^{
    // 线程常驻
    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
    [runloop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
    while (!self.isQuiting) {
        [runloop runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    }
}];
self.myThread.name = @"mx thread";
[self.myThread start];

// 执行任务    
[self performSelector:@selector(doWork) onThread:self.myThread withObject:nil waitUntilDone:NO];
```

退出 runloop 示例代码：

```objc
self.isQuiting = YES;
CFRunLoopStop(CFRunLoopGetCurrent());
```
## 后记

我是卖码维生的「穆哥」。我们下次见。

## 参考资料

- [NSThread](https://developer.apple.com/documentation/foundation/nsthread)
- [Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html#//apple_ref/doc/uid/10000057i-CH1-SW1)
- [Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1)