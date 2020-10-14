---
title: 『底层探索』8 - OC 消息发送流程之慢速查找
toc: true
donate: false
tags: []
date: 2020-09-23 09:39:55
categories: [底层探索]
---

在上篇 [OC 消息发送流程之快速查找](https://www.muhlenxi.com/2020/09/20/079-msg-send/) 中，如果最后没有找到方法的 imp，会跳转到 `CheckMiss` 或者 `JumpMiss`。今天将会探索这两个流程。

<!-- more -->

## 基础

在 WWDC 2020 的 [`Advancements in the Objective-C runtime`](https://developer.apple.com/videos/play/wwdc2020/10163/)中，Apple Runtime team 工程师阐述了，他们为了优化内存占用，提高 App 的性能，做了三个方面的改进。

- 1、修改了 runtime 中类的数据结构。
- 2、修改了 Objective-C method lists。
- 3、修改了 tagged pointers。

还给我们一个可靠的建议。**Don't read internal bits directly. Use the APIs.**


## 承上启下

首先看看 `CheckMiss` 和 `JumpMiss` 的汇编实现是什么样的。

```armasm
.macro CheckMiss
.if $0 == GETIMP
	cbz	p9, LGetImpMiss
.elseif $0 == NORMAL
	cbz	p9, __objc_msgSend_uncached
.elseif $0 == LOOKUP
	cbz	p9, __objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro

.macro JumpMiss
.if $0 == GETIMP
	b	LGetImpMiss
.elseif $0 == NORMAL
	b	__objc_msgSend_uncached
.elseif $0 == LOOKUP
	b	__objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro
```

也就是说在 `NORMAL` 模式下，这两个方法最终都会跳转到 `__objc_msgSend_uncached` 这里，那么这里又是啥呢？通过全局搜索，发现类似这个信息, 除此之外，再没有发现任何有价值信息。

```armasm
STATIC_ENTRY __objc_msgSend_uncached
UNWIND __objc_msgSend_uncached, FrameWithNoSaves
	
MethodTableLookup
TailCallFunctionPointer x17

END_ENTRY __objc_msgSend_uncached
```

在这里，我们发现了一个 `MethodTableLookup` , 字面意思是方法列表查找，这里面貌似有我们的方法，去看看实现。

```armasm
.macro MethodTableLookup // 汇编宏定义
	
	// push frame
	SignLR
	stp	fp, lr, [sp, #-16]!
	mov	fp, sp

	// save parameter registers: x0..x8, q0..q7
	sub	sp, sp, #(10*8 + 8*16)
	stp	q0, q1, [sp, #(0*16)]
	stp	q2, q3, [sp, #(2*16)]
	stp	q4, q5, [sp, #(4*16)]
	stp	q6, q7, [sp, #(6*16)]
	stp	x0, x1, [sp, #(8*16+0*8)]
	stp	x2, x3, [sp, #(8*16+2*8)]
	stp	x4, x5, [sp, #(8*16+4*8)]
	stp	x6, x7, [sp, #(8*16+6*8)]
	str	x8,     [sp, #(8*16+8*8)]

	// lookUpImpOrForward(obj, sel, cls, LOOKUP_INITIALIZE | LOOKUP_RESOLVER)
	// receiver and selector already in x0 and x1
	mov	x2, x16
	mov	x3, #3
	bl	_lookUpImpOrForward

	// IMP in x0
	mov	x17, x0
	
	// restore registers and return
	ldp	q0, q1, [sp, #(0*16)]
	ldp	q2, q3, [sp, #(2*16)]
	ldp	q4, q5, [sp, #(4*16)]
	ldp	q6, q7, [sp, #(6*16)]
	ldp	x0, x1, [sp, #(8*16+0*8)]
	ldp	x2, x3, [sp, #(8*16+2*8)]
	ldp	x4, x5, [sp, #(8*16+4*8)]
	ldp	x6, x7, [sp, #(8*16+6*8)]
	ldr	x8,     [sp, #(8*16+8*8)]

	mov	sp, fp
	ldp	fp, lr, [sp], #16
	AuthenticateLR

.endmacro
```

在这个实现了中，代码配置了相关参数后，就跳转到了 `_lookUpImpOrForward`, 搜一个这个 `lookUpImpOrForward` 关键在，在 `objc-runtime-new.mm` 的 6094 行附近找到了 `IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)`。

因此，我们猜测，如果汇编代码在缓存中找不到方法后，会调用这个函数继续查找，打个断点，看是否和我们分析的一致，经过断点调试后，的确调用了这个方法。

接下来，我们分析下 `lookUpImpOrForward` 方法的实现。

## lookUpImpOrForward

```c
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
    const IMP forward_imp = (IMP)_objc_msgForward_impcache;
    IMP imp = nil;
    Class curClass;

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup 如果 behavior 是 LOOKUP_CACHE，则从缓存中查找
    if (fastpath(behavior & LOOKUP_CACHE)) {
        imp = cache_getImp(cls, sel);
        if (imp) goto done_nolock;
    }
    
    runtimeLock.lock();
    
    checkIsKnownClass(cls);

    if (slowpath(!cls->isRealized())) {
        cls = realizeClassMaybeSwiftAndLeaveLocked(cls, runtimeLock);
    }

    if (slowpath((behavior & LOOKUP_INITIALIZE) && !cls->isInitialized())) {
        cls = initializeAndLeaveLocked(cls, inst, runtimeLock);
    }

    runtimeLock.assertLocked();
    curClass = cls;

    for (unsigned attempts = unreasonableClassCount();;) {
        // curClass method list. 方法列表查找
        Method meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            imp = meth->imp;
            goto done;
        }
        // curClass 赋值为父类，并且判断是否是 nil
        if (slowpath((curClass = curClass->superclass) == nil)) {
            // No implementation found, and method resolver didn't help.
            // Use forwarding.
            imp = forward_imp;
            break;
        }

        // Halt if there is a cycle in the superclass chain. 防止陷入死循环
        if (slowpath(--attempts == 0)) {
            _objc_fatal("Memory corruption in class list.");
        }

        // Superclass cache. 在父类缓存中查找
        imp = cache_getImp(curClass, sel);
        if (slowpath(imp == forward_imp)) {
            break;
        }
        if (fastpath(imp)) {
            goto done;
        }
    }

    // No implementation found. Try method resolver once.
    if (slowpath(behavior & LOOKUP_RESOLVER)) {
        behavior ^= LOOKUP_RESOLVER;
        return resolveMethod_locked(inst, sel, cls, behavior);
    }

 done:
    log_and_fill_cache(cls, imp, sel, inst, curClass);
    runtimeLock.unlock();
 done_nolock:
    if (slowpath((behavior & LOOKUP_NIL) && imp == forward_imp)) {
        return nil;
    }
    return imp;
}
```

在上述代码发现，有些代码的执行依赖于 behavior 这个参数，我们看看这个 behavior 的值分布。


```c
/* method lookup */
enum {
    LOOKUP_INITIALIZE = 1,  // 0001
    LOOKUP_RESOLVER = 2,    // 0010
    LOOKUP_CACHE = 4,       // 0100
    LOOKUP_NIL = 8,         // 1000
};
```

根据我的注释，有没有发现什么规律？就是这个枚举中任意两个元素，如果它们不同，那么它们 `&` 运算后的结果必然是 0。

`lookUpImpOrForward` 主要做的是在 cls 的继承链上调用 `getMethodNoSuper_nolock` 查找方法的 imp, 直到 superclass 为 nil 为止，如果找到则调用 `log_and_fill_cache` 缓存方法然后返回 imp，找不到则会调用 `resolveMethod_locked` 方法，并且这个方法只会被调用一次。

`log_and_fill_cache` 方法做了啥？这引起了我的好奇。

```c
static void
log_and_fill_cache(Class cls, IMP imp, SEL sel, id receiver, Class implementer)
{
#if SUPPORT_MESSAGE_LOGGING
    if (slowpath(objcMsgLogEnabled && implementer)) {
        bool cacheIt = logMessageSend(implementer->isMetaClass(), 
                                      cls->nameForLogging(),
                                      implementer->nameForLogging(), 
                                      sel);
        if (!cacheIt) return;
    }
#endif
    cache_fill(cls, sel, imp, receiver);
}
```

这个方法的作用是根据配置打印日志和调用 `cache_fill` 方法填充缓存。

```c
void cache_fill(Class cls, SEL sel, IMP imp, id receiver)
{
    runtimeLock.assertLocked();

#if !DEBUG_TASK_THREADS
    // Never cache before +initialize is done
    if (cls->isInitialized()) {
        cache_t *cache = getCache(cls);
#if CONFIG_USE_CACHE_LOCK
        mutex_locker_t lock(cacheUpdateLock);
#endif
        cache->insert(cls, sel, imp, receiver);
    }
#else
    _collecting_in_critical();
#endif
}
```

`cache->insert` 是不是很熟悉，这就是我们上篇文章探索的缓存插入方法。

接下来看看查找流程，`getMethodNoSuper_nolock` 是怎么查找 cls 中的 imp 呢？

```c
static method_t *
getMethodNoSuper_nolock(Class cls, SEL sel)
{
    runtimeLock.assertLocked();

    ASSERT(cls->isRealized());

    auto const methods = cls->data()->methods();
    for (auto mlists = methods.beginLists(),
              end = methods.endLists();
         mlists != end;
         ++mlists)
    {
        method_t *m = search_method_list_inline(*mlists, sel);
        if (m) return m;
    }

    return nil;
}
```

可以看出这段代码的操作是得到 methods 列表，从头到尾遍历这个列表，然后在子列表中继续调用 `search_method_list_inline` 方法查找 imp，找到返回 imp 否则返回 nil。继续看看 `search_method_list_inline` 的实现。

```c
ALWAYS_INLINE static method_t *
search_method_list_inline(const method_list_t *mlist, SEL sel)
{
    int methodListIsFixedUp = mlist->isFixedUp();
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);
    
    if (fastpath(methodListIsFixedUp && methodListHasExpectedSize)) {
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        // Linear search of unsorted method list
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }

#if DEBUG
    // sanity-check negative results
    if (mlist->isFixedUp()) {
        for (auto& meth : *mlist) {
            if (meth.name == sel) {
                _objc_fatal("linear search worked when binary search did not");
            }
        }
    }
#endif

    return nil;
}
```

`isFixedUp` 是 `static uint32_t fixed_up_method_list = 3` 常量。根据代码注释，可以得到这样的信息。

> Method lists from shared cache are 1 (uniqued) or 3 (uniqued and sorted).

这个方法根据列表是否同时满足下面两个条件，然后分为两个策略：

- 1、是否是独一无二的。
- 2、列表中单个元素的内存占用是否和 `method_t` 类型一致。

如果满足条件，则调用 `findMethodInSortedMethodList` 方法进行查找。不满足条件，则进行线性遍历进行逐个比较，线性查找是一种低效的策略。

`findMethodInSortedMethodList` 方法的实现如下：

```c
ALWAYS_INLINE static method_t *
findMethodInSortedMethodList(SEL key, const method_list_t *list)
{
    ASSERT(list);

    const method_t * const first = &list->first;
    const method_t *base = first;
    const method_t *probe;
    uintptr_t keyValue = (uintptr_t)key;
    uint32_t count;
    for (count = list->count; count != 0; count >>= 1) {
        probe = base + (count >> 1);
        
        uintptr_t probeValue = (uintptr_t)probe->name;
        
        if (keyValue == probeValue) {
            // 如果找到 method 的 name 和 key 相同，需要继续向前查找，看前面的 method 中 name 是否和 key 一样，要找到最前的那个 method
            while (probe > first && keyValue == (uintptr_t)probe[-1].name) {
                probe--;  
            }
            return (method_t *)probe;
        }
        
        if (keyValue > probeValue) {
            base = probe + 1;
            count--;
        }
    }
    
    return nil;
}
```

在一个有序的列表中，二分查找是一种高效的策略，上述的代码用的就是二分查找的策略，找到满足条件的 method 后，继续向前查找，找到左边边界，然后返回 method，没有则返回 nil。

到这一步，我们可以清晰的得到消息发送的慢速查找流程。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/msg_find_slow.png)

下篇文章将会探索发送消息时，当快速查找和慢速查找都都不到方法时，这时候应该怎么处理呢？

## 后记

我是穆哥，卖码维生的一朵浪花。我们下回见。
