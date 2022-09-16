---
title: Bazel学习笔记
date: 2022-08-31 17:30:01
tags:
---

# Bazel简介
Bazel 是一个开源构建和测试工具，类似于 Make、Maven 和 Gradle。它使用人类可读的简要构建语言。Bazel 支持多种语言的项目，并针对多个平台构建输出。Bazel 支持跨多个代码库和大量用户的大型代码库。

<!-- more -->
## Bazel的优点

- Bazel 会根据库、二进制文件、脚本和数据集的概念进行操作，暂时避免对编译器和编译器等工具进行单独调用的复杂性链接器
- Bazel 会缓存之前完成的所有工作，并跟踪对文件内容和构建命令所做的更改。 因此Bazel 仅重新构建更改及其依赖项的内容。如需进一步加快构建速度，可以将项目设置为以高度并行且增量的方式进行构建
- 跨平台，Bazel 可在 Linux、macOS 和 Windows 上运行
- 可扩展，支持扩展 Bazel 以支持任何其他语言或框架

## Bazel的一些概念

### Workspace（工作区）
工作区可以理解为一个repo或者一个工程，其包含了构建软件所需要的源文件；一个Workspace的根目录下需要有一个 `WORKSPACE` 文件，该文件可以为空，也可以包含构建输出所需的外部依赖项的引用。包含名为 WORKSPACE 的文件的目录被视为工作区的根。当一个目录基于包含 WORKSPACE 文件的子目录时，Bazel 将在该工作区中忽略任何子目录树
  
### Packages（包）
代码库中的代码组织的主要单元是软件包。软件包是相关文件的集合，以及有关如何使用这些文件生成输出工件的规范。一个包含 BUILD 文件的目录和其目录下的其他所有文件和子文件夹（不包含 BUILD 文件的子文件夹除外）

### Targets（目标）
软件包是目标的容器，在软件包的 BUILD 文件中定义。 通常target是指一个构建目标；由一个规则（rule）给出，必须有一个名字；

以官方示例代码为例：
```c++
cc_binary(      //编译规则，表示生成一个binary
    name = "hello-world",       //target名字，即生成的binary名字
    srcs = ["hello-world.cc"],      //源文件，是一个列表，可以是多个源文件一起
    deps = [
        ":hello-greet",         //依赖项，：hello-greet 表示依赖在同级目录下的 hello-greet 的 target
        "//lib:hello-time",     //依赖项，//lib:hello-time 表示依赖在项目根目录下的lib目录下的 hello-time 的target
    ],
)
```
- 可以使用 "..." 表示 package 中的所有 targets ，例如 `//test/...`表示 test package 中的所有targets.
引用Target的语法规则是：
`[@workspace][//package/path][:][target]`
当前主项目仓库 (repository) 的名称为 @，可以省略不写

### targets的可见性管理
bazel中所有的target都有可见性属性，如果没设置则用package中设置的默认可见性，package中可以设置以下4种可见性：

> "//visibility:public"  # 完全公开，本worksapce及其他workspace都可以访问  （#为BUILD里注释的方法）
"//visibility:private" # 私有，仅同一package内可访问
"@my_repo//foo/bar:__pkg__"    # 指定某些特定的Target可以访问，这里"__pkg__"是一种特殊语法，表示该package内所有Targets
"//foo/bar:__subpackages__" # 指定某些特定的Target可以访问，这里"__subpackages__"是一种特殊语法，表示该package及其内部所有package的所有Targets

# 使用Bazel构建C++项目
克隆官方给出的示例代码到本地：
`git clone https://github.com/bazelbuild/examples`
进入cpp-tutorial目录，代码的结构如下：
```shell
├── stage1
│   ├── main
│   │   ├── BUILD
│   │   └── hello-world.cc
│   ├── README.md
│   └── WORKSPACE
├── stage2
│   ├── main
│   │   ├── BUILD
│   │   ├── hello-greet.cc
│   │   ├── hello-greet.h
│   │   └── hello-world.cc
│   ├── README.md
│   └── WORKSPACE
└── stage3
    ├── lib
    │   ├── BUILD
    │   ├── hello-time.cc
    │   └── hello-time.h
    ├── main
    │   ├── BUILD
    │   ├── hello-greet.cc
    │   ├── hello-greet.h
    │   └── hello-world.cc
    ├── README.md
    └── WORKSPACE
```
## stage1 构建单个目标
在 stage1 中工作区目录下有一个 main 目录，其中包含了 BUILD 文件和 hello-world.cc 文件，BUILD 文件如下：
```c++
load("@rules_cc//cc:defs.bzl", "cc_binary")

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
)
```
BUILD 文件首先通过 load 函数将 @rules_cc//cc:defs.bzl 文件中的 cc_binary 函数导入，该函数用以定义构建 C++ 可执行文件的规则；而后调用 cc_binary 函数定义构建目标 (target) hello-world 的规则，其中参数 src 指定了源文件
运行以下指令进行构建：
`bazel build //main:hello-world`

