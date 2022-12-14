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

# Git配置
git 的设置使用 `git config` 命令，显示当前的 git 配置信息：
```shell
git config --list
```
编辑**当前**仓库的 git 配置文件
```shell
git config -e
```
编辑**系统上所有**仓库的 git 配置文件
```shell
git config -e --global
```
设置提交代码时的用户信息（去掉 --global 只对当前仓库有效）：
```
git config --global user.name "user"
git config --global user.email user@email.com

```

# Git Head
Head 指明了当前所在分支引用的指针，它总是指向某次commit，默认是上一次的commit。HEAD 保存在当前 git 的工作目录的 `.git/HEAD` 文件中，假设当前在 master 分支，打开 `.git/HEAD`的内容为：
```shell
ref: refs/heads/master
```
.git/refs目录下有3个文件夹 heads remotes tags
- heads 存储的是本地仓库的所有分支所对应的最新 commit 对象的 SHA-1 值
- remotes 存储的是远程仓库的所有分支所对应的最新 commit 对象的 SHA-1 值
- tags 存储的是所有tag所对应的 commit 对象的 SHA-1 值

当我们提交了新 commit, 切换了仓库、分支或回滚版本时，HEAD 都会对应发生改变

HEAD能够引用一个不与分支名称相关联的特定修订版。这种情况被称为分离的HEAD

# Git ignore
在工作区有一个 .gitignore 文件（若没有可以创建）。当我们有一些文件或文件夹不想被 git 跟踪时，需要把文件/文件夹所在地址写入 .gitignore 文件即可，但是若文件已经被 git 所追踪，则需要先在暂存区中删掉对应文件，再在 .gitignore 中加入对应文件

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
- \--oneline 查看简洁的版本
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

## git clone
使用 `git clone` 可以从现有的 Git 仓库中拷贝项目，命令格式为：
```shell
git clone <repo>
```
如：
```shell
git clone https://github.com/sogou/workflow.git
```
此时默认拷贝到运行 shell 命令所在的目录，如果要克隆到指定的目录，可以使用以下格式：
```shell
git clone <repo> <directory>
```
repo为仓库地址，directory为本地目录如
```shell
git clone https://github.com/sogou/workflow.git ~/my_repo
```
执行该命令后，会在当前目录下创建一个名为 my_repo 的目录，其中包含一个 .git 的目录，用于保存下载下来的所有版本记录

## git diff
### 工作区和暂存区之间的差异
git diff 用来显示工作区与暂存区之前的差异，指令为：
```shell
git diff
```
这个指令显示的是详细的差异，比如是某个文件的某些行有变化，有时候会很多很杂乱，可以加 --stat 来直接显示哪些文件有那些变化（添加 or 删除），即：
```shell
git diff --stat
```
想单独查看某个文件具体改动内容可以在git diff后加文件名，即：
```shell
git diff path_to_file
```

### 工作区与版本库之间的差异
查看工作区与版本库之间的差异的指令为：
```shell
git diff HEAD  
or
git diff commit-id
```
注意这两者的细微区别。当本地修改了3个文件，使用 git add 其中一个后，使用 `git diff --stat` 只会显示未 add 到暂存区的两个文件的不同，而使用 `git diff HEAD --stat` 则会显示尚未提交到版本库的3个文件的不同

### 暂存区和版本库之间的差异
如果要查看暂存区和版本库之间的差异，则需要添加 --cached ,如：
```shell
git diff --cached HEAD
or
git diff --cached commit-id
```
比较暂存区某个具体文件与版本库之间的差异：
```shell
git diff --cached HEAD path_to_file
or
git diff --cached commit-id path_to_file
```
其他的指令如 --stat 也可以组合使用

### 版本库之间提交记录的差异
查看版本库不同commit的所有差异指令为：
```shell
git diff commit-id commit-id
```
查看不同commit的某个文件的差异指令为：
```shell
git diff commit-id commit-id path_to_file
```

### 不同分支之间的比较
查看不同分支的差异指令为：
```shell
git diff branch1 branch2
```
查看不同分支文件的差异指令为：
```shell
git diff branch1 branch2 path_to_file
```

## git merge
git merge命令用于将两个或两个以上的开发分支合并到一个分支的操作。git merge 会将一系列的提交关联到一个统一的历史


