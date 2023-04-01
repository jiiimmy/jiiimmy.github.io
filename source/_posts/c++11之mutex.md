---
title: c++11之mutex详解
date: 2023-02-08 16:10:13
tags:
---

这篇博客详细解析c++11中的头文件 \<mutex\>
<!-- more -->

标准库头文件\<mutex>是c++11多线程支持库的一部分，它包含了以下部分：
**类**：
- mutex 提供基本互斥锁
- timed_mutex 提供互斥锁，并支持超时设置时间
- recursive_mutex 提供能被同一线程递归锁定的互斥锁
- recursive_timed_mutex 提供能被同一线程递归锁定的互斥锁，并支持超时设置时间
- lock_guard 模板类，实现严格基于作用域的互斥锁所有权包装器
- unique_lock 模板类，实现可移动的互斥锁所有权包装器
- scoped_lock (c++17) 用于多个互斥锁的免死锁RAII封装器
- once_flag 确保 call_once 只调用函数一次的辅助对象

**常量**：
- defer_lock 
- try_to_lock
- adopt_lock
这三个用于指定锁的策略

**函数**：
- lock  函数模板，锁定指定的多个互斥锁，若有任何一个不可用则阻塞
- try_lock  函数模板，通过不断的调用 try_lock 去获得多个互斥锁的所有权
- call_once 函数模板，仅调用函数一次，即使从多个线程调用
- std::swap(std::unique_lock) swap函数对 unique_lock 的特化，交互两个 unique_lock 的管理对象

# std::mutex
mutex 类是能用于保护共享数据免受从多个线程同时访问的同步原语
mutex 提供排他性非递归的所有权语义：
- 调用方线程从它成功调用 lock 或 try_lock 开始，到它调用 unlock 为止占有 mutex
- 线程占有 mutex 时，其他所有线程若视图要求 mutex 的所有权，则将阻塞 (对于 lock 的调用)或收到 false 的返回值（对于try_lock）
- 调用方线程在调用 lock 或 try_lock 前必须不占有 mutex

若 mutex 在仍为任何线程所占有时即被销毁，或在占有 mutex 时线程终止，则行为**未定义**

## 构造与析构
std::mutex 只支持默认构造和默认析构，不支持拷贝构造、移动构造、拷贝赋值和移动赋值

## lock 函数
锁定互斥锁，如果另一个线程已经锁定互斥，则 lock 的调用将阻塞执行，直到获得锁

## try_lock 函数
尝试锁定互斥锁并立即返回。成功获得锁时返回 true ，否则返回 false。若已占有 mutex 的线程调用 try_lock ，则行为未定义

## unlock 函数
解锁互斥锁，并且互斥锁必须为当前执行线程所锁定，否则行为未定义。

## native_handle 函数
返回底层实现定义的原生句柄对象

**通常不直接调用 lock 和 unlock，用 std::unique_lock 与 std::lock_guard 来管理互斥锁**

# timed_mutex 
mutex 类是能用于保护共享数据免受从多个线程同时访问的同步原语，类似于 mutex ，timed_mutex 提供排他性非递归的所有权语义。另外，timed_mutex 通过成员函数 try_lock_for() 和 try_lock_until() 来提供一个带超时时间获得 timed_mutex 所有权的能力。
timed_mutex 其他接口与 mutex 基本一致，这里只介绍 try_lock_for 和 try_lock_until

## try_lock_for
尝试锁住互斥锁，阻塞到经过指定的 timeout_duration 或者获取锁，取决于两者谁先来。成功获得锁时返回 true，否则返回 false
若 timeout_duration 小于或等于 timeout_duration.zero() ，则函数表现同 try_lock()。由于调度或资源争夺的延迟，该函数可能阻塞长于 timeout_duration
推荐用 steady_clock 度量时长。若实现用 system_clock 代替，则等待时间亦可能对时钟调整敏感

