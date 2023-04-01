---
title: cyber定时器
date: 2023-02-07 15:51:59
tags:
---

定时器
<!-- more -->

# TimerTask
从最简单的 TimerTask 开始看, 它定义了一个定时器任务的数据结构，具体如下：

```c++
struct TimerTask {
  explicit TimerTask(uint64_t timer_id) : timer_id_(timer_id) {}//构造函数
  uint64_t timer_id_ = 0;// 任务对应的id
  std::function<void()> callback;// Task所对应的回调函数 
  uint64_t interval_ms = 0; // 调用周期
  uint64_t remainder_interval_ms = 0; // 剩余时间间隔
  uint64_t next_fire_duration_ms = 0; //
  int64_t accumulated_error_ns = 0;  // 累积误差
  uint64_t last_execute_time_ns = 0; // 上次执行的时间
  std::mutex mutex;
};
```

# TimerBucket
```c++
class TimerBucket {
 public:
  void AddTask(const std::shared_ptr<TimerTask>& task) { //往 bucket 上加定时任务
    std::lock_guard<std::mutex> lock(mutex_);
    task_list_.push_back(task);
  }

  std::mutex& mutex() { return mutex_; }
  std::list<std::weak_ptr<TimerTask>>& task_list() { return task_list_; }

 private:
  std::mutex mutex_;// Bucket 自身的 mutex
  std::list<std::weak_ptr<TimerTask>> task_list_; // 任务队列
};
```

# TimingWheel
时间轮定义为:
```c++
class TimingWheel {
 public:
  ~TimingWheel() {
    if (running_) {
      Shutdown();
    }
  }

  void Start();

  void Shutdown();

  void Tick();

  void AddTask(const std::shared_ptr<TimerTask>& task);

  void AddTask(const std::shared_ptr<TimerTask>& task,
               const uint64_t current_work_wheel_index);

  void Cascade(const uint64_t assistant_wheel_index);

  void TickFunc();

  inline uint64_t TickCount() const { return tick_count_; }

 private:
  inline uint64_t GetWorkWheelIndex(const uint64_t index) {//快速取余，等价于  return index % WORK_WHEEL_SIZE
    return index & (WORK_WHEEL_SIZE - 1);
  }
  inline uint64_t GetAssistantWheelIndex(const uint64_t index) {
    return index & (ASSISTANT_WHEEL_SIZE - 1);
  }

  bool running_ = false;
  uint64_t tick_count_ = 0; //
  std::mutex running_mutex_; // 运行锁
  TimerBucket work_wheel_[WORK_WHEEL_SIZE]; // 工作轮数组
  TimerBucket assistant_wheel_[ASSISTANT_WHEEL_SIZE];// 辅助轮数组
  uint64_t current_work_wheel_index_ = 0; // 当前工作时间轮索引
  std::mutex current_work_wheel_index_mutex_; // 工作时间轮索引锁
  uint64_t current_assistant_wheel_index_ = 0; // 当前辅助时间轮索引
  std::mutex current_assistant_wheel_index_mutex_; // 辅助时间轮索引锁
  std::thread tick_thread_;// 时间线程

  DECLARE_SINGLETON(TimingWheel)
};
```

Cyber的时间轮中，存在工作轮和辅助轮两个时间轮，其中工作轮是真正去完成定时器的功能，辅助轮是为了延长定时器支持的最大周期，
工作轮走完完整一圈辅助轮才会走过一个bucket，我们看关键的 AddTask 代码：


```c++
void TimingWheel::AddTask(const std::shared_ptr<TimerTask>& task) {
  AddTask(task, current_work_wheel_index_);
}

void TimingWheel::AddTask(const std::shared_ptr<TimerTask>& task,
                          const uint64_t current_work_wheel_index) {
  if (!running_) {
    Start();
  }
  auto work_wheel_index = current_work_wheel_index +
                          static_cast<uint64_t>(std::ceil(
                              static_cast<double>(task->next_fire_duration_ms) /
                              TIMER_RESOLUTION_MS));//根据当前索引、下次调用时间和时间分辨率计算应该添加到哪个 bucket 上
  if (work_wheel_index >= WORK_WHEEL_SIZE) {//当计算得到的索引大于等于工作轮的最大限制
    auto real_work_wheel_index = GetWorkWheelIndex(work_wheel_index);//计算当前实际应该在的工作轮位置
    task->remainder_interval_ms = real_work_wheel_index;//更新task的剩余等待时间
    auto assistant_ticks = work_wheel_index / WORK_WHEEL_SIZE;//计算辅助时间轮
    if (assistant_ticks == 1 && //当辅助时间轮 tick 为1，实际工作的 bucket 编号小于当前 bucket 标号，直接加入到当前工作轮的 bucket 中
        real_work_wheel_index < current_work_wheel_index_) {//，因为转一圈后就为该task就该工作
      work_wheel_[real_work_wheel_index].AddTask(task);
    } else {//当辅助时间轮 tick 大于1
      auto assistant_wheel_index = 0;
      {
        std::lock_guard<std::mutex> lock(current_assistant_wheel_index_mutex_);
        assistant_wheel_index = GetAssistantWheelIndex(
            current_assistant_wheel_index_ + assistant_ticks);//根据当前辅助轮 bucket 索引和应该加的索引计算辅助时间轮的 bucket 索引
        assistant_wheel_[assistant_wheel_index].AddTask(task);//往对应辅助时间轮上加 task
      }

    }
  } else {//如果索引没有超过工作轮的最大限制，直接在对应索引的 bucket 上添加 task
    work_wheel_[work_wheel_index].AddTask(task);
  }
}

```
上面的代码就是往时间轮中加入 Task 的逻辑，接下来看如何将辅助轮中的任务加到工作轮当中

