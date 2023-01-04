---
title: 常用Linux指令
date: 2022-11-24 18:31:36
tags:
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
grep "match_pattern" file_1 file_2 file_3 ...
3. 输出除“match_pattern”之外的所有行
grep -v "match_pattern"  file_name
4. 使用正则表达式来模糊查找
grep -P "[1-9]$" file_name
5. 在管道grep中使用正则表达式
使用 grep -E "[1-9]+"  或者 egrep "[1-9]+"
如 echo this is a test line. | grep  -E "[a-z]+\."
6. 在grep时只输出匹配到的部分而不是一整行，加上-o参数
grep -o "match_pattern"  file_name
7. 统计文件或者文本中包含匹配字符串的行数，加上-c 选项
grep -c "match_pattern"  file_name
8. 输出包含匹配字符串的行数，加上-n选项
grep -n "match_pattern" file_name
9. 搜索多个文件并查到匹配文本在哪些文件出现：
grep -l "text" file1 file2 ...
10. 在多级目录中递归搜索
grep -r "match_pattern" . （.表示当前目录，也可以指定其他目录）
11. 忽略匹配样式中的大小写
grep -i "match_pattren" file_name
12. 在搜索时匹配多个样式
grep -e "match_pattern_1" -e "match_pattern_2"
也可以将多个match_pattern写在一个文件中如profile，然后用-f指定
grep -f profile file_name
13. grep静默输出，用来条件测试，命令运行成功返回0，失败则返回非0值
grep -q "match_pattern" file_name
14. 在grep搜索结果中包括或者排除指定文件
#只在当前目录中所的.json和.cpp文件中递归搜索字符串“match_pattern”
grep "match_pattern" . -r --include *.{json,cpp}
#在当前目录下除README文件中搜索“match_pattern”
grep "match_pattern" . -r --exclude "README"
#在当前目录下除filelist中的文件搜索“match_pattern”
grep "match_pattern" . -r --exclude-from filelist
15. 打印出匹配文本之前或者之后的行（如果有多个匹配项会以--作为各个匹配结果之间的分隔符）
#显示匹配某个结果之后的3行，使用-A选项：
grep "match_pattern" -A 3 file_name 
#显示匹配某个结果之前的3行，使用-B选项：
grep "match_pattern" -B 3 file_name 
#显示匹配某个结果之前之后的3行，使用-C选项：
grep "match_pattern" -C 3 file_name
16. 显示样式匹配所位于的字符或字节偏移
grep -b -o "match_pattern" file_name
#一行中字符串的字符偏移是从该行的第一个字符开始计算，起始值为0。选项  **-b -o**  一般总是配合使用。