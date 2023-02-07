---
title: C++11并发之future
date: 2023-01-31 10:26:49
tags:
---

本文将介绍 C++11 中的 \<future> 头文件中的类和相关函数

<!-- more -->
std::thread 是C++11中提供异步创建多线程的工具，只能是异步运行任务，却无法获取任务执行的结果，一般都是依靠全局对象，全局对象在多线程下是及其不安全的，为此标准库提供了 std::future 类模板来关联线程运行的函数和函数的返回结果，这种获取结果的方式是异步的
\<future> 头文件中包含了以下几个类和函数：
- Providers 类： std::promise, std::package_task
- Futures 类： std::future, std::shared_future
- Providers 类： std::async()
- 其他类型： std::future_error, std::future_errc, std::future_status, std::lauch

# std::promise 类
promise 对象可以保存某一类型 T 的值，该值可以被 future 对象读取（可能在另一线程中读取），因此 promise 也提供了一种线程同步的手段。在 promise 对象构造时可以和一个共享状态（通常是 std::future ）相关联，并可以在相关联的共享状态上保存一个类型为 T 的值。

可以通过 promise 的 get_future 接口来获取与改 promise 对象相关联的 future 对象，调用该函数后，两个对象（promise对象和future对象）的共享相同的共享状态(shared state)
- promise 对象是异步 Provider ，它可以在某一时刻设置共享状态的值
- future 对象可以异步返回共享状态的值，或者在必要时阻塞调用者并等待状态标志变为 ready 才获取共享状态的值

以一个简单的例子来说明上述关系：
```c++
#include <chrono>
#include <future>
#include <iostream>
#include <time.h>
using namespace std::chrono_literals;

struct A {
  int val;
  A() : val(0) {}
  A(int x) : val(x) {}
};//自定义一个 A 对象

void read(std::future<A> &fut) {//读线程
  auto start = std::chrono::high_resolution_clock::now();
  A x = fut.get();//获取共享状态的值
  std::cout << "val = " << x.val << std::endl;
  auto end = std::chrono::high_resolution_clock::now();
  std::chrono::duration<double, std::milli> elapsed = end - start; //计算获取到共享状态值的时间
  std::cout << "Waited " << elapsed.count() << " ms\n";
}

void write(std::promise<A> &prom) {//写线程
  std::this_thread::sleep_for(1000ms);//写线程阻塞1000ms
  A val(10);//构造 A 对象
  prom.set_value(val);//设置共享状态的值
}

int main() {
  std::promise<A> prom;//生成一个 std::promise<A> 对象
  std::future<A> fut = prom.get_future(); //和 future 相关联

  std::thread t1(read, std::ref(fut)); //将 future 交给读线程
  std::thread t2(write, std::ref(prom)); //将 promise 交给写线程

  t1.join();
  t2.join();
  return 0;
}
```
使用编译指令`g++ test.cpp -o a.out -pthread `然后运行此程序结果为：
```shell
val = 10
Waited 1000.32 ms
```
可以看到我们在写线程中主动阻塞了1000ms后再对 prom 设置状态，读线程里面被阻塞在了 `fut.get()`语句，因为共享状态在写线程 sleep 1000ms 后才被设置

## promise 构造函数
从 future 文件中可以看到 promise 构造函数有：
```c++
promise();
promise(promise&& __rhs) noexcept;
template<typename _Allocator>
promise(allocator_arg_t, const _Allocator& __a);

template<typename _Allocator>
promise(allocator_arg_t, const _Allocator&, promise&& __rhs)

promise(const promise&) = delete;
```
默认构造函数用来初始化一个空的共享状态
promise **支持移动构造**但是**禁止拷贝构造**
promise 还支持之定义的分配器来分配共享状态的默认构造和移动构造

