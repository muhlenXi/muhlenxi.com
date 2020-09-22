---
title: 『底层探索』7 - Objective-C 中的方法调用流程
toc: true
donate: false
tags: []
date: 2020-09-20 13:31:50
categories: [底层探索]
---

在 Objective-C 中，当我们调用一个对象的方法后，在底层经历怎样的流程呢？这就是我们今天要探索的。本文会先探索方法缓存查找，也就是快速查找流程。

<!-- more -->


## 基础知识

[Objective-C](https://zh.wikipedia.org/wiki/Objective-C) 是一门动态语言，也就是说在编译时无法确定方法的实现，在运行的时候才能确定。当你定义一个方法而没有实现时，在编译时是不会报错的。所有的方法在运行时才会处理，当无法处理时，程序会抛出异常。在底层是通过 [runtime](https://developer.apple.com/documentation/objectivec/objective-c_runtime) 来提供支持的。


## 初探

如下我们定义了一个继承 `NSObject ` 的 `RDPerson ` 类，定义了一个继承 `RDPerson ` 的 `RDStudent ` 类。


### 类定义

```objc
@interface RDPerson : NSObject

@property (nonatomic, copy) NSString * nickName;

- (void) sayHello;

@end

@implementation RDPerson

- (void)sayHello
{
    NSLog(@"RDPerson: Hello Everybody!");
}

@end
```

```objc
@interface RDStudent : RDPerson

- (void) goToSchool;

@end

@implementation RDStudent

- (void)goToSchool
{
    NSLog(@"RDStudent: Go to school every day!");
}

@end
```

### 对象创建

我们分别创建一个 `person` 对象，然后调用 `sayHello` 方法。创建一个 `student` 对象，然后调用 `sayHello` 和 `goToSchool` 方法，然后看输出是啥。

```objc
RDPerson *person = [[RDPerson alloc] init];
RDStudent *student = [[RDStudent alloc] init];
    
[person sayHello];
[student sayHello];
[student goToSchool];
```

调用方法后输出结果如下所示：

```shell
RDPerson: Hello Everybody!
RDPerson: Hello Everybody!
RDStudent: Go to school every day!
```

### 发送消息

我们使用 clang 对 `main.m` 进行重写，转换成 `c++` 的格式文件。

```shell
clang -rewrite-objc main.m -o main.cpp
```

在 `main.cpp` 中，我们找到了以上调用代码对应的版本。

```c
RDPerson *person = ((RDPerson *(*)(id, SEL))(void *)objc_msgSend)((id)((RDPerson *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("RDPerson"), sel_registerName("alloc")), sel_registerName("init"));
        RDStudent *student = ((RDStudent *(*)(id, SEL))(void *)objc_msgSend)((id)((RDStudent *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("RDStudent"), sel_registerName("alloc")), sel_registerName("init"));

((void (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("sayHello"));
((void (*)(id, SEL))(void *)objc_msgSend)((id)student, sel_registerName("sayHello"));
((void (*)(id, SEL))(void *)objc_msgSend)((id)student, sel_registerName("goToSchool"));
```

通过转换后的代码，我们发现，当我们调用一个对象的方法时，实际在底层是对这个对象发送消息，也就是调用 `objc_msgSend` 方法。我们找一下 `objc_msgSend` 的定义。

```c
objc_msgSend(id _Nullable self, SEL _Nonnull op, ...)
```

第一个参数是接收消息的对象，第二个参数是要发送的消息，也就是要调用的方法。

### 发送消息验证

我们直接在 `main` 函数中直接发送消息，看看是否能达到同样的效果。

```objc
objc_msgSend(person, sel_registerName("sayHello"));
struct objc_super father;
father.receiver = student;
father.super_class = [RDPerson class];
    
objc_msgSendSuper(&father, sel_registerName("sayHello"));
objc_msgSend(student, sel_registerName("goToSchool"));
```

调用上面的代码后，如果发生报错，要按照下图的图片修改下安全设置。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/strict.jpg)

这里和我们第一次打印的结果是一致的，这也证明了我们的分析。这说明方法调用的本质是消息发送。

```shell
RDPerson: Hello Everybody!
RDPerson: Hello Everybody!
RDStudent: Go to school every day!
```

## objc_msgSend

经过查找资料，发现 `objc_msgSend` 方法是用汇编语言实现的。我们先了解下 ARM 常用的汇编指令。

- `str` 读寄存器中的值，存到内存中
- `ldr` 读内存中的值，存到寄存器中
- `stp` 入栈指令，存入两个值
- `ldp` 出栈指令，取出两个值
- `lsr` 逻辑右移（logical shift right）
- `lsl` 逻辑左移 （logical shift left）
- `cmp` 比较
- `add` 相加
- `mov` 寄存器数据移动

### objc_msgSend 汇编分析

在 runtime 源码中，我们找到了`objc_msgSend` 的实现（ ARM64指令集架构的 ）, 对主要流程，我也添加了相关注释。

