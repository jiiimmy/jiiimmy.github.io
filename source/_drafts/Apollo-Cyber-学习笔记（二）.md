---
title: Apollo-Cyber-学习笔记（二）
date: 2022-09-06 17:00:57
tags:
---

上一篇详细介绍了 node 相关的，我们发现 reader 的很多动作是借助 blocker 来完成的，这一篇博客我们来看 blocker 做了些什么事情

<!-- more -->

blocker 目录结构为：
├── blocker.h
├── blocker_manager.cc
├── blocker_manager.h
├── blocker_manager_test.cc
├── blocker_test.cc
├── BUILD
├── intra_reader.h
└── intra_writer.h

我们详细看下每个文件的具体作用：
## blocker_manager.h
这个文件主要定义了 BlockerManager 类，可以看出这个类主要是用来管理 Blocker。
```c++
class BlockerManager {
 public:
  using BlockerMap =
      std::unordered_map<std::string, std::shared_ptr<BlockerBase>>;

  virtual ~BlockerManager();

  static const std::shared_ptr<BlockerManager>& Instance() {
    static auto instance =
        std::shared_ptr<BlockerManager>(new BlockerManager());
    return instance;
  }

  template <typename T>
  bool Publish(const std::string& channel_name,
               const typename Blocker<T>::MessagePtr& msg);

  template <typename T>
  bool Publish(const std::string& channel_name,
               const typename Blocker<T>::MessageType& msg);

  template <typename T>
  bool Subscribe(const std::string& channel_name, size_t capacity,
                 const std::string& callback_id,
                 const typename Blocker<T>::Callback& callback);

  template <typename T>
  bool Unsubscribe(const std::string& channel_name,
                   const std::string& callback_id);

  template <typename T>
  std::shared_ptr<Blocker<T>> GetBlocker(const std::string& channel_name);

  template <typename T>
  std::shared_ptr<Blocker<T>> GetOrCreateBlocker(const BlockerAttr& attr);

  void Observe();
  void Reset();

 private:
  BlockerManager();
  BlockerManager(const BlockerManager&) = delete;
  BlockerManager& operator=(const BlockerManager&) = delete;

  BlockerMap blockers_;
  std::mutex blocker_mutex_;
};
```
我们可以看到这里的 BlockerManager 是一个单例，其提供了一个静态函数 Instance 来获得单例的共享指针
私有变量 blockers_ 是一个 BlockerMap, 而 BlockerMap 是一个 string 与 BlockerBase 共享指针的一个键值对
BlockerManager 提供了 Pubulish, Subscribe, Unsubscribe, GetBlocker, GetOrCreateBlocker, Observe, Reset 接口

再看看具体实现：
**Publish**
```c++
template <typename T>
bool BlockerManager::Publish(const std::string& channel_name,
                             const typename Blocker<T>::MessagePtr& msg) {
  auto blocker = GetOrCreateBlocker<T>(BlockerAttr(channel_name));
  if (blocker == nullptr) {
    return false;
  }
  blocker->Publish(msg);
  return true;
}

template <typename T>
bool BlockerManager::Publish(const std::string& channel_name,
                             const typename Blocker<T>::MessageType& msg) {
  auto blocker = GetOrCreateBlocker<T>(BlockerAttr(channel_name));
  if (blocker == nullptr) {
    return false;
  }
  blocker->Publish(msg);
  return true;
}
```
Pulish 支持两种形式的 msg, 一种是 共享指针类型的T模板 msg, 一种是模板T类型的 msg, 其都是通过 channel_name 和 GetOrCreateBlocker 去获得 blocker 的共享指针，然后去调用对应 blocker 的 Pushlish 函数
**Subscribe**
```c++
template <typename T>
bool BlockerManager::Subscribe(const std::string& channel_name, size_t capacity,
                               const std::string& callback_id,
                               const typename Blocker<T>::Callback& callback) {
  auto blocker = GetOrCreateBlocker<T>(BlockerAttr(capacity, channel_name));
  if (blocker == nullptr) {
    return false;
  }
  return blocker->Subscribe(callback_id, callback);
}
```
Subscribe 函数也是去调用 blocker 的 Subscribe 函数，其传参为 callback_id 与 callback 函数