### git merge \<branch\>
该指令的作用是将 branch 的分支 merge 到当前分支。假设下面历史节点存在，并且当前所在分支为 master
```shell
     A---B---C dev
    /
D---E---F---G master
```
那么 `git merge dev` 命令会将 dev 与 master 分支共同节点（E节点）之后分离的节点（即 dev 分支上的A, B, C节点）重现在master分支上，按照ABCFG（一种存在的可能，**根据commit的提交时间来排序**），并且沿着master分支最后会创建一个记录合并结果的新节点H，该节点带有用户描述合并变化的信息，一种可能的合并后的 master 分支为（ABC和FG没有冲突时）：
```shell
     A---B---C dev
    /
D---E---A---B---C---F---G---H master
```

### git merge --abort
该命令仅在合并导致冲突时才使用； `git merge --abort` 将放弃合并过程并且尝试重建合并前的状态。但是，当合并开始前如果存在未 commit 的修改，`git merge --abort`在某些情况下将无法重现合并前的状态。（十分建议在 merge 前将本地未提交的改动 `git stash`, 在merge成功后再`git stash pop`）

### 解决冲突
在看到冲突以后，你可以选择以下两种方式：
方式一：
决定不合并，使用 `git merge --abort`
方式二：
当产生合并冲突时，使用git status 可以查看所有有冲突的文件。每个文件冲突的地方会以<<<<<<<, \=\=\=\=\=\=和 >>>>>>>表示。在\=\=\=\=\=\=\=之前的部分是当前分支的情况，在=======之后的部分是待merge分支的情况。需要自己手动修改冲突的地方，并使用 `git add`将有冲突的文件添加到暂存区，再使用 `git merge --continue`继续执行 merge 操作

### git merge 的几种形式
git merge --ff 命令
--ff 是值 fast-forward 命令。当使用 fase-forward 模式进行合并时，将不会创建一个新commit节点，默认情况下， git-merge 采用 fase-forward 模式
git merge --no-ff 命令
即使可以使用fast-forward 模式，也要创建一个新的合并节点。这是 git merge 在合并一个 tag 时的默认行为
git merge --ff-only
除非当前HEAD节点已经up-to-date（更新指向到最新节点）或者能够使用fast-forward模式进行合并，否则的话将拒绝合并，并返回一个失败状态

## git rebase
git rebase 是一个在另一个分支上重新应用当前分支提交的过程，它是一个线性的合并过程
其主要有2个应用场景：
### 整理 commit 记录
当我们基于主分支master进行开发时，往往会在完成开发任务中的某个小任务时进行一次提交来保存当前阶段的工作，等功能完全开发完后往往会有好几个commit，这个时候有把这几个commit合并成一个的需求。可以进行如下操作：
`git rebase -i master(或者 commit-id)`
此时会进入交互模式。在交互模式中将最后一次commit前面的指令从pick修改为s，将不想要的commit可以修改成d，保存退出即可(此时会进入到下一个交互中，是用来编写合并后的commit信息的)。完成编辑并保存退出后即可将几个commit合并成一个

### 分支合并
当我们从主分支上切出一个dev分支进行开发
```shell
      D---E---F dev
     /
A---B master
```
在开发完后发现其他同事已经合入了一个commit，此时分支为：
```shell
      D---E---F dev
     /
A---B---C master
```
此时想要合入代码时，就无法使用fast-forward模式合入代码。如果项目要求一个线性的集成分支提交记录，就不能直接 merge ，需要进行 `git rebase master` 操作，（在没有冲突时）然后分支变为
```shell
         D---E---F dev
        /
A---B---C master

```
完成 rebase 后 dev 分支会处于 fast-forward 可用模式

### rebase 发生了什么
假设当前分支状态如下，此时在 dev 分支执行 `git rebase master`操作
```shell
      D---E---F dev
     /
A---B---C master
```
- 首先，git 会将 dev 分支里面的 D,E,F 三个 commit 取消掉；
- 其次，把上面的操作临时保存成 patch 文件，存在 .git/rebase 目录下；
- 然后，把 dev 分支更新到最新的 master 分支，即将 C commit 加在 B 之后；
- 最后，把上面保存的 patch 文件按照 D,E,F 的顺序应用到 dev 分支上
若 C 与 D,E,F 均没有冲突（没有修改同一文件的同一行），那么 rebase 完成，此时 dev 分支的状态就是 `A---B---C---D---E---F`
若 C 与 D,E,F 任一 commit 有冲突，则 rebase 会停止，使用 git status 可以查看有冲突的文件，此时有两种选择
- 不知道该如何处理冲突，则直接使用 `git rebase --abort`，此时 dev 分支会回到 rebase 前的状态
- 知道如何处理冲突（即使用哪一个更改），先修改完所有冲突的地方，使用 `git add`将所有冲突文件添加到暂存区，最后使用 `git rebase --continue`完成 rebase ，此时 dev 分支的状态也是 `A---B---C---D---E---F`

