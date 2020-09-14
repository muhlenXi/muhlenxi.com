---
title: 底层探索 - 探究 alloc 做了啥 
toc: true
donate: false
tags: [alloc]
date: 2020-09-05 22:07:25
categories: [底层探索]
---

没有对象怎么办？new 一个，在 Objective-C 中我们可以通过 alloc 或 new 创建一个对象，那么问题来了？它底层是怎么实现的呢？

<!-- more -->

### overview

在 iOS 开发中，当我们创建一个类的实例的时候，会不假思索的使用如下的代码来创建对象。

```objc
[XXObject alloc] init];
[XXObject new];
```

### question

先来个思考题测试下，下面代码的输出结果与你想象的一样么？

```objc
RDPerson *p1 = [RDPerson alloc];
RDPerson *p2 = [p1 init];
RDPerson *p3 = [p1 init];
Log(@"%@ -- %p -- %p",p1, p1, &p1);
Log(@"%@ -- %p -- %p",p2, p2, &p2);
Log(@"%@ -- %p -- %p",p3, p3, &p3);
```

Log 语句打印的第一个是变量指向的对象，第二个是对象的内存地址，第三个是变量的内存地址。
其中 RDPerson 为继承于 NSObject 的一个类，目前没有任何属性和方法。

可能你对 `%@` `%p` 的格式含义不清楚。

- %@ 是打印指定对象的 description 属性，一般是一个字符串。
- %p 是以0x开头，打印出指针的值，也就是内存地址。
- 更多可以参考官网文档对 [String Format Specifiers](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Strings/Articles/formatSpecifiers.html) 的描述。

Log 是一个宏定义，是对 printf 函数的封装。

```objc
#ifdef DEBUG
#define Log(format, ...) printf("%s\n", [[NSString stringWithFormat:format, ## __VA_ARGS__] UTF8String]);
#else
#define Log(format, ...);
#endif
```

从左往右，第三列是打印变量 p1, p2, p3 的内存地址，肯定是不相同的。那么第一列和第二列的打印内容是否是相同的？或者不同的呢？记一下你现在的思考答案。

代码的运行结果出来了，看是不是出乎你所料。

```objc
<RDPerson: 0x10123c230> -- 0x10123c230 -- 0x7ffeefbff528
<RDPerson: 0x10123c230> -- 0x10123c230 -- 0x7ffeefbff520
<RDPerson: 0x10123c230> -- 0x10123c230 -- 0x7ffeefbff518
```

根据打印结果，可以看出，变量 p1, p2, p3 中存储的都是 RDPerson 对象的内存地址，也就是它们指向了同一个对象。

这三个变量地址是依次递减的，也印证了之前说的，变量是存储在栈上，栈底是高地址，栈顶是低地址，它们是由高到低进行分布的。

