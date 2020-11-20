---
title: Swift 访问控制权限
toc: false
donate: false
tags: [Swift]
date: 2019-05-14 20:26:21
categories: Swift
---

 Swift 的访问权限是怎么控制的？

<!-- more -->

## 访问控制权限介绍

### open

用 **open** 修饰的类、属性和方法可以被任何人（包括其他 module）使用，其他 module 可以继承和 override。

```swift
open class Person {
    open var name: String = "unknow"
    
    open func sayHello() {
        print("Hello")
    }
}
```

### public

用 **public** 修饰的类、属性和方法可以被任何人（包括其他 module）使用，但是其他 module 不能继承和 override。

```swift
public class Person {
    public var name: String = "unknow"
    
    public func sayHello() {
        print("Hello")
    }
}
```

### internal

用 **internal（默认访问级别）** 修饰的类、属性和方法只能在当前 module 内使用。

```swift
internal class Person {
    internal var name: String = "unknow"
    
    internal func sayHello() {
        print("Hello")
    }
}
```

### fileprivate

用 **fileprivate** 修饰的类、属性和方法只能在当前 swift 源文件中访问，其他 module 无法访问和继承。

```swift
fileprivate class Person {
    fileprivate var name: String = "unknow"
    
    fileprivate func sayHello() {
        print("Hello")
    }
}
```

### private

用 **private** 修饰的类、属性和方法只能在当前类访问。其他 module 无法访问和继承

```swift
private class Person {
    private var name: String = "unknow"
    
    private func sayHello() {
        print("Hello")
    }
}
```

### final

用 **final** 修饰的类不能被其他类继承。

```swift
final class Dog {
    var name: String
    var breed: String
    
    init(name: String, breed: String) {
        self.name = name
        self.breed = breed
    }
}
```

## 访问控制权限排序

因此访问权限由大及小依次为：

> open > public > internal > fileprivate > private 
> final