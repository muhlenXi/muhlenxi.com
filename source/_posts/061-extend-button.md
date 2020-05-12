---
title: 如何扩大 UIButton 的响应区域
toc: false
donate: false
tags: [UIButton]
date: 2019-054-14 21:14:41
categories: [Swift]
---

在不改变 UIButton frame 的情况下如何扩大它的响应区域?

<!-- more -->

## 方法

override point 方法

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