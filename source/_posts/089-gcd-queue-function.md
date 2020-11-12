---
title: 『底层探索』15 - GCD 中队列、Block 的用法和区别
toc: true
donate: false
tags: []
date: 2020-11-12 11:36:40
categories:
---

本文主要是探索和分析串行队列、并行队列、同步、异步之间的组合情况。

<!-- more -->

## GCD 简介

`GCD` 是 `Grand Central Dispatch` 的缩写，是 Apple 提供的一套多线程编程工具，我们可以通过 GCD 同步或异步地执行 Block 中的代码块。gcd 是使用 `C` 语言编写的。

使用 GCD 我们不需要关心线程的创建和状态维护，只需要创建合适的队列（Queue），或者使用系统提供的全局队列，然后将要执行的任务以同步或异步的方式添加到队列中即可。

在 GCD 中，主要有两种类型队列，`串行队列`（Serial）和 `并行队列`（Concurrent）。其中系统中的 `main queue` 属于特殊的串行队列，`global queue` 属于并行队列。

队列中的任务调度规则都是 FIFO（先进先出） 的。由于我们添加任务的方式有两种 `同步`（sync）和 `异步`，那么对于两种队列，就一共有 4 种情况。下面就分析这四种情况。

| 队列类型/提交方式 | 同步  | 异步  |
|:---------:|:---:|:---:|
| 串行队列      |   ---  |   ---  |
| 并行队列      |  ---   |  ---   |

## 系统队列获取

在 App 中，如果你没有指定队列，代码默认都是在系统的主线程中执行的，同时系统也提供了一个 `global queue` 全局并行队列供我们使用。它们是如何获取的呢？

```c
// 主队列
dispatch_queue_t mainQueue = dispatch_get_main_queue();
// 全局并行队列
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

`dispatch_get_global_queue` 方法中的第一个参数表示的是优先级，第二个参数是保留参数，未来可能会用到，现在传 0 就可以了。

## 队列的创建

如果我们不想用系统的队列，想创建自己的队列，也就是队列是如何创建的呢？

```c
// 串行
dispatch_queue_t serial = dispatch_queue_create("my serial queue", DISPATCH_QUEUE_SERIAL);
// 并行
dispatch_queue_t concurrent = dispatch_queue_create("my concurrent queue", DISPATCH_QUEUE_CONCURRENT);
```

可以通过 `dispatch_queue_create` 方法来创建队列。其中第一个参数是指定队列的名字，方便我们后续调试。第二个参数用于执行队列的类型。

`DISPATCH_QUEUE_SERIAL` 表示串行，`DISPATCH_QUEUE_CONCURRENT` 表示并行。我们有的时候会看到第二个参数传的是 `NULL`, 那么它是什么类型队列呢，其实这也是串行。我们不妨看看 `DISPATCH_QUEUE_SERIAL` 的声明：

```c
#define DISPATCH_QUEUE_SERIAL NULL
```

## 任务提交方式

我们可以将要执行的代码以 Block 的方式封装，然后将这个 block 添加到队列中，添加方式有同步和异步，这是怎么确定的呢？

```
// 同步添加
dispatch_sync(concurrent, ^{
    // do something
});
    
// 异步添加
dispatch_async(concurrent, ^{
    // do something
});
```

`dispatch_sync` 是以 `同步` 的方式将 block 添加到指定队列中。
`dispatch_async` 是以 `异步` 的方式将 block 添加到指定队列中。

那么同步和异步有什么区别呢？在官方文档中，我们可以看到这么一句话。

> you can use the `dispatch_sync` and `dispatch_sync_f` functions to add the task to the queue. These functions block the current thread of execution until the specified task finishes executing.

这句话的关键思想就是，同步提交到队列中的任务会阻塞当前线程，直到任务完成后才会解除阻塞状态。如果在主线程中执行 `dispatch_sync` 将会阻塞主线程。

在阻塞状态中，用户的事件将无法响应。因此苹果建议我们如果无特殊需要，尽可能以异步的方式去添加任务。

### 举个例子

`异步添加`

```c
// 默认在主线程
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
NSLog(@"执行任务 A");
// 同步添加
dispatch_async(globalQueue, ^{
    sleep(1);
    NSLog(@"执行任务 B");
});
dispatch_async(globalQueue, ^{
    sleep(2);
    NSLog(@"执行任务 C");
});
NSLog(@"执行任务 D");
```

这种异步的情况下，NSLog 的打印次序将是 `A D B C`。其中 `B C` 的执行顺序是不确定的。

异步输出结果如下：

```shell
执行任务 A
执行任务 D
执行任务 B
执行任务 C
```

`同步添加`。

也就是将上面的 `dispatch_async` 都改为 `dispatch_sync`, 那么 NSLog 的打印次序将是啥。由于同步会堵塞当前线程，也就是 `B C` 执行完后，才会执行 `D`, 因此将会是 `A B C D`。

同步输出结果如下：

```shell
执行任务 A
执行任务 B
执行任务 C
执行任务 D
```

输出结果也表明，执行 B C 的时候主线程确实被阻塞了，因此苹果强烈提醒我们。

> You should never call the `dispatch_sync` or `dispatch_sync_f` function from a task that is executing in the same queue that you are planning to pass to the function. 
> 
> This is particularly important for serial queues, which are guaranteed to deadlock, but should also be avoided for concurrent queues.


## GCD 进阶用法

### dispatch once

在 App 的生命周期中，block 只被执行一次，可以使用 `dispatch_once` 来实现。比如我们经常用到的单例模式，就经常用到 `dispatch_once`。

```c
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    // code to be executed once
});
```

### dispatch_after

延时去执行一个任务，可以用 `dispatch_after` 来实现。

```c
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    //code to be executed after a specified delay
});
```

`dispatch_after` 方法中，第一个参数用于指定任务执行时间，它是 `dispatch_time_t` 类型的。第二参数用于指定任务要添加的队列，第三个参数是要执行的 block。

### dispatch group

在并行队列中，有时候，我们可能会想要一组任务都完成的时候，再去做某件事。我们可以使用 `dispatch group` 来完成这个功能。

```c
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
dispatch_group_t group = dispatch_group_create();
    
