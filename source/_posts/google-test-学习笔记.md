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

# GTest中的概念

## 断言
在GTest中，是通过断言（Assertion）来判断代码实现的功能是否符合预期。断言的结果分为成功（success）、非致命错误（non-fatal failture）和致命错误（fatal failture）
- 成功：断言成功，程序的行为符合预期，程序允许继续向下运行。可以在代码中调用 SUCCEED() 来表示这个结果
- 非致命错误：断言失败，但是程序没有直接中止，而是继续向下运行。可以在代码中调用 FAIL() 来表示这个结果
- 致命错误：中止当前函数或者程序崩溃。可以在代码中调用 ADD_FAILURE() 来表示这个结果

GTest的断言是类似于函数调用的宏。通过对其行为进行断言来测试一个类或函数。当断言失败时，GTest会打印断言的源文件和行号位置以及失败消息。还可以提供自定义失败消息，该消息将附加到GTest的消息中

宏的格式有两种：
> EXPECT_* ：在失败时会产生非致命故障，不会中止当前功能
> ASSERT_*：在失败时会产生致命错误，并中止当前函数，在失败用例后面的语句将不再执行

通常 EXPECT_* 是首选，因为它们允许在测试中报告多个。但是如果在某条件不成立，程序就无法运行时，就应该使用 ASSERT_\*，需要注意的是 ASSERT_\* 是直接从当前函数返回的，可能会导致一些内存、文件资源没有释放，因此存在内存泄漏的问题。

