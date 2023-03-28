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


# awk
awk 是一种**编程语言**，用于在 Linux/Unix 下对文本和数据进行处理。数据可以来自标准输入(stdin)、一个或者多个文件，或其他命令的输出。它支持**用户自定义函数**和**动态正则表达式**等功能，它在命令行中使用，常被作为脚本使用

## awk命令格式和选项
语法形式
> awk [options] 'script' var=value file(s)
> awk [options] -f scriptfile var=value file(s)

常用命令选项
- -F fs fs指定输入分隔符，fs可以是字符串或正则表达式，如 -F :  ,默认的分隔符为空格或制表符
- -v var=value 赋值一个用户定义变量，将外部变量传给awk
- -f scriptfile 从脚本文件中读取awk命令


## awk模式和操作
awk模式可以是以下任意一个：
- /正则表达式/：使用通配符的扩展集
- 关系表达式：使用运算符进行操作，可以是字符串或数字的比较测试
- 模式匹配表达式：用运算符 `~`（匹配）和 `!~`（不匹配）
- BEGIN 语句块、pattern语句块、END语句块

操作由一个或多个命令、函数、表达式组成，之间由换行符或分号隔开，并位于大括号内，主要部分为：
- 变量或数组赋值
- 输出命令
- 内置函数
- 控制流语句

## awk脚本基本结构
`awk 'BEGIN{ print "start"} pattern{ commands } END{ print "end" }' file`
一个awk脚本通常由： BEGIN 语句块、能够使用模式匹配的通用语句块、END语句块3部分组成，这三个部分是可选的，任意一个部分都可以不出现在脚本中，脚本通常是在**单引号**中，如：
`awk 'BEGIN{i=0} {i++} END{print i} filename`

## awk的工作原理
- 第一步：执行 `BEGIN {command}`语句块中的语句
- 第二步：从文件或标准输入(stdin)读取一行，然后执行 pattern {commands}语句块，它逐行扫描文件，从第一行到最后一行重复这个过程，直到文件全部被读取完毕
- 第三步：当读至输入流末尾时，执行 END{ commands }语句块

当使用不带参数的print时，它就打印当前行，当print的参数是以逗号进行分隔时，打印时则以空格作为定界符。在awk的print语句块中双引号是被当作拼接符使用，例如：
```shell
echo | awk '{ var1="v1"; var2="v2"; var3="v3"; print var1,var2,var3; }' 
v1 v2 v3
echo | awk '{ var1="v1"; var2="v2"; var3="v3"; print var1"="var2"="var3; }'
v1=v2=v3
```

{}类似一个循环体，会对文件中的每一行进行迭代，通常变量初始化语句（如 i=0）以及打印文件头部的语句放入BEGIN语句块中，将打印的结果等语句放在END语句块中

## awk内置变量
```shell
ARGC               命令行参数个数
ARGV               命令行参数排列
ENVIRON            支持队列中系统环境变量的使用
FILENAME           awk浏览的文件名
FNR                浏览文件的记录数
FS                 设置输入域分隔符，等价于命令行 -F选项
NF                 浏览记录的域的个数，在执行过程中对应于当前的字段数
NR                 已读的记录数，在执行过程中对应于当前的行号
OFS                输出域分隔符(默认是一个空格)
ORS                输出记录分隔符(默认是一个换行符)
RS                 控制记录分隔符(默认是一个换行符)
```

示例：
```shell
echo -e "line1 f2 f3\nline2 f4 f5\nline3 f6 f7" | awk '{print "Line No:"NR", No of fields:"NF, "$0="$0, "$1="$1, "$2="$2, "$3="$3}'
Line No:1, No of fields:3 $0=line1 f2 f3 $1=line1 $2=f2 $3=f3
Line No:2, No of fields:3 $0=line2 f4 f5 $1=line2 $2=f4 $3=f5
Line No:3, No of fields:3 $0=line3 f6 f7 $1=line3 $2=f6 $3=f7
```
使用 print \$NF 可以打印出一行中的最后一个字段，使用 $(NF-1)则是打印倒数第二个字段，其他以此类推：
```shell
echo -e "line1 f2 f3\n line2 f4 f5" | awk '{print $NF}'
f3
f5
echo -e "line1 f2 f3\n line2 f4 f5" | awk '{print $(NF-1)}'
f2
f4
```

