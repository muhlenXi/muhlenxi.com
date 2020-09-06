---
title: 探究 alloc 做了啥 
toc: true
donate: false
tags: [alloc]
date: 2020-09-05 22:07:25
categories: [源码]
---

没有对象怎么办？new 一个，在 Objective-C 中我们可以通过 alloc 或 new 创建一个对象，那么问题来了？它底层是怎么实现的呢？

<!-- more -->

### overview

在 iOS 开发中，当我们创建一个类的实例的时候，会不假思索的使用如下的代码来创建对象。

```objc
[XXObject alloc] init];
[XXObject new];
```
在底层，alloc 的流程是什么样的呢？通过 Apple 开源的 runtime 源码 [objc4](https://opensource.apple.com/tarballs/objc4/), 我们可以一探究竟，本文参考的是 objc-781 版本。

### alloc

在 NSObject.mm 文件中，我们可以找到 alloc 的代码实现

```objc
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```

跟着这个调用路径，我们可以挖掘出如下调用流程：

```objc
1 _objc_rootAlloc(self)
2 callAlloc(cls, false, true)
3 _objc_rootAllocWithZone(cls, nil)
4 _class_createInstanceFromZone(cls, 0, nil, OBJECT_CONSTRUCT_CALL_BADALLOC)
```

比较关键的是方法是 `_class_createInstanceFromZone `, 在这个方法中，完成了内存空间计算、内存申请、对象关联等操作，完整代码如下：

```objc
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    // 1、计算需要开辟内存空间的大小，以16字节对齐
    size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (zone) {
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        // 2、申请内存空间
        obj = (id)calloc(1, size);
    }
    if (slowpath(!obj)) {
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    if (!zone && fast) {
        // 3、初始化isa，关联对象
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```

### new

通过 new 创建对象的流程是咋样的呢？在源码面前了无秘密。

```objc
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```

### init

init 的作用是啥呢？字面理解是初始化数据，看看源码是否如我们猜想的？

```objc
+ (id)init {
    return (id)self;
}

- (id)init {
    return _objc_rootInit(self);
}

id _objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```

居然什么也没做，仅仅返回了 self 而已，这是为什么呢？

### sum-up

通过一幅图，我们可以总结 alloc 的调用流程。


![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/oc-alloc.png)

我就是穆哥，我们下次见。