**Unsubscribe**
```c++
template <typename T>
bool BlockerManager::Unsubscribe(const std::string& channel_name,
                                 const std::string& callback_id) {
  auto blocker = GetBlocker<T>(channel_name);
  if (blocker == nullptr) {
    return false;
  }
  return blocker->Unsubscribe(callback_id);
}
```
Unsubscribe 函数通过 也是去调用 blocker 的 Unsubscribe 函数，其传参为 callback_id

**GetBlocker**
```c++
template <typename T>
std::shared_ptr<Blocker<T>> BlockerManager::GetBlocker(
    const std::string& channel_name) {
  std::shared_ptr<Blocker<T>> blocker = nullptr;
  {
    std::lock_guard<std::mutex> lock(blocker_mutex_);
    auto search = blockers_.find(channel_name);
    if (search != blockers_.end()) {
      blocker = std::dynamic_pointer_cast<Blocker<T>>(search->second);
    }
  }
  return blocker;
}
```
GetBlocker 的作用是通过 channel_name 获得对应的 blocker 指针，这里是直接在 blockers_ 里面查找

**GetOrCreateBlocker**
```c++
template <typename T>
std::shared_ptr<Blocker<T>> BlockerManager::GetOrCreateBlocker(
    const BlockerAttr& attr) {
  std::shared_ptr<Blocker<T>> blocker = nullptr;
  {
    std::lock_guard<std::mutex> lock(blocker_mutex_);
    auto search = blockers_.find(attr.channel_name);
    if (search != blockers_.end()) {
      blocker = std::dynamic_pointer_cast<Blocker<T>>(search->second);
    } else {
      blocker = std::make_shared<Blocker<T>>(attr);
      blockers_[attr.channel_name] = blocker;
    }
  }
  return blocker;
}
```
GetOrCreateBlocker 函数是在 blocker 中查找 channel_name 对应的 blocker，如果没有就创建一个 channel_name 对应的 blocker

## blocker_manager.cc
```c++
BlockerManager::BlockerManager() {}

BlockerManager::~BlockerManager() { blockers_.clear(); }

void BlockerManager::Observe() {
  std::lock_guard<std::mutex> lock(blocker_mutex_);
  for (auto& item : blockers_) {
    item.second->Observe();
  }
}

void BlockerManager::Reset() {
  std::lock_guard<std::mutex> lock(blocker_mutex_);
  for (auto& item : blockers_) {
    item.second->Reset();
  }
  blockers_.clear();
}
```
析构函数将 blockers 清空， Observe 是去调用每个 blocker 的 Observe 函数； Reset 函数先调用每个 blocker 的 Reset，再清空 blockers_

## blocker.h
```c++
class BlockerBase {
 public:
  virtual ~BlockerBase() = default;

  virtual void Reset() = 0;
  virtual void ClearObserved() = 0;
  virtual void ClearPublished() = 0;
  virtual void Observe() = 0;
  virtual bool IsObservedEmpty() const = 0;
  virtual bool IsPublishedEmpty() const = 0;
  virtual bool Unsubscribe(const std::string& callback_id) = 0;

  virtual size_t capacity() const = 0;
  virtual void set_capacity(size_t capacity) = 0;
  virtual const std::string& channel_name() const = 0;
};
```
BlockerBase 基类定义了一些接口
```c++
struct BlockerAttr {
  BlockerAttr() : capacity(10), channel_name("") {}
  explicit BlockerAttr(const std::string& channel)
      : capacity(10), channel_name(channel) {}
  BlockerAttr(size_t cap, const std::string& channel)
      : capacity(cap), channel_name(channel) {}
  BlockerAttr(const BlockerAttr& attr)
      : capacity(attr.capacity), channel_name(attr.channel_name) {}

  size_t capacity;
  std::string channel_name;
};
```
BlockerAttr 定义了 blocker 的属性，主要是 capacity 与 channel_name