```armasm
	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame

	cmp	p0, #0			                    // nil check and tagged pointer check，检查消息接收对象是否是 nil 和支持 taggedPointer
#if SUPPORT_TAGGED_POINTERS
	b.le	LNilOrTagged		            //  (MSB tagged pointer looks negative) 如果支持 taggedPointer，则跳转 LNilOrTagged
#else
	b.eq	LReturnZero                     // 如果不支持 taggedPointer，并且是 nil 则跳转 LReturnZero
#endif
	ldr	p13, [x0]		                    // p13 = isa 将消息接收对象的 isa，加载到 p13 中
	GetClassFromIsa_p16 p13	            	// p16 = class 获取 isa 中的 shiftcls，加载到 p16 中
LGetIsaDone:
                                            // calls imp or objc_msgSend_uncached
	CacheLookup NORMAL, _objc_msgSend

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
	b.eq	LReturnZero		                // 如果是 nil 则跳转 LReturnZero

	// tagged
	adrp	x10, _objc_debug_taggedpointer_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_classes@PAGEOFF
	ubfx	x11, x0, #60, #4
	ldr	x16, [x10, x11, LSL #3]
	adrp	x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGE
	add	x10, x10, _OBJC_CLASS_$___NSUnrecognizedTaggedPointer@PAGEOFF
	cmp	x10, x16
	b.ne	LGetIsaDone

                                            // ext tagged
	adrp	x10, _objc_debug_taggedpointer_ext_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_ext_classes@PAGEOFF
	ubfx	x11, x0, #52, #8
	ldr	x16, [x10, x11, LSL #3]
	b	LGetIsaDone
// SUPPORT_TAGGED_POINTERS
#endif

LReturnZero:                                // 返回值清零后返回
                                            // x0 is already zero
	mov	x1, #0
	movi	d0, #0
	movi	d1, #0
	movi	d2, #0
	movi	d3, #0
	ret

	END_ENTRY _objc_msgSend
```

用一个流程图可以简单概括上述汇编代码的流程。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/imgmsg_send1.png)

### CacheLookup 汇编分析

接下来我们看一下 `CacheLookup` 的实现。

```armasm
.macro CacheLookup
LLookupStart$1:

	// p1 = SEL, p16 = isa
	ldr	p11, [x16, #CACHE]				                            // p11 = mask|buckets

#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	and	p10, p11, #0x0000ffffffffffff	                            // p10 = buckets
	and	p12, p1, p11, LSR #48		                                // x12 = _cmd & mask p11右移 48 位后得到 mask，然后 & _cmd 得到 index
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	and	p10, p11, #~0xf			                                    // p10 = buckets
	and	p11, p11, #0xf			                                    // p11 = maskShift, 得到 计算 mask 需要的偏移量
	mov	p12, #0xffff
	lsr	p11, p12, p11				                                // p11 = mask = 0xffff >> p11
	and	p12, p1, p11				                                // x12 = _cmd & mask，得到 index
#else
#error Unsupported cache mask storage for ARM64.
#endif


	add	p12, p10, p12, LSL #(1+PTRSHIFT)                            // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT))，也就是 p12 = buckets + index * 16

	ldp	p17, p9, [x12]		                                        // {imp, sel} = *bucket
1:	cmp	p9, p1			                                            // 判断 bucket->sel == _cmd
	b.ne	2f			                                            // 不相等，跳转到下面的 2  scan more
	CacheHit $0			                                            // 相等，缓存命中 call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			                                        // 检查 bucket->sel == 0，等于 0，则调用相应方法
	cmp	p12, p10		                                            // 判断 当前 bucket == buckets（第一个）
	b.eq	3f                                                      // 相等，则跳转到下面的 3
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	                            // {imp, sel} = *--bucket，取前一个赋值
	b	1b			                                                // 跳转到上面的 1，继续 loop

3:	                                                                // wrap: p12 = first bucket, w11 = mask
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	add	p12, p12, p11, LSR #(48 - (1+PTRSHIFT))
                                                                    // p12 = buckets + (mask << 1+PTRSHIFT)，也就是 将buckets 最后一个元素的地址存到 p12 中
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	add	p12, p12, p11, LSL #(1+PTRSHIFT)
                                                                    // p12 = buckets + (mask << 1+PTRSHIFT)
#else
#error Unsupported cache mask storage for ARM64.
#endif

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	p17, p9, [x12]	                                        	// {imp, sel} = *bucket
1:	cmp	p9, p1			                                            // 判断 bucket->sel == _cmd
	b.ne	2f			                                            // 不相等，跳转到下面的 2  scan more
	CacheHit $0			                                            // 相等，缓存命中 call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			                                        // 检查 bucket->sel == 0，等于 0，则调用相应方法
	cmp	p12, p10		                                            // 判断 当前 bucket == buckets（第一个）
	b.eq	3f                                                      // 相等，则跳转到下面的 3
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	                            // {imp, sel} = *--bucket，取前一个赋值
	b	1b			                                                // 跳转到上面的 1，继续 loop

LLookupEnd$1:
LLookupRecover$1:
3:	// double wrap
	JumpMiss $0                                                     // 跳转 JumpMiss

.endmacro
```

总结一下方法缓存查找，也就是快速查找的流程。

- 1、通过对象的 isa 获取到所属的 class，接着获取到 class 中的 cache，也就是 buckets。
- 2、通过 sel & mask 得到 index，从而得到 buckets[index] 中的 bucket，然后对bucket 中的 sel 和调用的 _cmd 是否相同，如果相同则调用 `CacheHit` 结束流程，如果 sel 为 0，则调用 `CheckMiss` 结束流程。最后 index - 1，比较前一个元素。
- 3、当遍历到第一个元素时，如果还没找到 sel，则跳转到最后一个元素，接着往前遍历查找。如果找到则调用 `CacheHit` 结束流程，如果遇到 sel 为 0，则调用 `JumpMiss` 结束流程。


## 后记

本文我们探索了 `objc_msgSend` 中的缓存查找，也就是快速查找流程，下一篇文章中，我们将探索慢速查找流程，尽请期待。

我是穆哥，卖码维生的一朵浪花。我们下回见。


