---
title: zsh 的配置和 CocoaPods 和 zsh 的使用
date: 2015-10-05 16:17:11
categories: 工具集
tags: [zsh, CocoaPods]
donate: false
toc: true
---

对于一个爱折腾的人是闲不住的，最近经常和`终端(terminal)`打交道，看着这普普通通的界面，实在在是忍无可忍了，有一次网上查资料的时候，了解到`zsh`可以配制出高逼格的用户界面，就是配置有些复杂。但是程序猿都是一群聪明的家伙，研究出`oh-my-zsh`这个开源项目，让zsh配置降到了零门槛，而且完全兼容`bash`。
　　
　　<!-- more -->
　　
## 通过 oh-my-zsh 配置 zsh

### 配置步骤

打开我们亲爱的 `终端 Terminal`, 输入命令：`cat /etc/shells`

可以看到 Mac 内置了 6 种 shell

```
/bin/bash
/bin/csh
/bin/dash
/bin/ksh
/bin/sh
/bin/tcsh
```


1、安装 oh-my-zsh ，它会自动读取你的环境并帮你设置zsh, 在终端内输入以下命令 clone oh-my-zsh：

```
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh  
```
    
2、替换zshrc文件：

```
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```
    
3、切换到zsh模式：

```
chsh -s /bin/zsh
```
    
4、关闭并重新打开终端后，会发现变成 zsh 了，接下来是选择一款高逼格的主题！

oh-my-zsh [主题](https://github.com/robbyrussell/oh-my-zsh/wiki/themes)的主题和对应截图在在这里！

我选择的是ys主题，在终端中打开 oh-my-zsh 的配置文件，设置主题为 ys：

```
vim ~/.zshrc  //打开.zshrc文件
```
    
按 `字母i` 进入编辑模式，修改 `ZSH_THEME="ys"` 后，按 `esc` 退出编辑模式，再按 `shift + ：`，输入 `wq`, 就是保存并退出的意思。

    
## 常用 Terminal 命令

| Shortcut Keys | description |
| :--------- | :---------- |
|  control C  |  终止执行命令   |
|  control A  |  移动光标到 行首  |
|  control E |  移动光标到 行尾 |
|  control U |  删除整行命令 |
|  control K |  删除光标后的内容  |
|  clear |  清除屏幕或窗口内容 |
|  ls	 |  显示当前目录的内容 |
|  ls -ah	 |  全部显示当前目录的内容 |
|  rmdir	 |  删除一个目录 |
|  mvdir	 |  移动或重命名一个目录 |
| cd Desktop/ | 进入新目录 | 
|  cd ..  |  返回上一级目录 |
|  cd ~  |  返回主目录 |
| cp ~/Desktop/MyFile.rtf  ~/Documents | 拷贝文件 |
| cp ~/Desktop/MyFile.rtf  ~/Documents/MyFile1.rtf | 拷贝并重命名新文件 |
| mv ~/Desktop/MyFile.rtf  ~/Desktop/MyFile-old.rtf | 重命名文件 |
| mv ~/Desktop/MyFile.rtf  /Volumes/Backup/MyFolder | 移动文件 | 
|  cd	 |  改变当前目录 |
|  pwd |  显示当前目录的路径名 |
|  dircmp |  比较两个目录的内容  |
|  killall -KILL Finder |  重启Finder |
| defaults write com.apple.finder AppleShowAllFiles -bool true | 显示mac中隐藏文件(需重启Finder) |
| defaults write com.apple.finder AppleShowAllFiles -bool false |隐藏mac中隐藏文件(需重启Finder) |


## CocoaPods

### CocoaPods的安装

CocoaPods 是一个第三方库的依赖管理工具，可以自动更新第三方库，自动添加系统依赖库，自动设置编译选项，总的来说，就是能自动配置第三个方库的运行环境。他是一个命令行工具！[CocoaPods 官方指南](https://guides.cocoapods.org/)

#### 1. 配置 gem sources

RubyGems 是 Ruby 的一个包管理器，提供了分发 Ruby 程序和库的标准格式。[Ruby 维基百科](https://zh.wikipedia.org/wiki/RubyGems)

打开`终端`如下操作

1、移除默认的ruby源

```
gem sources --remove https://rubygems.org/
```
    
移除后会提示：`https://rubygems.org/ removed from sources`

2、添加 ruby-china 的源

```
gem sources -a https://gems.ruby-china.org/
```
    
注意：

1）如果曾经添加过淘宝的源，请执行如下操作，确保只有一个源。*
    
```
gem sources --add https://gems.ruby-china.org/ --remove http://ruby.taobao.org/
```
    
2）如果没有添加过 taobao 的源，可直接 安装 ruby-china 的源* 
    
```
gem sources --add https://gems.ruby-china.org/
```
     
添加后会提示相应的源已经 added to sources。
 
3、用 `gem sources -l` 查看当前的源，显示如下：

```shell
*** CURRENT SOURCES ***

https://gems.ruby-china.com
```

常用 gem 命令如下：

| 命令 | 简单说明 |
| :--------- | :---------- |
| gem sources --add https://gems.ruby-china.org/  | 添加源 |
| gem sources --remove https://rubygems.org/  | 移除源 |
| sudo gem update -n /usr/local/bin --system   | 更新gem |   
| gem -v  | 查看gem版本 |
 
#### 2、安装CocoaPods

安装命令：
 
```
sudo gem install -n /usr/local/bin cocoapods
```

输入 `gem list` 可查看 CocoaPods 的相关信息

 
####  3、CocoaPods 添加第三方库

1、查看一个库是不是支持 CocoaPods， 以AFNetworking为例

```
pod search AFNetworking
```
    
 
2、在Podfile文件添加相应版本的库，cd 到你的 Xcode 项目的目录下：
 
```
pod init
```
  
  
3、安装第三方库和依赖
 
```
pod install --verbose --no-repo-update
```

4、CocoaPods更新第三方库

当我们拿到别人的项目，或者自己的项目要添加新的第三方库，需要更新第三方库。

```
pod update --verbose --no-repo-update
```
    

### 常用 CocoaPods 命令

| Shortcut Keys | description |
| :--------- | :---------- |
|pod trunk register 邮箱 '用户名' --description='描述信息'|注册 pod|
| pod trunk me |  查看自己的 CocoaPods 信息  |
| pod lib lint PineappleKit.podspec | 检查 lib lint |
| pod trunk push PineappleKit.podspec --allow-warnings| 发布 podspec 到 CocoaPods|
| pod repo update | 更新本地  CocoaPods 的 spec 资源配置信息仓库|
| gem list --local \| grep cocoapods  | 查看安装的CocoaPods相关东西 |
| sudo gem uninstall cocoapods-core  | 卸载CocoaPods相关东西 |
| sudo gem install -n /usr/local/bin cocoapods  | 安装CocoaPods |
| sudo gem install cocoapods --version 1.6.1| 安装指定版本 CocoaPods |
| pod install --verbose --no-repo-update | 安装库和依赖 |
| pod init  | 生成默认的 Podfile配置文件 |
| pod search 库名|搜索第三方库|
| pod --version | 查看版本号|
| rm ~/Library/Caches/CocoaPods/search_index.json| 删除搜索缓存|