```c++
void TimingWheel::Cascade(const uint64_t assistant_wheel_index) {
  auto& bucket = assistant_wheel_[assistant_wheel_index];// 拿到辅助轮上对应的bucket
  std::lock_guard<std::mutex> lock(bucket.mutex());// 对 bucket 加锁
  auto ite = bucket.task_list().begin();
  while (ite != bucket.task_list().end()) {//遍历 bucket 上列表的所有节点
    auto task = ite->lock();// list 上存的是 weak_ptr，这里返回 shared_ptr
    if (task) {
      work_wheel_[task->remainder_interval_ms].AddTask(task);//往工作轮上添加 task
    }
    ite = bucket.task_list().erase(ite);//删除辅助轮对应的 task
  }
}
```

接下来是如何让时间轮转起来的 TickFunc：
```c++
void TimingWheel::TickFunc() {
  Rate rate(TIMER_RESOLUTION_MS * 1000000);  // ms to ns
  while (running_) {// 当处于 running 状态时
    Tick();//运行 Tick
    tick_count_++;
    rate.Sleep();//rate能保证按照预定的时间周期来run一次
    {
      std::lock_guard<std::mutex> lock(current_work_wheel_index_mutex_);
      current_work_wheel_index_ =
          GetWorkWheelIndex(current_work_wheel_index_ + 1);//更新工作轮索引
    }
    if (current_work_wheel_index_ == 0) {//当工作轮工作了一圈，辅助轮索引也更新一次
      {
        std::lock_guard<std::mutex> lock(current_assistant_wheel_index_mutex_);
        current_assistant_wheel_index_ =
            GetAssistantWheelIndex(current_assistant_wheel_index_ + 1);
      }
      Cascade(current_assistant_wheel_index_);//并且将当前辅助轮上的bucket的任务加到工作轮上
    }
  }
}
```

再看 Tick 函数做了什么事情：
```c++
void TimingWheel::Tick() {
  auto& bucket = work_wheel_[current_work_wheel_index_];
  {
    std::lock_guard<std::mutex> lock(bucket.mutex());
    auto ite = bucket.task_list().begin();
    while (ite != bucket.task_list().end()) {
      auto task = ite->lock();
      if (task) {
        auto* callback =
            reinterpret_cast<std::function<void()>*>(&(task->callback));
        cyber::Async([this, callback] {//在实车环境会将 callback 交给 task manager 做，仿真会另起线程去做
          if (this->running_) {
            (*callback)();
          }
        });
      }
      ite = bucket.task_list().erase(ite);
    }
  }
}
```
Tick 函数做的事情有两个：
1. 遍历当前工作轮的 bucket 上的所有 task_list, 封装 task 中的任务，然后调用 Async 函数去运行该任务
2. 在调用完该task后将其从工作轮中删除

# timer
最后看定时器的结构
```c++
struct TimerOption {// 定时器任务选项
  TimerOption(uint32_t period, std::function<void()> callback, bool oneshot)
      : period(period), callback(callback), oneshot(oneshot) {}

  TimerOption() : period(), callback(), oneshot() {}

  uint32_t period = 0;//运行周期

  std::function<void()> callback;//回调函数

  bool oneshot;//是否只跑一次
};

class Timer {
 public:
  Timer();

  explicit Timer(TimerOption opt);

  Timer(uint32_t period, std::function<void()> callback, bool oneshot);
  ~Timer();

  void SetTimerOption(TimerOption opt); //提供了一个设置 TimerOption 的接口

  void Start();

  void Stop();

 private:
  bool InitTimerTask();
  uint64_t timer_id_;
  TimerOption timer_opt_;
  TimingWheel* timing_wheel_ = nullptr;
  std::shared_ptr<TimerTask> task_;
  std::atomic<bool> started_ = {false};
};
```
头文件 TimerOption 结构体定义了定时器选项，其主要定义了运行周期，回调函数和是否只跑一次。
Timer支持三种构造函数，对外主要提供 Start 和 Stop 接口，对内有一个初始化 TimerTask 的接口，一个 timer_id，一个时间轮的指针，一个 TimerTask 的共享指针，一个 started_ 的原子变量