另外，std::promise 的 operator= 没有拷贝语义，**只有 move 语义**，即 std::promise 对象是禁止拷贝的，源码为：
```c++
promise& operator=(promise&& __rhs) noexcept;
promise& operator=(const promise&) = delete;
```
例子：
```c++
#include <iostream>
#include <thread>
#include <future>

std::promise<int> prom;

void read_global_promise () {
    std::future<int> fut = prom.get_future();
    int x = fut.get();
    std::cout << "value: " << x << '\n';
}

int main ()
{
    std::thread th1(read_global_promise);
    prom.set_value(10);
    th1.join();

    prom = std::promise<int>();    // prom 被move赋值为一个新的 promise 对象，如果注释这段，程序会 core dump

    std::thread th2 (read_global_promise);
    prom.set_value (20);
    th2.join();

  return 0;
}
```
## get_future 函数
该函数返回一个与 promise 共享状态相关联的 future 。返回的 future 对象可以访问由 promise 对象设置在共享状态上的值或者某个异常对象。只能从 promise 共享状态获取一个 future 对象。在调用该函数之后，promise 对象通常会在某个时间点准备好(设置一个值或者一个异常对象)，如果不设置值或者异常，promise 对象在析构时会自动地设置一个 future_error 异常(broken_promise)来设置其自身的准备状态

## set_value 函数
```c++
void  set_value(const _Res& __r);
void  set_value(_Res&& __r);
```
改函数设置共享状态的值，设置完后 promise 的共享状态变为 ready (还有2个模板特化版本的接口，未列出)

## set_exception 函数
为 promise 设置异常，此后 promise 的共享状态变标志变为 ready

## set_value_at_thread_exit 函数
```c++
void  set_value_at_thread_exit(const _Res& __r);
void  set_value_at_thread_exit(_Res&& __r);
```
设置共享状态的值，但是不将共享状态的标志设置为 ready，当线程退出时该 promise 对象会自动设置为 ready。如果某个 std::future 对象与该 promise 对象的共享状态相关联，并且该 future 正在调用 get，则调用 get 的线程会被阻塞，当线程退出时，调用 future::get 的线程解除阻塞，同时 get 返回 set_value_at_thread_exit 所设置的值。注意，该函数已经设置了 promise 共享状态的值，如果在线程结束之前有其他设置或者修改共享状态的值的操作，则会抛出 future_error( promise_already_satisfied )。

## set_exception_at_thread_exit 函数
其含义与 set_value_at_thread_exit 基本一致，只是设置的是异常

## swap
交换两个 swap 的共享状态

# std::packaged_task
std::packaged_task 封装一个可调用的对象，并且允许异步获取该可调用对象产生的结果，从封装可调用对象意义上来讲，std::packaged_task 与 std::function 类似，只不过 std::packaged_task 将其封装的可调用对象的执行结果传递给一个 std::future 对象（该对象通常在另外一个线程中获取 std::packaged_task 任务的执行结果）。

std::packaged_task 对象内部包含了两个最基本元素，一、被封装的任务(stored task)，任务(task)是一个可调用的对象，如函数指针、成员函数指针或者函数对象，二、共享状态(shared state)，用于保存任务的返回值，可以通过 std::future 对象来达到异步访问共享状态的效果。

可以通过 std::packged_task::get_future 来获取与共享状态相关联的 std::future 对象。在调用该函数之后，两个对象共享相同的共享状态，具体解释如下：
- std::packaged_task 对象是异步 Provider，它在某一时刻通过调用被封装的任务来设置共享状态的值。
- std::future 对象是一个异步返回对象，通过它可以获得共享状态的值，前提是共享状态标志变为 ready

std::packaged_task 的共享状态的生命周期一直持续到最后一个与之相关联的对象被释放或者销毁为止,例：
```c++
#include <chrono>
#include <future>
#include <iostream>

struct A {
  int val;
  A() : val(0) {}
  A(int x) : val(x) {}
};

A countdown(int time) {//计时 time 秒s返回一个 A 对象
  for (int i = 0; i != time; ++i) {
    std::cout << time - i << '\n';
    std::this_thread::sleep_for(std::chrono::seconds(1));
  }
  std::cout << "countdown Finished!\n";
  return A(12);
}

int main() {
  std::packaged_task<A(int)> task(countdown); // 设置 packaged_task
  std::future<A> ret = task.get_future(); // 获得与 packaged_task 共享状态相关联的 future 对象.

  std::thread th(std::move(task), 5); //创建一个新线程完成计数任务.

  A value = ret.get(); // 等待任务完成并获取结果.

  std::cout << "after count down A val is " << value.val << std::endl;

  th.join();
  return 0;
}
```
程序运行结果为：
```shell
5
4
3
2
1
countdown Finished!
after count down A val is 12
```

