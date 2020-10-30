---
title: SOLID 设计原则
toc: true
donate: false
tags: []
date: 2020-01-05 22:51:54
categories: 编程思想
---

在工作中，碰到令人赏心悦目的代码无疑是一件开心的事情，让人更开心的就是能够知道这些精彩的代码是如何想出来的。今天聊聊软件设计中一些经典的原则。

<!-- more -->

设计原则指的是设计软件架构中需要遵守的原则，遵循这些原则可以避免一些糟糕的设计。

## Single Responsibility Principle 单一职责原则


> A class should have one, and only one, reason to change.

如果一个类承担的职责过多，就等于把这些职责耦合在一起，一个职责的变化可能会削弱或者抑制这个类完成其他职责的能力。这种耦合会导致脆弱的设计，当变化发生时，设计会遭受到意想不到的破坏。

软件设计真正要做的许多内容，就是发现职责并把那些职责相互分离。

## Open Close Principle 开闭原则

> Software entities(classes, modules, functions, etc) should be open for extension, but closed for modification. 

面对需求，对程序的改动是通过增加新代码进行的，而不是更改现有的代码。

怎样的设计才能面对需求的改变却可以保持相对稳定，从而使得系统可以在第一个版本以后不断推出新的版本呢？

在我们最初编写代码时，假设变化不会发生。当变化发生时，我们就创建抽象来隔离以后发生的同类变化。

## Liskov Substitution Principle 里氏替换原则

> Derived types must be completely substitutable for their base types.

里氏代换原则：子类型必须能够替换掉它们的父类型。

## Interface Segregation Principle  接口分离原则

> Clients should not be forced to depend upon interfaces that they do not use.


## Dependency Inversion Principle 依赖倒置原则

> High-level modules should not depend on low-level modules. Both should depend on abstractions.
> Abstractions should not depend on details. Details should depend on abstractions.

高层模块不应该依赖低层模块。两个都应该依赖抽象。

抽象不应该依赖细节。细节应该依赖抽象。

## Other

### DRY

> Don't repeat yourself.

### KISS

> Keep it simple stupid.

## 参考资料

- [SOLID](https://www.oodesign.com/design-principles.html)





