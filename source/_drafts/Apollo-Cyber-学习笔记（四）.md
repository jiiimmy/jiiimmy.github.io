---
title: Apollo-Cyber-学习笔记（四）
date: 2022-10-11 14:05:48
tags:
---

此篇继续学习 transport
<!-- more -->
# message
这个目录主要是用来封装一些与 message 相关的函数

## history_attributes.h
```c++
struct HistoryAttributes {
  HistoryAttributes()
      : history_policy(proto::QosHistoryPolicy::HISTORY_KEEP_LAST),
        depth(1000) {}
  HistoryAttributes(const proto::QosHistoryPolicy& qos_history_policy,
                    uint32_t history_depth)
      : history_policy(qos_history_policy), depth(history_depth) {}

  proto::QosHistoryPolicy history_policy;
  uint32_t depth;
};
```
这个文件主要是封装了 history 的属性

## history.h
```c++
template <typename MessageT>
class History {
 public:
  using MessagePtr = std::shared_ptr<MessageT>;
  struct CachedMessage {
    CachedMessage(const MessagePtr& message, const MessageInfo& message_info)
        : msg(message), msg_info(message_info) {}

    MessagePtr msg;
    MessageInfo msg_info;
  };

  explicit History(const HistoryAttributes& attr);
  virtual ~History();

  void Enable() { enabled_ = true; }
  void Disable() { enabled_ = false; }

  void Add(const MessagePtr& msg, const MessageInfo& msg_info);
  void Clear();
  void GetCachedMessage(std::vector<CachedMessage>* msgs) const;
  size_t GetSize() const;

  uint32_t depth() const { return depth_; }
  uint32_t max_depth() const { return max_depth_; }

 private:
  bool enabled_;
  uint32_t depth_;
  uint32_t max_depth_;
  std::list<CachedMessage> msgs_;
  mutable std::mutex msgs_mutex_;
};

template <typename MessageT>
History<MessageT>::History(const HistoryAttributes& attr)
    : enabled_(false), max_depth_(1000) {
  auto& global_conf = common::GlobalData::Instance()->Config();
  if (global_conf.has_transport_conf() &&
      global_conf.transport_conf().has_resource_limit()) {
    max_depth_ =
        global_conf.transport_conf().resource_limit().max_history_depth();
  }

  if (attr.history_policy == proto::QosHistoryPolicy::HISTORY_KEEP_ALL) {
    depth_ = max_depth_;
  } else {
    depth_ = attr.depth;
    if (depth_ > max_depth_) {
      depth_ = max_depth_;
    }
  }
}

template <typename MessageT>
History<MessageT>::~History() {
  Clear();
}

template <typename MessageT>
void History<MessageT>::Add(const MessagePtr& msg,
                            const MessageInfo& msg_info) {
  if (!enabled_) {
    return;
  }
  std::lock_guard<std::mutex> lock(msgs_mutex_);
  msgs_.emplace_back(msg, msg_info);
  while (msgs_.size() > depth_) {
    msgs_.pop_front();
  }
}

template <typename MessageT>
void History<MessageT>::Clear() {
  std::lock_guard<std::mutex> lock(msgs_mutex_);
  msgs_.clear();
}

template <typename MessageT>
void History<MessageT>::GetCachedMessage(
    std::vector<CachedMessage>* msgs) const {
  if (msgs == nullptr) {
    return;
  }

  std::lock_guard<std::mutex> lock(msgs_mutex_);
  msgs->reserve(msgs_.size());
  msgs->insert(msgs->begin(), msgs_.begin(), msgs_.end());
}

template <typename MessageT>
size_t History<MessageT>::GetSize() const {
  std::lock_guard<std::mutex> lock(msgs_mutex_);
  return msgs_.size();
}
```
History 中定义了 CachedMessage 结构体 ，主要就是包含了一个 MessageT 的共享指针和 MessageInfo 变量
History 中的最关键的私有变量为 `std::list<CachedMessage> msgs_` ，它用 list 来缓存 message
History 的构造函数中将 enabled_ 设为false，默认最大缓存为 1000，然后根据 global_conf 和 history_attr 去设置 max_depth 和 depth
Add 函数就是将 msg 和 msg_info 给添加到缓存中，当 size 超过最大限制时弹出首个缓存， Clear 函数就是清空 msgs_ 
GetCachedMessage 是将 msgs_ 拷贝一份给传进来的 vector<CachedMessage> 指针， GetSize 返回 msgs_ 的长度