## std::packaged_task  构造函数
```c++
packaged_task() noexcept;

template<typename _Allocator>
packaged_task(allocator_arg_t, const _Allocator& __a) noexcept;

template<typename _Fn>
explicit packaged_task(_Fn&& __fn);

template<typename _Fn, typename _Alloc>
packaged_task(allocator_arg_t, const _Alloc& __a, _Fn&& __fn);

packaged_task(packaged_task&& __other) noexcept;

template<typename _Allocator>
packaged_task(allocator_arg_t, const _Allocator&, packaged_task&& __other) noexcept;

```

默认构造函数用来初始化一个空的共享状态，并且该对象无封装任务
有参构造初始化一个共享状态，并且封装任务由 __fn 来指定
packaged_task **支持移动构造**但是**禁止拷贝构造**
packaged_task 还支持之定义的分配器来分配共享状态的带参构造和移动构造

std::packaged_task 也和 std::promise 一样只允许 move 赋值运算

## valid 函数
检查当前 packaged_task 是否和一个有效的共享状态相关联，对于由默认构造函数生成的 packaged_task 对象，该函数返回 false，当对默认构造生成的对象进行了 move 赋值操作或者 swap 成有参构造生成的对象操作才能返回 true
例：
```c++
#include <chrono>
#include <future>
#include <iostream>

struct A {
  int val;
  A() : val(0) {}
  A(int x) : val(x) {}
};

std::future<A> launcher(std::packaged_task<A(int)> &task, int arg) {
  if (task.valid()) {
    std::future<A> ret = task.get_future();
    std::thread(std::move(task), arg).detach();
    return ret;
  }
  std::cout << "task is valid !\n";
  return std::future<A>();
}

int main() {
  std::packaged_task<A(int)> task(
      [](int x) { return A(x * x); }); // 设置 packaged_task
  std::packaged_task<A(int)> task2;    // 默认构造

  auto res = launcher(task, 10);
  auto res2 = launcher(task2, 10);
  std::cout << "res is " << res.get().val << std::endl;
  return 0;
}
```
运行结果为：
```shell
task is valid !
res is 100
```

## get_future 函数
与 std::promise 一样，get_future 返回一个与 packaged_task 对象共享状态相关的 future 对象。返回的 future 对象可以获得由另外一个线程在该 packaged_task 对象的共享状态上设置的某个值或者异常。


## operator(Args... args) 函数
packaged_task 重载了 operator() 函数，使用它能够调用该 packaged_task 对象所封装的对象（函数指针，函数对象，lambda表达式等），传入的参数为 args. 调用该函数一般会发生两种情况：

由于被封装的任务在 packaged_task 构造时指定，因此调用 operator() 的效果由 packaged_task 对象构造时所指定的可调用对象来决定：
- 如果被封装的任务是函数指针或者函数对象，调用 std::packaged_task::operator() 通过 std::forward 将参数完美转发给被封装的对象
- 如果被封装的任务是指向类的非静态成员函数的指针，那么 std::packaged_task::operator() 的第一个参数应该指定为成员函数被调用的那个对象，剩余的参数作为该成员函数的参数

例：
```c++
#include <chrono>
#include <future>
#include <iostream>

struct A {
  int val;
  A() : val(0) {}
  A(int x) : val(x) {}
};

A countdown(int time) {//计时 time 秒s返回一个 A 对象
  for (int i = 0; i != time; ++i) {
    std::cout << time - i << '\n';
    std::this_thread::sleep_for(std::chrono::seconds(1));
  }
  std::cout << "countdown Finished!\n";
  return A(12);
}

void read(std::future<A>& fut){
  A value = fut.get(); // 等待任务完成并获取结果.
  std::cout << "after count down A val is " << value.val << std::endl;
}

int main() {
  std::packaged_task<A(int)> task(countdown); // 设置 packaged_task
  std::future<A> ret = task.get_future(); // 获得与 packaged_task 共享状态相关联的 future 对象.

  // std::thread th(std::move(task), 5); //创建一个新线程完成计数任务.
  std::thread th(read, std::ref(ret));
  task(5);


  th.join();
  return 0;
}
```
这个版本和第一个版本功能一致，区别在于这个版本在主线程写，子线程读；而上个版本是在主线程写，子线程读