## try_lock_until
尝试锁住互斥锁，阻塞到指定的 timeout_time 或者获取锁，取决于两者谁先来。成功获得锁时返回 true，否则返回 false
若当前时间已经大于 timeout_time ，则函数表现同 try_lock()。同样由于调度或资源争夺的延迟，该函数可能阻塞到 timeout_time 之后
推荐用 steady_clock 来度量时长。若实现用 system_clock 代替，则阻塞时间亦可能对时钟调整敏感
用法如下：
```c++
#include <thread>
#include <iostream>
#include <chrono>
#include <mutex>
 
std::timed_mutex test_mutex;
 
void func()
{
    auto now=std::chrono::steady_clock::now();
    test_mutex.try_lock_until(now + std::chrono::seconds(10));
    std::cout << "hello world\n";
}
 
int main()
{
    std::lock_guard<std::timed_mutex> l(test_mutex);
    std::thread t(func);
    t.join();
}
```
该程序会在10s阻塞后输出 hello world，因为 test_mutex 在程序运行期间被 l 所持有

# std::recursive_mutex
recursive_mutex 类是能用于保护共享数据免受从多个线程同时访问的同步原语
recursive_mutex 提供排他性的所有权语义：
- 调用方线程在从它成功调用 lock 或 try_lock 开始的时期里占有 recursive_mutex。在此期间，线程可以对 lock 或 try_lock 进行再次调用，所有权的时期在线程调用相应次数的 unlock 时结束
- 线程占有 recursve_mutex 时，若其他线程试图获取 recursive_mutex 的所有权，则他们将阻塞(对于 lock )或收到 false 返回值(对于 try_lock)
- 可锁定 recursive_mutex 次数的最大值是未指定的，但抵达该数后，对 lock 的调用将抛出 std::system_error 异常，对 try_lock 的调用将返回 false

recursive_mutex 的使用场景之一是保护类中的共享状态，而类的成员函数可能相互调用，例：
```c++
#include <iostream>
#include <thread>
#include <mutex>
 
class X {
    std::recursive_mutex m;
    std::string shared;
  public:
    void fun1() {
      std::lock_guard<std::recursive_mutex> lk(m);
      shared = "fun1";
      std::cout << "in fun1, shared variable is now " << shared << '\n';
    }
    void fun2() {
      std::lock_guard<std::recursive_mutex> lk(m);
      shared = "fun2";
      std::cout << "in fun2, shared variable is now " << shared << '\n';
      fun1(); // 递归锁在此处变得有用
      std::cout << "back in fun2, shared variable is " << shared << '\n';
    };
};
 
int main() 
{
    X x;
    std::thread t1(&X::fun1, &x);
    std::thread t2(&X::fun2, &x);
    t1.join();
    t2.join();
}
```
可能的一种输出为：
```shell
in fun1, shared variable is now fun1
in fun2, shared variable is now fun2
in fun1, shared variable is now fun1
back in fun2, shared variable is fun1
```

recursive_mutex 只有 lock, try_lock, unlock, native_handle 四个函数，用法与 mutex 一致

# std::recursive_timed_mutex
std::recursive_timed_mutex 在 std::recursive_mutex 基础上增加了 try_lock_for 和 try_lock_until 两个接口，其用法与 std::timed_mutex 一致

# lock_guard
lock_guard 的定义为：
```c++
template<class Mutex>
class lock_guard;
```
lock_guard 类是互斥锁包装器，为在作用域块期间占有互斥锁提供便利的 RAII 机制
当一个 lock_guard 对象创建时，它就尝试获得给定的 mutex 的所有权，当控制权离开创建 lock_guard 对象的作用域时， lock_guard 会销毁并且释放 mutex
lock_guard 类不可复制

# unique_lock
unique_lock具有lock_guard的所有功能，而且更为灵活
unique_lock 定义为：
```c++
template <class Mutex>
class unique_lock;
```
unique_lock 类是一个通用的 mutex 所有权包装器，它允许延迟锁定，有时限的锁定，递归锁定，转移锁的所有权和与 condition_variable 一起使用
unique_lock 支持移动构造和移动赋值但不支持拷贝构造和拷贝赋值

其中 lock, try_lock, try_lock_for, try_lock_until, unlock 接口用法和其他锁一致

## swap
函数原型为：`void swap( unique_lock& other ) noexcept;`，作用为与其他 unique_lock 对象交换锁对象的内部状态

