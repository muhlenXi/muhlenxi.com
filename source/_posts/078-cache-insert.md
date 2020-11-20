---
title: 『底层探索』6 - Cache 中的 Insert Process
toc: true
donate: false
tags: []
date: 2020-09-19 17:11:48
categories: [底层探索]
---

在前面的文章了，我们探索了 objc_class 中的 superclass，bits，今天来探索 cache。

<!-- more -->

开始探索前，先了解下 CPU 指令集架构的相关知识。

## 指令集架构

### ARM

[ARM 架构](https://zh.wikipedia.org/wiki/ARM%E6%9E%B6%E6%A7%8B) (Advanced RISC Machine) 过去称作 `高级精简指令集机器`，是一个精简指令集（RSIC）处理器架构家族，广泛的应用在许多嵌入式系统设计中。ARM 处理器非常适用于移动通信领域，它符合设计目标为低成本、低功耗、低耗电的特性。

armv6、armv7、armv7s、arm64、arm64e都是arm处理器的指令集，所有指令集原则上都是向下兼容的。

### Intel

[`x86`](https://zh.wikipedia.org/wiki/X86) 泛指一系列英特尔公司开发的处理器的指令集架构。`i386` 是 32 位架构。 [x86-64](https://zh.wikipedia.org/wiki/X86-64) 是 x86 架构的64位扩展，向后兼容 16 位和 32 位架构。

### Apple

Mac 的处理器主要用的是 i386 和 x86-64 架构。iPhone 的处理器主要用的是 ARM 架构，具体可以从下面的列表中查看。

| ARM CPU的不同指令集 | 对应设备   |
|---------------|---------------|
| armv7         | iPhone 3GS，iPhone4，iPhone 4s，iPad，iPad2，iPad3(The New iPad)，iPad mini，iPod Touch 3G，iPod Touch4   |
| armv7s        | iPhone5， iPhone5C，iPad4，iPod5 |
| arm64         | iPhone5s，iPhone6、7、8，iPhone6、7、8 Plus，iPhone X，iPad Air，iPad mini2(iPad mini with Retina Display) |
| arm64e        | XS/XS Max/XR/ iPhone 11, iPhone 11 pro  |
|    x86_64      |  模拟器 64 位处理器 |
|     i386       |  模拟器 32 位处理器 |

## cache_t

通过前面的探索，我们了解到在底层是用 `objc_class` 表示类信息的。

```objc
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;      
    class_data_bits_t bits; 
}

```

之前我们探索了 superclass，bits，现在来探索 cache，cache 主要缓存的是调用过的方法。涉及到缓存，就想知道缓存的机制是什么？比如是如何保存的？如何查找的？

首先看看 `cache_t` 这个类型。

### 属性探索

```c
struct cache_t {
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED
    explicit_atomic<struct bucket_t *> _buckets;
    explicit_atomic<mask_t> _mask;
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
    explicit_atomic<uintptr_t> _maskAndBuckets;
    mask_t _mask_unused;
    
    static constexpr uintptr_t maskShift = 48;
    static constexpr uintptr_t maskZeroBits = 4;
    static constexpr uintptr_t maxMask = ((uintptr_t)1 << (64 - 
    static constexpr uintptr_t bucketsMask = ((uintptr_t)1 << (maskShift - maskZeroBits)) - 1;
    
    static_assert(bucketsMask >= MACH_VM_MAX_ADDRESS, "Bucket field doesn't have enough bits for arbitrary pointers.");
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
    explicit_atomic<uintptr_t> _maskAndBuckets;
    mask_t _mask_unused;

    static constexpr uintptr_t maskBits = 4;
    static constexpr uintptr_t maskMask = (1 << maskBits) - 1;
    static constexpr uintptr_t bucketsMask = ~maskMask;
#else
#error Unknown cache mask storage type.
#endif
    
#if __LP64__
    uint16_t _flags;
#endif
    uint16_t _occupied;

public:
    static bucket_t *emptyBuckets();
    
    struct bucket_t *buckets();
    mask_t mask();
    mask_t occupied();
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);
    void initializeToEmpty();

    unsigned capacity();
    bool isConstantEmptyCache();
    bool canBeFreed();
}

```

整体来分析，cache_t 根据 `CACHE_MASK_STORAGE` 的值不同，属性也不同，看看 `CACHE_MASK_STORAGE` 的声明是啥。

```c
#define CACHE_MASK_STORAGE_OUTLINED 1
#define CACHE_MASK_STORAGE_HIGH_16 2
#define CACHE_MASK_STORAGE_LOW_4 3

#if defined(__arm64__) && __LP64__
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_HIGH_16
#elif defined(__arm64__) && !__LP64__
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_LOW_4
#else
#define CACHE_MASK_STORAGE CACHE_MASK_STORAGE_OUTLINED
#endif

```

`LP64` 是数据模型中的一种，指的是 long 和 pointer 的字长是 64 位。 因此可以看出：

-  `CACHE_MASK_STORAGE_HIGH_16` 表示 ARM 64 架构、数据模型是 LP64
-  `CACHE_MASK_STORAGE_LOW_4` 表示 ARM 64 架构、数据模型不是 LP64
-  `CACHE_MASK_STORAGE_OUTLINED` 不属于以上两种的

也就是说，在 `CACHE_MASK_STORAGE_OUTLINED` 情况下，是使用 _buckets 和 _mask 两个属性来缓存数据的。在 `CACHE_MASK_STORAGE_HIGH_16` 情况下，是使用 _maskAndBuckets 一个属性来缓存的。在 `CACHE_MASK_STORAGE_LOW_4` 情况下，也是使用 _maskAndBuckets 一个属性来缓存的。其中 `uintptr_t` 是 `unsigned long` 的 typedef。`mask_t` 是 `uint32_t` 的 typedef。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/cache_t_struct.png)

那么 `bucket_t` 的定义是啥呢？

```c
struct bucket_t {
private:
#if __arm64__
    explicit_atomic<uintptr_t> _imp;
    explicit_atomic<SEL> _sel;
#else
    explicit_atomic<SEL> _sel;
    explicit_atomic<uintptr_t> _imp;
#endif

    public:
    inline SEL sel() const { 方法实现 }
    inline IMP imp(Class cls) const { 方法实现 }
    template <Atomicity, IMPEncoding>
    void set(SEL newSel, IMP newImp, Class cls);
    
    // 其他方法
}

```

可以看到 `bucket_t` 有两个关键的属性 `_sel` 和 `_imp`, 和 3 个关键的方法 `sel`, `imp`和 `set` 方法。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/bucket_t.png)

