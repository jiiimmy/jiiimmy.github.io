---
title: c++11之chrono
date: 2023-02-21 16:53:39
tags:
---


chrono库是一个灵活的类型集合，可以以不同的精度跟踪时间
<!-- more -->

chrono库定义了三种主要类型以及实用函数
- duration 时间段
- time points 时间点
- clocks 时钟

# duration 时间段
std::chrono::duration 用来表示一段时间，1小时，10分钟，0.01秒钟都可以表示。不同的时间单位都能换算成秒， duaration 正是通过秒来表示其他时间段

duration 是一个模板类，它的定义如下：
```c++
template<typename Rep, typename Period = std::ratio<1>>
class duration;

template<std::intmax_t Num, std::intmax_t Denom = 1>
class ratio;
```
Period 是一个 std::ratio 类型， std::ratio 表示一个分数，第一个参数是分子，第二个是分母，默认为1， `duration<Rep, Period>`就表示这个时间段有 Period 秒。模板参数 Rep 是一种数据类型，用来表示 Period 的数量

```c++
std::chrono::duration<int, std::ratio<60, 1>> minute(1) // 1 分钟
std::chrono::duration<int, std::ratio<1, 1000>> ms(2) // 2 毫秒

std::chrono::duration<int, std::ratio<1, 1>> n_second(22); // 22秒
std::chrono::duration<double, std::ratio<60, 1>> n_minute(n_second); // 用 22 秒初始化此时间段
// duration.count() 用于返回该时间段的数量，比如有多少个一分钟，几个一小时
std::cout << n_minute.count() << std::endl;  // 0.366667 - 22 秒为 0.367 分钟
```

chrono中定义了常见的 duration 类型，下至纳秒上至小时：
```c++
chrono::nanoseconds
chrono::microseconds;
chrono::milliseconds;
chrono::seconds;
chrono::minutes;
chrono::hours;
```

## 时间段的转换
使用 std::chrono::duration_cast 可以将不同刻度的时间进行转换，但是由精度细的时间段往精度粗的时间段转换时会丢失精度，如：
```c++
#include <iostream>
#include <chrono>

int main() {
  std::chrono::seconds t_s(122);
  std::chrono::minutes t_min = std::chrono::duration_cast<std::chrono::minutes>(t_s);

  std::cout <<t_min.count() <<std::endl;
  t_s = std::chrono::duration_cast<std::chrono::seconds>(t_min);
  std::cout << t_s.count() <<std::endl;
  return 0;
}
```
程序运行结果为：
```shell
2
120
```
当我们用 122s 去转换成分钟数时会丢失掉精度转换成 2min，即使再用相反的方法转换回 seconds 也无法将丢失的精度找回

## 时间段的运算
不同时间类型的时间段可以进行加减除取模等操作，也可以和常数相乘，运算结果是符合直觉的，即时间段与时间段相加还是时间段，时间段与时间段相除则是一个数值，如：
```c++
#include <chrono>
#include <iostream>

int main() {
  std::chrono::seconds t_s(12);
  std::chrono::minutes t_min(2);

  std::cout << (t_min + t_s).count() << std::endl;
  std::cout << (t_min - t_s).count() << std::endl;
  std::cout << (t_min / t_s) << std::endl;
  std::cout << (t_min * 2).count() << std::endl;
  std::cout << (t_min % t_s).count() << std::endl;

  return 0;
}
```
程序运行结果为：
```shell
132
108
10
4
0
```

# time point 时间点
时间点表示一个具体的时刻。它的定义为：
```c++
template<typename Clock, typename Duration = typename Clock::duration>
class time_point;
```
其中 Clock 表示一个时钟， Duration 表示从时钟的起始点开始经过的时间，如：
```c++
std::chrono::time_point<std::chrono::system_clock,std::chrono::seconds> tp(std::chrono::seconds(1));
```
这里 tp 表示时间起点后的 1 秒，通常是 1970-01-01 00:00:01

## 时间点的运算
一个常用的时间点就是当前时间点，可以通过如下方式获得：
```c++
std::chrono::system_clock::time_point now = std::chrono::system_clock::now();
auto tomorrow = now + std::chrono::hours(24);//明日此时

decltype(now)::duration dur = now.time_since_epoch();//获得时间点距离时钟起始时刻的 duration
std::cout << dur.count() << std::endl;
```
**时间点和时间点**可以做减法得到时间段，**时间段加时间段**可以得到新的时间点

epoch 的意思是纪元，即某个时刻的开始。不同的时钟对 epoch 选择不同， system_clock 的 epoch 是 1970-01-01 00:00:00， steady_clock 的 epoch 是系统启动时间
时间点之间采用的 duration 可能不同，此时可用 time_point_cast 进行不同 duration 的转换，如：
```c++
std::chrono::time_point<std::chrono::system_clock> tp(std::chrono::hours(1));
std::cout << tp.time_since_epoch().count()<<std::endl;//结果为 3600000000000
auto tp_sec = std::chrono::time_point_cast<std::chrono::minutes>(tp);
std::cout << tp_sec.time_since_epoch().count()<<std::endl;//结果为 60
```
这里转换同样存在精度丢失问题

# clocks 时钟
c++ 11 中存在三个时钟
- chrono::system_clock
- chrono::steady_clock
- chrono::high_resolution_clock


## system_clock
std::chrono::system_clock 表示当前的系统时钟，系统中运行的所有进程使用 now() 得到的时间是一致的，其 epoch 为 `1970-01-01 00:00:00`。
system_clock 提供了静态方法可以获取当前时间，并且提供了与 time_t 类型的相互转化的接口，这样就可以利用 ctime 函数库中的时间格式化、转化时区等接口，如：
```c++
//当前时间点
std::chrono::system_clock::time_point now = std::chrono::system_clock::now();

//转化为 time_t 类型
time_t now_t = std::chrono::system_clock::to_time_t(now);

//从 time_t 转化为 time_point
time_t t = time(nullptr);

std::chrono::system_clock::time_point Now = std::chrono::system_clock::from_time_t(t);
```

## steady_clock
字面上意思是稳定的时钟，与 system_clock 的区别在于 steady_clock 的 epoch 是开机时间，因此就算用户修改了系统时间也不会影响到这个时钟，它的计时始终是增加的，但是如果把系统时间前向设置到2020年，那么通过 system_clock 获取到的系统时间一下就比先前小了，如：
```c++
auto now = std::chrono::steady_clock::now().time_since_epoch();

// using hours = std::chrono::duration<double, std::ratio<3600, 1>>;

auto now_h = std::chrono::duration_cast<std::chrono::hours>(now);

std::cout << now_h.count()<<std::endl; // 315 即我的机器已经开机315个小时了
```
这个时钟通常用来对程序运行时间进行计时，或者用来设置定时器的时长：
```c++
#include <chrono>
#include <iostream>
#include <thread>

using namespace std::chrono_literals;
int main() {
  std::chrono::steady_clock::time_point start = std::chrono::steady_clock::now();
  std::this_thread::sleep_for(2ms);
  std::chrono::steady_clock::time_point end = std::chrono::steady_clock::now();
  std::cout << "sleep time is " << std::chrono::duration_cast<std::chrono::milliseconds>((end - start)).count() << " ms\n";

  return 0;
}
```
运行结果为：
```shell
sleep time is 2 ms
```
steady_clock 没有与 time_t 相互转换的接口

## high_resolution_clock
high_resolution_clock 表示实现提供的具有最小滴答周期的时钟，它可能是 std::chrono::system_clock 或 std::chrono::steady_clock 的别名或者第三个独立时钟