## release
返回指向互斥锁的指针（若有关联互斥锁），并打断任何与 *this 和 关联mutex 的关联，如果在 release 前 *this 拥有关联 mutex的所有权，那么需要调用方负责解锁互斥锁

## mutex
返回指向关联互斥锁的指针，若没有则返回空指针

## owns_lock
测试 *this 是否锁住其关联的 mutex

## operator bool
检查 *this 是否锁住其关联的 mutex。等效地调用 owns_lock() 

用法示例：
```c++
#include <iostream>
#include <thread>
#include <mutex>
 
struct Box{
 explicit Box(int num): num_things(num) {}
 int num_things;
 std::mutex m;
};

void transfer(Box& from, Box& to, int num){
  //仍未实际取锁
  std::unique_lock<std::mutex> lock1(from.m, std::defer_lock);
  std::unique_lock<std::mutex> lock2(to.m, std::defer_lock);

  //锁两个 unique_lock 而不死锁
  std::lock(lock1, lock2);

  from.num_things -= num;
  to.num_things += num;

  // from.m 与 to.m 解锁于 unique_lock 的析构函数
}

int main() 
{
    Box acc1(100);
    Box acc2(50);
    std::thread t1(transfer, std::ref(acc1), std::ref(acc2),10);
    std::thread t2(transfer, std::ref(acc1), std::ref(acc2),5);
    t1.join();
    t2.join();
    std::cout << "acc1.num_things = " << acc1.num_things <<std::endl;
    std::cout << "acc2.num_things = " << acc2.num_things <<std::endl;
}
```
程序运行结果为：
```shell
acc1.num_things = 85
acc2.num_things = 65
```

## std::scoped_lock
定义为:
```c++
template<calss... MutexTypes>
class scoped_lock;
```
scoped_lock 是在它作用域存在期间为0个或者多个 mutex 提供便利 RAII 机制的互斥锁包装器
创建 scoped_lock 对象时，它试图取得给定互斥锁的所有权。控制流离开创建 scoped_lock 对象的作用域时，析构 scoped_lock 并释放互斥锁。若给出数个互斥锁，则使用免死锁算法，如同以 std::lock 

## std::derfer, std::try_to_lock, std::adopt_lock
这三个的定义为：
```c++
constexpr std::defer_lock_t derfer_lock {};
constexpr std::try_to_lock_t try_to_lock {};
constexpr std::adopt_lock_t adopt_lock {};
```
即 std::defer_lock 、 std::try_to_lock 和 std::adopt_lock 分别是空结构体标签类型 std::defer_lock_t 、 std::try_to_lock_t 和 std::adopt_lock_t 的实例，他们用于为 std::lock_guard, std::unique_lock 及 std::shared_lock 指定锁定策略
| 类型          | 效果                               |
| ------------- | ---------------------------------- |
| defer_lock_t  | 不立即获得互斥锁的所有权               |
| try_to_lock_t | 尝试获得互斥锁的所有权而不阻塞     |
| adopt_lock_t  | 假设调用方线程已拥有互斥锁的所有权 |

## std::try_lock
std::try_lock 定义为：
```c++
template< class Lockable1, class Lockable2, class... LockableN >
int try_lock( Lockable1& lock1, Lockable2& lock2, LockableN&... lockn );
```
尝试锁定每个给定的可锁定 (Lockable) 对象 lock1、 lock2、 ...、 lockn ，通过以从头开始的顺序调用 try_lock 。
若调用 try_lock 失败，则不再进一步调用 try_lock ，并对任何已锁对象调用 unlock ，返回锁定失败对象的 0 底下标。
若调用 try_lock 抛出异常，则在重抛前对任何已锁对象调用 unlock 


## std::lock
std::lock 定义为：
```c++
template< class Lockable1, class Lockable2, class... LockableN >
void lock( Lockable1& lock1, Lockable2& lock2, LockableN&... lockn );
```
用免死锁算法锁定给定的可锁定 (Lockable) 对象 lock1 、 lock2 、 ... 、 lockn 来避免死锁。
以对 lock 、 try_lock 和 unlock 的未指定系列调用锁定对象。若调用 lock 或 unlock 导致异常，则在重抛异常前会对任何已锁的对象调用 unlock 
std::scoped_lock 提供此函数的 RAII 包装，通常它比裸调用 std::lock 更好