## git reset
git reset 命令通常被用来重置更改，即操作 HEAD 将其指向特定的 commit。从 git 角度出发 git reset 是一个复杂的、多功能的撤销修改的工具。它充当了 git 的时间机器。让我们可以在各种提交之间来回跳跃。
git reset 有三种核心调用形式，如下：
### git reset --hard
使用`git reset --hard commit-id`它将移动 Head 到指定 commit-id，然后用提交的内容更新暂存区。这是最直接、最不安全、也是最常用的选项。--hard会改变提交历史，HEAD指针会更新到指定的commit。同时暂存区和工作区会重置成与 指定 commit-id 相匹配的状态，即所有的与指定 commit-id 不同的改动都会丢失。

### git reset --mixed 
--mixed 是 git reset 命令的一个默认选项。`git reset --mixed commit-id` 会将指定 commit-id 与 当前 HEAD 所在 commit-id 之间所有的改动放在工作区，同时会将暂存区中所有的文件退回到工作区，原来工作区的文件不会发生改动，此时使用 git status 可以看到所有有改动的文件

因此可以用 git reset 来**取消刚刚加入到暂存区的文件**，指令为：
`git reset HEAD path_to_file`
### git reset --soft
`git reset --soft commit-id` 只将 HEAD 指向到 commit-id ，不会对工作区和暂存区做任何修改

### 版本回滚
使用 git reset 可以进行版本回滚，可以有两种方式，当已知分支名 dev 时，可以用如下指令进行回滚：
```shell
git reset dev
```
当只是想回滚当前版本前一个版本时，可以使用如下指令：
```shell
git reset HEAD^
```
所有方式如下：
| 方式    | 意义       |
| ------- | ---------- |
| HEAD    | 当前版本   |
| HEAD^   | 上一个版本 |
| HEAD^^  | 上上个版本 |
| HEAD~10 | 上10个版本 |
| HEAD~n  | 上n个版本  |
| 版本号  | 指定版本   |

这些方式均可搭配 –mixed, --soft, --hard 来达到不同的效果

### git reflog
`git reflog`用来记录我们的每一次命令
当我们回滚到前几个版本后又想回到最新版本时，使用git log 已经找不到回滚之前的commit-id，此时使用 git reflog 即可以查到回滚前对应的 commit-id，然后再使用 git reset --hard commit-id 即可回到最开始的状态
## git rm
`git rm file` 命令用来删除**工作区和暂存区**中的文件
其和 `rm file`的区别在于 rm 只会删除工作区的 file, 在暂存区其还没删除，需要使用 `git add file`才能进行提交，而使用 `git rm file` 后可以直接进行提交

### git rm --cached file_name
当我们想在 git 上删除文件，但同时又想把文件保留在本地仓库，即不想在 git 上分享该文件，此时使用 `git rm --cached file_name`，这个指令将删除操作只作用于暂存区，而不是工作区

## git fetch
git fetch 用来将远程仓库的最新内容拉取到本地, 但**不会更新你本地 repo 的工作状态**

`git fetch -all` 可以用来拉取远程仓库的所有分支
`git fetch origin branch` 可以用来拉取指定远程分支 branch 到本地，不会影响任何本地 repo 状态
git fetch还可以只拉取远程仓库的某一个commit
`git fetch origin commit-id`
## git cherry-pick
`git cherry-pick` 通常与 `git fetch` 配合使用
一种场景是当我们在某个分支上进行开发，已经提交了几个commit，此时发现某个功能存在 bug ，这个 bug 被另一个同事在另一个分支上进行了修复，但是该分支有好几个 commit ，我们只需要使用其修复 bug 的 commit 来对我们的功能进行验证， cherry-pick 的作用就是添加某一个 commit 到当前分支上，其用法为：
```shell
git cherry-pick commit-id1 commit-id2
```
cherry-pick 支持同时 pick 几个 commit-id 用空格分开即可，当进行 cherry-pick 时也有可能发生冲突，其解决方案与 git rebase 发生冲突基本一致：
- 当不知道该如何解决冲突时，直接 git cherry-pick --abort，此时 git 回退到进行 cherry-pick 前的状态
- 确定该如何解决冲突时，先解决冲突的文件，然后使用 git add 将改动添加，再使用 git cherry-pick --continue 即可 pick 成功

