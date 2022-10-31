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

## const与指针变量
const 修饰指针变量有以下三种情况
- const 修饰指针指向的内容，此时内容为不可变量，通常称为常量指针,如：
```c++
#include <iosteam>
int main(){
    int a = 3;
    int b = 4;
    const int *p = &a; //通常称p为常量指针
    //int const *p = &a 也可以这样写
    std::cout << " *p = " << *p << std::endl;
    //*p = 5; 错误，不能通过指针p来修改p所指对象
    a = 5;
    std::cout << " *p = " << *p << std::endl;
    p = &b;
    std::cout << " *p = " << *p << std::endl;
    return 0;
}
```
这段代码的输出为
```shell
 *p = 3
 *p = 5
 *p = 4
```
这里指针常量的意思是指不能通过指针 p 来修改指针所指的变量a的内容，但仍然可以直接通过直接修改变量 a 来修改内容。同时，常量指针可以让其指向另一个变量

- const 修饰指针，此时指针为不可变量，通常称为指针常量
看下面这段代码：
```c++
int main()
{
    int a = 3;
    int b = 4;
    int* const p = &a;
    //p = &b; 错误，const修饰p，p只能指向a所在的地址
    std::cout << " *p = " << *p << std::endl;
    *p = 5;
    std::cout << " *p = " << *p << std::endl;
    return 0;
}
```
输出为：
```shell
 *p = 3
 *p = 5
```
这里常量指针的意思是指不能修改指针 p 所指的地址，但是可以通过 *p 来改变其所指地址里的内容。
- const 同时修饰指针和指向的内容，指针本身是一个常量，其所指内容也是一个常量，又称为指向常量的指针常量。

>**总结**：左定值，右定向，const修饰不变量（左右是指 const 与 * 的位置关系）

## const与普通函数参数
对于 const 修饰函数参数大致可以分为四种情况
1. 值传递
2. 常量指针
3. 指针常量
4. 引用传递
```c++
void test1(const int val) {
    //++val; 不允许
    std::cout << "val ptr = " << &val << std::endl;
}
void test2(const int* val) {
    //++(* val); 不允许
    std::cout << "val ptr = " << val << std::endl;
    int b=6;
    val = &b;
    std::cout << "b   ptr = " << &b << std::endl;
    std::cout << "val ptr = " << val << std::endl;
}
void test3(int* const val) {
    ++(* val);
    //int b = 3; 不允许
    //val = &b;
    std::cout << "val ptr = " << val << std::endl;
}
void test4(const int& val) {
    //++val; 不允许
    std::cout << "val ptr = " << &val << std::endl;
}

int main()
{
    int a = 3;
    std::cout << "a   ptr = " << &a << std::endl;
    std::cout << "---------------" << std::endl;
    test1(a);
    std::cout << "---------------" << std::endl;
    test2(&a);
    std::cout << "---------------" << std::endl;
    test3(&a);
    std::cout << "---------------" << std::endl;
    test4(a);
    return 0;
}
```
代码的输出为：
```shell
a   ptr = 000000E1BFEFFA94
---------------
val ptr = 000000E1BFEFFA70
---------------
val ptr = 000000E1BFEFFA94
b   ptr = 000000E1BFEFF974
val ptr = 000000E1BFEFF974
---------------
val ptr = 000000E1BFEFFA94
---------------
val ptr = 000000E1BFEFFA94
```
由上面的代码和输出我们可以得出以下结论：
- 使用 const 值传递参数时，在函数内不能修改 val ，并且 val 与 a 的地址不一样，即生成了一个临时变量
- 使用常量指针作为参数时，在函数内不能通过指针来修改变量 val ，但是可以修改指针本身指向不同的变量
- 使用指针常量作为参数时，在函数内可以通过指针来修改变量 val ，但是不能修改指针本身所指的变量
- 使用 const 引用作为参数时，在函数内不能修改 val, 同时 val 的地址与 a 一样，没有产生临时变量