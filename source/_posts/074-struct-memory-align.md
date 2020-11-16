---
title: 『底层探索』2 - struct 内存对齐
toc: true
donate: false
tags: []
date: 2020-09-08 14:50:53
categories: [底层探索]
---

当我们定义一个 struct 的时候，它在内存中是怎么存储的？占用了多少字节的内存空间呢？这就是我们今天要探索的问题。

<!-- more -->

计算机中数据处理的最小单位是 Byte （字节），8 个二进制位称为一个字节。比如 `255` 用二进制表示为 `11111111`,用十六进制表示为 `0xFF`。

## 基本数据类型

首先我们看下基本数据类型的内存占用大小。本文使用的是 64 bit 的操作系统。

```c
char a;
short b;
int c;
long d;
float e;
double f;

printf("char   -> %lu \n", sizeof(a));
printf("short  -> %lu \n", sizeof(b));
printf("int    -> %lu \n", sizeof(c));
printf("long   -> %lu \n", sizeof(d));
printf("float  -> %lu \n", sizeof(e));
printf("double -> %lu \n", sizeof(f));
```

### sizeof

The sizeof operator yields an integer equal to the size of the specified object or type in bytes. 

`sizeof` 是一个关键字，是一个编译时运算符，用于判断变量或数据类型的字节大小。

运行上面的代码，我们可以得到基本数据类型的内存占用字节大小。

```c
char   -> 1 
short  -> 2 
int    -> 4 
float  -> 4 
long   -> 8 
double -> 8 
```

## struct

```c
struct A {
    int a; 
    char b; 
    short c; 
    long d; 
    char e; 
};

struct B {
    char b; 
    int a; 
    short c; 
    long d; 
    char e; 
};
```

上面我们定义了结构体 A 和 B，我们用 `sizeof` 来分别计算它们的内存占用字节大小，你猜猜会是什么呢？

```c
struct A sa;
struct B sb;
    
printf("A size -> %lu \n", sizeof(sa));
printf("b size -> %lu \n", sizeof(sb));
```

你可能会想，A 和 B 都包含 int， char， short， long 四种类型。根据结构体中的元素所属的类型依次进行求和，4 + 1 + 2 + 8 + 1 = 16，结果就是都打印 16 了。是这样的么？我们看看实际打印结果。

```c
A size -> 24 
b size -> 32
```
是不是有些意外？为什么呢？为了提高 CPU 的数据访问效率，系统对 struct 在内存中的存储做了内存对齐处理操作。更多对齐知识可以 google 一下。

### 原则一

接下来我们看看这个 struct 内存占用大小是如何计算出来的。这就引出了内存对齐的第一个原则。

> 原则一：结构体中的元素是按照定义的顺序一个一个放到内存中去的，不是我们想的紧密排列的那样。从结构体存储的首地址开始，每个元素放到内存中时，都是以当前元素的大小来划分的。因此一定是放到当前元素大小的整数倍上的地址上的。（以结构体变量首地址为 0 开始计算）

```c
struct A {
    int a;      // [0123]
    char b;     // [4]
    short c;    // 5[67]
    long d;     // [8,9,10,11,12,13,14,15]
    char e;     // [16]
};
```

对于 struct A:

- 元素 a 占用 4 个字节，刚好放到地址 0-3 中.
- 元素 b 占用 1 个字节，此时地址 4，刚好是 1 的整数倍，所以 b 放在地址 4 中。
- 元素 c 占用 2 个字节，此时地址 5，不是 2 的整数倍，跳过，到地址6，6 刚好是 2 的整数倍。所以 c 放到地址 6-7 中。
- 元素 d 占用 8 个字节，此时地址 8，刚好是 8 的整数倍，所以 d 放到地址 8-15 中。
- 元素 e 占用 1 个字节，放到 16 中。

根据原则一，所以 struct A 总共占用 0-16，总共 17 个字节。

```c
struct B {
    char b;    // [0]
    int a;     // 123[4567]
    short c;   // [89]
    long d;    // 10,11,12,13,14,15 [16,17,18,19,20,21,22,23]
    char e;    // [24]
};
```

同理，对于 struct B：

- 元素 b 占用 1 个字节，放到地址 0 中。
- 元素 a 占用 4 个字节，此时地址是 1，不满足条件，依次跳过。直到地址 4，所有放到地址 4-7 中。
- 元素 c 占用 2 个字节，此时地址 8，刚好是 2 的整数倍，所以 c 放在地址 8-9 中。
- 元素 d 占用 8个字节. 此时地址 10，不是 8 的整数倍，跳过，直到地址16，16 刚好是 8 的整数倍。所以 d 放到地址 16-23 中。
- 元素 e 放到地址 24 中。