alloc 的流程是什么样的呢？ init 又做了啥呢？通过 Apple 开源的 runtime 源码 [objc4](https://opensource.apple.com/tarballs/objc4/), 我们可以一探究竟，本文参考的是 objc-781 版本。

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

fastpath 和 slowpath 是啥？

```objc
#define fastpath(x) (__builtin_expect(bool(x), 1))
#define slowpath(x) (__builtin_expect(bool(x), 0))
```

可以看出，这是两个宏定义。`__builtin_expect` 又是什么呢？通过查询 gcc 资料得知，__builtin_expect 是 gcc 编译器的内置函数，用于程序员将分支预测信息告诉编译器，从而让编译器优化我们的代码，提高指令跳转性能。在底层我们知道，分支语句都是通过 jmp 跳转指令来实现的，jmp 要跳转的指令地址越近，做的计算越少，则性能越好。

第一步，对象占用内存的空间是怎么计算的呢？这引起了我的兴趣。

```objc
size_t instanceSize(size_t extraBytes) const {
    if (fastpath(cache.hasFastInstanceSize(extraBytes))) {
        return cache.fastInstanceSize(extraBytes);
    }

    size_t size = alignedInstanceSize() + extraBytes;
    // CF requires all objects be at least 16 bytes.
    if (size < 16) size = 16;
    return size;
}
```

size_t 是啥？

```objc
typedef __SIZE_TYPE__ size_t;
```

```
#ifndef __SIZE_TYPE__
#define __SIZE_TYPE__ long unsigned int
#endif
```

在 stddef.h 文件中可以找到 __SIZE_TYPE__ 的宏定义，因此 size_t 类型也就是 long unsigned int 的 typedef。

接着看看 cache.fastInstanceSize 做了啥？

```objc
size_t fastInstanceSize(size_t extra) const
{
    ASSERT(hasFastInstanceSize(extra));

    if (__builtin_constant_p(extra) && extra == 0) {
        return _flags & FAST_CACHE_ALLOC_MASK16;
    } else {
        size_t size = _flags & FAST_CACHE_ALLOC_MASK;
        // remove the FAST_CACHE_ALLOC_DELTA16 that was added
        // by setFastInstanceSize
        return align16(size + extra - FAST_CACHE_ALLOC_DELTA16);
    }
}
```
__builtin_constant_p 是 gcc 内置函数，用于判断是否是编译器常量。

解开神秘计算的最后一道面纱了，align16 函数。

```objc
static inline size_t align16(size_t x) {
    return (x + size_t(15)) & ~size_t(15);
}
```

看上去一眼懵逼，知道是个计算，啥意思，笨办法，代入 x 一个值，看看啥效果。这里假如 x = 9，别问为啥是 9，我的幸运数字。当然你可以随便选个数字。

```
0000 0000 0000 0000 0000 0000 0000 1001   // 9 用 32 位二进制表示
0000 0000 0000 0000 0000 0000 0000 1111   // 15 用32位二进制表示
0000 0000 0000 0000 0000 0000 0001 1000   // 9 + 15 = 24
1111 1111 1111 1111 1111 1111 1111 0000   // 15 取反的二进制表示
0000 0000 0000 0000 0000 0000 0001 0000   // 按位与的结果，是 16，也就是输入9，输出 16  
```

我们可以得出一个结论，align16 的作用是以16为基准，进行向上取整。小于 16 的数字，取 16，大于 16 小于 32 的数字取 32，以此类推。

为什么要这么处理呢？这就涉及到了内存字节对齐的知识了。感兴趣的可以查查为什么内存要字节对齐呢。

第二步中，calloc 是 c 标准库函数，`void	*calloc(size_t __count, size_t __size)` 分配内存空间，并返回一个指向它的指针。其中第一个参数是要被分配的元素的个数，第二个参数是元素的大小。

第三步，initIsa 是生成 isa 数据，通过源码，发现最终调用的是这个函数。

```objc
inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    ASSERT(!isTaggedPointer()); 
    
    if (!nonpointer) {
        isa = isa_t((uintptr_t)cls);
    } else {
        ASSERT(!DisableNonpointerIsa);
        ASSERT(!cls->instancesRequireRawIsa());

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA
        ASSERT(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa;
    }
}
```

其中 uintptr_t 和 isa_t 的定义如下：

```objc
typedef unsigned long           uintptr_t;
```

```
//isa_t 定义 联合
union isa_t {
    isa_t() { } // 构造函数 1
    isa_t(uintptr_t value) : bits(value) { } // 构造函数 2

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};
```

发现根据是否是 nonpointer 走不同的 isa 生成逻辑。这下真相大白了。

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

居然什么也没做，仅仅返回了 self 而已，这是为什么呢？这是一个不错的思考题。

### sum-up

通过一幅图，我们可以总结 alloc 的调用流程。


![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/oc-alloc.png)

### reference

这我写这篇文章用到的资料，感兴趣可以看看。

- 1、[objc4 781](https://opensource.apple.com/tarballs/objc4/)
- 2、[String Format Specifiers](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Strings/Articles/formatSpecifiers.html)
- 3、[6.59 Other Built-in Functions Provided by GCC](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html#index-g_t_005f_005fbuiltin_005fexpect-4159)
- 4、[stddef.h](https://sites.uclouvain.be/SystInfo/usr/include/stddef.h.html)

我是穆哥，卖码维生的一朵小浪花，我们下期见。