dispatch_group_async(group, globalQueue, ^{
    sleep(2);
    NSLog(@"执行任务 A %@", NSThread.currentThread);
});
dispatch_group_async(group, globalQueue, ^{
    sleep(3);
    NSLog(@"执行任务 B %@", NSThread.currentThread);
});
dispatch_group_async(group, globalQueue, ^{
    sleep(1);
    NSLog(@"执行任务 C %@", NSThread.currentThread);
});
    
dispatch_group_notify(group, globalQueue, ^{
    NSLog(@"A B C 都执行完了, 可以去做其他事情了。 %@", NSThread.currentThread);
});
```

在 `dispatch_group_notify` 方法中，block 中的代码会在组中最后一个完成任务的线程中执行。如果我们要刷新 UI，需要将 block 异步添加到主队列中才行。

使用 `dispatch_group_wait` 方法可以阻塞当前线程，直到 group 中的任务都完成或者到达截止时间。

上面的代码还有一种繁琐的写法。

```c
dispatch_group_enter(group);
dispatch_async(globalQueue, ^{
    sleep(2);
    NSLog(@"执行任务 A %@", NSThread.currentThread);
    dispatch_group_leave(group);
});
    
    
dispatch_group_enter(group);
dispatch_group_async(group, globalQueue, ^{
    sleep(3);
    NSLog(@"执行任务 B %@", NSThread.currentThread);
    dispatch_group_leave(group);
});

    
dispatch_group_enter(group);
dispatch_group_async(group, globalQueue, ^{
    sleep(1);
    NSLog(@"执行任务 C %@", NSThread.currentThread);
    dispatch_group_leave(group);
});
    
dispatch_group_notify(group, globalQueue, ^{
    NSLog(@"A B C 都执行完了 %@", NSThread.currentThread);
});
    
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
```

`dispatch_group_enter` 用于显示的声明接下来的 block 进入到 group 中。
`dispatch_group_leave` 用于显示的声明 group 中的当前 block 已经执行完毕。

### dispatch semaphore 

如果你添加到并行队列的任务要访问的资源是有限的，这时候你想要控制同时访问该资源的任务数，这时候可以用 `dispatch semaphore` 来实现这个功能。也就是传说中的信号量。

基本使用方式如下：

- 用 `dispatch_semaphore_create(count)` 方法可以创建一个信号量，其中 count 是同时访问资源的任务数，也就是同时访问的线程数量。
- 在每个任务用 `dispatch_semaphore_wait` 等待信号量，获取到信号量。 并且信号量大于 0，则会执行这个任务。如果信号量小于 0，则会进入等待状态。
- 当任务执行完毕后，需要用 `dispatch_semaphore_signal` 发送信号量。此时处于等待状态的其他线程会重新等待信号量。

1、如果我们在并行异步线程中，想要控制任务的顺序执行，使用信号量可以完成这个需求。

```c
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

for(int i = 0; i < 10; i++) {
    dispatch_async(concurrent, ^{
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        
        sleep(1);
        NSLog(@"执行任务 %d thread = %@", i, NSThread.currentThread);
        
        dispatch_semaphore_signal(semaphore);
    });
}
```

2、如果我们在并行异步线程中想要两个任务 A 和 B 之间有依赖关系，比如只有 B 任务完成后， A 任务才可以执行。这使用信号量也可以完成的, 注意此时的 count 需要传 0。

```c
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

dispatch_async(concurrent, ^{
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    sleep(1);
    NSLog(@"执行任务 A");
});
dispatch_async(concurrent, ^{
    sleep(2);
    NSLog(@"执行任务 B");
    dispatch_semaphore_signal(semaphore);
});
```

通过上面的案例，我们不难猜测，调用 `dispatch_semaphore_wait` 会使 count 减 1，调用 `dispatch_semaphore_signal` 会使 count 加 1。

关于 GCD 的底层原理，下篇文章将会探索。

### dispatch semaphore 

## 大脑热身赛

下面我们通过几道题来巩固下上面的知识。

## 参考资料

- [1. Dispatch Queues](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW1)
