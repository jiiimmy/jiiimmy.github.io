---
title: C++11之std::function与std::bind
date: 2022-09-16 15:38:22
tags: ["c++","function"]
---

简介：c++11中新引入的std::function是一个可调用对象包装器，它是一个类模板，可以容纳除了类成员函数指针之外的所有可调用对象，它可以用统一的方式处理函数、函数对象、函数指针、lambda表达式以及std::bind对象等，并允许保存和延迟它们的执行。
<!-- more -->
使用时需要包括头文件 <functional>

# std::function

```c++
#include <functional>
#include <iostream>

void foo(int a) { std::cout << " call normal func , val = " << a << std::endl; }

template <typename T> void foo1(T a) {
  std::cout << " call normal template func , val = " << a << std::endl;
}

class Foo {
public:
  void operator()(int a) {
    std::cout << " call functor() , val = " << a << std::endl;
  }
};

template <typename T> class Foo1 {
public:
  void operator()(T a) {
    std::cout << " call template functor() , val = " << a << std::endl;
  }
};

class Bar {
public:
  void func(int a) {
    std::cout << " call class member func , val = " << a << std::endl;
  }

  static void func1(int a) {
    std::cout << " call class member static func , val = " << a << std::endl;
  }
};

template <typename T> class Bar1 {
public:
  void func(T a) {
    std::cout << " call template class member func , val = " << a << std::endl;
  }

  static void func1(T a) {
    std::cout << " call template class member static func , val = " << a
              << std::endl;
  }
};

template<typename T>
using Callbacker = std::function<void(T)> ;

int main() {
  std::function<void(int)> cb;
  //Callbacker<int> cb; 也可以这样声明

  cb = foo;
  cb(0); //普通函数

  void (*fnc_ptr)(int);
  fnc_ptr = foo;
  cb(1); //普通函数指针

  cb = [](int a) {
    std::cout << " call lambda func, val = " << a << std::endl;
  };
  cb(2); // lambda表达式

  cb = foo1<int>;
  cb(3); //模板函数

  cb = Foo();
  cb(4);

  cb = Foo1<int>();
  cb(5);

  Bar b;
  // cb = Bar::func;
  // cb = b.func;
  cb = std::bind(&Bar::func, &b, std::placeholders::_1);
  cb(6);

  cb = Bar::func1;
  cb(7);

  Bar1<int> b1;
  cb = std::bind(&Bar1<int>::func, &b1, std::placeholders::_1);
  cb(8);

  cb = Bar1<int>::func1;
  cb(9);

  return 0;
}
```
程序的输出为：
```shell
 call normal func , val = 0
 call normal func , val = 1
 call lambda func, val = 2
 call normal template func , val = 3
 call functor() , val = 4
 call template functor() , val = 5
 call class member func , val = 6
 call class member static func , val = 7
 call template class member func , val = 8
 call template class member static func , val = 9
```
在上面的代码中 function 的声明方式为 std::function<void(int)>, 一般形式的声明为 std::function<函数类型>, 括号里的是函数的**参数**，括号外的是函数**返回值**
对于类的非静态成员函数, function 必须借用 std::bind 才能够进行封装,在封装模板函数时，需要显示的将模板函数实例化
在上面的代码中，我们显示的定义了函数的参数类型和返回值类型，我们也能够对 function 模板化，如：
```c++
template<typename T,typename T2>
using Callbacker = std::function<T2(T)> ;

double convert(int a){
    return static_cast<double>(a);
}

void test(){
    Callbacker<int,double> cb1;//声明一个参数为 int, 返回值为 double 的 function 对象
    cb1 = convert;
}
```
我们可以根据实际需求来定义模板的个数

# std::bind
std::bind 可以看作一个通用的函数适配器，**它转化一个可调用对象成一个新的可调用对象，来匹配原对象的参数列表**，std::bind 可以将可调用对象与其参数一起进行绑定，绑定后的结果可以用 std::function 保存，其主要有如下作用：
- **将可调用对象与其参数绑定形成一个仿函数**
- **绑定部分参数为定值，减少可调用对象的入参个数**
- **调整函数入参顺序**

使用 bind 的一般形式为：
`auto newCallback = std::bind(callback, arg_list);`
即新的可调用对象是封装了旧调用对象和它的参数列表，在 arg_list 中有时会包含形如 _n 的名字，其中 n 为整数，它表示占位符，表示 newCallback 的参数，他们占据了传递给 newCallback 参数的问题。 std::placeholders::_1 表示 newCallback 的第一个参数，std::placeholders::_2为第二个，以此类推，看如下代码：

```c++
#include <functional>
#include <iostream>

void func(int a, double b, char c) {
  std::cout << "call normal func" << std::endl;
  std::cout << "a = " << a << std::endl;
  std::cout << "b = " << b << std::endl;
  std::cout << "c = " << c << std::endl;
}

class Foo {
public:
  void operator()(int a, double b, char c) {
    std::cout << "call functor()" << std::endl;
    std::cout << "a = " << a << std::endl;
    std::cout << "b = " << b << std::endl;
    std::cout << "c = " << c << std::endl;
  }
};

class Bar {
public:
  void func(int& a, double& b, char c) {
    std::cout << "call class member func " << std::endl;
    std::cout << "a = " << a << std::endl;
    std::cout << "b = " << b << std::endl;
    std::cout << "c = " << c << std::endl;
    std::cout << "b ptr = " << &b << std::endl;
  }
};

int main() {
  double b = 2.1;
  std::cout << "b ptr = " << &b << std::endl;
  auto bindfunc = std::bind(func, std::placeholders::_1, b, 'y');
  bindfunc(2);

  auto bindfunc2 =
      std::bind(Foo(), std::placeholders::_1, std::placeholders::_2, 'y');
  bindfunc2(10, 10.1);

  Bar bar;
  auto bindfunc3 = std::bind(&Bar::func, &bar, std::placeholders::_2,
                             std::placeholders::_1, 'y');
  bindfunc3(20.1, 20);

  auto cb = [](int a, double b, char c) {
    std::cout << "call lambda func " << a << std::endl;
    std::cout << "a = " << a << std::endl;
    std::cout << "b = " << b << std::endl;
    std::cout << "c = " << c << std::endl;
  };
  std::function<void(int,double,char)> bindfunc4 = std::bind(cb, std::placeholders::_3, std::placeholders::_2,
                             std::placeholders::_1);
  bindfunc4('z', 30.1, 30);
}
```
程序的输出为:
```shell
a ptr = 0x7fffe8c7dffc
b ptr = 0x7fffe8c7e000
call normal func
a = 2
b = 2.1
c = y
a ptr = 0x7fffe8c7dffc
b ptr = 0x7fffe8c7e030
call functor()
a = 10
b = 10.1
c = y
call class member func 
a = 20
b = 20.1
c = y
call lambda func 30
a = 30
b = 30.1
c = z
```
由上面的代码可以得出以下结论：
- 预绑定的参数是以**值传递**的形式，不预绑定的参数要用 std::placeholders::_n 的形式占位（即占位符），从_1开始，依次递增，是以**引用传递**的形式；
- std::placeholders_n 表示新的可调用对象的第几个参数，而且与原函数的该占位符所在位置的进行匹配，可以调整 _n 的顺序来改变原函数的入参顺序；
- **bind绑定类成员函数时，第一个参数表示对象的成员函数的指针，第二个参数表示对象的地址，这是因为对象的成员函数需要有this指针**。并且编译器不会将对象的成员函数隐式转换成函数指针，需要通过&手动转换；
- std::bind的返回值是可调用实体，可以直接赋给 std::function。
