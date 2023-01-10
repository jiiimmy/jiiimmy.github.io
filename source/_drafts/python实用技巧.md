---
title: python实用技巧
tags:
---


# 数值、日期及时间

## 对数值进行取整
当想要把一个浮点数取到固定小数位时，可以使用内建的 `round(value, ndigits)`函数，如：
```shell
>>> round(1.23,1)
1.2
>>> round(1.26,1)
1.3
>>> round(-1.26,1)
-1.3
```
由上面的例子可以看到，使用 round 函数时会对指定位数后面进行**四舍五入**，这个四舍五入与平时的四舍五入又有细微差异。
- 当某个值恰好为两个整数的一半时，取整操作会取离该值最近的那个偶数上，如：
```shell
>>> round(1.5,0)
2.0
>>> round(2.5,0)
2.0
```
- 四舍五入当该位正好为5时，不会发生“入”而是发生“舍”的操作，如：
```shell
>>> round(1.251,1)
1.3
>>> round(1.25,1)
1.2
```
传给 round 的参数 ndigits 可以是负数，这种情况下会相应的取整到十位、百位、千位等，如：
```shell
>>> var = 1627731
>>> round(var,-1)
1627730
>>> round(var,-2)
1627700
>>> round(var,-3)
1628000
```

## 执行精确的小数计算
使用原生数据类型进行计算的时候会有精度问题，如：
```shell
>>> a = 4.2
>>> b = 2.1
>>> a+b
6.300000000000001
>>> (a+b) == 6.3
False
```
如果想得到更高的精度（并为此牺牲部分性能），可以使用 decimal 模块，如：
```shell
>>> from decimal import Decimal
>>> a = Decimal('4.2')
>>> b = Decimal('2.1')
>>> a+b
Decimal('6.3')
>>> print(a+b)
6.3
>>> (a+b)==Decimal('6.3')
True
>>> (a+b)==6.3
False
```
使用 Decimal 时用的会创建 Decimal 对象，因此不能直接与数值进行比较，decimal 模块的主要功能是允许控制计算过程中的各个方面，包括数字的位数和四舍五入，如：
```shell
>>> from decimal import localcontext
>>> a = Decimal('1.3')
>>> b = Decimal('1.7')
>>> print(a/b)
0.7647058823529411764705882353
>>> with localcontext() as ctx:
...   ctx.prec = 3
...   print(a/b)
... 
0.765
>>> with localcontext() as ctx:
...   ctx.prec = 50
...   print(a/b)
... 
0.76470588235294117647058823529411764705882352941176

```

## 对数值进行格式化输出
要对单独的一个数值进行格式化输出，使用内建的 format() 函数即可，如：
```shell
>>> x = 1234.5678
#保留两位小数
>>> format(x, '0.2f')
'1234.57'
#保留1位小数，并且占10个字符位，右对齐
>>> format(x, '>10.1f')
'    1234.6'
#保留1位小数，并且占10个字符位，左对齐
>>> format(x, '<10.1f')
'1234.6    '
#保留1位小数，并且占10个字符位，居中对齐
>>> format(x, '^10.1f')
'  1234.6  '
#千分位形式
>>> format(x, ',')
'1,234.5678'
#千分位形式并且保留1位小数
>>> format(x, '0,.1f')
'1,234.6'
```
若想用科学计数法，只需要将 f 改成 E/e 即可，如：
```shell
>>> format(x, 'e')
'1.234568e+03'
>>> format(x, '0.2E')
'1.23E+03'
```

## 十进制二、八和十六进制之间的转化
要将一个整数转化为二进制、八进制或十六进制的文本字符串形式，只需使用内建的 bin(), oct(), hex() 函数即可，如：
```shell
>>> x=1234
>>> bin(x)
'0b10011010010'
>>> oct(x)
'0o2322'
>>> hex(x)
'0x4d2'
```
若不希望出现前缀，可以使用 format() 函数，如：
```shell
>>> format(x, 'b')
'10011010010'
>>> format(x, 'o')
'2322'
>>> format(x, 'x')
'4d2'
```
要将字符串形式的整数转化为不同的进制，只需要使用 int() 函数再配合适当的进制即可，如：
```shell
>>> int('4d2', 16)
1234
>>> int('10011010010', 2)
1234
```

## 处理无穷大和NaN
浮点数中有无穷大、负无穷大和 NaN (not a number)
Python中没有特殊的语法来表示这些特殊的浮点数值，但可以通过 float() 来创建，如：
```shell
>>> a=float('inf')
>>> b=float('-inf')
>>> c=float('nan')
>>> a
inf
>>> b
-inf
>>> c
nan
>>> 
```
要检测是否出现了这些值，可以使用 math.isinf() 和 math.isnan() 函数，如：
```shell
>>> import math
>>> math.isinf(a)
True
>>> math.isnan(a)
False
>>> math.isinf(c)
False
>>> math.isnan(c)
True
```
无穷大值在数学计算中会进行传播，NaN会通过所有操作进行传播，且不会引发任何异常

## 随机选择
当我们想随机挑选出元素或者想生成随机数时，可以使用 random 模块，如：
```shell
>>> val = [1,2,3,4,5,6]
# 随机选出一个元素
>>> random.choice(val)
1 
>>> random.choice(val)
6
>>> random.choice(val)
4

# 随机选出多个元素
>>> random.sample(val,2)
[1, 6]
>>> random.sample(val,2)
[2, 5]
>>> random.sample(val,3)
[5, 2, 3]
>>> random.sample(val,3)
[6, 5, 4]

# 随机打乱顺序（洗牌）
>>> random.shuffle(val)
>>> val
[2, 4, 1, 5, 3, 6]

# 产生随机整数， randint(a,b)内指定范围，产生的值是 [a,b]闭区间
>>> random.randint(0,10)
2
>>> random.randint(0,10)
4
>>> random.randint(0,10)
3
>>> random.randint(0,10)
6

# 产生0-1之间均匀分布的浮点数值
>>> random.random()
0.5298177119965545
>>> random.random()
0.056028968927269496
>>> random.random()
0.33530335693862534

# 随机产生由N个比特位表示的整数
>>> random.getrandbits(5)
15
>>> random.getrandbits(5)
8
>>> random.getrandbits(5)
23
>>> random.getrandbits(5)
7

# 设置随机种子
random.seed(12345)

```

## 