打印每一行的第二和第三个字段：
```shell
awk '{ print $2,$3 }' filename
```
统计文件中的行数：
```shell
awk 'END{ print NR }' filename
```

## 将外部变量值传递给awk
借助 -v 选项，可以将外部值传递给awk：

```shell
VAR=1000;
echo | awk -v VARIABLE=$VAR '{ print VARIABLE }'
10000
```
或者：
```shell
var1="aaa"
var2="bbb"
echo | awk '{ print v1,v2 }' v1=$var1 v2=$var2
aaa bbb
```

## awk运算与判断
作为一种程序设计语言所应具有的特点之一，awk支持多种运算，这些运算与C语言提供的基本相同。awk还提供了一系列内置的运算函数（如log、sqr、cos、sin等）和一些用于对字符串进行操作（运算）的函数（如length、substr等等）。这些函数的引用大大的提高了awk的运算功能

### 算术运算符
| 运算符 | 描述                       |
| ------ | -------------------------- |
| + -    | 加、减                     |
| * / %  | 乘、除与求余               |
| + - !  | 一元加，减和逻辑非         |
| ^ **   | 求幂                       |
| ++ --  | 自增和自减，作为前缀或后缀 |
例：
```shell
awk 'BEGIN{a="b";print a++,++a;}'
0 2
```
注意：所有用作算术运算符进行操作，操作数自动转为数值，所有非数值都变为0。这里先将 a转为0，然后执行++操作，故打印的第一个为0，并且此时a=1，再执行++a使a=2，最后打印a的值为2

### 赋值运算符
| 运算符                  | 描述     |
| ----------------------- | -------- |
| = += -= *= /= %= ^= **= | 赋值语句 |
这里的运算符与C语言基本一致，区别是 ^=,**=是幂等于的意思，例
```shell
awk 'BEGIN{a=3;b=2;a^=2;b**=3;print a,b}'
9 8
```

### 逻辑运算符
| 运算符 | 描述   |
| ------ | ------ |
| \|\|   | 逻辑或 |
| &&     | 逻辑与 |
例：
```shell
awk 'BEGIN{a=1;b=2;print (a>5 && b<=2),(a>5 || b<=2);}'
0 1
```

### 正则运算符
| 运算符 | 描述                             |
| ------ | -------------------------------- |
| ~ !~   | 匹配正则表达式和不匹配正则表达式 |

```
^ 行首
$ 行尾
. 除了换行符以外的任意单个字符
* 前导字符的零个或多个
.* 所有字符
[] 字符组内的任一字符
[^]对字符组内的每个字符取反(不匹配字符组内的每个字符)
^[^] 非字符组内的字符开头的行
[a-z] 小写字母
[A-Z] 大写字母
[a-Z] 小写和大写字母
[0-9] 数字
\< 单词头单词一般以空格或特殊字符做分隔,连续的字符串被当做单词
\> 单词尾
```

正则需要用两个'/'封住，即 /正则表达式/，例：
```shell
awk 'BEGIN{a="100testa";if(a ~ /^100*/){print "ok";}}'
```

### 关系运算符
| 运算符          | 描述       |
| --------------- | ---------- |
| < <= > >= != == | 关系运算符 |
例：
```shell
awk 'BEGIN{a=11;if (a>=9){print "ok";}}'
ok
```
>注：> < 可以作为字符串比较，也可以用作数值比较，关键看操作数如果是字符串就会转换为字符串比较。两个都为数字才转为数值比较。字符串比较：按照ASCII码顺序比较

### 其他运算符
| 运算符 | 描述                 |
| ------ | -------------------- |
| $      | 字段引用             |
| 空格   | 字符串连接符         |
| ?:     | C条件表达式          |
| in     | map中是否存在某键值 |
例：
```shell
awk 'BEGIN{a="b";print a=="b"?"ok":"err";}'
ok
awk 'BEGIN{a="b";arr[0]="b";arr[1]="c";print (a in arr);}'
0
awk 'BEGIN{a="b";arr[0]="b";arr["b"]="c";print (a in arr);}'
1
```


