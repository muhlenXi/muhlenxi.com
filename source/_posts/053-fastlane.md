---
title: fastlane 的使用
toc: false
donate: false
tags: [fastlane]
date: 2019-11-12 19:33:49
categories: 工具集
---


fastlane 是自动部署和发布 iOS 和 Android  App 的一个工具，用于处理一些单调乏味的工作，比如生成应用截图、代码签名和应用发布等。更多信息请访问官网 [fastlane](https://fastlane.tools/)。

<!-- more -->

## 如何安装

安装 fastlane

```shell
sudo gem install -n /usr/local/bin fastlane --verbose
```

查看 fastlane 版本号

```shell
fastlane -v
```

安装蒲公英的 Fastlane 插件

```shell
fastlane add_plugin pgyer
```

## 如何使用

match 证书和描述文件

```shell
fastlane match development --readonly true

fastlane match adhoc --readonly true

fastlane match appstore --readonly true
```