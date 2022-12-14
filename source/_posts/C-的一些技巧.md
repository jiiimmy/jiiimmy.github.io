---
title: C++的一些技巧
date: 2022-12-06 15:53:43
tags:
---

<!-- more -->
# C++中 # 和 ## 的用法和作用
## 一个 # 号
在 c/c++ 的宏中， # 的功能是将其后面的宏参数进行字符串化操作，即对其引用的宏变量通过替换后再在其左右各加上一个双引号，如：
```c++
#include <iostream>
#include <string>

#define PRINT_VAR(NAME)                                                        \
  { std::cout << #NAME << " = " << NAME << std::endl; }

int main() {
  int a = 10;
  std::string str = "test str";
  PRINT_VAR(a)
  PRINT_VAR(str)
  return 0;
}
```
运行此程序的输出为：
```shell
a = 10
str = test str
```
我们定义的 PRINT_VAR 宏首先将变量名打印出来，再将变量对应的值给打印出来，这里打印变量名用到了 # 号，将 NAME 所对应的宏变量转化成了字符串

## 两个 # 号
\#\#被称为链接符，##的作用是将##前面与##后面的内容做连接，构成一个新的值，这个新的值不是一个字符串，其可以作为变量名,类名,方法名等
所以在C++中我们可以用##来进行连接，组成一些方法名，变量名，或者类名来简化代码的写法,如：

```c++
#include <iostream>
#include <string>
#include <vector>
#define PRINT_VAR(NAME)                                                        \
  { std::cout << #NAME << " = " << NAME << std::endl; }

class Handlerbase {
public:
  virtual ~Handlerbase() {}
  virtual void callfunc() = 0;
};

static std::vector<Handlerbase *> all_handler;

#define RegisterHandler(NAME)                                                   \
  {                                                                            \
    class NAME##HANDLER : public Handlerbase {                                 \
      virtual void callfunc() {                                                \
        std::cout << #NAME << "HANDLER callfunc called" << std::endl;          \
      }                                                                        \
    };                                                                         \
    all_handler.push_back(new NAME##HANDLER());                                \
  }

int main() {
    RegisterHandler(A);
    RegisterHandler(B);
    RegisterHandler(C);
    for (auto i : all_handler) {
      i->callfunc();
    }

  return 0;
}
```
编译运行结果为：
```shell
AHANDLER callfunc called
BHANDLER callfunc called
CHANDLER callfunc called 
```
这里利用 ## 很方便的创建了几个不同的Handlerbase派生类
