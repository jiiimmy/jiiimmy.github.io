---
title: 常用Linux指令
date: 2022-10-31 08:32:30
tags: ["linux"]
---


这篇博客总结一些场景的Linux指令
<!-- more -->

# grep
1. 在文件中搜索一个单词，命令会返回一个包含“match_pattren”文本行：
```shell
grep match_pattern file_name
grep "match_pattern" file_name
```
2. 在多个文件中查找：
```shell
grep "match_pattern" file_1 file_2 file_3 ...
```
3. 输出除“match_pattern”之外的所有行
```shell
grep -v "match_pattern"  file_name
```
4. 使用正则表达式来模糊查找
```shell
grep -P "[1-9]$" file_name
```
5. 在管道grep中使用正则表达式
```shell
使用 grep -E "[1-9]+"  或者 egrep "[1-9]+"
如 echo this is a test line. | grep  -E "[a-z]+\."
```
6. 在grep时只输出匹配到的部分而不是一整行，加上-o参数
```shell
grep -o "match_pattern"  file_name
```
7. 统计文件或者文本中包含匹配字符串的行数，加上-c 选项
```shell
grep -c "match_pattern"  file_name
```
8. 输出包含匹配字符串的行数，加上-n选项
```shell
grep -n "match_pattern" file_name
```
9. 搜索多个文件并查到匹配文本在哪些文件出现：
```shell
grep -l "text" file1 file2 ...
```
10. 在多级目录中递归搜索
```shell
grep -r "match_pattern" . （.表示当前目录，也可以指定其他目录）
```
11. 忽略匹配样式中的大小写
```shell
grep -i "match_pattren" file_name
```
12. 在搜索时匹配多个样式
```shell
grep -e "match_pattern_1" -e "match_pattern_2"
也可以将多个match_pattern写在一个文件中如profile，然后用-f指定
grep -f profile file_name
```
13. grep静默输出，用来条件测试，命令运行成功返回0，失败则返回非0值
```shell
grep -q "match_pattern" file_name
```
14. 在grep搜索结果中包括或者排除指定文件
```shell
#只在当前目录中所的.json和.cpp文件中递归搜索字符串“match_pattern”
grep "match_pattern" . -r --include *.{json,cpp}
#在当前目录下除README文件中搜索“match_pattern”
grep "match_pattern" . -r --exclude "README"
#在当前目录下除filelist中的文件搜索“match_pattern”
grep "match_pattern" . -r --exclude-from filelist
```
15. 打印出匹配文本之前或者之后的行（如果有多个匹配项会以--作为各个匹配结果之间的分隔符）
```shell
#显示匹配某个结果之后的3行，使用-A选项：
grep "match_pattern" -A 3 file_name 
#显示匹配某个结果之前的3行，使用-B选项：
grep "match_pattern" -B 3 file_name 
#显示匹配某个结果之前之后的3行，使用-C选项：
grep "match_pattern" -C 3 file_name
```
16. 显示样式匹配所位于的字符或字节偏移
```shell
grep -b -o "match_pattern" file_name
#一行中字符串的字符偏移是从该行的第一个字符开始计算，起始值为0。选项  **-b -o**  一般总是配合使用。
```

# tar
tar 命令可以为Linux的文件和目录创建归档，可以利用tar来为某一特定文件创建档案，也可以在档案中改变文件甚至新增文件。利用tar命令可以把一大堆**文件和目录**全部打包成一个文件

**压缩和打包的区别**
打包是指将一大堆文件或目录变成一个总的文件；压缩则是将一个大的文件通过压缩算法变成一个小的文件

## tar语法
tar的语法为
```shell
tar [options...] [file...]
```
tar 支持的 options 有：
| 选项     | 描述                         |
| -------- | ---------------------------- |
| -A       | 追加tar文件至归档            |
| -c (***) | 创建一个新归档               |
| -d       | 找出归档与文件系统的差异     |
| --delete | 从归档中删除                 |
| -r       | 追加文件至归档结尾           |
| -t       | 列出归档内容                 |
| -u       | 仅追加比归档中副本更新的文件 |
| -x       | 从归档中解出文件             |
| -z       | 通过gzip过滤归档             |
| -C DIR   | 改变目录至DIR                |
| -v       | 详细的列出处理的文件         |
| -f       | 指定压缩后的文件             |
