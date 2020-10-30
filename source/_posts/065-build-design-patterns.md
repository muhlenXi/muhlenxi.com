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

## The Simple Factory

The Simple Factory isn’t actually a Design Pattern; it’s more of a programming idiom.

**角色**：`SimpleFactory`

**Demo**: [SimpleFactory.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/SimpleFactory.swift)

**核心**：简单工厂实际上就是用一个单独的类来生成不同对象的实例。

简单工厂模式的最大优点在于工厂类中包含了必要的判断逻辑，根据客户端传入的 Type 来实例化对应的对象。对于客户端来说，这么做剥离了与具体产品的依赖。

在用到简单工厂的地方，我们可以考虑用反射技术来消除 switch 或 if，解除分支判断带来的耦合。这样在需求变化的时候，可以减少这个类的修改。

## Factory Method Pattern

The Factory Method Pattern defines an interface for creating an object, but lets subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

工厂方法模式是定义一个用于创建对象的接口，让子类决定实例化哪一个类。工厂方法模式使一个类的实例化延迟到其子类。

**角色**：`Factory` `Concrete Factory`

- `Factory` 是一个接口，接口中声明了生成产品的方法。
- `Concrete Factory` 实现了 `Factory Protocol`, 对于不同的工厂，调用方法后生成的产品也是不同的。

**Demo**: [FactoryMethod.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/FactoryMethod.swift)

**核心**：针对每个要创建的对象都会提供一个工厂类，这个工厂类都实现了创建产品的方法。

工厂方法克服了简单工厂违背 `开放-封闭原则` 的缺点，又保持了封装对象创建过程的优点。

也就是说，当我们想要哪种类型的对象，只要生成对应类型的工厂，然后由该工厂生成对应对象即可。

##  Abstract Factory Pattern

The Abstract Factory Pattern provides an interface for creating families of related or dependent objects without specifying their concrete classes.

**角色**： `AbstractFactory`  `ConcreteFactory` `AbstractProduct` `ConcreteProduct`

- `AbstractFactory` 是一个抽象工厂的接口，接口中定义一组生成抽象产品的方法。
- `AbstractProduct` 是抽象产品，拥有不同产品的抽象特征。
- `ConcreteFactory` 是实现了接口的具体工厂，生成一组具体对象。
- `ConcreteProduct` 是具体的产品，也就是我们业务中定义的不同的类。

**Demo**: [AbstractFactory.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/AbstractFactory.swift)

抽象工厂模式主要是解决涉及多个产品系列的问题，通常是在运行时再创建一个 ConcreteFactory 类的实例，这个具体的工厂再创建具有特定产品实现的产品对象。

也就是说，当我们创建不同的产品对象，客户端应使用不同的具体工厂，当需求发生变化时，直接更改具体工厂就可以了。

## Prototype Pattern

Prototype Pattern: Specify the kinds of objects to create using a prototypical instance, and create new objects by coping this prototype.

**角色**：`Client` `Prototype` `ConcretePrototype`

**Demo**：[Prototype.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Prototype.swift)

**核心**：通过 `复制` 一个已经存在的实例来返回新的实例，而不是新建实例。

被复制的实例就是我们说的原型，这个原型是可以定制的。

案列： copy、mutableCopy

## Builder Pattern

Builder Pattern: 将一个复杂对象的构建和它的表示分离，使得同样的构建过程可以创建不同的表示。

**角色**： `Director` `Builder` `ConcreteBuilder` `Product`

- `Product` 是最后要生成的产品。
- `Builder` 是一个构造的接口，定义了一组构造 `Product` 的过程方法。
- `ConcreteBuilder` 是具体的构造器，实现了构造接口中的构造方法。
- `Director` 是指挥者，用于生成具体的构造器。

**Demo**：[Builder.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Builder.swift)

建造者模式的好处就是使得建造代码与表示代码分离，由于建造者隐藏了该产品是如何组装的，所有若需要改变一个产品的内部表示，只需要再定义一个具体的建造者就可以了。

案例：麦当劳 肯德基

## Singleton Pattern

The Singleton Pattern ensures a class has only one instance, and provides a global point of access to it.

**Demo**：[Singleton.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Singleton.swift)

**核心**：单例模式确保了一个类的实例有且仅有一个，并且提供了一个全局访问入口。

单例模式封装了它的唯一实例，这样它可以严格地控制客户怎样访问它以及何时访问它。简单地说就是对唯一实例的受控访问。
