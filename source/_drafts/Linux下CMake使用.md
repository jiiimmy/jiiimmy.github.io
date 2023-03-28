---
title: Linux下CMake使用
tags: [Cmake]
---

# 简介
CMake是一个跨平台的编译工具，可以用简单的语句来描述所有平台的编译过程。他能够输出各种各样的makefile或者project文件，能测试编译器所支持的C++特性。CMake 的组态档取名为 CMakeLists.txt。Cmake 并不直接建构出最终的软件，而是产生标准的建构档，然后再以一般的建构方式使用。

下面以几个例子来说明 Linux 下 CMake 的使用

<!-- https://github.com/Akagi201/learning-cmake/blob/master/docs/cmake-practice.pdf -->

# FirstPro
第一个项目将一个源代码文件生成可执行文件，对于这么简单的项目，在 CMakeLists.txt 文件中只需三行即可

在本地输入 `cmake --version`即可看到目前 CMake的版本信息
新建一个 firstpro目录，目录下存在 CMakeLists.txt 和 first.cpp 两个文件

## CMakeLists.txt

```
cmake_minimum_required(VERSION 3.15)

# set the project name
project(FisrtPro)

# add the executable
add_executable(FisrtPro first.cpp)
```
cmake_minimum_required 用来指定 CMake 的最低版本号， project 指定项目名称， add_executable 用来生成可执行文件，需要在这指定生成的可执行文件名称和相关的源文件

## first.cpp

first.cpp 文件用来计算输入程序数字的平方根并打印出来，代码为：
```c++
#include <cmath>
#include <cstdlib>
#include <iostream>
#include <string>

v  const double res = sqrt(val);
  std::cout << "The square root of " << val << " is " << res << std::endl;
  return;
}

int main(int argc, char *argv[]) {
  if (argc < 2) {
    std::cout << "input value for calculation" << std::endl;
    return 1;
  }

  for (int i = 1; i < argc; ++i) {
    const double input = atof(argv[i]);
    CalculateSqrt(input);
  }

  return 0;
}
```

##  构建、编译与运行
进入 firstpro目录，创建一个 build 目录，然后进入 build 目录并运行 CMake 来配置项目，生成构建系统：
```shell
mkdir build && cd build
cmake ..
```
构建系统需要指定 CMakeLists.txt 所在路径，此时在 build 目录而 CMakeLists.txt 在上级目录故使用 `cmake ..`来构建，构建完后build目录会存在以下文件：
```shell
CMakeCache.txt  CMakeFiles  cmake_install.cmake  Makefile
```
此时 build 目录会生成 Makefil 文件，然后调用编译器来实际编译和链接项目：
```shell
cmake --build .
```
--build 指定编译生成的文件存放目录，一般就指定为 Makefile 所在目录，此时 build 目录下会生成我们的目标程序 FirstPro ，执行它：
```shell
./FisrtPro 2 3 4
The square root of 2 is 1.41421
The square root of 3 is 1.73205
The square root of 4 is 2
```
此程序分别计算 2，3，4的平方根，从输出结果看得到了正确的结果，此时的目录结构为：
```shell
.
├── build
├── CMakeLists.txt
└── first.cpp
```
这里我们将编译产物放在了新建的 build 目录下，可以避免编译产物和代码文件混在一起

## 优化 CMakeLists.txt 文件

### set 与 PROJECT_NAME
之前的 CMakeLists.txt 为：
```
cmake_minimum_required(VERSION 3.15)

# set the project name
project(FisrtPro)

# add the executable
add_executable(FisrtPro first.cpp)
```
project指令隐式的定义了两个 cmake 变量:
<projectname>_BINARY_DIR 以及<projectname>_SOURCE_DIR，
这里就是 FirstPro_BINARY_DIR 和 FirstPro_SOURCE_DIR，同时 cmake 系统也帮助我们预定义了 PROJECT_BINARY_DIR 和 PROJECT_SOURCE_DIR
变量，他们的值与 FirstPro_BINARY_DIR, FirstPro_SOURCE_DIR 一致，建议使用 PROJECT_BINARY_DIR 和 PROJECT_SOURCE_DIR

指定了项目名后，后面可能会有多个地方用到这个项目名，当需要改名字时就需要同时改多个地方，可以使用 `PROJECT_NAME`来表示项目名，如：
```
add_executable(${PROJECT_NAME} first.cpp)
```

当生成可执行文件需要指定多个可执行文件，需要用空格隔开，如：
```
add_executable(${PROJECT_NAME} a.cpp b.cpp c.cpp)
```
可以通过 set 来指定一个变量来表示多个源文件来防止修改时同时修改多个地方，如
```
set(SRC_FILES a.cpp b.cpp c.cpp)
add_executable(${PROJECT_NAME} ${SRC_FILES})
```

修改后的 CMakeLists.txt 为：
```
cmake_minimum_required(VERSION 3.15)

# set the project name
project(FisrtPro)

set(SRC_FILES first.cpp)

# add the executable
add_executable(FisrtPro ${SRC_FILES})
```

