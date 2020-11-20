---
title: 聊聊 iOS 证书、ID、描述文件、通知推送
toc: true
donate: false
tags: []
date: 2019-06-01 20:32:22
categories: iOS
---

一篇关于 iOS 开发中涉及的证书、ID、描述文件。通知推送相关的记录。

<!-- more -->


iOS 的代码签名用来控制一个 App 是哪个开发者开发的，可以运行到哪些设备上。

Apple 用 Bundle Identifier 来唯一标识一个 App，这是一个独一无二的字符串。

开发者通过创建 Certificates 来向 Apple 提交身份信息。

Apple 用 Provisioning Profiles 文件来指定 App 的 Bundle Identifier，App 的签名证书，可安装这个 App 的设备ID集。

###  安装、分发、提交商店

App Provisioning Profile 的类型有四种，分别是 :

- Development
- Ad hoc
- App Store
- in-house

根据  Certificate 的类型和分发方式可以分为以下四种情况：

#### Development

也就是通过 Xcode 安装到真机上，证书和描述文件需要满足如下条件

```
Certificate: Development
Provisioning Profile: Development
```

#### Distribution  Ad Hoc

通过这种方式分发的 App，需要将待安装的真机的 UDID 注册到对应的开发者账号中，最多只能注册 100 台。 证书和描述文件需要满足如下条件：

```
Certificate: Distribution
Provisioning Profile: Ad Hoc
```

#### Distribution Enterprise

这是企业分发方式，这种方式生成的 App 不能提交到 App Store，可以通过蒲公英来分发到不同的设备中，不需要在开发者账号中注册设备的 UDID。 证书和描述文件需要满足如下条件：

```
Certificate: Distribution (Enterprise)
Provisoning Profile: Universal Distribution
```

#### Distribution App Store

提交 App Store 的 App， 证书和描述文件需要满足如下条件：

```
Certificate: Distribution
Provisioning Profile: App Store
```

### 推送通知

用于 apns 推送的证书类型有两种，一种是开发证书，一种是生产证书。

1、用 Xcode 安装的 App，如果想要收到通知，需要满足以下条件：

- Signing certificate 必须是开发证书，也就是 Apple Development：XXX
- Provisioning Profile 的类型是 Development
- 推送的后台是用 开发证书 推送的通知。

2、蒲公英分发的 ad hoc 类型的 App，如果想要收到通知，需要满足以下条件：

- Signing certificate 必须是分发证书，也就是 Apple Distribution
- Provisioning Profile 的类型是 Ad hoc
- 推送的后台是用 生产证书 推送的通知

3、App Store 下载的 App，如果想要收到通知，需要满足以下条件：

- Signing certificate 必须是分发证书，也就是 Apple Distribution
- Provisioning Profile 的类型是 App Store
- 推送的后台是用 生产证书 推送的通知

###  如何生成相应文件

1、[如何生成 App ID](https://customersupport.doubledutch.me/hc/en-us/articles/229488228-iOS-How-to-Create-an-App-ID)
2、[如何生成 Distribution Certificate](https://customersupport.doubledutch.me/hc/en-us/articles/360001189514-iOS-How-to-Create-a-Distribution-Certificate)
3、[如何生成 Push Notification Certificate](https://customersupport.doubledutch.me/hc/en-us/articles/229495568-iOS-How-to-Create-a-Push-Notification-Certificate)
4、[如何生成 Provisioning Profile](https://customersupport.doubledutch.me/hc/en-us/articles/229496268-iOS-How-to-Create-a-Provisioning-Profile)