---
title: 『底层探索』0 - iOS Responder Chain
toc: true
donate: false
tags: []
date: 2020-09-01 16:10:23
categories: [底层探索]
---

今天开始我们的底层探索旅程，第一篇我们将探索 iOS 的时间响应链，也就是探索事件是怎么产生和传递的？

<!-- more -->

## 概述

首先来说一下事件是什么？事件（event）是一个用于描述我们与 App 的交互的对象。比如，当我们触摸一下 iPhone 的屏幕，就产生了一个触摸事件。我们的 App 可以接收许多不同的事件，比如 `touch events`, `motion events`, `remote-control events` 和 `press events`。其中 touch events (触摸事件) 是最常见的事件，本文探索的就是触摸事件。

在 iOS 中， App 通过 responder 对象来接收和处理事件。在这里，responder 可以称为响应者。一个响应者可以是 `UIResponder` 类的实例，也可以是继承 `UIResponder` 的类的实例，比如 `UIView`, `UIViewController` 和 `UIApplication`。UIKit 是基于 responder 来自动管理事件的。

对于事件（event）来说，UIKit 会指定一个第一响应者（first responder），然后将事件分发给第一响应者。如果事件的类型不同，那么指定的第一响应者也会不同。

UIKit 使用一种 `view-based hit testing` 机制来判断产生事件的 view 是哪一个。说具体些就是在 view 的层次结构中，view 层级结构是一种树状结构，UIKit 会逐个比较触摸位置（touch location）和各个 view 的 bounds，从而找到产生事件的 view。

比较方法是在 view 树结构中，不断调用 view 中的 `hitTest:withEvent:` 方法来寻找包含 touch location 的最深叶子节点。这个叶子节点所对应的 view 将会成为第一响应者。

如果 touch location 在 view bounds 之外，`hitTest:withEvent:` 会忽略这个 view 及其子视图。

举个例子，如果有一个视图 A，它的 `clipsToBounds` 属性是 `false`, 视图 A 中有一个视图 B 的部分视图超出了 A 的 bounds，那么超出的部分即使包含了 touch location，那么它也不会响应事件的。下图中超出 orange view 的 blue view 则不会响应事件。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/blueViewjpg.jpg)

## 探索 UIView

在 iOS 中，我们使用的 view 都是继承于 UIView 的，下面我们来看下 UIView 的类信息。

```swift
open class UIView : UIResponder, 遵循的协议列表 {

    open class var layerClass: AnyClass { get }
    
    public init(frame: CGRect)
    public init?(coder: NSCoder)
    
    // 其他属性和方法
}
```

根据上述代码我们可以知道，UIView 继承于 UIResponder，这也是 view 可以接收和处理事件的原因，接着，我们看下 UIResponder 的类信息。

```swift
open class UIResponder : NSObject, UIResponderStandardEditActions {

    open var next: UIResponder? { get }
    open var canBecomeFirstResponder: Bool { get } // default is NO

    open func becomeFirstResponder() -> Bool
    open var canResignFirstResponder: Bool { get } // default is YES
    open func resignFirstResponder() -> Bool
    open var isFirstResponder: Bool { get }
    
    open func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?)
    open func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?)
    open func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?)
    open func touchesCancelled(_ touches: Set<UITouch>, with event: UIEvent?)
    
    // 其他方法
}
```

通过上述代码，我们可以看到哪些有效信息呢？

- UIResponder 是一个链表节点，有一个 next 属性指向下一个 UIResponder。
- UIResponder 提供了四个 `touch` 方法来处理事件，如果我们有 UIResponder 子类，需要 override 这四个方法来处理自己的事件。

接下来，看看我们常用的 View 中，哪些继承于 UIView，通过查阅代码，我们可以得到如下的一个图。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/UIKit.png)

由图看出，我们用的所有视图都能够接收和处理 event，如果当前 view 不能处理某个 event，就会将 event 通过 next 传递给下一个 View，如果下一个 view 仍然不能处理 event，将会继续传递给下一个 view。也就是说 event 会在响应链上不断传播，直到 event 被处理或丢弃。下面我们通过一个例子详细说明。

## 举例

如图我们创建一个包含 label、textField、button 和 2 个 background view 的 App。那么它的事件传递响应链是这样的。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/responder-demo.png)

- 如果 view 是 ViewController 的 root view，那么它的 next responder 是 ViewController。
- 如果 view 不是 ViewController 的 root view，那么它的 next responder 是它的 superview。
- 如果 ViewController 是 window 的 root view controller，那么它的 next responder 是 window。
- 如果 ViewController 是被另一个 ViewController A present 出来的，那么它的 next responder 是 ViewController A。
- window 的 next responder 是 UIApplication。
- 如果 UIApplication 的 delegate 是 UIResponder 的实例，那么 UIApplication 的 next responder 是 delegate，否则就是 nil。

## 应用

在一些不规则视图中，我们有的时候，想扩大或缩小它的响应热区。那么我们就可以在这个事件响应链上搞点事情。

比如，我们想在不改变 frame 的情况下想扩大 UIButton 的响应热区，也就是点击它的边缘外的视图也可以触发点击事件，我们可以这么做。

```swift
class ExtendButton: UIButton {
    override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        // 要扩大的事件响应范围偏移
        let offset: CGFloat = -10
        let bounds = self.bounds.insetBy(dx: offset, dy: offset)
        return bounds.contains(point)
    }
}
```

弄懂 event 的响应和传递机制后，除此之外，还可以做许多有趣的事情。

## 后记

我是穆哥，卖码维生的一朵浪花。我们下回见。

## 参考

- [0. UIEvent](https://developer.apple.com/documentation/uikit/uievent)
- [1. UIResponder](https://developer.apple.com/documentation/uikit/uiresponder)
- [2. Using Responders and the Responder Chain to Handle Events](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/using_responders_and_the_responder_chain_to_handle_events)
- [3. hitTest(_:with:)](https://developer.apple.com/documentation/uikit/uiview/1622469-hittest?)
- [4. point(inside:with:)](https://developer.apple.com/documentation/uikit/uiview/1622533-point)
- [5. Handling UIKit Gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures)
