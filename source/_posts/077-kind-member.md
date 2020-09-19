---
title: 『底层探索』5  - 「类和方法」的归属问题
toc: true
donate: false
tags: []
date: 2020-09-15 20:42:24
categories: [底层探索]
---

本文主要探索的是类的归属和方法的归属问题。

<!-- more -->

今天探索下类的归属和方法的归属。在 Objective-C 中，对于某个对象，

- 可以用 `class` 方法来获取这个对象所属的类
- 可以用 `isKindOfClass` 方法来判断是否是某类的子类
- 可以用 `isMemberOfClass` 方法来判断是否是某类的实例

对于某个类，

- 可以用 `class_getInstanceMethod` 来查找是否存在某个实例方法
- 可以用 `class_getClassMethod` 来查找是否存在某个类方法

接下来将会探索以上提到的这些方法的底层实现。

### RDTeacher

首先声明一个继承于 `NSObject` 的 `RDTeacher` 类, 它:

- 实现了一个 `sayStandUp` 的类方法
- 实现了一个 `sayByebye` 的实例方法

完整代码如下。

```objc
@interface RDTeacher : NSObject

+ (void)sayStandUp;
- (void)sayByebye;
- 
@end

@implementation RDTeacher

+ (void)sayStandUp {
    NSLog(@"Stand up");
}

- (void)sayByebye {
    NSLog(@"Good bye");
}

@end
```

### class

首先探索 `class` 方法，看看下面的 `testClass` 方法，你觉得 cls0 和 cls1 的地址会相同么？那么 cls2 和 cls3 呢？记住你此时的答案。

```objc
void testClass() {
    RDTeacher *person = [[RDTeacher alloc] init];
    Class cls0 = [person class];
    Class cls1 = [RDTeacher class];
    NSLog(@"%p -- %p", cls0, cls1);  // objc_opt_class
    
    NSObject *object = [[NSObject alloc] init];
    Class cls2 = [object class];
    Class cls3 = [NSObject class];
    NSLog(@"%p -- %p", cls2, cls3);
}
```

看看打印的结果是啥？

```shell
0x100003550 -- 0x100003550
0x100335140 -- 0x100335140
```

有没有出乎你的意外，为什么实例对象和类对象的 class 是一样的呢。在调用 class 的地方打个断点，运行到断点处。然后打开 `Xcode` -> `Debug` -> `Debug workflow` -> `Always Show Disassembly` 后，编辑区的代码会变成汇编形式的。我们会发现调用 `class` 方法实际在底层调用的是 `objc_opt_class` 方法。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/objcclass.jpg)

