---
title: 常用Git指令
date: 2022-08-20 09:51:29
tags: git
description: 记录工作中常用的git指令
---
## 工作中常用的git指令

<!-- more -->
## git stash 常用指令 
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