GTest提供了一组断言，可以用各种方式来验证代码的行为。包括检查布尔条件、基于关系运算符比较值、验证字符串值、浮点值等等。甚至还有一些断言可以通过提供自定义谓词来验证更复杂的状态。有关 GoogleTest 提供的断言的完整列表，可以参考官方文档：[Assertions Reference](https://google.github.io/googletest/reference/assertions.html)

如果想自定义失败时的信息，只需要使用流操作符 << 将这些信息流到断言宏中，并且这些信息只有在失败时会显示出来，例如：
```c++
ASSERT_EQ(x.size(), y.size()) << "Vectors x and y are of unequal length";

for (int i = 0; i < x.size(); ++i) {
  EXPECT_EQ(x[i], y[i]) << "Vectors x and y differ at index " << i;
}
```
任何可以流向 std::ostream 的东西都可以流向断言宏，特别是 C 语言的字符串（char*）和 std::string 对象。

## TEST 宏函数
如果想要创建测试，可以使用宏函数 TEST，它具有以下特点：
- TEST 是一个没有返回值的宏函数
- 我们可以在该函数中使用断言来检测代码是否符合预期，测试的结果由断言决定。如果测试中的任何断言失败（致命或非致命），或者如果测试崩溃，测试接管会报失败

TEST函数定义为：
```c++
TEST(TestSuiteName, TestName) {
  ... test body ...
}
```
TestSuiteName 对应测试用例集名称，TestName 是归属的测试用例名称。这两个名称都必须是有效的 C++ 标识符，并且它们不应包含任何下划线。测试的全名由其包含的测试用例集及其测试名称组成。来自不同测试用例集的测试可以具有相同的名称。

这里以我自己写的一个简单版本的判断一个数是否为质数函数为例：

```c++
#include <gtest/gtest.h>

bool is_prime(int num) {
  if (num < 2) {
    return false;
  }
  for (int i = 2; i * i <= num; ++i) {
    if (num % i == 0) {
      return false;
    }
  }
  return true;
}

//测试负数
TEST(PRIMETEST, Negative) {
  for (int i = -10; i < -1; ++i) {
    EXPECT_FALSE(is_prime(i)) << "i index is" << i;
  }
}

//测试0
TEST(PRIMETEST, Zero) { EXPECT_FALSE(is_prime(0)); }v

//测试正数
TEST(PRIMETEST, Positive) {
    EXPECT_FALSE(is_prime(1));
    EXPECT_TRUE(is_prime(2));
    EXPECT_FALSE(is_prime(4));
    EXPECT_TRUE(is_prime(7));
    EXPECT_TRUE(is_prime(6203));
    EXPECT_FALSE(is_prime(10000));
}
```

这里为了能够更好的组织测试用例，将数据根据正负数划分为三个测试用例集，每一个用例集中都是相同类型的测试用例，执行测试的输出如下：
```shell
Running main() from gmock_main.cc
[==========] Running 3 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 3 tests from PRIMETEST
[ RUN      ] PRIMETEST.Negative
[       OK ] PRIMETEST.Negative (0 ms)
[ RUN      ] PRIMETEST.Zero
[       OK ] PRIMETEST.Zero (0 ms)
[ RUN      ] PRIMETEST.Positive
[       OK ] PRIMETEST.Positive (0 ms)
[----------] 3 tests from PRIMETEST (0 ms total)

[----------] Global test environment tear-down
[==========] 3 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 3 tests.
```
这就表示我们的代码通过了我给出的所有测试用例

## TEST_F 宏函数
当我们写了两个或多个对相似数据进行操作的测试，可以使用test fixture来为多个测试重用这些相同的配置，其对应的宏函数为TEST_F，函数定义为：
```c++
TEST_F(TestFixtureName, TestName) {
  ... test body ...
}
```
TestFixtureName 对应一个test fixture类的名称。因此我们需要自己去定义一个这样的类，并让它继承testing::Test类，然后根据我们的需要实现下面这两个虚函数：
- `virtual void SetUp()` ：类似于构造函数，在TEST_F之前运行
- `virtual void TearDown()` ：类似于析构函数，在TEST_F之后运行

此外`testing::Test`基类还提供了两个静态函数：
- `static void SetUpTestSuite()` ：在第一个TEST之前运行
- `static void TearDownTestSuite()`：在最后一个TEST之后运行

除了这两种，还有一种全局事件，即继承``，并实现以下两个虚函数：
`virtual void SetUp()`：在所有用例之前运行
`virtual void TearDown()`：在所有用例之后运行

以GTest官网的sample3为例，在queue.h中实现如下：
```c++
#pragma once 

#include <stddef.h>

template <typename E>
class Queue;

template <typename E>
class QueueNode {
  friend class Queue<E>;

 public:
  const E& element() const { return element_; }

  QueueNode* next() { return next_; }
  const QueueNode* next() const { return next_; }

 private:
  explicit QueueNode(const E& an_element)
      : element_(an_element), next_(nullptr) {}

  const QueueNode& operator=(const QueueNode&);
  QueueNode(const QueueNode&);

  E element_;
  QueueNode* next_;
};

template <typename E>  
class Queue {
 public:
  Queue() : head_(nullptr), last_(nullptr), size_(0) {}

  ~Queue() { Clear(); }

  void Clear() {
    if (size_ > 0) {
      QueueNode<E>* node = head_;
      QueueNode<E>* next = node->next();
      for (;;) {
        delete node;
        node = next;
        if (node == nullptr) break;
        next = node->next();
      }

      head_ = last_ = nullptr;
      size_ = 0;
    }
  }

  size_t Size() const { return size_; }

  QueueNode<E>* Head() { return head_; }
  const QueueNode<E>* Head() const { return head_; }

  QueueNode<E>* Last() { return last_; }
  const QueueNode<E>* Last() const { return last_; }

  void Enqueue(const E& element) {
    QueueNode<E>* new_node = new QueueNode<E>(element);

    if (size_ == 0) {
      head_ = last_ = new_node;
      size_ = 1;
    } else {
      last_->next_ = new_node;
      last_ = new_node;
      size_++;
    }
  }

  E* Dequeue() {
    if (size_ == 0) {
      return nullptr;
    }

    const QueueNode<E>* const old_head = head_;
    head_ = head_->next_;
    size_--;
    if (size_ == 0) {
      last_ = nullptr;
    }

    E* element = new E(old_head->element());
    delete old_head;

    return element;
  }

  template <typename F>
  Queue* Map(F function) const {
    Queue* new_queue = new Queue();
    for (const QueueNode<E>* node = head_; node != nullptr;
         node = node->next_) {
      new_queue->Enqueue(function(node->element()));
    }

    return new_queue;
  }

 private:
  QueueNode<E>* head_;
  QueueNode<E>* last_;
  size_t size_;
  Queue(const Queue&);
  const Queue& operator=(const Queue&);
};
```
对应的queue_test.cpp文件如下，为了了解其构建对象的顺序，我们加上了构造和析构函数：
```c++
#include "queue.h"
#include <gtest/gtest.h>

namespace {
class QueueTestSmpl3 : public testing::Test {
public:
  QueueTestSmpl3() {
    std::cout << "call queue test ctor" <<std::endl;
  }
  ~QueueTestSmpl3() {
    std::cout << "call queue test dtor" <<std::endl;
  }
protected:
  void SetUp() override {
    q1_.Enqueue(1);
    q2_.Enqueue(2);
    q2_.Enqueue(3);
  }

  static int Double(int n) { return 2 * n; }

  // 测试 Queue::Map() 时用到的辅助函数
  void MapTester(const Queue<int> *q) {
    const Queue<int> *const new_q = q->Map(Double);

    ASSERT_EQ(q->Size(), new_q->Size());

    for (const QueueNode<int> *n1 = q->Head(), *n2 = new_q->Head();
         n1 != nullptr; n1 = n1->next(), n2 = n2->next()) {
      EXPECT_EQ(2 * n1->element(), n2->element());
    }

    delete new_q;
  }

  Queue<int> q0_;
  Queue<int> q1_;
  Queue<int> q2_;
};

//测试默认构造函数
TEST_F(QueueTestSmpl3, DefaultConstructor) { EXPECT_EQ(0u, q0_.Size()); }

//测试Dequeue
TEST_F(QueueTestSmpl3, Dequeue) {
  int *n = q0_.Dequeue();
  EXPECT_TRUE(n == nullptr);

  n = q1_.Dequeue();
  ASSERT_TRUE(n != nullptr);
  EXPECT_EQ(1, *n);
  EXPECT_EQ(0u, q1_.Size());
  delete n;

  n = q2_.Dequeue();
  ASSERT_TRUE(n != nullptr);
  EXPECT_EQ(2, *n);
  EXPECT_EQ(1u, q2_.Size());
  delete n;
}

//测试Map函数
TEST_F(QueueTestSmpl3, Map) {
  MapTester(&q0_);
  MapTester(&q1_);
  MapTester(&q2_);
}
} // namespace
```
我们看一下QueueTestSmpl3的实现，这个类中实现了一些测试时用到的辅助函数，以及使用 SetUp预置了一些测试数据。（除了有特殊需求，则不需要实现TearDown，因为析构函数已经帮我们释放了资源）
该目录下的BUILD文件为：
```shell
cc_test(
  name = "queue_test",
  size = "small",
  srcs = ["queue_test.cpp"],
  deps = ["@com_google_googletest//:gtest_main",
          "//:queue"
  ],
)

cc_library(
  name = "queue",
  hdrs = ["queue.h"]
)
```
然后运行以下指令执行测试：
```shell
bazel test --test_output=all //:queue_test
```
得到的输出为：
```shell
Running main() from gmock_main.cc
[==========] Running 3 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 3 tests from QueueTestSmpl3
[ RUN      ] QueueTestSmpl3.DefaultConstructor
call queue test ctor
call queue test dtor
[       OK ] QueueTestSmpl3.DefaultConstructor (0 ms)
[ RUN      ] QueueTestSmpl3.Dequeue
call queue test ctor
call queue test dtor
[       OK ] QueueTestSmpl3.Dequeue (0 ms)
[ RUN      ] QueueTestSmpl3.Map
call queue test ctor
call queue test dtor
[       OK ] QueueTestSmpl3.Map (0 ms)
[----------] 3 tests from QueueTestSmpl3 (0 ms total)

[----------] Global test environment tear-down
[==========] 3 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 3 tests.
```

从输出中以 DefaultConstructor 为例，来分析一下它的执行流程：
1. QueueTestSmpl3 调用构造函数来构造出一个对象
2. QueueTestSmpl3 对象调用SetUp函数进行初始化
3. DefaultConstructor 开始执行并结束测试
4. QueueTestSmpl3 对象调用隐式生成的 TearDown 进行清理
5. QueueTestSmpl3 调用析构函数，析构对象

这里我们写了3个TEST_F函数，每一个函数的流程都是如此，从输出结果可以看到这个流程是按顺序执行

## 运行测试
TEST 和 TEST_F 宏函数会隐式地向 GoogleTest 注册这些测试函数，因此我们不需要为了运行这些它们而进行枚举。定义完测试后，你可以用 RUN_ALL_TESTS 来运行它们，此时会运行所有的测试，如果全部成功则返回 0，反之则返回 1。其执行流程如下：
1. 保存所有 GoogleTest 标志的状态
2. 为第一个测试创建一个 test fixture 对象
3. 通过 SetUp 初始化 test fixture 对象
4. 在 test fixture 对象上进行测试
5. 通过 TearDown 清理 test fixture
6. 删除 test fixture
7. 恢复所有 GoogleTest 标志的状态
8. 对下一个测试重复上述步骤，直到所有测试都运行完毕

如果发生 fatal failure ，则将跳过后续步骤

## 编写main()函数
用户不需要编写自己的 main 函数，编译器会自动将它链接到 gtest_main。如果用户有特殊需求，需要编写一个 main 函数，调用 RUN_ALL_TESTS() 来运行所有测试，并且只能调用一次 RUN_ALL_TESTS()函数，同时必须要在main()函数中返回 RUN_ALL_TESTS()的返回值
例如：
```c++
int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```
`testing::InitGoogleTest`函数会解析 GoogleTest 标志的命令行，并删除所有识别的标志。这允许用户通过各种标志控制测试程序的行为。必须在调用RUN_ALL_TESTS之前调用此函数 ，否则标志将无法正确初始化。

参考文献：
[Googletest Primer](https://google.github.io/googletest/primer.html)
[一文掌握谷歌 C++ 单元测试框架 GoogleTest](https://zhuanlan.zhihu.com/p/544491071)