接下来在包含 `objc 781` 源码的 `HelloWorld` 工程中全局搜索 `objc_opt_class` 关键词，会找到如下的源码。可编译的 `HelloWorld` 工程在这里下载 [objc-runtime](https://github.com/muhlenXi/objc-runtime)。

```objc
// Calls [obj class]
Class
objc_opt_class(id obj)
{
#if __OBJC2__
    if (slowpath(!obj)) return nil;
    Class cls = obj->getIsa();
    if (fastpath(!cls->hasCustomCore())) {
        return cls->isMetaClass() ? obj : cls;
    }
#endif
    return ((Class(*)(id, SEL))objc_msgSend)(obj, @selector(class));
}
```

可以看出，在 \_\_OBJC2__ 中主要是通过 `getIsa()` 获取对象的 `class`，然后根据 `cls` 是否是 `meta class` 返回不同的 `class`, 如果是类对象，则返回本身，否则返回该对象所属的类。为什么要这么设计呢？当你看到本文最后的 isa 走向图，你就会恍然大悟的，暂时先记着这个逻辑。

### class_getInstanceMethod class_getClassMethod

下面代码块创建了一个 `RDTeacher` 类型的对象 t，然后得到对象 t 的类 cls 和元类 metaCls，然后我们分别在这两个 class 中查找 `sayByebye` 实例方法和查找 `sayStandUp` 实例方法，猜猜能找到么？

```objc
void testGetMethod() {
    RDTeacher *t = [[RDTeacher alloc] init];
    Class cls = [t class];
    
    const char* className = class_getName(cls);
    Class metaCls = objc_getMetaClass(className);
    
    Method method0 = class_getInstanceMethod(cls, @selector(sayByebye));
    Method method1 = class_getInstanceMethod(metaCls, @selector(sayByebye));
    NSLog(@"getInstanceMethod --> %p %p \n", method0, method1);  // yes no
    
    Method method2 = class_getClassMethod(cls, @selector(sayStandUp));
    Method method3 = class_getClassMethod(metaCls, @selector(sayStandUp));
    NSLog(@"getClassMethod --> %p %p \n", method2, method3);     // yes yes
}
```

根据方法的功能定义，method0 能找到，method1 找不到。method2 能找到，method3 找不到。看看打印结果，是否是和我们的分析一致。

```shell
getInstanceMethod --> 0x100003118 0x0
getClassMethod --> 0x100003098 0x100003098
```

为什么 method3 能找到，也就是说，对于一个元类，也能找到它的类方法？看看源码是怎么实现的。

```objc
Method class_getInstanceMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;
    lookUpImpOrForward(nil, sel, cls, LOOKUP_RESOLVER);
    return _class_getMethod(cls, sel);
}
```

以上是 `class_getInstanceMethod` 的实现，`_class_getMethod` 的实现就不贴了，感兴趣的可以自己找找，这里不影响我们的分析。 

```objc
Method class_getClassMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;
    return class_getInstanceMethod(cls->getMeta(), sel);
}
```

`class_getClassMethod` 的实现比较巧妙，寻找类的类方法，相当于寻找元类的实例方法。因为对象所属的类是类，类对象所属的类是元类。再看看 `getMeta()` 方法做了啥？

```objc
Class getMeta() {
    if (isMetaClass()) return (Class)this;
    else return this->ISA();
}
```

**如果类是元类返回本身，否则返回所属的类。** 根据这个逻辑，对类对象和该类对象所属的元类查找相同的类方法，底层调用逻辑是一样的。所以 method3 也能找到类方法的原因水落石出了。

### isKindOfClass

`isKindOfClass` 用于判断一个对象的类是否是某个类的子类。看看下面的打印结果是否如你所想的一样。首先看看 `isKindOfClass` 底层实现。

```objc
// Calls [obj isKindOfClass]
BOOL
objc_opt_isKindOfClass(id obj, Class otherClass)
{
#if __OBJC2__
    if (slowpath(!obj)) return NO;
    Class cls = obj->getIsa();
    if (fastpath(!cls->hasCustomCore())) {
        for (Class tcls = cls; tcls; tcls = tcls->superclass) {
            if (tcls == otherClass) return YES;
        }
        return NO;
    }
#endif
    return ((BOOL(*)(id, SEL, Class))objc_msgSend)(obj, @selector(isKindOfClass:), otherClass);
}
```

该方法的核心逻辑是通过 `getIsa` 找到该对象所属的类，然后在该类的 superclass 继承链上寻找是否有类等于指定的类。

```objc
void testKindOf() {
    BOOL res0 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
    BOOL res1 = [(id)[RDTeacher class] isKindOfClass:[RDTeacher class]];
    // 1 0
    
    BOOL res2 = [(id)[NSObject alloc] isKindOfClass:[NSObject class]];
    BOOL res3 = [(id)[RDTeacher alloc] isKindOfClass:[RDTeacher class]];
    // 1 1
    NSLog(@"%d %d %d %d", res0, res1, res2, res3);
}
```

以上的代码的打印结果是 `1 0 1  1`, 为什么会是这个结果呢？请看下面的分析：

- 对于 res0，`[NSObject class]` 得到的是 `NSObject 类`，类对象 `getIsa` 得到的是 `NSObject 根元类`，根元类的 `superclass` 是 `NSObject 类`，在根元类的 superclass 继承链上可以找到 `NSObject 类`。 所以得到的结果是 **1**。
- 对于 res1，`[RDTeacher class]` 得到的是 `RDTeacher 类`，类对象 `getIsa` 得到的是 `RDTeacher 根元类`，在 `RDTeacher 根元类`的 superclass 继承链上找不到 `RDTeacher 类`。所以得到的结果是 **0**。

- 对于 res3，`[NSObject alloc]` 得到的是 `NSObject 对象`，对象 `getIsa` 得到的是 `NSObject 类`，`NSObject 类`的 superclass 继承链上可以找到 `NSObject 类`。 所以得到的结果是 **1**。
- 对于 res4，`[RDTeacher alloc]` 得到的是 `RDTeacher 对象`，对象 `getIsa` 得到的是 `RDTeacher 类`，在 `RDTeacher 类`的 superclass 继承链上可以找到 `RDTeacher 类`。所以得到的结果是 **1**。

### isMemberOfClass

`isMemberOfClass` 用于判断一个对象是否是某个类的实例。看看 `isMemberOfClass` 底层实现。

```objc
+ (BOOL)isMemberOfClass:(Class)cls {
    return self->ISA() == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
```

通过源码，我们可以得出该方法的核心逻辑：

- 如果是类对象，则判断类对象 isa 指向的类是否和指定类相同。
- 如果是对象，则判断对象所属的类是否和指定类相同。

看看下面的代码打印结果是否如你所想的一样？

```objc
void testMemberOf() {
    BOOL res4 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
    BOOL res5 = [(id)[RDTeacher class] isMemberOfClass:[RDTeacher class]];
    // 0 0
       
    BOOL res6 = [(id)[NSObject alloc] isMemberOfClass:[NSObject class]];
    BOOL res7 = [(id)[RDTeacher alloc] isMemberOfClass:[RDTeacher class]];
    // 1 1
       
    NSLog(@"%d %d %d %d", res4, res5, res6, res7);
}
```

以上的代码的打印结果是 `0 0 1  1`, 为什么会是这个结果呢？请看下面的分析：

- 对于 res4，`[NSObject class]` 得到的是 `NSObject 类`，类对象 `ISA()` 得到的是 `NSObject 根元类`，`NSObject 根元类` 和 `NSObject 类` 是不同的 ，所以得到的结果是 **0**。
- 对于 res5，`[RDTeacher class]` 得到的是 `RDTeacher 类`，类对象 `ISA()` 得到的是 `RDTeacher 根元类`，`RDTeacher 根元类` 和 `RDTeacher 类` 是不相同的。所以得到的结果是 **0**。

- 对于 res6，`[NSObject alloc]` 得到的是 `NSObject 对象`，对象 `ISA()` 得到的是 `NSObject 类`，`NSObject 类` 和 `NSObject 类` 是相同的。 所以得到的结果是 **1**。
- 对于 res7，`[RDTeacher alloc]` 得到的是 `RDTeacher 对象`，对象 `ISA()` 得到的是 `RDTeacher 类`，在 `RDTeacher 类` 和 `RDTeacher 类` 是相同的。所以得到的结果是 **1**。

最后附上 Apple 经典的 isa 走位图，如果你能把这张图了然于胸，上面的这些对你来说就是 small case。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/isadir.png)

### 后记

我是穆哥，卖码维生的一朵浪花。我们下回见。