## git branch
git branch 用来进行与分支相关的操作
### 创建新分支
```shell
git branch branch_name
```
该命令将会创建一个新的名为 branch_name 的分支，但不会切到新的分支上去，若要在创建的同时切换到新分支可以使用 `git branch -m branch_name`，其作用与 `git checkout -b branch_name`一致

### 列出本地所有分支
```shell
git branch --list 或者
git branch
```
即可列出本地仓库的所有分支以及当前所在分支

### 删除本地分支
使用 `git branch -d branch_name` 会删除本地的 branch_name 对应的分支，但是如果该分支有提交未进行合并，则会删除失败
使用 `git branch -D branch_name` 会强制删除本地的 branch_name 对应的分支，即使该分支有提交未进行合并，也会删除成功

### 删除远程分支
可以在本地删除一个远程分支，命令为：
```shell
git push origin -delete branch_name
```

### 重命名分支
我们可以在 branch 命令帮助下重命名该分支，指令为：
```shell
git branch -m old_branch new_branch
```
## git pull
git pull 从远程获取最新分支代码并合并到本地的分支上，其实是 `git fetch`和`git merge FETCH_HEAD`的简写
命令格式如下：
```shell
git pull <远程主机名> <远程分支名>:<本地分支名>
```
如果远程分支名与本地分支名一致，可以省略冒号后面部分
```shell
git pull origin master
```
上面指令的意思为取回 origin/master 分支并与本地的 master 分支合并

### 强制pull
当远程主分支发生了大改，而本地的主分支还是大改前的状态，此时用 git pull 会有冲突，而我们只想强制将本地分支更新到与远程分支一致，可以用以下两种方法（以master分支为例）：
方式一：
```shell
git checkout -b tmp
git branch -D master
git fetch origin master && git checkout master
```
方式二：
```shell
git fetch origin master
git reset --hard master
```

## git push
git push 用来将本地仓库的内容上传到远程仓库
git push 的用法为：
```shell
git push <option> <repository> <branch_name>
```
repository : 指推送的目的地，可以是一个URL，也可以是远程仓库的名称
常用的 option 有：
| option  | 意义                       |
| ------- | -------------------------- |
| -all    | 推所有分支                 |
| -prune  | 删除本地没有对应的远程分支 |
| -mirror | 用来将版本库镜像到远程     |
| -tags   | 推所有的本地标签           |
| -delete | 删除指定分支               |
| -u      | 创建一个上游跟踪链接       |

当我们修改了本地的某个 commit 导致和远程分支出现不一致，此时用 git push 会失败，此时可以使用强制push
```shell
git push origin dev -f
```
## git revert
git revert 命令是用来撤销某个 commit 的改动，在这个过程中不会删除任何数据；相反，它会创建一个具有相反效果的新 commit 来撤销原 commit 的改动，其用法为：
```shell
git revert commit-id
```
使用 git revert 后会新生成一个 commit，如果不想生成新 commit 可以加上 -n 参数，此时会将撤销的改动添加在暂存区：
```shell
git revert commit-id -n
```

在 revert 的时候可能也会产生冲突，解决方案与 git rebase 冲突时一致

### git revert 与 git reset 的区别
- git revert 是用一次新的 commit 来回滚之前的 commit，此次提交之前的commit都会被保留；
- git reset 是回到某次提交，提交及之前的 commit 都会被保留，但是此 commit 之后的修改都会被删除

## git remote
通常使用 `git remote -v` 检查版本库的远程名字以及其对应URL

我们可以显示的为一个仓库添加远程仓库连接，指令为：
```shell
git remote add <name> <remote URL>
```
在添加完后就可以在 git 中任何需要使用 URL 的地方用 name 来代替 URL 

可以显示添加就可以显示删除一个远程仓库连接，指令为：
```shell
git remote rm <name>
```

同时，支持重命名远程仓库的名称，指令为：
```shell
git remote rename <old_name> <new_name>
```

remote 也支持修改 URL, 指令为：
```shell
git remote set-url <name> <newURL>
```