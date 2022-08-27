---
title: Git学习笔记
date: 2022-08-20 09:51:29
tags: git
description: 记录工作中常用的git指令
---

# Git简介
Git 是一个开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目，Git 是 Linus Torvalds 为了帮助管理 Linux 内核开发而开发的一个开放源码的版本控制软件
<!-- more -->
# Git基本概念
**工作区：** 本地存在的目录，即用git管理的文件夹
**暂存区：** 英文叫 stage 或 index。一般存放在 .git 目录下的 index 文件（.git/index）中，所以把暂存区有时也叫作索引（index）
**版本库：** 工作区有一个隐藏目录 .git，这个不算工作区，而是 Git 的版本库，也可以叫做本地仓库。

# Git基本指令
## git init
在本地某个目录下新建一个文件夹并进入`mkdir repo && cd repo` , 使用 `git init` 命令来初始化一个 Git 仓库，Git 的很多命令都需要在 Git 的仓库中运行，所以 git init 是使用 Git 的第一个命令。

在执行完成 git init 命令后，Git 会在当前目录下生成一个 .git 目录，该目录包含了资源的所有元数据，其他的项目目录保持不变。即每一个被git管理的目录都会有自己的独立的暂存区和本地仓库。

也可以直接在当前目录下执行`git init newrepo`，git会自动创建一个newrepo的文件夹并进行初始化
## git add
在git init后本地仓库是空的，当我们新增了文件或者修改了本地已经存在的文件并且想要git保存这个更改到本地仓库时，需要使用git add指令，比如在仓库目录下新建了一个test.py文件，使用指令：
```shell
git add test.py
```
就可以将新增的文件添加到**暂存区**，当修改文件时也是同样的指令将修改添加到暂存区

* 关于git add的一些**小技巧**
当我们修bug或者做功能开发一次修改了多个目录下的多个文件，一个一个文件去添加很麻烦，这里有两种方法进行一次添加多个文件的操作
1. 在.git所在的目录即repo根目录下使用 `git add . `，即添加当前目录下的所有改动，但是不推荐这么使用，因为往往我们会在仓库里面放一些临时文件或者不想被git追踪的文件，此操作只在工作区所有文件都需要添加到暂存区时使用
2. 使用`git add directory`指令，使用时将directory替换成工作区对应的文件夹，就可以将directory目录下的所有文件添加到暂存区，不过这里也要注意临时文件的问题

## git commit
git commit 的作用是将暂存区的修改提交到本地仓库，并且会生成一个独一无二的40位的commit id，commit id在版本回退的时候非常有用，它相当于一个快照，可以在未来的任何时候通过git reset的组合命令回到你想要的commit id上。git commit 最简单的用法为
```shell
git commit -m "message"
```
message用来存储你对这个commit的描述，最好是将这个提交做了什么事情描述清楚，这样在版本回退的时候能清楚的知道要回到哪个commit，光靠commit id很难知道这个commit做了什么事情，**强烈建议在message上写上有意义的信息**

git commit 其他参数
* -a 带上-a参数时可以将所有已跟踪文件中的执行修改或删除操作的文件都提交到本地仓库，即使它们没有经过git add添加到暂存区，但是新加的文件（即没有被git系统管理的文件）是不能被提交到本地仓库的。
* --amend 也叫追加提交,它可以在不增加一个新的commit id的情况下将新修改的代码追加到前一次的commit id中。具体操作时先将改动用git add 添加到暂存区，再使用`git commit --amend`去提交修改。注：`git commit --amend`是支持空提交的，即没有在本地add任何修改也可以使用，这时候的作用就是去修改 "message"里填的信息
* -s 带上你的签名，在多人合作开发项目时推荐使用

## git status
git status 命令用于显示工作目录和暂存区的状态。使用 git status 命令能看到哪些修改被暂存到了，哪些没有，哪些文件没有被git追踪到
用法为：
```shell
git status
```
使用git status返回结果分为3种
- 拟提交的改动  这是已经放入暂存区，等待使用git commit提交的变更
- 未暂存的改动  这是工作目录与暂存区直接存在差异的文件列表
- 未跟踪的文件  这类文件对于git来说是未知的，git不会追踪这些文件的改动
  