### 方法探索

> iPhone 的 CPU 指令集架构是 ARM 64，所以下面探索的是在 ARM 64 架构上的实现。

在上面的 `cache_t` 中列出了一些核心方法，接下来我们看看这些方法的实现，这些方法在 `objc-cache.m` 文件中可以找到。

当缓存中新增方法时，缓存的容量势必增加。通过方法名可以猜测 `incrementOccupied` 方法的功能是缓存容量增加占用。

全局搜索 `incrementOccupied()` 方法调用，发现只有一处，在 `void cache_t::insert(Class cls, SEL sel, IMP imp, id receiver)` 方法中，也就是说，insert 方法也肯定会调用，打断点，也发现会运行到这里。insert 方法源码如下。

```c
ALWAYS_INLINE
void cache_t::insert(Class cls, SEL sel, IMP imp, id receiver)
{
#if CONFIG_USE_CACHE_LOCK
    cacheUpdateLock.assertLocked();
#else
    runtimeLock.assertLocked();
#endif

    ASSERT(sel != 0 && cls->isInitialized());

    mask_t newOccupied = occupied() + 1;   // 已占用空间大小
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
    if (slowpath(isConstantEmptyCache())) { // 没有初始化缓存空间
        // 申请容量为 4 的内存空间
        if (!capacity) capacity = INIT_CACHE_SIZE;
        reallocate(oldCapacity, capacity, /* freeOld */false);
    }
    else if (fastpath(newOccupied + CACHE_END_MARKER <= capacity / 4 * 3)) {
        // 如果已占用空间大小 + 1 小于或等于总容量的四分之三，不扩容
    }
    else {  // 如果已占用空间大小 + 1 大于总容量的四分之三，扩容为之前总容量的2倍，缓存全清
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;
        if (capacity > MAX_CACHE_SIZE) {
            capacity = MAX_CACHE_SIZE; // 需要申请总容量大于最大容量，则为最大容量
        }
        reallocate(oldCapacity, capacity, true);  // 申请新的内存空间
    }

    bucket_t *b = buckets();   // 获取 buckets
    mask_t m = capacity - 1;   // 计算 mask
    mask_t begin = cache_hash(sel, m);  // 计算 index
    mask_t i = begin;

    // 将 sel 和 imp 放到到空余 index
    do {
        if (fastpath(b[i].sel() == 0)) { // index 中没有方法缓存
            incrementOccupied();         // 增加已占用容量
            b[i].set<Atomic, Encoded>(sel, imp, cls);  // 设置 sel 和 imp
            return;
        }
        if (b[i].sel() == sel) {  // 该方法缓存已存在直接返回
            return;
        }
    } while (fastpath((i = cache_next(i, m)) != begin));  // 如果 index 中有缓存，则计算下一个 index, 直到找到空余 index

    cache_t::bad_cache(receiver, (SEL)sel, cls);
}

```

