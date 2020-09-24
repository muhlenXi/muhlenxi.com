---
title: 『底层探索』9 - OC 消息发送流程之消息转发
toc: true
donate: false
tags: []
date: 2020-09-24 21:54:24
categories: [底层探索]
---

当向一个对象发送消息时，当快速查找和慢速查找都没有找到方法的 imp 时，在程序 crash 之前，还有一个消息转发流程来进行挽救，接下来我们探索一下消息转发流程。对于不开源的代码，将会用 `Hopper Disassembler` 来反编译可执行文件进行探索。

<!-- more -->


## 测试对象

首先定义一个 RDPerson 类，仅仅声明一个 `sayNB` 的对象方法，而不做任何实现。

```objc
@interface RDPerson : NSObject

- (void)sayNB;

@end

@implementation RDPerson

@end 
```

运行如下的测试代码, 在 `sayNB` 方法调用处打断点开始调试。

```objc
RDPerson *p1 = [[RDPerson alloc] init];
[p1 sayNB];
```

在 [8 - OC 消息发送流程之慢速查找](https://www.muhlenxi.com/2020/09/23/080-msg-send-slow-find/) 中，最后我们得知，当要调用的方法，在快速查找和慢速查找都找不到 imp，此时会调用 `resolveMethod_locked(inst, sel, cls, behavior)` 方法。先看看这个方法的实现。

```objc
static NEVER_INLINE IMP
resolveMethod_locked(id inst, SEL sel, Class cls, int behavior)
{
    runtimeLock.assertLocked();
    ASSERT(cls->isRealized());

    runtimeLock.unlock();

    if (! cls->isMetaClass()) {
        resolveInstanceMethod(inst, sel, cls);
    } 
    else {
        resolveClassMethod(inst, sel, cls);
        if (!lookUpImpOrNil(inst, sel, cls)) {
            resolveInstanceMethod(inst, sel, cls);
        }
    }

    return lookUpImpOrForward(inst, sel, cls, behavior | LOOKUP_CACHE);
}
```

这个方法的主要逻辑是，根据参数中的类是否是元类，然后调用不同的 resolve 方法。

- 如果 cls 不是元类，则会调用 `resolveInstanceMethod` 方法。最后
- 如果 cls 是元类，则会调用 `resolveClassMethod` 方法。

特别是元类，调用完 `resolveClassMethod` 方法后，还会调用 `lookUpImpOrNil` 来查找是否有 imp，如果没找到，则会调用 `resolveInstanceMethod` 方法。这里这么做的原因是，在 isa 走位图中，NSObject 根元类的对象的 superclass 是 NSObject。NSObject 是类，所以需要调用 `resolveInstanceMethod` 方法。

`lookUpImpOrNil` 的具体实现如下，也是对 `lookUpImpOrForward` 方法的调用，仅仅是参数不同。

```c
static inline IMP
lookUpImpOrNil(id obj, SEL sel, Class cls, int behavior = 0)
{
    return lookUpImpOrForward(obj, sel, cls, behavior | LOOKUP_CACHE | LOOKUP_NIL);
}
```

最后会调用 `lookUpImpOrForward` 方法进行最后一次方法查找。下面我们看看 `resolveInstanceMethod` 方法做了啥？

## 动态方法决议

```objc
static void resolveInstanceMethod(id inst, SEL sel, Class cls)
{
    runtimeLock.assertUnlocked();
    ASSERT(cls->isRealized());
    SEL resolve_sel = @selector(resolveInstanceMethod:);

    // 1、判断类对象的元类中是否有 `resolveInstanceMethod` 方法。也就是类对象中是否实现了 + resolveInstanceMethod 方法。
    if (!lookUpImpOrNil(cls, resolve_sel, cls->ISA())) {
        // Resolver not implemented.
        return;
    }

    // 2、有实现，则通过发送消息的方式来调用。
    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(cls, resolve_sel, sel);

    // 3、再次查找 sel 方法的实现
    IMP imp = lookUpImpOrNil(inst, sel, cls);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}
```

经过分析，`resolveInstanceMethod` 方法主要做了三件事，在上面代码中有备注。也就是说，我们可以在类中实现 `resolveInstanceMethod` 方法，然后做一些处理，最好是能让找到 sel 对应的 imp。`resolveClassMethod` 方法的实现逻辑大同小异。

比如我们可以通过调用 `class_addMethod` 方法来动态添加方法，这样可以在 crash 前进行挽救处理。这种策略也就是动态方法决议。

如果 `lookUpImpOrForward` 还是得不到 sel 对应的 imp，最后的 imp 将会是 `_objc_msgForward_impcache`，接下来将会走消息转发流程。

我们在 RDPerson 中实现 `resolveInstanceMethod` 方法，然后在里面打个断点。然后用 `bt` 看看此时的方法调用堆栈，结果如下所示，也验证了我们的分析。

```shell
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 10.1
  * frame #0: 0x0000000100003884 HelloWorld`+[RDPerson resolveInstanceMethod:](self=RDPerson, _cmd="resolveInstanceMethod:", sel="sayNB") at RDPerson.m:19:45
    frame #1: 0x0000000100313d47 libobjc.A.dylib`resolveInstanceMethod(inst=0x0000000100760090, sel="sayNB", cls=RDPerson) at objc-runtime-new.mm:6000:21
    frame #2: 0x00000001002ff7c3 libobjc.A.dylib`resolveMethod_locked(inst=0x0000000100760090, sel="sayNB", cls=RDPerson, behavior=1) at objc-runtime-new.mm:6042:9
    frame #3: 0x00000001002ff0ec libobjc.A.dylib`lookUpImpOrForward(inst=0x0000000100760090, sel="sayNB", cls=RDPerson, behavior=1) at objc-runtime-new.mm:6191:16
    frame #4: 0x00000001002da1d9 libobjc.A.dylib`_objc_msgSend_uncached at objc-msg-x86_64.s:1101
    frame #5: 0x00000001000033ff HelloWorld`testMessageForward at main.m:164:5
    frame #6: 0x0000000100003444 HelloWorld`main(argc=1, argv=0x00007ffeefbff478) at main.m:169:9
    frame #7: 0x00007fff6b73ecc9 libdyld.dylib`start + 1
    frame #8: 0x00007fff6b73ecc9 libdyld.dylib`start + 1
```

## 消息转发

经过测试，发现 `resolveInstanceMethod` 方法会被调用两次。我们在第二次调用的地方，看看调用堆栈的情况。

```shell
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 10.1
  * frame #0: 0x0000000100003884 HelloWorld`+[RDPerson resolveInstanceMethod:](self=RDPerson, _cmd="resolveInstanceMethod:", sel="sayNB") at RDPerson.m:19:45
    frame #1: 0x0000000100313d47 libobjc.A.dylib`resolveInstanceMethod(inst=0x0000000000000000, sel="sayNB", cls=RDPerson) at objc-runtime-new.mm:6000:21
    frame #2: 0x00000001002ff7c3 libobjc.A.dylib`resolveMethod_locked(inst=0x0000000000000000, sel="sayNB", cls=RDPerson, behavior=0) at objc-runtime-new.mm:6042:9
    frame #3: 0x00000001002ff0ec libobjc.A.dylib`lookUpImpOrForward(inst=0x0000000000000000, sel="sayNB", cls=RDPerson, behavior=0) at objc-runtime-new.mm:6191:16
    frame #4: 0x00000001002d8cc9 libobjc.A.dylib`class_getInstanceMethod(cls=RDPerson, sel="sayNB") at objc-runtime-new.mm:5921:5
    frame #5: 0x00007fff316a1c68 CoreFoundation`__methodDescriptionForSelector + 282
    frame #6: 0x00007fff316bd57c CoreFoundation`-[NSObject(NSObject) methodSignatureForSelector:] + 38
    frame #7: 0x00007fff31689fc0 CoreFoundation`___forwarding___ + 408
    frame #8: 0x00007fff31689d98 CoreFoundation`__forwarding_prep_0___ + 120
    frame #9: 0x00000001000033ff HelloWorld`testMessageForward at main.m:164:5
    frame #10: 0x0000000100003444 HelloWorld`main(argc=1, argv=0x00007ffeefbff478) at main.m:169:9
    frame #11: 0x00007fff6b73ecc9 libdyld.dylib`start + 1
    frame #12: 0x00007fff6b73ecc9 libdyld.dylib`start + 1
```

在堆栈中我们发现，当动态方法决议后，如果仍然没找到 imp 后，此时会调用 `CoreFoundation` 中的 `__forwarding_prep_0___` 来执行消息转发流程。

## 反编译探索

由于 `CoreFoundation` 没有开源，我们可以通过 [Hopper Disassembler](https://www.hopperapp.com/) 工具来反编译 `CoreFoundation` 的可执行文件，通过阅读反汇编伪代码的方式来继续探索。

首先全局搜索 `__forwarding_prep_0___`, 可找到如图的伪代码。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/forwarding-prep.jpg)

图中发现将会继续调用 `___forwarding___` 方法，我们双击进去看看是啥。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/forwarding.jpg)

在这里我们找到了会调用 `forwardingTargetForSelector` 方法。如果这个方法返回的是 nil 然后顺着跳转逻辑。来到了这里，也就是将会调用 `methodSignatureForSelector` 方法。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/method_sign.jpg)

根据图中的跳转，我们发下会调用 `__methodDescriptionForSelector` 方法。看看这个方法的实现，是调用了 `class_getInstanceMethod` 方法。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/desforselector.png)

`class_getInstanceMethod` 的实现如下:

```c
Method class_getInstanceMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;
        
    // Search method lists, try method resolver, etc.
    lookUpImpOrForward(nil, sel, cls, LOOKUP_RESOLVER);

    return _class_getMethod(cls, sel);
}
```

这个方法以 `LOOKUP_RESOLVER` 的方式再次调用了 `lookUpImpOrForward` 的方法，从而会再次调用 `resolveMethod_locked` 方法，也就会调用 `resolveInstanceMethod` 方法。到目前为止，`resolveInstanceMethod` 方法在消息转发流程中会调用两次的原因找到了。

当 `methodSignatureForSelector` 方法的返回值不是 `nil` 时,将会调用 `forwardInvocation` 方法，如果返回的是 `nil` 时，将会通过发送消息的方式调用 `doesNotRecognizeSelector` 方法，从而 crash。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/invocation.jpg)

## 总结

通过以上的探索，我们可以得到完整的消息转发流程图。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/message-forward.png)

## 后记

我是穆哥，卖码维生的一朵浪花。我们下回见。



