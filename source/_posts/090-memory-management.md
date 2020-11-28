---
title: 『底层探索』16 - iOS 内存管理底层探索
toc: true
donate: false
tags: []
date: 2020-11-25 16:25:24
categories: [底层探索]
---

本系列第一篇文章，我们探索了对象是怎么创建的的，那么操作系统是怎么管理内存中对象的声明周期呢？也就是内存管理，本文会探索内存管理的底层原理。

<!-- more -->

## 目标

本文的主要探索如下：

- 1、MRC 是什么？
- 2、ARC 是什么？与 MRC 的区别？
- 3、TaggedPointer 对象？
- 3、普通对象的引用计数是怎么管理的？
- 4、普通对象时怎么销毁的？
- 5、strong weak assign copy？
- 6、自动释放池的工作原理

## MRC

在 iOS 中，引用类型对象的内存管理是通过引用计数（Reference Counting）来管理的。每个对象在使用的过程中都有各自的引用计数，当引用计数为 0 的时候，对象会被销毁，对象占用的内存将会被释放。

管理引用计数主要有两种方式，MRC 和 ARC。 MRC 是 Manual Reference Counting 的缩写，就是手动管理对象的引用计数。在早期的开发中，可以使用以下的代码管理引用计数。

```objc
MyClass *foo = [[MyClass alloc] init];
[foo retain];
[foo release];
MyClass *bar = [foo copy];
[foo autorelease];
```

## ARC

在现在的开发中，是使用 `ARC` (Automatic Reference Counting) 技术来管理 App 中对象的内存使用的。

我们无需在代码中添加上述的引用计数管理代码了，这件事情编译器会在编译期帮我们做。也就是在编译的时候，编译器会在我们代码中的合适位置添加上 `retain`, `release`, `copy`, `autorelease` 和 `autoreleasepool` 语句。这样我们就可以专注于功能开发了。

### TaggedPointer 

#### LittleEndian or BigEndian

我们先了解下字节序，我们都知道数据在内存和磁盘中都是按照字节来存储的，其中存储的顺序就是字节序。现在有两种字节序，分别是 LittleEndian（小端序）和 BigEndian（大端序）。更多资料见 [字节顺序](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82%E5%BA%8F).

小端序的特点：数据低字节数据在内存低地址中存储，高字节数据在内存高地址中存储。
大端序的特点：刚好与小端序相反。网络中传输的数据是大端序。

x86 和 ARM 系列处理器使用的是小端序，下面我们通过代码验证下：

方式 1：调用 API

```objc
if (NSHostByteOrder() == NS_LittleEndian) {
    NSLog(@"LittleEndian");
} else if(NSHostByteOrder() == NS_BigEndian){
    NSLog(@"BigEndian");
} else {
    NSLog(@"Unknow");
}
```

方式 2：看内存中的数据排列方式

```objc
UInt32 value = 0xa1b2c3d4;
NSLog(@"%p", &value);
NSLog(@"%d", value);
```

在 NSLog 地方下断点，然后使用 lldb 指令查看内存数据。

```shell
(lldb) p &value
(UInt32 *) $0 = 0x00007ffeeacf010c
(lldb) x 0x00007ffeeacf010c
0x7ffeeacf010c: d4 c3 b2 a1 b0 77 40 6c 94 7f 00 00 48 54 f1 04  .....w@l....HT..
0x7ffeeacf011c: 01 00 00 00 29 8c 35 5e ff 7f 00 00 b0 77 40 6c  ....).5^.....w@l
```

输出的内存结果是 `d4 c3 b2 a1`, 也就是数据低字节数据存储在内存低地址中，所以是小端序。

#### 标记指针

为了提高性能和减少内存使用，Apple 开发出一种叫 tagged pointer 的技术。这项被应用在一些系统提供的一些小对象上，比如 `NSString`, `NSDate`, `NSNumber` 等。

前面我们了解到对象在内存中是对齐分布的，也就是它们的地址通常是 16 的倍数。比如对象指针是一个 64 位的整数，为了对齐，内存地址的一些位永远是 0。我们看一下如下对象的地址。

```objc
NSObject *objc = [[NSObject alloc] init];
UILabel *label = [[UILabel alloc] init];
UIView *view = [[UIView alloc] init];
UIButton *button = [[UIButton alloc] init];
UIViewController *controller = [[UIViewController alloc] init];
    
NSArray *array = @[objc, label, view, button, controller];
for (NSObject *obj in array) {
    NSLog(@"%p", obj);
}
```

打印结果如下:

```shell
2020-11-26 20:39:58.776037+0800 OCDemo[14579:347545] 0x60000308c6f0
2020-11-26 20:39:58.776203+0800 OCDemo[14579:347545] 0x7ffd8e40b2d0
2020-11-26 20:39:58.776281+0800 OCDemo[14579:347545] 0x7ffd8e40d1f0
2020-11-26 20:39:58.776380+0800 OCDemo[14579:347545] 0x7ffd8e40d360
2020-11-26 20:39:58.776481+0800 OCDemo[14579:347545] 0x7ffd8e40df90
```

从打印结果看，内存地址的值没有完全占满 8 个字节 64 位。所以肯定有空余字节上全是 0 。

Tagged Pointer 正好使用了这一特点，它给那些最高位不为 0 的对象赋予了特别的含义。在 Objective-C 的 64 位实现中，最高位为 1 的的对象指针就被视为 Tagged Pointer（标记指针）。剩下的 3 位为已标记类表的索引，该索引用于查找标记指针的类。剩下的 60 位用来存储标记类的数据。

在 Objective-C 中，tagged pointer 的地址值做了混淆处理，混淆的代码实现是这样的：

```objc
extern uintptr_t objc_debug_taggedpointer_obfuscator;

static inline void * _Nonnull
_objc_encodeTaggedPointer(uintptr_t ptr)
{
    return (void *)(objc_debug_taggedpointer_obfuscator ^ ptr);
}
```

所以，想要看到 tagged pointer 的真实地址值，我们需要对混淆值进行还原：

```objc
static inline uintptr_t
_objc_decodeTaggedPointer(const void * _Nullable ptr)
{
    return (uintptr_t)ptr ^ objc_debug_taggedpointer_obfuscator;
}
```

sidetable

retain release retainCount autoRelease dealloc 

weak autoReleasePool

## 参考资料

- [0、About Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html)
- [1、Automatic Reference Counting](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html)
- [2、Resolving Strong Reference Cycles Between Class Instances](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html#ID52)
- [3、Transitioning to ARC Release Notes](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html)