## std::once_flag
std::once_flag 类是给 std::call_once 的辅助类，传递给多个 std::call_once 调用的 std::once_flag 对象允许那些调用彼此协调，从而只令调用之一实际运行完成
std::once_flag 既不可复制亦不可移动

## std::call_once
函数定义为：
```c++
template< class Callable, class... Args >
void call_once( std::once_flag& flag, Callable&& f, Args&&... args );
```
它的作用为准确执行一次可调用(Callable)对象 f，即使同时从多个线程调用，具体细节为：
- 若在调用 call_once 的时候，flag 指示 f 已经被调用，那么 call_once 直接返回（称对 call_once 的这种调用为消极）
- 否则，call_once 以参数 `std::forward<Args>(args)... `调用 `std::forward<Callable>(f)` （如同用 std::invoke ）。不同于 std::thread 构造函数或 std::async ，不移动或复制参数，因为不需要转移它们到另一线程去执行（称这种对 call_once 的调用为积极）。
  - 若该调用抛异常，则传播异常给 call_once 的调用方，并且不翻转 flag ，将尝试调用另一个调用（称这种对 call_once 的调用为异常）
  - 若该调用正常返回（称这种对 call_once 的调用为返回），则翻转 flag ，并保证以同一 flag 对 call_once 的其他调用为消极

示例：
```c++
#include <iostream>
#include <thread>
#include <mutex>
 
std::once_flag flag1, flag2;
void simple_do_once(){
  std::call_once(flag1, [](){std::cout <<"Simple example call once\n";});
}

void  may_throw_function(bool do_throw){
  if(do_throw){
    std::cout << "throw : call_once will retry\n";
    throw std::exception();
  }
  std::cout << "Didn't throw, call_once will not attempt again\n";
}

void do_once(bool do_throw){
  try{
    std::call_once(flag2, may_throw_function,do_throw);
  } catch(...){
  }
}

int main() 
{
    std::thread st1(simple_do_once);
    std::thread st2(simple_do_once);
    std::thread st3(simple_do_once);
    std::thread st4(simple_do_once);
    st1.join();
    st2.join();
    st3.join();
    st4.join();
 
    //以下代码在我的机器会出现程序卡死，具体原因不明
    // std::thread t1(do_once, true);  
    // std::thread t2(do_once, true);
    // std::thread t3(do_once, false);
    // std::thread t4(do_once, true);
    // t1.join();
    // t2.join();
    // t3.join();
    // t4.join();
    return 0;
}
```
程序的运行结果为：
```shell
Simple example call once
```

# shared_mutex
shared_mutex (c++17) 类是一个同步原语，可用于保护共享数据不被多个线程同时访问。与便于独占访问的其他互斥类型不同，shared_mutex 拥有二个访问级别：
- 共享（读） 多个线程能共享同一个互斥锁的所有权
- 独占（写） 仅一个线程能占有互斥锁
若一个线程已获取独占性锁（通过 lock 、 try_lock ），则无其他线程能获取该锁（包括共享的）
仅当任何线程均未获取独占性锁时，共享锁能被多个线程获取（通过 lock_shared 、 try_lock_shared
在一个线程内，同一时刻只能获取一个锁（共享或独占性）
std::shared_mutex 在由多个线程同时读共享数据时特别有用，但是一个线程只能在没有其他线程进行读写时进行写操作

std::shared_mutex 比 std::mutex 多了 lock_shared, try_lock_shared, unlock_shared 三个接口，对应着共享方式的 lock, try_lock 和 unlock

# shared_timed_mutex
shared_timed_mutex（c++14） 在 shared_mutex 基础上增加了  try_lock_for() 、 try_lock_until() 、 try_lock_shared_for() 、 try_lock_shared_until() 来增加其超时获取所有权的能力

