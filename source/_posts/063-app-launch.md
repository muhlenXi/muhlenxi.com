---
title: iOS App 启动流程和优化策略
toc: true
donate: false
tags: []
date: 2020-05-15 22:22:23
categories: iOS
---

当我们拿起手机, 从主页点击 App icon 开始刷刷刷的时候,肯定有一小部分人对这 App 的启动过程产生好奇的, 今天我们就来聊聊 App 是怎么启动的，在开始之前，我们要了解一些预备知识。

<!-- more -->

## 预备知识

在探究 App 的启动流程前，先了解一下本文提到的基本概念

### Mach-O

`Mach-O`-  是 Mach Object 的缩写，是 mac os 和 iOS 中可执行文件、动态库的文件格式。主要包括以下几种文件：

- `Executable` - Main binary for application
- `Dylib`— Dynamic library
- `Bundle` - Dylib that cannot be linked, only dlopen(), e.g. plug-ins
- `Image` - 上面提到的三者都可以称为 Image
- `Framework` - Dylib with directory for resources and headers, 也就是携带 header 和资源文件的 Dylib

每个 Mach-O Image 文件分为 `__TEXT`,`__DATA`,`__LINKEDIT` 这三段，每段的大小是内存 page 的整数倍。

- `__TEXT` has header, code, and
read-only constants
- `__DATA` has all read-write content:
globals, static variables, etc
- `__LINKEDIT` has "meta data” about
how to load the program

### Virtual Memory

Virtual Memory 是一个中间层，主要功能是用于将每个进程的地址映射到实际物理内存地址。更多信息可以访问 [Wiki 虚拟内存](https://zh.wikipedia.org/zh-hans/%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98)。

## 启动类型

启动类型根据在内存中的状态可以分为 冷启动 和 热启动。

## 启动流程

App 启动流程可以分为三个阶段，第一阶段是 pre-main, 第二阶段是 main，第三阶段是 main 之后。

在 pre-main 阶段，Kernel 会调用 Dyld 依次执行以下流程：

- Map all dependent dylibs, recurse
- Rebase all images
- Bind all images
- ObjC prepare images
- Run initializers

### Load dylibs

在 `Load dylibs` 流程，App 依赖的所有 Dylib 会一次被加载在内存中，为了安全起见，Dylib 加载方式是采用 ASLR 的方式，也就是 Dylib 在内存中的加载地址是随机的。在此过程中主要做了以下几件事情：

- 解析 Dylib 的依赖列表
- 读取 Dylib 的开始文件
- 校验 mach-o
- 注册代码签名
- 调用 mmap() 函数映射每个段的内存地址

### Rebase

由于 Dylib 在内存中地址是随机的，因此要对 Dylib 中 `__DATA` 的指针进行修复，也就是 rebasing。说的通俗点就是对其中的所有指针加上 slide value （新旧地址的差值）。

```
Slide = actual_address - preferred_address
```

### Bind

Dylib 中会有调用其他 Dylib 的代码，Dylib 对其他 dylib 的引用是符号化的，dylib 需要寻找对应的 symbol name，这就是所谓的 binding 流程， binding 比 rebasing 的计算要多些。

### Objc

在 rebase 和 bind 的流程中，大部分的 Objc 类已经配置完了，所有的类已经注册了。在 Objc 阶段，做的事情主要有：

- 将 Category 中的方法插入到 method list 中
- 更新 Non-fragile ivars offsets
- 校验 Selector 的唯一性

### Initialization

`Initializer` 流程中主要进行如下的初始化：

`libSystem Init`：

- initializes the interfaces with low level system components

`Runtime Init`:

- initializes the language runtime
- Invokes all class static load methods

`UIkit init`:

- Instantiates the UIApplication and UIApplicationDelegate
- Begins event processing and integration with the system

`Application init`:

- Invokes UIApplicationDelegate app lifecycle callbacks
- Invokes UIApplicationDelegate UI lifecycle callbacks
- Invokes UISceneDelegate UI lifecycle callbacks for each scene (iOS 13)

初始化流程结束后，就开始了首页面的渲染，当用户看到首页面后，启动流程就结束了。

## 启动优化策略

针对以上启动流程，针对不同的阶段，有不同的优化策略。如下所示

1. Avoid linking unused frameworks
2. Avoid dynamic library loading during launch
3. Hard link all your dependencies
4. Expose dedicated initialization API in frameworks
5. Reduce impact to launch by avoiding `+[Class load]`
6. Use `+[Class initialize]` to lazily conduct static initialization
7. Minimize work in UIApplication subclass
8. Minimize work in UIApplicationDelegate initialization
9. Defer unrelated work
10. Share resources between scenes
11. Flatten view hierarchies and lazily load views
12. Optimize auto layout usage
13. Leverage os_signpost to measure work

如果有我理解不到位的地方，希望各位大佬不吝赐教。

## 参考资料

- [WWDC 2016 406](https://developer.apple.com/videos/play/wwdc2016/406/)
- [WWDC 2019 423](https://developer.apple.com/videos/play/wwdc2019/423/)
