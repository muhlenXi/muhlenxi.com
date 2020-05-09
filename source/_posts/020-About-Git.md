---
title: 聊聊 Git
date: 2016-12-15 10:20:19
categories: 工具集
tags: [Git]
---

 *版权声明：本文为 muhlenXi 原创文章，欢迎转载，转载请注明来源。*

### 导语：

       Git 是目前世界上最先进的分布式版本控制系统。

　　要有这样的理念：**活到老，学到老，不要停止，一直保持前进！**

<!-- more -->

### 常见 git 本地仓库操作

#### 创建版本库 (repository) 并初始化

    mkdir  zhangsan     //创建zhangsan文件夹
    cd     zhangsan
    pwd                 //打印当前路径
    git init            //初始化git仓库
    
    ls -ah              //查看隐藏文件
 
　　*Unix 的哲学是“没有消息就是好消息”*，没有消息说明操作成功。
   
####  添加文件到版本库

	 touch readme.text            //创建readme文件
	 git add readme.text          //添加文件
	 git commit -m "这是标记信息"   //提交文件
	 
	 git status                   //查看工作区的状态
	 git diff                     //查看修改了哪些内容
	 
	 git log                      //查看历史提交日志记录
	 
#### 版本回退
	 
	 git reset --hard HEAD^       //回退到上一版本
	 git reset --hard commit_id(版本号)       //回退到某一版本
	 git reflog                              //查看命令历史
	 
	 cat readme.text              //查看文件内容
	 
	 
#### 撤销修改

【场景1】：当你改乱了工作区的某个文件的内容，想直接丢弃工作区的修改时，用命令 `git checkout -- filename`。

【场景2】：当你不但改乱了工作区某个文件的内容，还添加到了暂存区，想丢弃修改，分两步走，第一步用命令 `git reset HEAD file` ,就回到了【场景1】，第二步按【场景1】的操作。

【场景3】：已经提交了不合适的修改到版本库时，想要撤销本次修改，参考 `版本回退`，不过前提是还没有推送到远程库。

#### 删除文件

    git rm readme.text     //只删除工作区中的文件
    
    git rm readme.text    //从版本库中删除该文件
    git commit -m "remove readme.text"
    
    //恢复误删的文件，前提是没有从版本库中删除该文件
    git checkout -- readme.text
    
### git远程仓库操作

　　本地 Git 仓库和 GitHub 仓库之间的传输是通过 ssh 加密的，所以要设置 ssh。

【1】创建 SSHkey 

    ssh-keygen -t rsa -C "你注册的邮箱地址"
    
　　提示：1、创建完成后，会在主目录中生成 `.ssh` 目录，里面有 `id_rsa（秘钥）`和`id_rsa.pub（公钥）`。

　　2、cd 到 .ssh 目录下，用命令 `cat id_rsa.pub` 查看公钥内容。

【2】登陆 GitHub，打开 Account setting , 在 SSH keys 界面 粘贴公钥里的内容。


#### 关联远程仓库

　　在 GitHub 上创建一个新的仓库后，然后可以进行如下操作。

    git remote add origin  仓库地址   //关联远程仓库
    
    git push -u origin master       //第一次推送Master分支的所有内容
    
    git push origin master       //以后推送最新修改
    
    git clone 仓库地址            //克隆远程仓库到本地
    
 
    
### git 分支管理

#### 分支操作

    git branch                //查看分支
    git branch 分支名          //创建分支
    git checkout 分支名        //切换分支
    
    git checkout -b 分支名     //创建并切换分支
    git merge 分支名           //合并某分支到当前分支
    git branch -d 分支名       //删除分支
    
　　**注意：合并分支时，如有冲突，需要先解决冲突，然后再add和commit。**

    git log --graph          //查看分支合并图 
    
　　*合并分支时，如果有可能，git会用 `Fast forward ` 模式，在该模式下，删除分支后，会丢失分支信息。*

　　**强制禁用 `Fast forward ` 模式，git会在merge时生成一个新的commit，这样从分支历史上就可以看出分支信息。**

    git merge --no-ff -m "提示标记" 分支名
    
#### 分支策略

* 1、首先， `master` 分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；
* 2、新建分支 `dev`，干活都是在 dev 分支上，也就是说 dev 分支是不稳定的，当发布新版本的时候，才把dev 分支合并到 master 上，在 master 上发布新版本。
* 3、不同的人干活，都新建 `自己的分支`，然后时不时往 `dev` 分支上合并就可以了。

#### 通过 bug 分支来修复 bug


    git stash         //保存当前工作现场
    git stash list    //查看工作现场保存记录
    
    git stash apply   //恢复工作现场
    git stash drop    //删除stash内容
    
    git stash pop     //恢复工作现场的同时删除stash内容
    
    git stash apply stash@{0}  //恢复指定的stash
    
　　*每开发一个新功能的时候，最好新建一个功能分支！*

    git branch -D 分支名     //强行删除一个未合并的分支

#### 标签的操作

	 git tag v1.0      //给最新的commit版本打一个标签，v1.0为标签名
	 
	 git tag           //查看所有的标签 
	 
	 git log --pretty=oneline --abbrev-commit     //查看历史提交的commit id
	 git tag 标签名 commit_id      //给指定的commit打标签
	 
	 git show 标签名               //查看标签信息
	 
	 git tag -a 标签名 -m “说明文字” commit_id    //打带说明文字的标签
	 
	 git tag -s 标签名 -m “说明文字” commit_id    //打用PGP签名标签
	 
	 git tag -d 标签名             //删除标签
	 
	 git push origin 标签名        //推送某个标签到远程仓库
	 
	 git push origin --tags       //推送全部标签到远程仓库
	 
	 git push origin :refs/tags/标签名   //删除远程标签，需先删除本地标签
	 
	   

### 多人协作工作模式

* 1、首先，试着用 `git push origin 分支名` 推送自己的修改；
* 2、如果推送失败，则因为远程分支比你的本地分支要新，需要先用 `git pull` 试图合并。
* 3、如果合并有冲突，则解决冲突，并在本地提交；
* 4、没有冲突或解决掉冲突后，再用 `git push origin 分支名` 推送，就能推送成功。

　　*注意：如果 `git pull` 提示 “no tracking information”，则说明本地分支和远程分支的链接关系没有创建，用命令 `git branch --set-upstream 远程分支名 本地分支名`*


### 关于 GitHub

* 在 GitHub 上，可以任意 Fork 开源仓库；
* 自己拥有 Fork 后的仓库的读写权限
* 可以推送 pull request 给 开源项目 来贡献代码

### 配置别名


【1】配置命令：


    git config --global alias.st status
    git config --global alias.co checkout
    git config --global alias.cm commit
    git config --global alias.br branch
    
    git config --global alias.last 'log -1'  //显示最近一次的提交
    
　　*`--global` 参数是全局参数，也就是这些命令在这台电脑的所有 Git 仓库下都有用。*

【2】配置文件的路径

 　　**每个仓库的Git配置文件都放在.git/config文件中：**

 　　用命令 `cat .git/config ` 可以查看。

　　*别名就在 [alias] 后面，要删除别名，直接把对应的行删掉即可。*


　　**当前用户的 Git 配置文件放在`用户主目录`下的一个隐藏文件 `.gitconfig` 中：**

　　用命令 `cat .gitconfig ` 可以查看。


### 结束语

*欢迎在本文下面留言一起交流心得...*

*如果本文能给你带来一定的帮助，在自己有能力的情况下，不妨赞助一下，表示对博主辛勤耕作的支持！*
    