## message_info.h
```c++
class MessageInfo {
 public:
  MessageInfo();
  MessageInfo(const Identity& sender_id, uint64_t seq_num);
  MessageInfo(const Identity& sender_id, uint64_t seq_num,
              const Identity& spare_id);
  MessageInfo(const MessageInfo& another);
  virtual ~MessageInfo();

  MessageInfo& operator=(const MessageInfo& another);
  bool operator==(const MessageInfo& another) const;
  bool operator!=(const MessageInfo& another) const;

  bool SerializeTo(std::string* dst) const;
  bool SerializeTo(char* dst, std::size_t len) const;
  bool DeserializeFrom(const std::string& src);
  bool DeserializeFrom(const char* src, std::size_t len);

  // getter and setter
  const Identity& sender_id() const { return sender_id_; }
  void set_sender_id(const Identity& sender_id) { sender_id_ = sender_id; }

  uint64_t channel_id() const { return channel_id_; }
  void set_channel_id(uint64_t channel_id) { channel_id_ = channel_id; }

  uint64_t seq_num() const { return seq_num_; }
  void set_seq_num(uint64_t seq_num) { seq_num_ = seq_num; }

  const Identity& spare_id() const { return spare_id_; }
  void set_spare_id(const Identity& spare_id) { spare_id_ = spare_id; }

  static const std::size_t kSize;

 private:
  Identity sender_id_;
  uint64_t channel_id_ = 0;
  uint64_t seq_num_ = 0;
  Identity spare_id_;
};
```
MessageInfo 主要是记录了 sender_id_, channel_id_, seq_num(消息编号), sqare_id_；主要看一下它的序列化和反序列化函数：
```c++
bool MessageInfo::SerializeTo(std::string* dst) const {
  RETURN_VAL_IF_NULL(dst, false);

  dst->assign(sender_id_.data(), ID_SIZE);
  dst->append(reinterpret_cast<const char*>(&seq_num_), sizeof(seq_num_));
  dst->append(spare_id_.data(), ID_SIZE);

  return true;
}

bool MessageInfo::SerializeTo(char* dst, std::size_t len) const {
  if (dst == nullptr || len < kSize) {
    return false;
  }

  char* ptr = dst;
  std::memcpy(ptr, sender_id_.data(), ID_SIZE);
  ptr += ID_SIZE;
  std::memcpy(ptr, reinterpret_cast<const char*>(&seq_num_), sizeof(seq_num_));
  ptr += sizeof(seq_num_);
  std::memcpy(ptr, spare_id_.data(), ID_SIZE);

  return true;
}

bool MessageInfo::DeserializeFrom(const std::string& src) {
  return DeserializeFrom(src.data(), src.size());
}

bool MessageInfo::DeserializeFrom(const char* src, std::size_t len) {
  RETURN_VAL_IF_NULL(src, false);
  if (len != kSize) {
    return false;
  }

  char* ptr = const_cast<char*>(src);
  sender_id_.set_data(ptr);
  ptr += ID_SIZE;
  std::memcpy(reinterpret_cast<char*>(&seq_num_), ptr, sizeof(seq_num_));
  ptr += sizeof(seq_num_);
  spare_id_.set_data(ptr);

  return true;
}
```
支持从 string 类型或 char* 类型的序列化与反序列化，主要调用的是 std::memcpy 函数