## make_ready_at_thread_exit  函数
该函数会调用被封装的任务，并向任务传递参数，类似 std::packaged_task 的 operator() 成员函数。但是与 operator() 函数不同的是，make_ready_at_thread_exit 并不会立即设置共享状态的标志为 ready，而是在线程退出时设置共享状态的标志

## reset 函数
该函数会重置 packaged_task 的共享状态，但是会保留之前的被封装的任务，如以下例子， packaged_task 被重用了2次：
```c++
#include <chrono>
#include <future>
#include <iostream>

int doubleVal(int x) { return x * 2; }

int main() {
  std::packaged_task<int(int)> task(doubleVal); // 设置 packaged_task
  std::future<int> ret =
      task.get_future(); // 获得与 packaged_task 共享状态相关联的 future 对象.

  task(5);
  std::cout << "double of the value is " << ret.get() << std::endl;

  // reuse task
  task.reset();
  ret = task.get_future();
  std::thread(std::move(task), 10).detach();

  std::cout << "double of the value is " << ret.get() << std::endl;

  return 0;
}
```

## swap 函数
该函数用于交换两个 packaged_task 的共享状态


# std::future 
前文已经多次使用 std::future ，其作用就是可以用来获取异步任务的结果，因此可以把它当成一个简单的线程同步的手段。std::future 通常由某个 Provider 创建（一个异步任务的提供者），Provider 在某个线程中设置共享状态的值，与该共享状态相关联的 std::future 对象调用 get（通常在另外一个线程中） 获取该值，如果共享状态的标志不为 ready，则调用 std::future::get() 会阻塞当前的调用者，直到 Provider 设置了共享状态的值（此时共享状态的标志变为 ready），std::future::get 返回异步任务的值或异常（如果发生了异常）。

一个有效(valid)的 future 对象通常由以下三种 Provider 创建，并和某个共享状态相关联，provider 为：
1. std::async 函数
2. std::promise::get_future, get_future 为 promise 类的成员函数
3. std::packaged_task::get_future, get_future 为 packaged_task 的成员函数

一个 std::future 对象只有在有效(valid)的情况下才有用，由 std::future 默认构造函数创建的 future 对象不是有效的（除非当前非有效的 future 对象被 move 赋值另一个有效的 future 对象）

在一个有效的 future 对象上调用 get 会阻塞当前的调用者，直到 Provider 设置了共享状态的值或异常（此时共享状态的标志变为 ready），std::future::get 将返回异步任务的值或异常（如果发生了异常）。

## 构造函数
constexpr future();

future(future&& );
future& operator=(future&&);

future(const future&) = delete;
future& operator=(const future&) = delete;

future 提供了默认构造和移动构造函数，禁止拷贝构造和拷贝赋值，只支持移动赋值

## share 函数
该函数返回一个 std::share_future 对象，调用该函数之后，该 std::future 对象本身已经不和任何共享状态相关联，因此该 std::future 的状态不再是 valid，例：
```c++
#include <future>
#include <iostream>

int doubleVal(int x) { return x * 2; }

int main() {

  std::future<int> ret = std::async(doubleVal, 20);

  std::shared_future<int> share_fut = ret.share();

  if (ret.valid()) {
    std::cout << "ret is valid\n";
  } else {
    std::cout << "ret not valid\n";
  }

  //share future 对象可以被多次访问
  std::cout << "value: " << share_fut.get() << std::endl;
  std::cout << "value *2 :" << share_fut.get() * 2 << std::endl;

  return 0;
}
```
程序运行结果为：
```shell
ret not valid
value: 40
value *2 :80
```
## get 函数
当与该 std::future 对象相关联的共享状态标志变为 ready 后，调用该函数将返回保存在共享状态中的值，如果共享状态的标志不为 ready，则调用该函数会**阻塞**当前的调用者，而此后一旦共享状态的标志变为 ready，get 返回 Provider 所设置的共享状态的值或者异常（如果抛出了异常)