```c++
template <typename T>
class Blocker : public BlockerBase {
  friend class BlockerManager;

 public:
  using MessageType = T;
  using MessagePtr = std::shared_ptr<T>;
  using MessageQueue = std::list<MessagePtr>;
  using Callback = std::function<void(const MessagePtr&)>;
  using CallbackMap = std::unordered_map<std::string, Callback>;
  using Iterator = typename std::list<std::shared_ptr<T>>::const_iterator;

  explicit Blocker(const BlockerAttr& attr);
  virtual ~Blocker();

  void Publish(const MessageType& msg);
  void Publish(const MessagePtr& msg);

  void ClearObserved() override;
  void ClearPublished() override;
  void Observe() override;
  bool IsObservedEmpty() const override;
  bool IsPublishedEmpty() const override;

  bool Subscribe(const std::string& callback_id, const Callback& callback);
  bool Unsubscribe(const std::string& callback_id) override;

  const MessageType& GetLatestObserved() const;
  const MessagePtr GetLatestObservedPtr() const;
  const MessagePtr GetOldestObservedPtr() const;
  const MessagePtr GetLatestPublishedPtr() const;

  Iterator ObservedBegin() const;
  Iterator ObservedEnd() const;

  size_t capacity() const override;
  void set_capacity(size_t capacity) override;
  const std::string& channel_name() const override;

 private:
  void Reset() override;
  void Enqueue(const MessagePtr& msg);
  void Notify(const MessagePtr& msg);

  BlockerAttr attr_;
  MessageQueue observed_msg_queue_;
  MessageQueue published_msg_queue_;
  mutable std::mutex msg_mutex_;

  CallbackMap published_callbacks_;
  mutable std::mutex cb_mutex_;

  MessageType dummy_msg_;
};
```
Blocker 类主要是实现了 BlockerBase 定义的一些接口、一些类型定义、一些访问接口以及一些私有接口

**构造函数**
```c++
template <typename T>
Blocker<T>::Blocker(const BlockerAttr& attr) : attr_(attr), dummy_msg_() {}
```
构造函数必须有 BlockerAttr ，同时调用 T 类型的默认构造给 dummy_msg_
**析构函数**
```c++
template <typename T>
Blocker<T>::~Blocker() {
  published_msg_queue_.clear();
  observed_msg_queue_.clear();
  published_callbacks_.clear();
}
```
将 published_msg_queue_, observed_msg_queue_, published_callbacks_ 清空
**Publish**
```c++
template <typename T>
void Blocker<T>::Publish(const MessageType& msg) {
  Publish(std::make_shared<MessageType>(msg));
}

template <typename T>
void Blocker<T>::Publish(const MessagePtr& msg) {
  Enqueue(msg);
  Notify(msg);
}
```
这里我们看到 Pushlish 函数是先调用 Enqueue(msg), 再调用 Notify(msg)

