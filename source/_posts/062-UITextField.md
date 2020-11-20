---
title: UITextField 粘贴指定字符
toc: false
donate: false
tags: [UITextField]
date: 2019-04-04 21:21:27
categories: iOS
---

UITextField 有的时候需要粘贴前 n 个字符，那么怎么做呢？

<!-- more -->

指定粘贴前前 n 个字符，比如总共允许输入 20 个字符，现在输入框中有 15 个字符，现在待粘贴的文本有 8 个字符，则取前 5 个字符，剩下的丢弃。

## UITextField 添加扩展方法

```swift
public extension UITextField {
    /// 粘贴指定前n个字符
    ///
    /// - Parameters:
    ///   - range: 替换区间
    ///   - pastedString: 待粘贴的字符串
    ///   - prefixNumber: 最多粘贴的长度
    func pastedPrefixNumberText(range: NSRange, pastedString: String, prefixNumber: Int) {
        guard prefixNumber >= 0, prefixNumber <= pastedString.count else {
            return
        }
        let oldText = self.text ?? ""
        var cursorPositionIndex: Int = prefixNumber
        let newText = (oldText as NSString).replacingCharacters(in: range, with: String(pastedString.prefix(prefixNumber)))
        if let selectedRange = self.selectedTextRange {
            let oldCursorPositionIndex = self.offset(from: self.beginningOfDocument, to: selectedRange.start)
            cursorPositionIndex += oldCursorPositionIndex
        }
        self.text = newText
        if let newPosition = self.position(from: self.beginningOfDocument, in: UITextLayoutDirection.right, offset: cursorPositionIndex) {
            DispatchQueue.main.async {
                self.selectedTextRange = self.textRange(from: newPosition, to: newPosition)
            }
        }
    }
}
```