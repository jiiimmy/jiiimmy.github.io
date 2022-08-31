---
title: 浅谈const在C++中的使用
date: 2022-08-29 06:31:40
tags: ["const","c++"]
---

最近在工作中用const的时候发现自己对const的理解是一种一知半解的状态，打算用这篇博客系统性总结const的用法
<!-- more -->
# 基本概念
在C++中 const 允许指定一个语义约束，编译器会强制实施这个约束，允许程序员告诉编译器某值是保持不变的。如果在编程中确实有某个值保持不变，就应该明确使用const，这样可以获得编译器的帮助。其可以用来修饰基础类型变量、自定义对象、成员函数、返回值、函数参数

# 不同场景的const用法
## const与基础类型变量
```c++
const int a = 7;
int b = a; //正确
a = 8; //错误，不能改变
```
在上面这段代码中，a被定义为一个const常量，可以将a赋值给b，但是不能对a再次赋值，因为a被编译器认为是一个常量，其值不可以修改。再看下面一段代码：
```c++
#include <iostream>

int main()
{
    const int a =7;
    int* p = (int*)&a;
    *p = 8;
    std::cout << "a = " <<a <<std::endl;

    volatile const int b =7;
    int* p1 = (int*)&b;
    *p1 = 8;
    std::cout << "b = " <<b <<std::endl;

    return 0;
}
```
得到的输出为：
```shell
a = 7
b = 8
```
用gdb在第9行打断点查看程序运行过程中变量a的变化情况，如下：

```shell
(gdb) n
5           const int a =7;
(gdb) n
6           int* p = (int*)&a;
(gdb) print(a)
$1 = 7
(gdb) n
7           *p = 8;
(gdb) n
8           std::cout << "a = " <<a <<std::endl;
(gdb) print(a)
$2 = 8
(gdb) n
a = 7
```
对于变量a，我们声明的是const，然后我们通过将a的指针强转成int*给p，通过\*p=8来改变a指针指的内容为8，通过gdb我们发现a的值确实被修改成了8，但是因为我们对a加了const关键字，会让编译器对其进行优化，在程序端看a一直为7，所以我们看到输出 `a = 7`。对于变量b，在加上const的时候同时加上了volatil让编译器不要进行优化，这里我们通过指针去修改b的值就能体现到程序中的b中，但是这样就失去了const本身的意义，故我们在确定一个变量为const后，就不要试图通过指针去修改它，这样往往会产生意想不到的行为