**重置函数**
```c++
template <typename T>
void Blocker<T>::Reset() {
  {
    std::lock_guard<std::mutex> lock(msg_mutex_);
    observed_msg_queue_.clear();
    published_msg_queue_.clear();
  }
  {
    std::lock_guard<std::mutex> lock(cb_mutex_);
    published_callbacks_.clear();
  }
}

template <typename T>
void Blocker<T>::ClearObserved() {
  std::lock_guard<std::mutex> lock(msg_mutex_);
  observed_msg_queue_.clear();
}

template <typename T>
void Blocker<T>::ClearPublished() {
  std::lock_guard<std::mutex> lock(msg_mutex_);
  published_msg_queue_.clear();
}
```
这里是根据需要先获得对应的锁，在清空对应的队列
**Observe函数**
```c++
template <typename T>
void Blocker<T>::Observe() {
  std::lock_guard<std::mutex> lock(msg_mutex_);
  observed_msg_queue_ = published_msg_queue_;
}
```
这里值得注意，当我们进行 Oberve 的时候，会先获得 msg 的锁，然后将 published_msg_queue_ 赋值给 observed_msg_queue_
**Subscribe函数**
```c++
template <typename T>
bool Blocker<T>::Subscribe(const std::string& callback_id,
                           const Callback& callback) {
  std::lock_guard<std::mutex> lock(cb_mutex_);
  if (published_callbacks_.find(callback_id) != published_callbacks_.end()) {
    return false;
  }
  published_callbacks_[callback_id] = callback;
  return true;
}
```
Subscribe 首先去 published_callbacks_ 中寻找是否有 callback_id 字段，有的话说明 callback_id 已经订阅并设置过 callback ，这里不支持修改并返回false；否则在 published_callbacks_ map中加上对应的键值对
**Unsubscribe函数**
```c++
template <typename T>
bool Blocker<T>::Unsubscribe(const std::string& callback_id) {
  std::lock_guard<std::mutex> lock(cb_mutex_);
  return published_callbacks_.erase(callback_id) != 0;
}
```
Unsubscribe 就是从 published_callbacks_ 中删除 callback_id 所对应的键
**Enqueue函数**
```c++
template <typename T>
void Blocker<T>::Enqueue(const MessagePtr& msg) {
  if (attr_.capacity == 0) {
    return;
  }
  std::lock_guard<std::mutex> lock(msg_mutex_);
  published_msg_queue_.push_front(msg);
  while (published_msg_queue_.size() > attr_.capacity) {
    published_msg_queue_.pop_back();
  }
}
```
Enqueue 函数首先判断 T 类型的 blocker 的容量是否为0，为0的话直接 return ，然后获取 msg 锁，将 msg 放在 published_msg_queue_ 的队首，同时判断此时的队列大小是否超过了 blocker 的容量，超过了的话将 published_msg_queue_ 队尾的数据扔掉
**Notify函数**
```c++
template <typename T>
void Blocker<T>::Notify(const MessagePtr& msg) {
  std::lock_guard<std::mutex> lock(cb_mutex_);
  for (const auto& item : published_callbacks_) {
    item.second(msg);
  }
}
```
Notify 函数就是获得回调函数锁，然后去用 msg 做参数去调用各个订阅的 callback
还有一些其他函数只是单纯的功能实现，这里就不再赘述
## intra_reader.h
intra_reader 是在仿真模式下用的 reader, 我们先看定义：
```c++
template <typename MessageT>
class IntraReader : public apollo::cyber::Reader<MessageT> {
 public:
  using MessagePtr = std::shared_ptr<MessageT>;
  using Callback = std::function<void(const std::shared_ptr<MessageT>&)>;
  using Iterator =
      typename std::list<std::shared_ptr<MessageT>>::const_iterator;

  IntraReader(const proto::RoleAttributes& attr, const Callback& callback);
  virtual ~IntraReader();

  bool Init() override;
  void Shutdown() override;

  void ClearData() override;
  void Observe() override;
  bool Empty() const override;
  bool HasReceived() const override;

  void Enqueue(const std::shared_ptr<MessageT>& msg) override;
  void SetHistoryDepth(const uint32_t& depth) override;
  uint32_t GetHistoryDepth() const override;
  std::shared_ptr<MessageT> GetLatestObserved() const override;
  std::shared_ptr<MessageT> GetOldestObserved() const override;
  Iterator Begin() const override;
  Iterator End() const override;

 private:
  void OnMessage(const MessagePtr& msg_ptr);

  Callback msg_callback_;
};
```
可以看到 intra_reader 基本上只是 override reader 的一些接口, 多了一个 OnMessage 接口

再看一下具体实现：
```c++
template <typename MessageT>
IntraReader<MessageT>::IntraReader(const proto::RoleAttributes& attr,
                                   const Callback& callback)
    : Reader<MessageT>(attr), msg_callback_(callback) {}

template <typename MessageT>
IntraReader<MessageT>::~IntraReader() {
  Shutdown();
}
```
构造函数中去构造基类对象并设置 callback, 析构函数调用 Shutdown

