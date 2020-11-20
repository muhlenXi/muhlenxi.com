---
title: 两种创建 Swift 单例 的方式
toc: false
donate: false
tags: [Swift]
date: 2019-05-12 20:43:49
categories: Swift
---

项目中通常通过一个单例来访问共享的资源，那么如何定义一个 Swift 单例呢？

<!-- more -->

## 定义一个单例类

1、无需初始化单例属性的单例类

```swift
class Singleton {
    static let sharedInstance = Singleton()
}
```

2、需要初始化单力属性的单例类

```swift
class Singleton {
    static let sharedInstance: Singleton = {
        let instance = Singleton()
        instance.setup()
        return instance
    }()
    
    private func setup() {
        // 初始化单例内的属性
    }
}
```


## 参考来源

- [Managing a Shared Resource Using a Singleton](https://developer.apple.com/documentation/swift/cocoa_design_patterns/managing_a_shared_resource_using_a_singleton)