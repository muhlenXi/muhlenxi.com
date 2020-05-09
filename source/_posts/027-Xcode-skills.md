---
title: 聊聊 Xcode skills
date: 2017-03-21 15:25:58
categories: 工具集
tags: [Xcode]
---

 *版权声明：本文为 muhlenXi 原创文章，欢迎转载，转载请注明来源。*

#### 详情简介：

> 本文记录的是 Xcode 软件的使用技巧。

学习贵在记录和总结！

<!-- more -->　　

### Xcode

#### 技巧篇

##### Xcode 快捷键 详细说明：

* `Command + F`		当前文件中查找
* `Command + Option + F`		当前文件中查找和替换

* `Command + Shift + F`		项目中查找
* `Command + Shift + Option + F`		项目中查找和替换

* `Command + 0`		显示或隐藏左边导航栏
* `Command + 1 - 8 `		选择左边导航栏中的选项

* `Command + Option + 0`		显示或隐藏右边的导航栏
*  `Command + Option + 1 - 9`		选择右边导航栏中的选项

* `Command + Option + /`		给函数添加注释

* `Command + B`		编译
* `Command + R` 		运行
* `Command + .` 		停止运行
* `Command + Shift + K` 		清理缓存

* `Command + Shift + Y` 		显示或隐藏控制台
* `Command + K`		清理控制台内容

* `Command + Option + Return`		打开 Xib 相关联的 ViewController
* `Command + Return`		关闭 Xib 相关联的文件

* `Command + [` 		`左移`选中的代码块
* `Command + ]` 		`右移`选中的代码块
* `Command + \` 		注释或取消注释 代码块
* `Command + Option + [` 		`上移`代码块
* `Command + Option + ]`			`下移`代码块 

* `Command + Shift + O` 		根据文件名快速切换文件

* `Command + Control + space` 呼出 emoji 表情输入

* `Control + A` 光标移到本行 `行首`
* `Control + E` 光标移到本行 `行尾`
* `Control + N` 光标跳到 `下一行`
* `Control + P` 光标跳到 `上一行`

* `Command + 方向左键 `  光标移到本行代码开头
* `Command + 方向 →`		光标移到本行代码结尾
* `Command + L`		快速跳到某一行



##### Xcode 快捷键图示

<div align=center>
<img src="http://7xvffo.com1.z0.glb.clouddn.com/Xcode%20Keys.jpg" width="736" height="1041" alt="选项列表图"/>
</div>


#### 配置篇

##### Provisioning Profiles 文件目录

如图：`/用户/yangxi/资源库/MobileDevice/Provisioning Profiles/`

<div align=center>
<img src="http://7xvffo.com1.z0.glb.clouddn.com/Provisioning.png" width="949" height="213" alt="选项列表图"/>
</div>

##### Xcode 代码块 文件目录

如图：`/用户/yangxi/资源库/Developer/Xcode/UserData/CodeSnippets/`

<div align=center>
<img src="http://7xvffo.com1.z0.glb.clouddn.com/codeSniper.png" width="753" height="123" alt="选项列表图"/>
</div>

##### 查找项目中的中文字符串

*如图所示：点击放大镜并切换到 `Find > Regular Expression` 模式下，输入 `@"[^"]*[\u4E00-\u9FA5]+[^"\n]*?"`回车即可！Swift则要去掉@符号。* 

<div align=center>
<img src="http://7xvffo.com1.z0.glb.clouddn.com/Xcode.png" width="363" height="165" alt="选项列表图"/>
</div>

##### 导出 本地化语言配置 xiff 文件


*如图所示：1、选中项目后，然后点击 `Editor`,选择 红色方框 中的选项，然后执行步骤2。* 

<div align=center>
<img src="http://7xvffo.com1.z0.glb.clouddn.com/localized1.png" width="565" height="291" alt="选项列表图"/>
</div>

*2、填入文件名和路径，勾选要导出的语言，保存就可以了。。* 

<div align=center>
<img src="http://7xvffo.com1.z0.glb.clouddn.com/localized2.png" width="565" height="291" alt="选项列表图"/>
</div>

##### Objective-C 调用 Swift

【1】确保将项目的`Target`的 `Build Setting -> Packaging -> Defines Module` 设置为 `YES`。
【2】修改`Build Setting` 中的 `Product Module Name` 为 `项目名`。
【3】在调用的地方 导入 `#import <项目名-Swift.h>`即可。

##### pch文件路径

*$(SRCROOT)/项目名称/pch文件名*


##### Xcode8控制台输出信息过多解决方法

Product -> Scheme -> Edit Scheme... -> Run -> Arguments, 在Environment Variables里边添加 OS_ACTIVITY_MODE ＝ disable

