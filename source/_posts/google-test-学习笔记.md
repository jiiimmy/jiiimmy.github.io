---
title: Google Test 学习笔记
date: 2022-08-28 05:43:25
tags: ["gtest","c++"]
---


# 简介
Google Test（简称GTest）是谷歌开源的一个跨平台的（Linux、Mac、Windows等）C++单元测试框架，能够很方便的帮助我们测试自己写的C++程序是否符合预期。不仅如此，它还提供了丰富的断言、致命和非致命判断、参数化、“死亡测试”等。

<!-- more -->
[GTest 官网](https://google.github.io/googletest/)
[GTest 仓库](https://github.com/google/googletest)

# 单元测试
单元测试（unit testing）是指对软件的最小可测试单元进行正确性测试，以检验程序单元是否满足功能、性能、接口、设计规约等要求。单元可以是一个函数、一个方法、一个类、一个功能模块或者一个子系统

单元测试本质上也是代码，与普通代码的区别在于它是验证代码正确性的代码。单元测试有如下优点：
- 便于后期重构。单元测试可以为代码的重构提供保障，只要重构代码之后单元测试全部运行通过，那么在很大程度上表示这次重构没有引入新的BUG，当然这是建立在完整、有效的单元测试覆盖率的基础上。
- 优化设计。编写单元测试将使用户从调用者的角度观察、思考，会让使用者把程序设计成易于调用和可测试，并且尽量降低软件中的耦合。
- 文档记录。单元测试就是一种无价的文档，它是展示函数或类如何使用的最佳文档，这份文档是可编译、可运行的、并且它保持最新，永远与代码同步。
- 具有回归性。自动化的单元测试避免了代码出现回归，编写完成之后，可以随时随地地快速运行测试。
  
# GTest的优点
1. 测试时独立和可重复的，GTest使每个测试用例运行在不同的对象上，从而使测试之间相互独立。当测试失效时，GTest允许单独运行它以进行快速调试
2. 测试有良好的组织，可以反映被测试代码的结构。 GoogleTest 将相关测试划分到一个测试组内，组内的测试能共享数据，使测试易于维护。
3. 测试是可移植的和可重复使用的。 与平台无关的代码，其测试代码也应该和平台无关，GoogleTest 能在不同的操作系统下工作，并且支持不同的编译器。
4. 当测试用例执行失败时，提供尽可能多的有效信息，以便定位问题。 GoogleTest 不会在第一次测试失败时停止。相反，它只会停止当前测试并继续下一个测试。还可以设置报告非致命故障的测试，然后继续当前测试。因此，您可以在单个运行-编辑-编译周期中检测和修复多个错误。

# 搭建环境
这里我选择Bazel作为编译工具，后面专门写一个关于Bazel的学习博客。首先在我的home目录下创建workspace
```shell
cd ~ && mkdir workspace && cd workspace
```
然后在workspace下创建一个WORKSPACE 文件，并在其中引入外部依赖 GoogleTest，此时Bazel 会自动去Github 上拉取GTest的相关文件，WORKSPACE中的内容为：
```shell
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
  name = "com_google_googletest",
  urls = ["https://github.com/google/googletest/archive/609281088cfefc76f9d0ce82e1ff6c30cc3591e5.zip"],
  strip_prefix = "googletest-609281088cfefc76f9d0ce82e1ff6c30cc3591e5",
)
```
然后在workspace下创建first_gtest.cc文件：
```c++
#include <gtest/gtest.h>

// Demonstrate some basic assertions.
TEST(HelloTest, BasicAssertions) {
  // Expect two strings not to be equal.
  EXPECT_STRNE("hello", "world");
  // Expect equality.
  EXPECT_EQ(7 * 6, 42);
  EXPECT_EQ(1+1, 3);
}
```

同时在目录下创建BUILD文件并说明目标编译的规则：
```shell
cc_test(
    name = "first_gtest",
    size = "small",
    srcs = ["first_gtest.cc"],
    deps = ["@com_google_googletest//:gtest_main"],
)
```
在终端执行以下命令即可构建并运行测试程序：
`bazel test --test_output=all //:hello_test `
注：若想再次运行程序，可以在同目录的bazel-bin文件下找到first_gtest的程序 ：)
程序的输出为：
```shell
Running main() from gmock_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from HelloTest
[ RUN      ] HelloTest.BasicAssertions
first_gtest.cc:9: Failure
Expected equality of these values:
  1+1
    Which is: 2
  3
[  FAILED  ] HelloTest.BasicAssertions (1 ms)
[----------] 1 test from HelloTest (1 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (1 ms total)
[  PASSED  ] 0 tests.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] HelloTest.BasicAssertions

 1 FAILED TEST
```
我们并没有在first_gtest.cc中定义main()函数，第1行的输出说明GTest已经提供了main()入口，我们写的TEST函数在gmock_main.cc中被调用，这个函数是在BUILD的`"@com_google_googletest//:gtest_main"`中所引入。 第5行表明开始RUN我们写的TEST函数，然后输出了一个FAILED消息，因为我们在first_gtest.cc的第9行写了1+1的期望值是3，但是实际输出值是2，故该TEST没有通过测试，虽然第7、8行的代码通过了测试。



参考文献：
[一文掌握谷歌 C++ 单元测试框架 GoogleTest](https://zhuanlan.zhihu.com/p/544491071)