另外两种常用的用法：
`git status -s ` 或者 `git status --short` 会得到格式更为紧凑的输出
`git status --ignored` 列出被git忽略的文件

## git checkout
git checkout最为常用的两种情形是创建分支和切换分支。
- 创建分支，其实包含了两个动作，一是新建一个分支，二是切换到新建的分支，用法为
`git checkout -b newbranch`
- 切换分支（必须是本地存在的分支）
`git checkout localbranch`
如果只是想创建新分支并不想切换到新分支，则使用
`git branch newbranch`
使用git checkout来创建新分支时，新的分支commit和原来分支的commit是一样的，只是做修改时只会提交到新的分支而不会同步到原来的分支

当修改了多个文件发现其中某几个文件改错了，并且还没有将改动添加到暂存区，想要在**工作区**放弃对这几个文件的修改时，使用如下指令可以支持放弃对多个文件的改动
`git checkout -- file1 file2`

- 有事进行分支切换时，checkout会报错，通常是因为工作区有未提交的改动，而改动的文件在目标分支也对其有改动造成冲突，这时要么将工作区的改动提交到当前分支后再checkout，要么用git stash将改动存在stash栈中后再checkout

## git log
git log的作用主要是显示所有的历史commit，不带参数使用时，它会列出所有历史记录，最近的排在最上方，显示提交对象的哈希值，作者、提交日期、和提交说明，如果记录过多，则按Page Up、Page Down、↓、↑来控制显示
按q退出历史记录列表。当我们想要查找某个commit时可以通过筛选参数来查找：

筛选参数
- -n 显示前n条log
- \--after="2022-8-20" 显示2022年8月20后的所有commit，日期还能用相对时间表示，如"yesterday" ，"1 week ago"
- \--before="date" 与after相反，显示date之前的所有commit。
- \--since 与\--after一样
- \--until 与\--before一样
- \--author= 按作者查找，如 `git log author="WeiZheng"` ，作者名不需要精确匹配，只需要包含就行了，同时还支持正则表达式
- \--grep= 按commit描述查找，如`git log --grep "bug fix"`，可以传入-i来忽略大小写。如果想同时使用--grep和--author，必须在附加一个--all-match参数
- \--(空格) file1 file2 按文件来查找
还有其他的参数，具体可以参考这篇博客
[git log解析](https://www.cnblogs.com/bellkosmos/p/5923439.html)


## git stash
应用场景：当在一个分支下进行开发，但是代码还没有完全写完，
- 把当前所有改动暂存在stash栈中,并在执行暂存时添加备注，方便查找；只用git stash也可以，但很久没用时会忘记这个暂存修改了什么东西
```shell
git stash save "save message"
```
- 查看当前栈所有的stash存储
```shell
git stash list
```
- 查看stash栈顶哪些文件做了改动，默认show第一个存储,如果要显示其他存储，后面加stash@{num}，比如第二个:
```shell
git stash show
git stash show stash@{1}
```
- 查看stash栈顶存储的具体改动，如果想显示其他存储的具体改动，需要在后面加stash@{num} 命令： `git stash show  stash@{num}  -p`，比如第二个:
```shell
git stash show -p
git stash show -p stash@{1}
```
- 应用stash栈某个存储到工作区,但不会把存储从stash栈中删除，默认使用第一个存储，即stash@{0}，如果要使用其他则需要在后面加stash@{num}命令，`git stash apply stash@{num}` ，比如第二个:
```shell
git stash apply
git stash apply stash@{1} 
```
- 应用stash栈顶的改动到工作目录，将缓存堆栈中的对应stash删除，与git stash apply的区别是应用后会在stash栈中删除对应的存储
```shell
git stash pop
git stash pop stash@{1}
```
- 丢弃stash栈中的第num个存储:
```shell
git stash drop stash@{num}
```
- 删除stash栈中所有的缓存:
```shell
git stash clear
```