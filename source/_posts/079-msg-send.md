---
title: 『底层探索』7 - Objective-C 中的方法调用流程
toc: true
donate: false
tags: []
date: 2020-09-20 13:31:50
categories: [底层探索]
---

在 Objective-C 中，当我们调用一个对象的方法后，在底层经历怎样的流程呢？这就是我们今天要探索的

<!-- more -->


## 初探

如下我们定义了一个继承 `NSObject ` 的 `RDPerson ` 类，定义了一个继承 `RDPerson ` 的 `RDStudent ` 类。


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

这里和我们第一次打印的结果是一致的，这也证明了我们的分析。

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

在 runtime 源码中，我们找到了`objc_msgSend` 的实现（ ARM64指令集架构的 ）, 对主要流程，我也添加了相关注释。

```
	ENTRY _objc_msgSend           // 方法入口
	UNWIND _objc_msgSend, NoFrame

	cmp	p0, #0			   // 检查消息接收对象是否是 nil 和支持 taggedPointer
#if SUPPORT_TAGGED_POINTERS
	b.le	LNilOrTagged		// 如果支持 taggedPointer，则跳转 LNilOrTagged
#else
	b.eq	LReturnZero      // 如果不支持 taggedPointer，并且是 nil 则跳转 LReturnZero
#endif
	ldr	p13, [x0]		   // 将消息接收对象的 isa，加载到 p13 中
	GetClassFromIsa_p16 p13		// 获取 isa 中的 shiftcls，加载到 p16 中
LGetIsaDone:
	CacheLookup NORMAL, _objc_msgSend  // class 获取完毕后，跳转到 CacheLookup 

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
	b.eq	LReturnZero		// 如果是 nil 则跳转 LReturnZero

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
#endif

LReturnZero:  // 返回值清零后返回
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

接下来我们看一下 `CacheLookup` 的实现。

```c
.macro CacheLookup
LLookupStart$1:
	// p1 = SEL, p16 = isa
	ldr	p11, [x16, #CACHE]				// p11 = mask|buckets

#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
	and	p12, p1, p11, LSR #48		// x12 = _cmd & mask
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	and	p10, p11, #~0xf			// p10 = buckets
	and	p11, p11, #0xf			// p11 = maskShift
	mov	p12, #0xffff
	lsr	p11, p12, p11				// p11 = mask = 0xffff >> p11
	and	p12, p1, p11				// x12 = _cmd & mask
#else
#error Unsupported cache mask storage for ARM64.
#endif


	add	p12, p10, p12, LSL #(1+PTRSHIFT)
		             // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT))

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// wrap: p12 = first bucket, w11 = mask
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	add	p12, p12, p11, LSR #(48 - (1+PTRSHIFT))
					// p12 = buckets + (mask << 1+PTRSHIFT)
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	add	p12, p12, p11, LSL #(1+PTRSHIFT)
					// p12 = buckets + (mask << 1+PTRSHIFT)
#else
#error Unsupported cache mask storage for ARM64.
#endif

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

LLookupEnd$1:
LLookupRecover$1:
3:	// double wrap
	JumpMiss $0

.endmacro
```
