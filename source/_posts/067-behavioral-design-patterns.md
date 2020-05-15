---
title: 聊聊 行为型的设计模式
toc: true
donate: false
tags: []
date: 2020-01-20 23:18:17
categories: 编程思想
---

行为型的设计模式有 10 个，包括 `观察者模式` `模板方法模式` `命令模式` `状态模式` `职责链模式` `解释器模式` `中介者模式` `访问者模式` `策略模式` `备忘录模式` `迭代器模式` 

<!-- more -->

###  Observer Pattern

The Observer Pattern defines a one-to-many dependency between objects so that when one object changes state, all of its dependents are notified and updated automatically.

[Observer.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Observer.swift)

将一个系统分割成一系列相互协作的类有一个不好的副作用，那就是要维护相关对象间的一致性。我们不希望为了维护一致性而使各类紧密耦合，这样会给维护、扩展和重用都带来不便。

观察者模式所做的工作就是在解除耦合。让耦合的双方都依赖于抽象，而不是依赖于具体。从而使得各自的变化都不会影响另一边的变化。

案例：通知中心 报纸

## Template Method Pattern

The Template Method Pattern defines the skeleton of an algorithm in a method, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm’s structure.

[TemplateMethod.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/TemplateMethod.swift)

模板方法模式是通过把不变行为搬移到超类，去除子类中的重复代码来体现它的优势。

## Command Pattern

The Command Pattern encapsulates a request as an object, thereby letting you parameterize other objects with different requests, queue or log requests, and support undoable operations.

角色: `Command` `Receiver` `Invoker`

[Command.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Command.swift)

命令模式把请求一个操作的对象与知道怎么执行一个操作的对象分隔开。

如果在某个系统中使用命令模式时，需要实现命令的撤销功能，可以使用备忘录模式来存储可撤销操作的状态。

## State Pattern

1The State Pattern allows an object to alter its behavior when its internal state changes. The object will appear to change its class.

[State.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/State.swift)

状态模式将特定的状态相关的行为都放入一个对象中，由于所有与状态相关的代码都存在于某个 ConcreteState 中， 所以通过定义新的子类可以很容易地增加新的状态和转换。

状态模式通过把各种状态转移逻辑分布到 State 的子类之间，来减少相互间的依赖。

当一个对象的行为取决于它的状态，并且它必须在运行时刻根据状态改变它的行为时，就可以考虑使用状态模式了。

## Chain of Responsibility Pattern

Chain of Responsibility Pattern：使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这个对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。

角色： `Handler` `ConcreteHandler`

[ChainOfResponsibility.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/ChainOfResponsibility.swift)

当客户提交一个请求时，请求是沿链传递直到有一个 ConcreteHandler 对象处理它。

客户端可以随时增加或修改处理一个请求的结构，增强了给对象指派职责的灵活性。


##  Interpreter Pattern

Interpreter Pattern：给定一个语言，定义它的文法的一种表示，并定义一个解释器，这个解释器使用该表示来解释语言中的句子。

[Interpreter.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Interpreter.swift)

通常当有一个语言需要解释执行，并且你可将该语言中的句子表示为一个抽象语法树时，可使用解释器模式。

案例：正则表达式

## Mediator Pattern

Mediator Pattern：用一个中介对象来封装一系列的对象交互。中介者使各个对象不需要显示地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。

角色： `Mediator` `Colleague`

[Mediator.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Mediator.swift)

## Visitor Pattern

Visitor Pattern：表示一个作用于某对象结构中的各元素的操作。它使你可以在不改变各元素类的前提下定义作用于这些元素的新操作。

角色： `ObjectStructure` `Element` `Visitor`

[Visitor.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Visitor.swift)

访问者模式适用于数据结构相对稳定的系统，它把数据结构和作用于数据结构上的操作之间的耦合解脱开，使得操作集合可以相对自由地演化。

## Strategy Pattern

The Strategy Pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

角色：

[Strategy](https://github.com/muhlenXi/design-patterns/blob/master/demos/Strategy.swift)

哪些情况的代码包含策略模式的特征？

- 遵循了相同协议的不同子类


## Command Pattern

The Command Pattern encapsulates a request as an object, thereby letting you parameterize other objects with different requests, queue or log requests, and support undoable operations.

角色: `Command` `Receiver` `Invoker`

[Command.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Command.swift)

命令模式把请求一个操作的对象与知道怎么执行一个操作的对象分隔开。

如果在某个系统中使用命令模式时，需要实现命令的撤销功能，可以使用备忘录模式来存储可撤销操作的状态。

##  Iterator Pattern

The Iterator Pattern provides a way to access the elements of an aggregate object sequentially without exposing its underlying representation.

[Iterator.swift](https://github.com/muhlenXi/design-patterns/blob/master/demos/Iterator.swift)

迭代器模式分离了集合对象的遍历行为，抽象出一个迭代器类来负责，这样既可以不暴露集合的内部结构，又可让外部代码透明地访问集合内部的数据。

案例：for-in