## valid 函数
检查当前的 std::future 对象是否有效，一个有效的 std::future 对象只能通过 std::async(), std::future::get_future 或者 std::packaged_task::get_future 来初始化。另外由 std::future 默认构造函数创建的 std::future 对象是无效(invalid)的，同时通过 std::future 的 move 赋值后该 std::future 对象也可以变为 valid。
例：
```c++
#include <future>
#include <iostream>

int doubleVal(int x) { return x * 2; }

int main() {
  std::future<int> foo, bar;
  foo = std::async(doubleVal, 20);

  if (foo.valid()) {
    std::cout << "ret is valid\n";
  } else {
    std::cout << "ret not valid\n";
  }

  if (bar.valid()) {
    std::cout << "bar is valid\n";
  } else{
    std::cout << "bar not valid\n";
  }

  bar = std::move(foo);

  if (foo.valid()) {
    std::cout << "ret is valid\n";
  } else {
    std::cout << "ret not valid\n";
  }

  if (bar.valid()) {
    std::cout << "bar is valid\n";
  } else{
    std::cout << "bar not valid\n";
  }

  return 0;
}
```
运行结果为：
```shell
ret is valid
bar not valid
ret not valid
bar is valid
```

## wait 函数
该函数的作用是等待与当前 future 对象相关联的共享状态的标志变为 ready。如果当前共享状态不是ready，调用该函数会阻塞当前线程，直到共享状态变为 ready。一旦共享状态的标志变为 ready, wait()函数返回，当前线程被解除阻塞，但是 wait()并不读取共享状态的值或异常

## wait_for 函数
wait_for 的功能与 wait 类似，即等待与改 future 对象相关联的共享状态的标志变为 ready，函数原型为：
```c++
template<typename _Rep, typename _Period>
future_status
wait_for(const chrono::duration<_Rep, _Period>& __rel) const;
```
wait_for 可以设置一个超时时间，如果共享状态的标志在该时间段结束之前没有被 Provider 设置为 ready, 则调用 wait_for 的线程被阻塞，在等待了 __rel 时间后会有一个返回值，具体如下：
| 返回值                  | 含义                                                                   |
| ----------------------- | ---------------------------------------------------------------------- |
| future_status::ready    | 共享状态的标志已经变为 ready，即 Provider 在共享状态上设置了值或者异常 |
| future_status::timeout  | 超时，即在指定时间内共享状态的标志没有被设置为 ready                   |
| future_status::deferred | 共享状态包含一个 deferred 函数                                         |

例：
```c++
#include <chrono>
#include <future>
#include <iostream>

int testDouble(int x) {

  std::chrono::milliseconds sleep_time(1000);
  std::this_thread::sleep_for(sleep_time);
  return x * 2;
}

int main() {
  std::future<int> foo;
  foo = std::async(testDouble, 20);
  std::chrono::milliseconds time_out(200);

  while(foo.wait_for(time_out) == std::future_status::timeout){//每隔 time_out 时间调用一次 wait_for
    std::cout << "wating ...\n" ;
  }

  std::cout << "double val = " << foo.get()<<std::endl;

  return 0;
}
```
运行结果为：
```shell
wating ...
wating ...
wating ...
wating ...
double val = 40
```

## wait_until
与 std::future::wait() 的功能类似，即等待与该 std::future 对象相关联的共享状态的标志变为 ready; 不同的是，wait_until() 可以设置一个系统绝对时间点 abs_time，如果共享状态的标志在该时间点到来之前没有被 Provider 设置为 ready，则调用 wait_until 的线程被阻塞，在 abs_time 这一时刻到来之后会返回，返回值通 wait_for()

# shared_future
std::shared_future 与 std::future 类似，但是 std::shared_future 可以拷贝、多个 std::shared_future 可以共享某个共享状态的最终结果(即共享状态的某个值或者异常)。shared_future 可以通过某个 std::future 对象构造，或者通过 std::future::share() 显示转换，无论哪种转换，被转换的那个 std::future 对象都会变为 not-valid.