```c++
template <typename MessageT>
bool IntraReader<MessageT>::Init() {
  if (this->init_.exchange(true)) {
    return true;
  }
  return BlockerManager::Instance()->Subscribe<MessageT>(
      this->role_attr_.channel_name(), this->role_attr_.qos_profile().depth(),
      this->role_attr_.node_name(),
      std::bind(&IntraReader<MessageT>::OnMessage, this,
                std::placeholders::_1));
}

template <typename MessageT>
void IntraReader<MessageT>::Shutdown() {
  if (!this->init_.exchange(false)) {
    return;
  }
  BlockerManager::Instance()->Unsubscribe<MessageT>(
      this->role_attr_.channel_name(), this->role_attr_.node_name());
}
```
Init 函数主要是在 BlockerManager 单例中订阅 channel_name 对应的消息，同时将 OnMessage 设置为回调函数
Shutdown 函数主要是在 BlockerManager 中取消订阅 channel_name 对应的消息
```c++
template <typename MessageT>
void IntraReader<MessageT>::OnMessage(const MessagePtr& msg_ptr) {
  this->second_to_lastest_recv_time_sec_ = this->latest_recv_time_sec_;
  this->latest_recv_time_sec_ = apollo::cyber::Time::Now().ToSecond();
  if (msg_callback_ != nullptr) {
    msg_callback_(msg_ptr);
  }
}
```
OnMessage 函数主要就是调用初始化时候设置的回调函数
其他的一些函数基本都是先从 BlockerManager 中获得对应 blocker, 再转调对应 blocker 的函数

## intra_writer.h
```c++
template <typename MessageT>
class IntraWriter : public apollo::cyber::Writer<MessageT> {
 public:
  using MessagePtr = std::shared_ptr<MessageT>;
  using BlockerManagerPtr = std::shared_ptr<BlockerManager>;

  explicit IntraWriter(const proto::RoleAttributes& attr);
  virtual ~IntraWriter();

  bool Init() override;
  void Shutdown() override;

  bool Write(const MessageT& msg) override;
  bool Write(const MessagePtr& msg_ptr) override;

 private:
  BlockerManagerPtr blocker_manager_;
};
template <typename MessageT>
IntraWriter<MessageT>::IntraWriter(const proto::RoleAttributes& attr)
    : Writer<MessageT>(attr) {}

template <typename MessageT>
IntraWriter<MessageT>::~IntraWriter() {
  Shutdown();
}

template <typename MessageT>
bool IntraWriter<MessageT>::Init() {
  {
    std::lock_guard<std::mutex> g(this->lock_);
    if (this->init_) {
      return true;
    }
    blocker_manager_ = BlockerManager::Instance();
    blocker_manager_->GetOrCreateBlocker<MessageT>(
        BlockerAttr(this->role_attr_.channel_name()));
    this->init_ = true;
  }
  return true;
}

template <typename MessageT>
void IntraWriter<MessageT>::Shutdown() {
  {
    std::lock_guard<std::mutex> g(this->lock_);
    if (!this->init_) {
      return;
    }
    this->init_ = false;
  }
  blocker_manager_ = nullptr;
}

template <typename MessageT>
bool IntraWriter<MessageT>::Write(const MessageT& msg) {
  if (!WriterBase::IsInit()) {
    return false;
  }
  return blocker_manager_->Publish<MessageT>(this->role_attr_.channel_name(),
                                             msg);
}

template <typename MessageT>
bool IntraWriter<MessageT>::Write(const MessagePtr& msg_ptr) {
  if (!WriterBase::IsInit()) {
    return false;
  }
  return blocker_manager_->Publish<MessageT>(this->role_attr_.channel_name(),
                                             msg_ptr);
}
```
intra writer 就是借助 blocker_manager_ 来 Publish 对应的消息，不需要经过传输