## 添加版本号和配置文件
我们可以在 CMakeLists.txt 为可执行文件和项目提供一个版本号。首先在 CMakeLists.txt 文件中使用 project 命令设置项目名称和版本号，如：
```
cmake_minimum_required(VERSION 3.15)

project(FisrtPro, VERSION 1.0)
```
然后配置头文件将版本号传递给源代码：
```
configure_file(FirstProConfig.h.in FirstProConfig.h)
```
由于 FirstProConfig.h 文件会自动写入 build 目录，因此必须将该目录添加到搜索头文件的路径列表中，添加方法为将下列语句添加到 CMakeLists.txt 末尾：
```
target_include_directories(${PROJECT_NAME} PUBLIC ${PROJECT_BINARY_DIR})
```
PROJECT_BINARY_DIR 即为生成的可执行文件所在路径，这里就是指 build 所在路径

然后在项目目录下创建 FirstProConfig.h.in 文件，内容如下：
```
#define FirstPro_VERSION_MAJOR @FirstPro_VERSION_MAJOR@
#define FirstPro_VERSION_MINOR @FirstPro_VERSION_MINOR@
```
用 CMake 构建完项目后，在 build 目录下会生成一个 FirstProConfig.h 文件，文件内容为：
```
#define FirstPro_VERSION_MAJOR 1
#define FirstPro_VERSION_MINOR 0
```
然后在 first.cpp 中包含头文件 FirstProConfig.h ，通过一下代码即可打印出版本号：
```c++
std::cout << "Version " << FirstPro_VERSION_MAJOR << "." << FirstPro_VERSION_MINOR << std::endl;
```

## 使用 c++11
将 first.cpp 的 atof 改为 std::atod(), 这是 C++11 的特性，并且删除 `#include<cstdlib>`
```c++
const double input = std::stod(argv[i]);
```

在 CMake 中支持特定 C++标准的最简单方法是使用 CMAKE_CXX_STANDARD 标准变量。在 CMakeLists.txt 中设置 CMAKE_CXX_STANDARD 为11，CMAKE_CXX_STANDARD_REQUIRED 设置为True。确保在 add_executable 命令之前添加 CMAKE_CXX_STANDARD_REQUIRED 命令


## 添加库
当工程逐渐变大时，往往需要将不同功能的代码写在不同的目录，接下来在项目中我们写一个用二分法实现求平方根的接口而不是使用标准库里的接口，在 firstpro 目录下新建 mysqrt 文件夹，然后在该目录下新建 mysqrt.h 和 mysqrt.cpp，主要是实现 mysqrt 函数接口，对应的文件为：
mysqrt.h
```c++
#pragma once
double mysqrt(const double val);
```
mysqrt.cpp
```c++
#include "mysqrt.h"
#include <cmath>

double mysqrt(const double val) {
  const double kThreshold = 1e-9;
  double low = 0, hi = val;
  while (std::fabs(hi - low) > kThreshold) {
    double mid = (low + hi) / 2;
    if (mid * mid > val) {
      hi = mid;
    } else {
      low = mid;
    }
  }
  return low;
}
```
然后在该目录新建 CMakeLists.txt 文件，内容为：
```
add_library(mysqrt mysqrt.cpp)
```
上面的语法表示添加一个叫做 mysqrt 的库文件，其对应的源文件为 mysqrt.cpp。

**在 CMake 中生成的 target 有可执行文件和库文件，分别使用 add_executable 和 add_libaray 命令，语法是先指定文件名/库文件名，然后指定所有相关的源文件**
此时的文件目录结构为：
```shell
.
├── build
├── CMakeLists.txt
├── first.cpp
├── FirstProConfig.h.in
└── mysqrt
    ├── CMakeLists.txt
    ├── mysqrt.cpp
    └── mysqrt.h
```
为了在 first.cpp 中使用 mysqrt 这个库，需要在最顶级 CMakeLists.txt 中添加 `add_subdirectory(mysqrt)` 指令来指定库所在的子目录，该子目录下即包含了对应的 CMakeLists.txt 文件和对应代码文件
要使生成的 binary 使用库文件，需要知道库文件和对应头文件位置，可以分别通过 `target_link_libraries` 和 `target_include_directories` 来指定
最后顶层的 CMakeLists.txt 为：
```
cmake_minimum_required(VERSION 3.15)

# set the project name
project(FirstPro VERSION 1.1)


configure_file(FirstProConfig.h.in FirstProConfig.h)

set(SRC_FILES first.cpp)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)


add_subdirectory(mysqrt)

# add the executable
add_executable(FirstPro ${SRC_FILES})

target_link_libraries(${PROJECT_NAME} mysqrt)

target_include_directories(${PROJECT_NAME} PUBLIC 
                           ${PROJECT_BINARY_DIR}
                           ${PROJECT_SOURCE_DIR}/mysqrt
                           )

```

mysqrt 库添加完成，接下来需要在 first.cpp 中使用该函数，即在头文件中加入:
```c++
#include "mysqrt.h"
```
再使用自定义接口
```c++
const double res = mysqrt(val);
```
接下来重新编译得到 binary 即可

## 使用宏将库设置为可选项
对于大型项目来说，通常需要一份代码适配多种需求，此时可以通过宏来控制最终生成的 binary 使用哪个版本的 sqrt 接口