---
title: 『Swift 进阶』1 - Swift 源码编译
toc: true
donate: false
tags: []
date: 2020-12-11 16:14:22
categories: [Swift 进阶]
---

本文主要是编译 Apple Swift 开源代码的相关内容。

<!-- more -->

如果想让自己的编码技术更进一步，多看看比你优秀的代码是比较简单粗暴的方式。从这一篇文章开始，我们开始探索 Swift 这一门语言，探索方式主要如下：

- 查看 Swift 开源代码逻辑。
- 查看 Swift 编译后的中间代码 SIL（Swift Intermediate Language） 的逻辑。

## 环境准备

### 参考

写这个博文的时候，作者本次编译时的环境是这样的，仅供参考。

- macOS Catalina 10.15.7
- Xcode Version 12.2 (12B45b)
- Visual Studio Code Version: 1.51.1
- cmake version 3.19.1
- Python 3.8.0
- ninja 1.10.2
- sccache 0.2.13

### 检查

本次编译，我们会用到 `cmake`, `python3`, `ninja`, `sccache`。可以使用如下终端命令检测是否安装：

```shell
cmake --version # This should be 3.18.1 or higher for macOS
python3 --version
ninja --version
sccache --version
```

如果以上的工具没有安装，可以使用 `Homebrew` 安装。

```shell
brew install cmake ninja sccache
```

## 编译步骤

我们本次编译的是 `Swift 5.3.1 Release` 版本的源码。

### 下载源码

1、创建 `swift-project` 目录，进入该目录下。

```shell
mkdir -p swift-project/swift
cd swift-project/swift
```

2、clone 源码。

```shell
git clone --branch swift-5.3.1-RELEASE https://github.com/apple/swift.git
```

3、更新 Swift 相关的库，不然会导致后面编译失败。确保在 `swift-project/swift` 目录下，执行如下命令。

```shell
./swift/utils/update-checkout --tag swift-5.3.1-RELEASE --clone
```

4、开始编译源码。确保在 `swift-project/swift` 目录下，执行如下命令。

```shell
./swift/utils/build-script -r --debug-swift-stdlib --lldb
```

## 调试

### VSCode

####  安装依赖 CodeLLDB

我们后面会使用 `Visual Studio Code` 来调试 `Swift`。所以需要装个 `CodeLLDB` 插件来方便调试。

打开 `VSCode` 后，`Command + shift + x` 切换到 `Extensions` 后，在搜索框中搜索 `CodeLLDB` 后点击 `install` 即可。

#### 开始调试

`VSCode` open 我们的 `swift-project` 目录后，需要配置编译 `launch.json`。配置如下，要注意路径一致。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug",
            "program": "${workspaceFolder}/swift/build/Ninja-RelWithDebInfoAssert+stdlib-DebugAssert/swift-macosx-x86_64/bin/swift",
            "args": [],
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

`Command + shift + D` 切换到 `Run` 后，点击左上秒的 `Debug` 按钮即可。

## 参考资料

- [apple/swift](https://github.com/apple/swift)
- [How to Set Up an Edit-Build-Test-Debug Loop](https://github.com/apple/swift/blob/main/docs/HowToGuides/GettingStarted.md)
- [Swift Intermediate Language (SIL)](https://github.com/apple/swift/blob/main/docs/SIL.rst)