```c++
namespace {
std::atomic<uint64_t> global_timer_id = {0};
uint64_t GenerateTimerId() { return global_timer_id.fetch_add(1); }
}  // namespace
```
timer.cc 中定义了一个原子变量来生成每个 timer 的 id 值

```c++
Timer::Timer() {
  timing_wheel_ = TimingWheel::Instance();
  timer_id_ = GenerateTimerId();
}

Timer::Timer(TimerOption opt) : timer_opt_(opt) {
  timing_wheel_ = TimingWheel::Instance();
  timer_id_ = GenerateTimerId();
}

Timer::Timer(uint32_t period, std::function<void()> callback, bool oneshot) {
  timing_wheel_ = TimingWheel::Instance();
  timer_id_ = GenerateTimerId();
  timer_opt_.period = period;
  timer_opt_.callback = callback;
  timer_opt_.oneshot = oneshot;
}
```
构造函数中将时间轮指针指向 TimingWheel 的单例，生成对应的 timer_id ，设置 opt

```c++
void Timer::Start() {
  if (!common::GlobalData::Instance()->IsRealityMode()) {//非实车模式直接返回
    return;
  }

  if (!started_.exchange(true)) {//如果该任务还没有 start
    if (InitTimerTask()) {//调用初始化函数
      timing_wheel_->AddTask(task_);//将 task 加入到时间轮
    }
  }
}

void Timer::Stop() {
  if (started_.exchange(false) && task_) {//如果 start 为 true 并且 task 不为空指针
    auto tmp_task = task_;//用一个共享指针去获得 task_ 里面的锁再调用 task_ 的 reset
    {
      std::lock_guard<std::mutex> lg(tmp_task->mutex);
      task_.reset();
    }
  }
}
```
最后看核心部分的 InitTimerTask 代码：
```c++
bool Timer::InitTimerTask() {
  if (timer_opt_.period == 0) {//周期为0时直接返回 false
    return false;
  }

  if (timer_opt_.period >= TIMER_MAX_INTERVAL_MS) {//周期超过最大限制时直接返回 false
    return false;
  }

  task_.reset(new TimerTask(timer_id_));
  task_->interval_ms = timer_opt_.period;
  task_->next_fire_duration_ms = task_->interval_ms;//下次激活间隔
  if (timer_opt_.oneshot) {//如果只运行一次
    std::weak_ptr<TimerTask> task_weak_ptr = task_;//用 weak_ptr 指向 TimerTask
    task_->callback = [callback = this->timer_opt_.callback, task_weak_ptr]() {//设置callback
      auto task = task_weak_ptr.lock();
      if (task) {
        std::lock_guard<std::mutex> lg(task->mutex);
        callback();
      }
    };
  } else {//重复task
    std::weak_ptr<TimerTask> task_weak_ptr = task_;
    task_->callback = [callback = this->timer_opt_.callback, task_weak_ptr]() {
      auto task = task_weak_ptr.lock();
      if (!task) {//如果共享指针管理对象已被释放，直接return
        return;
      }
      std::lock_guard<std::mutex> lg(task->mutex);
      auto start = Time::MonoTime().ToNanosecond();
      callback();
      auto end = Time::MonoTime().ToNanosecond();
      uint64_t execute_time_ns = end - start;//计算callback一共花费了多少时间，纳秒级别
      uint64_t execute_time_ms =
        std::llround(static_cast<double>(execute_time_ns) / 1e6);//转换成毫秒级别的时间
      if (task->last_execute_time_ns == 0) {//第一次执行的时候，先将 last_execute_time 设置成 start
        task->last_execute_time_ns = start;
      } else {
        task->accumulated_error_ns +=//计算累积误差（纳秒级），即相邻两次调用时间与周期时间的差异
            start - task->last_execute_time_ns - task->interval_ms * 1000000;
      }
      task->last_execute_time_ns = start;//给上次执行时间赋值
      if (execute_time_ms >= task->interval_ms) {//如果计算时间超过了 task 的周期时间
        task->next_fire_duration_ms = TIMER_RESOLUTION_MS;// 将下次激活时间间隔设置为最小分辨率
      } else {//如果在task的周期内完成计算
        int64_t accumulated_error_ms = std::llround(
            static_cast<double>(task->accumulated_error_ns) / 1e6);//计算累积误差（毫秒级）
        if (static_cast<int64_t>(task->interval_ms - execute_time_ms -//如果运行周期减执行时间再减最小时间分辨率大于等于累积误差
                                 TIMER_RESOLUTION_MS) >= accumulated_error_ms) {
          task->next_fire_duration_ms =//下次激活时间间隔为周期时间减执行时间再减计算误差
              task->interval_ms - execute_time_ms - accumulated_error_ms;
        } else {//否则将下次激活时间设置为最小时间分辨率，即下个bucket就运行
          task->next_fire_duration_ms = TIMER_RESOLUTION_MS;
        }
      }
      TimingWheel::Instance()->AddTask(task);//向时间轮中添加task
    };
  }
  return true;
}
```
