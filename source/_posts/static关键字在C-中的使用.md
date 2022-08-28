---
title: static关键字在C++中的使用
date: 2022-08-27 01:44:27
tags: ["c++","static"]
---

static 是C++的一个关键字，它被用来控制变量的存储方式和可见性
<!-- more -->
# 程序在内存中的模型
首先，一个完成的程序在内存中的内存分布情况如下：
![内存模型](https://s1.imagehub.cc/images/2022/08/28/b2cd5e7c672d9afa4de5f4ef3b09cd13.md.jpg)
**代码区**：存放程序的代码，即CPU执行的机器指令，并且是只读的
**常量区**：存放常量(程序在运行的期间不能够被改变的量，例如: 10，字符串常量”abcde”， 数组的名字等)
**静态区（全局区**）：静态变量和全局变量的存储区域是一起的，一旦静态区的内存被分配, 静态区的内存直到程序全部结束之后才会被释放，此区的变量默认初始化为0
**堆区**：由程序员调用malloc()函数来主动申请的，需使用free()函数来释放内存，若申请了堆区内存，之后忘记释放内存，很容易造成内存泄漏。在c++中一般是用new和delete来申请释放内存
**栈区**：存放函数内的局部变量，形参和函数返回值。栈区之中的数据的作用范围过了之后，系统就会回收自动管理栈区的内存(分配内存 , 回收内存),不需要开发人员来手动管理。
# static与变量
- 当static修饰变量时，变量会保存在静态区，并且只会对其初始化一次，变量的生命周期会直到程序结束，我们来看代码
```c++
#include <iostream>

class A {
public:
    int get_val() { return val; }
    A() {
        std::cout << "call A ctor" << std::endl;
        val = 0;
    }
    A& operator++() {
        ++val;
        return *this;
    }
private:
    int val;
};

void test() {
    static A a;
    ++a;
    std::cout << "a val = " << a.get_val() << std::endl;
}

void test2() {
    A aa;
    ++aa;
    std::cout << "aa val = " << aa.get_val() << std::endl;
}

int main()
{
    for (int i = 0; i < 4; ++i) {
        test();
    }

    std::cout << "-------------" << std::endl;

    for (int i = 0; i < 4; ++i) {
        test2();
    }
    return 0;
}
```
得到的输出为：
```shell
call A ctor
a val = 1
a val = 2
a val = 3
a val = 4
-------------
call A ctor
aa val = 1
call A ctor
aa val = 1
call A ctor
aa val = 1
call A ctor
aa val = 1
```
从代码中我们可以看到，虽然test()函数中的a是一个局部变量，但它被static修饰，因此保存在静态区，生命周期并不会随着第一次test()函数的结束而结束。第一次调用test()函数时，会调用A的构造函数并将其保存在静态区。当我们第二次调用test()函数时，因为静态区已经存在一个A对象的a变量，因为不会再去调用A的构造函数。
而对于普通变量aa，在调用test2()时，会在栈区去创建变量aa，其生命随着test()函数的结束而结束，因此我们可以看到调了4次A的构造函数，并且每次构造完后val的值都是0

- 静态全局变量和静态局部变量
我们来看下面一段代码
```c++
#include <iostream>

class A {
public:
    A() {
        std::cout << "call A ctor" << std::endl;
    }
    ~A() {
        std::cout << "call A dtor" << std::endl;
    }
};

class B {
public:
    B() {
        std::cout << "call B ctor" << std::endl;
    }
    ~B() {
        std::cout << "call B dtor" << std::endl;
    }
};

class C {
public:
    C() {
        std::cout << "call C ctor" << std::endl;
    }
    C(C& other) {
        std::cout << "call C copy ctor" << std::endl;
    }
    ~C() {
        std::cout << "call C dtor" << std::endl;
    }
};

static C c;
void test() {
    static A a;
    C c2(c);
    //B b2(b); 编译不通过
}


int main()
{
    std::cout << "start call main func !" << std::endl;
    static B b;

    test();
    return 0;
 
}
```
程序的输出为：
```shell
call C ctor
start call main func !
call B ctor
call A ctor
call C copy ctor
call C dtor
call A dtor
call B dtor
call C dtor
```
由这段代码及输出我们可以得到以下结论
> 1. static并不会改变变量的作用域，我们定义了静态全局变量c，main()函数里的静态局部变量b，test()里的静态局部变量a和由c经过拷贝构造的静态局部变量c2。我们无法在test中调用b变量，即使test函数是在main函数中被调用。
> 2. 静态全局变量会在main函数之前构造，而静态局部变量只会在第一次声明它的时候开始构造，是在main函数之后构造的
> 3. 虽然全局静态变量和局部静态变量都存在静态区，但是析构也是有顺序的，越后声明的静态局部变量越先析构

<!-- 同时静态变量使得该变量在静态存储区分配内存;在声明该变量的整个文件中都是可见的，而在文件外是不可见的 -->

# static与普通函数
被static修饰的普通函数在整个文件中都是可见的，但是在文件外时不可见的。在a.h中定义了static void func()，在b.cpp中include "a.h"也无法使用func函数。可以避免多人合作时的函数冲突

# static与成员变量
被声明成static的成员变量只被初始化一次，因为它也被保存在静态区。包含静态成员变量的类的所有实体共享一个静态成员变量，故在一定程度上能减小对象的大小。正因为此，静态成员变量需要在**类内声明，类外定义**，但是单成员变量时const的时候可以在类内声明时就给初值，其整个生命周期值不会变化。具体可以看以下代码：

```c++

#include <iostream>

class A {
public:
    A() {
        std::cout <<"num = " << ++num << std::endl;
    }
    static int num;//const static int num =10;在定义常量时可以赋初始值
    int val;
};
int A::num = 0;//类外初始化

class B{
    B(){
        std::cout <<"num = " << ++num <<std::endl;
    }
    int num;
    int val;
};
int main()
{
    for (int i = 0; i < 4; ++i) {
        A a;
    }
    std::cout << "size A = " << sizeof(A) <<std::endl;
    std::cout << "size B = " << sizeof(B) <<std::endl;
    return 0;
}
```
程序的输出为：
```shell
num = 1
num = 2
num = 3
num = 4
size A = 4
size B = 8
```

# static与成员函数
static修饰的类成员函数属于类，不属于任何对象，因此static类成员函数是没有this指针的，this指针是指向本对象的指针。正因为没有this指针，所以static类成员函数不能访问非static的类成员，只能访问 static修饰的类成员（变量和函数）。
同时，static成员函数不能被virtual修饰，static成员不属于任何对象或实例，所以加上virtual没有任何实际意义。类的static函数可以在没有构造任何类实例之前被调用