根据原则一，所以 struct B 总共占用 0-24，总共 25 个字节。

打印一下结果，验证一下前面逻辑的分析。

```c
printf("a -> %lu, b -> %lu, c -> %lu, d -> %lu, e -> %lu \n",
               (void*)&sa.a - (void*)&sa,
               (void*)&sa.b - (void*)&sa,
               (void*)&sa.c - (void*)&sa,
               (void*)&sa.d - (void*)&sa,
               (void*)&sa.e - (void*)&sa);
    
printf("b -> %lu, a -> %lu, c -> %lu, d -> %lu, e -> %lu \n",
			(void*)&sb.b - (void*)&sb,
			(void*)&sb.a - (void*)&sb,
			(void*)&sb.c - (void*)&sb,
			(void*)&sb.d - (void*)&sb,
			(void*)&sb.e - (void*)&sb);
```

打印结果如下，符合我们原则一的分析。

```c
a -> 0, b -> 4, c -> 6, d -> 8, e -> 16 
b -> 0, a -> 4, c -> 8, d -> 16, e -> 24 
```

### 原则二

根据原则一得到的结果是 17 和 25，不是我们前面打印的 24 和 32，所以又引出了内存对齐的第二个原则。

> 原则二:  判断经过原则一计算出的总长度 m，是否是 struct 中最宽元素的长度 n 的整数倍。满足条件则结束，不满足，则要补齐为最宽长度 n 的整数倍。

struct A 和 struct B 中的最宽元素都是 d，长度是 8。

struct A 根据原则一计算的是 17 ，不满足原则二，要补全对齐到 24。
struct B 根据原则一计算的是 25，不满足原则二，要补全对齐到 32。

### struct 嵌套

如果一个 struct 中包含另一个 struct 呢？这时候的内存是怎么对齐的呢？

```c
struct RDPoint {
    long x;    // [0...7]
    char y;    // [9]
};  // 16

struct RDPoint1 {
    int x;     // [0...3]
    char y;    // [4]
};  // 8

struct A {
    char a;                    // 0
    struct RDPoint point;      // [8...23]
    struct RDPoint1 point1;    // [24...31]
    char b;                    // [32]
};  // 40
```

猜猜下面的代码会输出什么？

```c
struct A sa;
struct RDPoint point;
struct RDPoint1 point1;
    
printf("RDPoint size -> %lu \n", sizeof(point));
printf("RDPoint1 size -> %lu \n", sizeof(point1));
printf("A size -> %lu \n", sizeof(sa));
    
printf("a -> %lu, point -> %lu, point1 -> %lu, b -> %lu  \n",
           (void*)&sa.a - (void*)&sa,
           (void*)&sa.point - (void*)&sa,
           (void*)&sa.point1 - (void*)&sa,
           (void*)&sa.b - (void*)&sa);
```

### 原则三

这就引出了内存对齐的第三条原则。

> 原则3：如果 struct A 中含包含 struct B 元素，则 struct B 的起始地址必须是 struct B 中的最宽元素的整数倍。

根据原则一和二，我们得出 struct RDPoint 占用内存 16 个字节。struct RDPoint1 占用内存 8 个字节，对于 struct A ：

- 元素 a 占用 1 个字节，放到地址 0 中。
- 元素 point 占用 16 个字节，最宽元素的长度是 8 个字节，此时地址是 1，不满足原则三，依次跳过。直到地址 8，所以放到地址 8-23 中。
- 元素 point1 占用 8 个字节，最宽元素的长度是 4 个字节，此时地址是24，满足原则三，所以放到地址 24-31 中。
- 元素 b 占用一个字节，放到地址 32 中。

所以根据原则一、二、三 struct A 占用内存大小应该是 40 个字节。

打印下实际运行结果，看是否符合我们的分析。

```c
RDPoint size -> 16 
RDPoint1 size -> 8 
A size -> 40 
a -> 0, point -> 8, point1 -> 24, b -> 32 
```

## 总结

struct 内存对齐主要遵循三个原则：

- 1、struct 中每个元素的存放地址必须是当前元素类型大小的整数倍。
- 2、struct 的总大小必须是其中最宽元素的整数倍。
- 3、如果 struct 中有嵌套 struct，嵌套的 struct 的起始地址必须是它的最宽元素大小的整数倍。

如果你能弄懂了这三个原则，那么 struct 的内存占用大小问题应该就一清二楚了。

## 后记

我是穆哥，卖码维生的一朵浪花。我们下期见。