# shared_lock
shared_lock 类是通用的共享互斥所的所有权包装器，允许延迟锁定、定时锁定和锁的所有权转移。锁定 shared_lock 会以共享模式锁定关联的共享互斥锁（std::unique_lock 可用于以排他性模式锁定）
除了锁定的锁类型为 shared_mutex 和 shared_timed_mutex 外，其他接口和功能和 unique_lock 基本一致



```c++
#include <iostream>
#include <mutex>
#include <shared_mutex>
#include <thread>
 
class ThreadSafeCounter {
 public:
  ThreadSafeCounter() = default;
 
  // 多个线程/读者能同时读计数器的值。
  unsigned int get() const {
    std::shared_lock<std::shared_mutex> lock(mutex_);//读时用 shared_lock
    return value_;
  }
 
  // 只有一个线程/写者能重置/写线程的值。
  void increment() {
    std::unique_lock<std::shared_mutex> lock(mutex_);//写时用 unique_lock
    ++value_;
  }
 
  // 只有一个线程/写者能重置/写线程的值。
  void reset() {
    std::unique_lock<std::shared_mutex> lock(mutex_);
    value_ = 0;
  }
 
 private:
  mutable std::shared_mutex mutex_;
  unsigned int value_ = 0;
};
 
int main() {
  ThreadSafeCounter counter;
 
  auto increment_and_print = [&counter]() {
    for (int i = 0; i < 3; i++) {
      counter.increment();
      std::cout << std::this_thread::get_id() << ' ' << counter.get() << '\n';
 
      // 注意：写入 std::cout 实际上也要由另一互斥同步。省略它以保持示例简洁。
    }
  };
 
  std::thread thread1(increment_and_print);
  std::thread thread2(increment_and_print);
 
  thread1.join();
  thread2.join();
}
```
可能的一个结果为：
```shell
140644930766592 2
140644930766592 3
140644930766592 4
140644922373888 4
140644922373888 5
140644922373888 6
```


# mutex 与 atomic 的区别

在C++多线程编程中经常遇到的问题就是共享资源的安全问题，即多个线程如何安全高效的操作某一个共享变量。本文以自带数据类型和自定义数据类型来做比较

```c++
#include <chrono>
#include <iostream>
#include <mutex>
#include <thread>
#include <atomic>

std::atomic<int> num{0};
std::mutex mtx;

static const int numthread = 4;

void click() { //统计点击量
  for (int i = 0; i < 10000000; ++i) {
    ++num;//对于atomic<int>类型有重载 ++ 运算符
  }
}

int main() {
  auto start = std::chrono::steady_clock::now();

  std::thread threads[numthread];// 4个线程，每个线程点击1000000次

  for (int i = 0; i < numthread; ++i) {
    threads[i] = std::thread(click);
  }

  for (int i = 0; i < numthread; ++i) {
    threads[i].join();
  }

  auto end = std::chrono::steady_clock::now();

  auto duration =
      std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count();
  std::cout << "duration time: " << duration << " ns\n";

  std::cout << "result = " << num << std::endl;

  return 0;
}
```
程序运行结果为：
```shell
duration time: 674953294 ns
result = 40000000
```
使用原子变量

将原子变量改成普通变量，再用mutex来对每次写动作做保护，代码为：
```c++
#include <chrono>
#include <iostream>
#include <mutex>
#include <thread>
#include <atomic>

int num = 0;
std::mutex mtx;

static const int numthread = 4;

void click() { //统计点击量
  for (int i = 0; i < 10000000; ++i) {
    std::lock_guard<std::mutex> lk(mtx);
    ++num;
  }
}

int main() {
  auto start = std::chrono::steady_clock::now();

  std::thread threads[numthread];// 4个线程，每个线程点击1000000次

  for (int i = 0; i < numthread; ++i) {
    threads[i] = std::thread(click);
  }

  for (int i = 0; i < numthread; ++i) {
    threads[i].join();
  }

  auto end = std::chrono::steady_clock::now();

  auto duration =
      std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count();
  std::cout << "duration time: " << duration << " ns\n";

  std::cout << "result = " << num << std::endl;

  return 0;
}
```

运行结果为：
```shell
duration time: 3902193722 ns
result = 40000000
```
原因就在于锁机制，虽然确保了数据安全，但是却也导致性能急速下降
