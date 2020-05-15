---
title: 聊聊 结构型的设计模式
toc: true
donate: false
tags: []
date: 2020-01-15 23:17:30
categories: 编程思想
---

结构型的设计模式有 7 个，包含 `外观模式` `装饰模式` `适配器模式` `桥接模式` `组合模式`  `享元模式` `代理模式`

<!-- more -->

## Facade Pattern

The Facade Pattern provides a unified interface to a set of interfaces in a subsystem. Facade defines a higherlevel interface that makes the subsystem easier to use.

[Facade.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Facade.swift)

外观模式为子系统的方法提供了一个统一的接口。这个接口使子系统更加容易使用。

案例：中间层 图形用户界面

## Decorator Pattern

The Decorator Pattern attaches additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

[Decorator.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Decorator.swift)

装饰模式是为已有功能动态地添加更多功能的一种方式。

装饰模式是利用 SetComponent 来对对象进行包装的。这样每个装饰对象的实现就和如何使用这个对象分离开了，每个装饰对象只关心自己的功能，不需要关心如何被添加到对象链中。

## Adapter Pattern

The Adapter Pattern converts the interface of a class into another interface the clients expect. Adapter lets classes work together that couldn’t otherwise because of incompatible interfaces.

[Adapter.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Adapter.swift)

适配器模式主要用于希望复用一些现有的类，但是接口又与复用环境要求不一致的情况。

案例：电源适配器

## Bridge Pattern

Bridge Pattern: 将抽象部分与它的实现部分分离，使它们都可以独立地变化。

[Bridge.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Bridge.swift)

实现系统可能有多角度分类，每一种分类都有可能变化，那么就把这种多角度分离出来让它们独立变化，减少它们之间的耦合。

桥接模式是合成/聚合规则的应用。

## Composite Pattern

The Composite Pattern allows you to compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

[Composite.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Composite.swift)

组合模式让客户可以一致地使用组合结构和单个对象。

使用场景：当需求中是体现部分与整体层次的结构时，以及你希望用户可以忽略组合对象和单个对象的不同，统一地使用组合结构中的所有对象时，就应该考虑用组合模式了。

案例：文件夹和文件

##  Flyweight Pattern

Flyweight Pattern：运用共享技术有效地支持大量细颗粒度的对象。

角色： `FlyweightFactory` `Flyweight` `ConcreteFlyweight`

[Flyweight.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Flyweight.swift)

使用条件：

- 如果一个应用程序使用了大量的对象，而大量的这些对象造成了很大的存储开销时。
- 对象的大多数状态可以外部状态，如果删除对象的外部状态，那么可以用相对较少的共享对象取代很多组对象。

## Proxy Pattern

The Proxy Pattern provides a surrogate or placeholder for another object to control access to it.

[Proxy.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Proxy.swift)

代理模式为其他对象提供一种代理以控制对这个对象的访问。

代理模式特点：Proxy 和 RealSubject 实现了一套通用 Interface。Proxy 保存了 RealSubject 的引用，当代理方法被调用时，代理则调用 Subject 的相同方法。

案例： 远程代理、虚拟代理、安全代理等