## 构造函数
shared_future 有四种构造函数和两种移动函数
```c++
constexpr shared_future();
shared_future(const shared_future& __sf);
shared_future(future<_Res&>&& __uf);
shared_future(shared_future&& __sf);
shared_future& operator=(const shared_future& __sf);
shared_future& operator=(shared_future&& __sf);
```
注意从 future 构造 shared_future 时是右值，构造完后原 future 对象变成 invalid 状态

## 其他函数
shared_future 的其他函数接口与 future 基本一致

# std::async 函数
函数原型为：
```c++
template <class Fn, class... Args>
  future<typename result_of<Fn(Args...)>::type>
    async(Fn&& fn, Args&&... args);

template <class Fn, class... Args>
  future<typename result_of<Fn(Args...)>::type>
    async(launch policy, Fn&& fn, Args&&... args);
```
两者的差别在于第一类 async 没有指定异步任务的启动策略（launch policy），而第二类函数指定了启动策略，可以为 launch::async, launch::deferred，以及两者的按位或 (|)

fn 和 args 参数用来指定异步任务及其参数，其返回值是一个 std::future 对象，通过该对象可以获取异步任务的值或者异常

例：
```c++
#include <cmath>
#include <chrono>
#include <future>
#include <iostream>

double ThreadTask(int n) {
    std::cout << std::this_thread::get_id()
        << " start computing..." << std::endl;

    double ret = 0;
    for (int i = 0; i <= n; i++) {
        ret += std::sin(i);
    }

    std::cout << std::this_thread::get_id()
        << " finished computing..." << std::endl;
    return ret;
}

int main(int argc, const char *argv[])
{
    std::future<double> f(std::async(std::launch::async, ThreadTask, 100000000));

    while(f.wait_until(std::chrono::system_clock::now() + std::chrono::seconds(1))
            != std::future_status::ready) {
        std::cout << "task is running...\n";
    }

    //另一种方式
    // while(f.wait_for(std::chrono::seconds(1))
    //         != std::future_status::ready) {
    //     std::cout << "task is running...\n";
    // }

    std::cout << f.get() << std::endl;

    return 0;
}
```
运行结果为：
```shell
139854862804736 start computing...
task is running...
task is running...
139854862804736 finished computing...
1.71365
```

# 与 future 相关的枚举类型
主要有 3 种枚举类型与 future 相关
## std::future_errc
future_errc 主要有 4 种状态
| 类型                      | 含义                                                                                           |
| ------------------------- | ---------------------------------------------------------------------------------------------- |
| broken_promise            | 与该 future 共享状态相关联的 promise 对象在设置值或异常之前被销毁                              |
| future_already_retrieved  | 与该 future 对象相关联的共享状态的值已经被当前 Provider 获取了，即调用了 std::future::get 函数 |
| promise_already_satisfied | std::promise 对象已经对共享状态设置了某一值或者异常                                            |
| no_state                  | 无共享状态                                                                                     |

## std::future_status
std::future_status 类型主要用在 std::future(或std::shared_future)中的 wait_for 和 wait_until 两个函数中
| 类型                    | 含义                                                                                                  |
| ----------------------- | ----------------------------------------------------------------------------------------------------- |
| future_status::ready    | wait_for(或wait_until) 因为共享状态的标志变为 ready 而返回                                            |
| future_status::timeout  | 超时，即 wait_for(或wait_until) 因为在指定的时间段（或时刻）内共享状态的标志依然没有变为 ready 而返回 |
| future_status::deferred | future_status::deferred                                                                               |

## std::launch
该枚举类型主要是在调用 std::async 设置异步任务的启动策略
| 类型             | 含义                                                                                   |
| ---------------- | -------------------------------------------------------------------------------------- |
| launch::async    | Asynchronous: 异步任务会在另外一个线程中调用，并通过共享状态返回异步任务的结果         |
| launch::deferred | Deferred: 异步任务将会**在共享状态被访问时调用**，相当与按需调用（即延迟(deferred)调用 |