## listener_handler.h
```c++
using apollo::cyber::base::AtomicRWLock;
using apollo::cyber::base::ReadLockGuard;
using apollo::cyber::base::WriteLockGuard;

class ListenerHandlerBase;
using ListenerHandlerBasePtr = std::shared_ptr<ListenerHandlerBase>;

class ListenerHandlerBase {
 public:
  ListenerHandlerBase() {}
  virtual ~ListenerHandlerBase() {}

  virtual void Disconnect(uint64_t self_id) = 0;
  virtual void Disconnect(uint64_t self_id, uint64_t oppo_id) = 0;
  inline bool IsRawMessage() const { return is_raw_message_; }
  virtual void RunFromString(const std::string& str,
                             const MessageInfo& msg_info) = 0;

 protected:
  bool is_raw_message_ = false;
};

template <typename MessageT>
class ListenerHandler : public ListenerHandlerBase {
 public:
  using Message = std::shared_ptr<MessageT>;
  using MessageSignal = base::Signal<const Message&, const MessageInfo&>;

  using Listener = std::function<void(const Message&, const MessageInfo&)>;
  using MessageConnection =
      base::Connection<const Message&, const MessageInfo&>;
  using ConnectionMap = std::unordered_map<uint64_t, MessageConnection>;

  ListenerHandler() {}
  virtual ~ListenerHandler() {}

  void Connect(uint64_t self_id, const Listener& listener);
  void Connect(uint64_t self_id, uint64_t oppo_id, const Listener& listener);

  void Disconnect(uint64_t self_id) override;
  void Disconnect(uint64_t self_id, uint64_t oppo_id) override;

  void Run(const Message& msg, const MessageInfo& msg_info);
  void RunFromString(const std::string& str,
                     const MessageInfo& msg_info) override;

 private:
  using SignalPtr = std::shared_ptr<MessageSignal>;
  using MessageSignalMap = std::unordered_map<uint64_t, SignalPtr>;
  // used for self_id
  MessageSignal signal_;
  ConnectionMap signal_conns_;  // key: self_id

  // used for self_id and oppo_id
  MessageSignalMap signals_;  // key: oppo_id
  // key: oppo_id
  std::unordered_map<uint64_t, ConnectionMap> signals_conns_;

  base::AtomicRWLock rw_lock_;
};

template <>
inline ListenerHandler<message::RawMessage>::ListenerHandler() {
  is_raw_message_ = true;
}

template <typename MessageT>
void ListenerHandler<MessageT>::Connect(uint64_t self_id,
                                        const Listener& listener) {
  auto connection = signal_.Connect(listener);
  if (!connection.IsConnected()) {
    return;
  }

  WriteLockGuard<AtomicRWLock> lock(rw_lock_);
  signal_conns_[self_id] = connection;
}

template <typename MessageT>
void ListenerHandler<MessageT>::Connect(uint64_t self_id, uint64_t oppo_id,
                                        const Listener& listener) {
  WriteLockGuard<AtomicRWLock> lock(rw_lock_);
  if (signals_.find(oppo_id) == signals_.end()) {
    signals_[oppo_id] = std::make_shared<MessageSignal>();
  }

  auto connection = signals_[oppo_id]->Connect(listener);
  if (!connection.IsConnected()) {
    return;
  }

  if (signals_conns_.find(oppo_id) == signals_conns_.end()) {
    signals_conns_[oppo_id] = ConnectionMap();
  }

  signals_conns_[oppo_id][self_id] = connection;
}

template <typename MessageT>
void ListenerHandler<MessageT>::Disconnect(uint64_t self_id) {
  WriteLockGuard<AtomicRWLock> lock(rw_lock_);
  if (signal_conns_.find(self_id) == signal_conns_.end()) {
    return;
  }

  signal_conns_[self_id].Disconnect();
  signal_conns_.erase(self_id);
}

template <typename MessageT>
void ListenerHandler<MessageT>::Disconnect(uint64_t self_id, uint64_t oppo_id) {
  WriteLockGuard<AtomicRWLock> lock(rw_lock_);
  if (signals_conns_.find(oppo_id) == signals_conns_.end()) {
    return;
  }

  if (signals_conns_[oppo_id].find(self_id) == signals_conns_[oppo_id].end()) {
    return;
  }

  signals_conns_[oppo_id][self_id].Disconnect();
  signals_conns_[oppo_id].erase(self_id);
}

template <typename MessageT>
void ListenerHandler<MessageT>::Run(const Message& msg,
                                    const MessageInfo& msg_info) {
  signal_(msg, msg_info);
  uint64_t oppo_id = msg_info.sender_id().HashValue();
  ReadLockGuard<AtomicRWLock> lock(rw_lock_);
  if (signals_.find(oppo_id) == signals_.end()) {
    return;
  }

  (*signals_[oppo_id])(msg, msg_info);
}

template <typename MessageT>
void ListenerHandler<MessageT>::RunFromString(const std::string& str,
                                              const MessageInfo& msg_info) {
  auto msg = std::make_shared<MessageT>();
  if (message::ParseFromHC(str.data(), static_cast<int>(str.size()),
                           msg.get())) {
    Run(msg, msg_info);
  }
}
```
**ListenerHandlerBase** 中定义了 Disconnect, RunFromString, IsRawMessage 接口
**MessageSignal** 是 Signal 类的特化，其参数类型为 Message 和 MessageInfo 
**Listener** 是回调函数的封装
**MessageConnection** 是 Connection 类的特化，其参数类型为 Message 和 MessageInfo 
**SignalPtr** 是 MessageSignal 类的共享指针
**MessageSignalMap** 是 id 与 SignalPtr 的 map
Signal, Slot, Connection 的定义在 base/signal.h中，简单来说就是一个**信号-槽**的机制
**ConnectionMap** 是一个 id 与 MessageConnection 的 map
ListenerHandler 的对外接口主要为：
Connect, Disconnect, Run, RunFromString

ListenerHandler Connect 函数是调用 signal 的 Connect 函数，并将其记录在 signal_conns_ 中
ListenerHandler Disconnect 函数也是调用对应 signal 的 Disconnect，并将其在 signal_conns_ 中删除

Run 函数中首先调用 `signal_(msg, msg_info)`，这里 signal 类重载了()运算符，重载中主要做的事情是会去拷贝一份 Signal 对应的槽函数并调用一次，然后清除 Disconnect 的槽函数
然后调用 oppo_id 所对应的 signal 也调用一次()的重载

RunFromString 函数先将 string 构造成 Message 对象，再调用 Run 函数

# rtps
rtps 里封装了一些 rtpsdispatcher 中使用的类

## attributes_filler.h
```c++
class AttributesFiller {
 public:
  AttributesFiller();
  virtual ~AttributesFiller();

  static bool FillInPubAttr(const std::string& channel_name,
                            const QosProfile& qos,
                            eprosima::fastrtps::PublisherAttributes* pub_attr);

  static bool FillInSubAttr(const std::string& channel_name,
                            const QosProfile& qos,
                            eprosima::fastrtps::SubscriberAttributes* sub_attr);
};
```
AttributesFiller 主要是将 QosProfile 中的设置项去匹配设置 fastrtps 中的对应项