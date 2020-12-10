---
title: 『底层探索』17 - iOS 内存管理 - 强引用分析
toc: true
donate: false
tags: []
date: 2020-12-10 14:49:59
categories: [底层探索]
---

本文主要以 NSTimer 为例，分析循环引用产生的原因和不同的改进方案。

<!-- more -->

## strong or weak

当我们在 class 中声明一个 property 的时候，经常会用 `strong` 和 `weak` 去修饰对象类型的属性。那么它们之间有什么区别呢？

结论：

- 用 `strong` 修饰的属性指向的对象的引用计数会 + 1。
- 用 `weak` 修饰的属性指向的对象的引用计数会保持不变。

我们可以通过下面的代码观察到这一现象。

```objc
// 声明属性
@property (nonatomic, strong) NSObject *strongObject;
@property (nonatomic, weak) NSObject *weakObject;

- (void)test {
    NSObject *object = [[NSObject alloc] init];
    NSLog(@"rc = %ld", [object retainCount]);
    
    self.strongObject = object;
    NSLog(@"after strong rc = %ld", [object retainCount]);
    
    self.weakObject = object;
    NSLog(@"after weak rc = %ld", [object retainCount]);
}
```

打印结果如下，很明显可以得出上面的结论。

```shell
rc = 1
after strong rc = 2
after weak rc = 2
```

## 循环引用

了解了 `strong` 和 `weak` 的不同作用后，下面我们分析下循环引用问题。

当内存中的两个对象互相持有的时候，也就是都有一个强引用指向对方，这就产生了一个引用环。这两个对象都在等待对方先释放，会一直在内存中，导致内存泄漏。如图所示：

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/ref-cycle.png)

要解决这种问题的方法是，将其中的一个引用用 weak 修饰，改为弱引用。

## NSTimer

我们一般会按照如下的方式，开启一个定时器。

```objc
self.timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(fire) userInfo:Nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
```

上面代码，会每隔 1 秒调用我们的 fire 方法。如果在控制器 A 中，我们持有一个 NSTimer  的强引用。

```
@interface AViewController ()
@property (nonatomic, strong) NSTimer *timer;
@end
```

然后我们不做任何处理，直接返回上一个控制器，你觉得 A 控制器会被释放么，也就是会调用 `dealloc` 方法么？

为了避免循环引用，我们一般还会在 dealloc 中销毁我们的定时器。代码如下：

```objc
- (void)dealloc
{
    NSLog(@"AViewController dealloc");
    [self.timer invalidate];
    self.timer = nil;
}
```

实际情况是，dealloc 并没有被调用，当我返回上一个控制器时，fire 方法还是持续的被调用。为什么会这样呢？

在 `NSTimer` 的 `timerWithTimeInterval:target:selector:userInfo:repeats` 官方文档中，我们找到如下描述：

> The object to which to send the message specified by aSelector when the timer fires. The timer maintains a strong reference to this object until it (the timer) is invalidated.

就是说，如果 timer 没有销毁，那么 timer 对 target（self）也会持有一个强引用。按照前面的逻辑，这就产生了一个循环引用。

这个时候，有同学说，使用 weakSelf 不就打破引用环了么。也就是：

```objc
__weak typeof(self)  weakSelf = self;
self.timer = [NSTimer timerWithTimeInterval:1.0 target:weakSelf selector:@selector(fire) userInfo:Nil repeats:YES];
```

到这里并没有万事大吉。情况还是如前一样，dealloc 没有被调用，fire 还是持续输出。为什么会这样呢？

原因是，target 持有的强引用是 weakSelf 指向的对象，也就是 `AViewController` 本身。除此之外，当前的 runloop 对 timer 也持有一个 strong 引用，并且 runloop 的生命周期比 `AViewController` 还长。该信息来源于：

> Timers work in conjunction with run loops. Run loops maintain strong references to their timers, so you don’t have to maintain your own strong reference to a timer after you have added it to a run loop.

所以，我们要在合适的时机，手动的将 timer 进行销毁。首先放在 `dealloc` 中是行不通的。下面我们讨论下几种打破循环引用的方案。

