---
title: C++11之std::function与std::bind
date: 2022-09-16 15:38:22
tags:
---

简介：c++11中新引入的std::function是一个可调用对象包装器，它是一个类模板，可以容纳除了类成员函数指针之外的所有可调用对象，它可以用统一的方式处理函数、函数对象、函数指针、lambda表达式等，并允许保存和延迟它们的执行。
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