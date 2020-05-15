---
title: 聊聊 构建型的设计模式
toc: true
donate: false
tags: []
date: 2020-01-10 23:15:50
categories: 编程思想
---

构建型的设计模式主要有 6 个，包含 `简单工厂模式` `工厂方法模式` `抽象工厂模式` `原型模式` `建造者模式` `单例模式`

<!-- more -->

### The Simple Factory

The Simple Factory isn’t actually a Design Pattern; it’s more of a programming idiom.

[SimpleFactory.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/SimpleFactory.swift)

简单工厂实际上就是用一个单独的类来生成不同对象的实例。

简单工厂模式的最大优点在于工厂类中包含了必要的判断逻辑，根据客户端传入的 Type 动态实例化相关的类。对于客户端来说，去除了与具体产品的依赖。

所有在用简单工厂的地方，都可以考虑用反射技术来去除 switch 或 if，解除分支判断带来的耦合。

## Factory Method Pattern

The Factory Method Pattern defines an interface for creating an object, but lets subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

[FactoryMethod.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/FactoryMethod.swift)

工厂方法模式是定义一个用于创建对象的接口，让子类决定实例化哪一个类，工厂方法模式使一个类的实例化延迟到其子类。

工厂方法克服了简单工厂违背 开放-封闭 原则的缺点，又保持了封装对象创建过程的优点。

##  Abstract Factory Pattern

The Abstract Factory Pattern provides an interface for creating families of related or dependent objects without specifying their concrete classes.

角色： `AbstractFactory`  `ConcreteFactory` `AbstractProduct Product`

[AbstractFactory.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/AbstractFactory.swift)

抽象工厂模式主要是解决涉及多个产品系列的问题，通常实在运行时再创建一个 ConcreteFactory 类的实例，这个具体的工厂再创建具有特定产品实现的产品对象。也就是说，为创建不同的产品对象，客户端应使用不同的工厂。

## Prototype Pattern

Prototype Pattern: 用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的的对象。

[Prototype.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Prototype.swift)

案列： copy 方法

## Builder Pattern

Builder Pattern: 将一个复杂对象的构建和它的表示分离，使得同样的构建过程可以创建不同的表示。

结构角色： `Director` `Builder` `ConcreteBuilder` `Product`

[Builder.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Builder.swift)

建造者模式是在当创建复杂对象的算法应该独立于该对象的组成部分以及它们的装配方式时适用的模式。

建造者模式的好处就是使得建造代码与表示代码分离，由于建造者隐藏了该产品是如何组装的，所有若需要改变一个产品的内部表示，只需要再定义一个具体的建造者就可以了。

案例：麦当劳 肯德基

## Singleton Pattern

The Singleton Pattern ensures a class has only one instance, and provides a global point of access to it.

[Singleton.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Singleton.swift)

单例模式封装了它的唯一实例，这样它可以严格地控制客户怎样访问它以及何时访问它。简单地说就是对唯一实例的受控访问。