## 解决方案

### viewWillDisappear

第一种方案是，在 `ViewController` 的 `viewWillDisappear` 方法中，调用 timer 的 `invalidate` 方法和设置为 nil。

这种情况下，如果你再 push 到另一个控制器时，`AViewController` 仍然在导航栏的栈中，这时候 timer 已经被销毁了。显然这种方案是不完美的。

### didMoveToParentViewController

第二种方案是，重写 `didMoveToParentViewController` 的方法，在这个方法中，条件调用 timer 的 `invalidate` 方法和设置为 nil。`didMoveToParentViewController` 在如下情况会被调用：

> Called after the view controller is added or removed from a container view controller.

当 `parent` 为 `nil` 的时候，也就是这个控制器被移除了，所以我们的代码可以这么写：

```objc
- (void)didMoveToParentViewController:(UIViewController *)parent {
    if (parent == nil) {
        [self.timer invalidate];
        self.timer = nil;
    }
}
```

### 自定义封装 MLXTimerWrapper 

方案三的思路是：

- 在 MLXTimer 中持有一个 target 控制器，让 timer 持有的 target 是自己本身。
- 在每次定时器事件触发的时候，先判断 taget 控制是是否还存在。也就是是否是 `nil`。
    - 存在的话，转发消息给 target 控制器。触发 target 控制器中的方法。
    - 不存在的话，调用 timer 的 `invalidate` 方法和设置为 nil。

### 中介者模式 NSProxy

方案三的思路是，让 timer 持有的 target 不是当前 `AViewController`, 而是一个中间对象。这样 `AViewController` pop 的时候，就可以被销毁了，也就是 dealloc 会被调用了。在 dealloc 中，我们就可以调用 timer 的 `invalidate` 方法和设置为 nil，从而打破循环引用。

中间者的作用是被 timer 持有，然后将 timer 产生的定时器事件 ，转发给`AViewController`, 从而触发 `fire` 方法。

在 Objective-C 中，有一个和 `NSObject` 平级的抽象类 `NSProxy`, `NSProxy` 仅仅实现了 root class 的基本方法，比如 `NSObjectProtocol ` 中的方法。它的实用特性是比 `NSObject` 轻量级，并且实现了消息转发机制。

我们可以实现一个 `NSProxy` 的子类 `MLXProxy`, 作为 timer 的 target，重写消息转发方法，将 timer 产生的事件转发给 `AViewController`。具体的代码如下：

`MLXProxy.h` 文件如下：

```
#import <Foundation/Foundation.h>

@interface MLXProxy : NSProxy

+ (instancetype)proxyWithObject:(id) object;

@end
```

`MLXProxy.m` 文件如下：

```objc
#import "MLXProxy.h"

@interface MLXProxy ()

// 注意这里要是弱引用对象
@property (nonatomic, weak) id object;

@end

@implementation MLXProxy

+ (instancetype) proxyWithObject:(id) object {
    MLXProxy *proxy = [MLXProxy alloc];
    proxy.object = object;
    return proxy;
}

- (id)forwardingTargetForSelector:(SEL) aSelector {
    return self.object;
}

- (void)dealloc
{
    NSLog(@"MLXProxy dealloc");
}

@end
```

这样，我们使用 time 的方式就变成了这样：

```objc
@property (nonatomic, strong) NSTimer *timer;
@property (nonatomic,strong) MLXProxy *proxy;
```

```objc
self.proxy = [MLXProxy proxyWithObject:self];
self.timer = [NSTimer timerWithTimeInterval:1.0 target:self.proxy selector:@selector(fire) userInfo:Nil repeats:YES];
       
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
```

## 参考资料

- [didMoveToParentViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621405-didmovetoparentviewcontroller)
- [timerWithTimeInterval:target:selector:userInfo:repeats](https://developer.apple.com/documentation/foundation/nstimer/1408356-timerwithtimeinterval)
- [NSProxy](https://developer.apple.com/documentation/foundation/nsproxy)