根据代码中的注释很容易理解 `insert` 的逻辑了，这个方法主要做了三件事情：

- 1、如果缓存空间没有初始化，则申请容量为 4 的缓存空间。
- 2、占用空间+1后，如果占用空间大于或等于总容量的 3/4 , 则扩容为之前容量的 2 倍，否则不扩容。
- 3、将要调用的方法添加到缓存中，如果缓存中存在该方法则直接返回。

上述代码用于生成 index 的 `cache_next` 方法的实现也不是很复杂，就是简单的取前一个 index，如果到第一个元素的下标了，下一个下标将是数组最后一个元素的 index。

```c
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;
}
``` 

接着看看 `reallocate` 方法的实现，在第一次申请缓存空间和扩容会调用这个方法。

```c
ALWAYS_INLINE
void cache_t::reallocate(mask_t oldCapacity, mask_t newCapacity, bool freeOld)
{
    bucket_t *oldBuckets = buckets();  // 获取旧的 buckets
    bucket_t *newBuckets = allocateBuckets(newCapacity);  // 申请新的 buckets
    ASSERT(newCapacity > 0);
    ASSERT((uintptr_t)(mask_t)(newCapacity-1) == newCapacity-1);
    // 设置新的 buckets
    setBucketsAndMask(newBuckets, newCapacity - 1);
    // 释放旧的 buckets
    if (freeOld) {  
        cache_collect_free(oldBuckets, oldCapacity);
    }
}

```

这个函数的主要功能是申请和配置新的 buckets，然后释放旧的 buckets。 `allocateBuckets` 方法是怎么实现的呢？

```c
bucket_t *allocateBuckets(mask_t newCapacity)
{
    if (PrintCaches) recordNewCache(newCapacity);
    return (bucket_t *)calloc(cache_t::bytesForCapacity(newCapacity), 1);
}

```

这个方法首先通过 `bytesForCapacity` 计算需要占用的内存空间大小, 然后调用 `calloc` 开辟内存空间。`bytesForCapacity` 的实现如下。

```c
size_t cache_t::bytesForCapacity(uint32_t cap)
{
    return sizeof(bucket_t) * cap;
}

```

`setBucketsAndMask` 方法是怎么实现的呢？

```c
void cache_t::setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask)
{
    uintptr_t buckets = (uintptr_t)newBuckets;
    uintptr_t mask = (uintptr_t)newMask;
    
    ASSERT(buckets <= bucketsMask);
    ASSERT(mask <= maxMask);
    // (newMask 左移 48 位) | (newBuckets) 运算后就是 _maskAndBuckets 的新值
    _maskAndBuckets.store(((uintptr_t)newMask << maskShift) | (uintptr_t)newBuckets, std::memory_order_relaxed);
    _occupied = 0;  // 占用清零处理
}

```

通过以上的分析，我们可以得到一个方法 ARM64 指令集架构下 cache-insert 的流程图。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/cache_insert.png)

本文我们探索了 cache insert 流程，下一篇文章中，我们将探索 cache 的查找流程。

## 后记

我是穆哥，卖码维生的一朵浪花。我们下回见。 