# sed
sed 是一种流编辑器，它是文本处理中非常重要的工具，能够完美的配合正则表达式使用。处理时，把当前处理的行存储在临时缓冲区中，称为“模式空间”（pattern space），接着用sed命令处理缓冲区中的内容，处理完成后，把缓冲区的内容送往屏幕。接着处理下一行，这样不断重复，直到文件末尾。文件内容并没有改变，除非使用重定向存储输出。Sed主要用来自动编辑一个或多个文件；简化对文件的反复操作；编写转换程序等。

## sed的选项、命令、替换标记
sed的命令格式有两种：
```shell
sed [options] 'command' file(s)
sed [options] -f scriptfile file(s)
```

常用选项
```shell
-e<script>或--expression=<script>：以选项中的指定的script来处理输入的文本文件；
-f<script文件>或--file=<script文件>：以选项中指定的script文件来处理输入的文本文件；
-h或--help：显示帮助；
-n或--quiet或——silent：仅显示script处理后的结果；
--version：显示版本信息。
-i 此选项会直接修改源文件，需慎用
```

sed 命令
```shell
a\ # 在当前行下面插入文本。
i\ # 在当前行上面插入文本。
c\ # 把选定的行改为新的文本。
d # 删除，删除选择的行。
D # 删除模板块的第一行。
s # 替换指定字符
h # 拷贝模板块的内容到内存中的缓冲区。
H # 追加模板块的内容到内存中的缓冲区。
g # 获得内存缓冲区的内容，并替代当前模板块中的文本。
G # 获得内存缓冲区的内容，并追加到当前模板块文本的后面。
l # 列表不能打印字符的清单。
n # 读取下一个输入行，用下一个命令处理新的行而不是用第一个命令。
N # 追加下一个输入行到模板块后面并在二者间嵌入一个新行，改变当前行号码。
p # 打印模板块的行。
P # (大写) 打印模板块的第一行。
q # 退出Sed。
b lable # 分支到脚本中带有标记的地方，如果分支不存在则分支到脚本的末尾。
r file # 从file中读行。
t label # if分支，从最后一行开始，条件一旦满足或者T，t命令，将导致分支到带有标号的命令处，或者到脚本的末尾。
T label # 错误分支，从最后一行开始，一旦发生错误或者T，t命令，将导致分支到带有标号的命令处，或者到脚本的末尾。
w file # 写并追加模板块到file末尾。  
W file # 写并追加模板块的第一行到file末尾。  
! # 表示后面的命令对所有没有被选定的行发生作用。  
= # 打印当前行号码。  
# # 把注释扩展到下一个换行符以前。  
```
## sed用法实例
### 替换操作：s命令
替换文本中的字符串：
```shell
sed 's/book/books/' file
```
将 file 中的 book 替换为books
**-n 选项**和 **p命令**一起使用表示只打印那些发生替换的行：
`sed -n 's/book/books/p' file`
直接编辑文件 **选项-i**，会匹配file中每一行的所有book替换为books：
`sed -i 's/book/books/g' file`

使用后缀/g标记会替换每一行中的所有匹配：
`sed 's/book/books/g' file`
当需要从第N处匹配开始替换时，可以使用 /Ng:
```shell
echo sksksksksksk | sed 's/sk/SK/2g'
skSKSKSKSKSK
echo sksksksksksk | sed 's/sk/SK/3g'
skskSKSKSKSK
echo sksksksksksk | sed 's/sk/SK/4g'
skskskSKSKSK
```
### 定界符
以上命令中字符 '/'在sed中作为定界符使用，也可以使用任意的定界符：
```shell
echo test |sed 's:test:TEST:g'
TEST
echo test |sed 's|test|TEST|g'
TEST
```
当定界符出现在样式内部时，需要进行转义，或者换一个定界符：
```shell
echo /bin |sed 's/\/bin/\/usr\/local\/bin/g'
/usr/local/bin
echo /bin | sed 's#/bin#/usr/local/bin#g'
/usr/local/bin
```

### 删除操作：d 命令
删除空白行：
`sed '/^$/d' file`
删除文件的第2行：
`sed '2d' file`
删除文件的第2行到末尾的所有行：
`sed '2,$d' file`
删除文件最后一行：
`sed '$d' file`
删除文件中所有开头是test的行：
`sed '/^test/d' file`