输出如下：
```shell
INFO: Analyzed target //main:hello-world (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 0.168s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
```
在构建成功后会在项目根目录（WORKSPACE同级目录）下生成`bazel-bin`, `bazel-out`, `bazel-stage1`, `bazel-testlogs` 四个文件夹，用 ll 可以看出他们只是软链，真正保存的地方是在home目录的.cache/bazel目录下。我们想要的`hello-world`可执行程序在`bazel-bin/main`目录下，在项目根目录下执行`bazel-bin/main/hello-world`
可以得到输出：
```shell
Hello world
Thu Sep  1 14:10:02 2022
```
stage1 的依赖关系图为：
![hello-world关系依赖图](https://bazel.build/static/docs/images/cpp-tutorial-stage1.png)

## stage2 构建多个目标
stage2中在main文件下多了`hello-greet.h`和`hello-greet.cc`文件，对应的BUILD文件为：
```c++
load("@rules_cc//cc:defs.bzl", "cc_binary", "cc_library")

cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
    ],
)
```
在 load() 函数中，引入了 "cc_binary" 和 "cc_library" 规则，这里首先将`hello-greet.h`和`hello-greet.cc`编译成一个名为`hello-greet`的 library target ，然后在`hello-world`中将其作为依赖项写在`deps = []`中，因为他们与`hello-world.cc`在同级目录下，因此只需写成`:hello-greet`即可。编译方法与之前一样，这里不再赘述
当我们修改 `hello-greet.cc` 并重新构建项目，Bazel 只会重新编译该文件
stage2 的依赖图关系为：
![hello-world关系依赖图](https://bazel.build/static/docs/images/cpp-tutorial-stage2.png)

## stage3 构建多个软件包
在 stage3 的目录下有两个目录，每个目录都有自己的 `BUILD` 文件，因此工作区存在两个软件包： `lib`和`main`
lib/BUILD的文件为：
```c++
load("@rules_cc//cc:defs.bzl", "cc_library")

cc_library(
    name = "hello-time",
    srcs = ["hello-time.cc"],
    hdrs = ["hello-time.h"],
    visibility = ["//main:__pkg__"],
)
)
```
main/BUILD的文件为：
```c++
load("@rules_cc//cc:defs.bzl", "cc_binary", "cc_library")

cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
        "//lib:hello-time",
    ],
)
```
在这个例子中，主软件包`hello-world`目标依赖与 lib 软件包中的`hello-time`目标，这也是通过`deps`来让bazel知道
为了成功构建，我们需要使用可见性属性让 `lib/BUILD` 中的 `//lib:hello-time` 目标对 `main/BUILD` 中的目标显示可见，因为默认情况下，目标仅对同一 `BUILD` 文件中的其他目标可见
具体的做法是在 `hello-time` 的编译规则中加入了 `visibility` ,如下所示：
```c++
visibility = ["//main:__pkg__"]
```
stage3 的依赖图关系为：
![hello-world关系依赖图](https://bazel.build/static/docs/images/cpp-tutorial-stage3.png)

# 常见 C++ 构建用例
Bazel build C++有多种规则，详情参考下面的链接：
[Bazel C/C++ 规则](https://bazel.build/reference/be/c-cpp#cc_library)
## 在目标中包含多个文件
可以使用 `glob` 在单个目标中包含多个文件，如：
```c++
cc_library(
    name = "build-all-the-files",
    srcs = glob(["*.cc"]),
    hdrs = glob(["*.h"]),
)
```
Bazel在构建时会在包含此 `BUILD` 文件所在目录（不含子目录）中构建所有的 `.cc` 和 `.h` 文件

## 使用传递性包含
当我们在.h或者.cc/.cpp文件包含某个target里面的.h文件时，我们就应该在deps中加上对该target的依赖，只有直接依赖项需要指定为依赖项，例如，在`sandwich.h`中 #include `bread.h`，在`bread.h`中 #include `flour.h` ，但是 `sandwich.h`中没有直接 #include `flour.h`，故 `BUILD` 文件如下：

```c++
cc_library(
    name = "sandwich",
    srcs = ["sandwich.cc"],
    hdrs = ["sandwich.h"],
    deps = [":bread"],
)

cc_library(
    name = "bread",
    srcs = ["bread.cc"],
    hdrs = ["bread.h"],
    deps = [":flour"],
)

cc_library(
    name = "flour",
    srcs = ["flour.cc"],
    hdrs = ["flour.h"],
)
```
在这里，sandwich 库依赖于 bread 库，后者依赖于 flour 库。虽然没有直接 include 但是我们可以使用在 flour 中定义的全局变量

<!-- ## 添加 include 路径 -->

## 包含外部库
以 Gtest 为例，如果我们想要使用Gtest ，只需在 `WORKSPACE` 文件添加代码库函数下载
```c++
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "com_google_googletest",
  urls = ["https://github.com/google/googletest/archive/609281088cfefc76f9d0ce82e1ff6c30cc3591e5.zip"],
  strip_prefix = "googletest-609281088cfefc76f9d0ce82e1ff6c30cc3591e5",
)
```
然后在需要使用 GTest 的 `BUILD` 的文件的 `deps` 中加上即可
```c++
"@com_google_googletest//:gtest_main",
```

## 添加对预编译库的依赖关系
如果使用只包含已编译版本的库（例如头文件和 .so 文件），需要将其封装在 cc_library 规则中：
```c++
cc_library(
    name = "mylib",
    srcs = ["mylib.so"],
    hdrs = ["mylib.h"],
)
```

参考文献：
[bazel官网](https://bazel.build/)


