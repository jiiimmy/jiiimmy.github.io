---
title: Apollo Cyber 学习笔记（一）
date: 2022-09-02 13:49:14
tags:
---

这篇来介绍对 node 的理解
<!-- more -->

node 的目录结构如下：
```shell
├── BUILD
├── node.cc
├── node_channel_impl.h
├── node_channel_impl_test.cc
├── node.h
├── node_service_impl.h
├── node_test.cc
├── reader_base.h
├── reader.h
├── reader_test.cc
├── writer_base.h
├── writer.h
├── writer_reader_test.cc
└── writer_test.cc
```
# Node的作用
Node 是Cyber RT的基本构建块，每个模块都包含并通过节点进行通信。通过在节点中定义读/写 或 服务/客户端，模块可以具有不同类型的通信。

首先看一下 node.h 的定义，简化掉了部分注释
```c++
class Node {
 public:
  template <typename M0, typename M1, typename M2, typename M3>
  friend class Component;
  friend class TimerComponent;
  friend bool Init(const char*);
  friend std::unique_ptr<Node> CreateNode(const std::string&,
                                          const std::string&);
  virtual ~Node();

  const std::string& Name() const;

  template <typename MessageT>
  auto CreateWriter(const proto::RoleAttributes& role_attr)
      -> std::shared_ptr<Writer<MessageT>>;

  template <typename MessageT>
  auto CreateWriter(const std::string& channel_name)
      -> std::shared_ptr<Writer<MessageT>>;

  template <typename MessageT>
  auto CreateReader(const std::string& channel_name,
                    const CallbackFunc<MessageT>& reader_func = nullptr)
      -> std::shared_ptr<cyber::Reader<MessageT>>;

  template <typename MessageT>
  auto CreateReader(const ReaderConfig& config,
                    const CallbackFunc<MessageT>& reader_func = nullptr)
      -> std::shared_ptr<cyber::Reader<MessageT>>;

  template <typename MessageT>
  auto CreateReader(const proto::RoleAttributes& role_attr,
                    const CallbackFunc<MessageT>& reader_func = nullptr)
      -> std::shared_ptr<cyber::Reader<MessageT>>;

  template <typename Request, typename Response>
  auto CreateService(const std::string& service_name,
                     const typename Service<Request, Response>::ServiceCallback&
                         service_callback)
      -> std::shared_ptr<Service<Request, Response>>;

  template <typename Request, typename Response>
  auto CreateClient(const std::string& service_name)
      -> std::shared_ptr<Client<Request, Response>>;

  void Observe();

  void ClearData();

  template <typename MessageT>
  auto GetReader(const std::string& channel_name)
      -> std::shared_ptr<Reader<MessageT>>;

 private:
  explicit Node(const std::string& node_name,
                const std::string& name_space = "");

  std::string node_name_;
  std::string name_space_;

  std::mutex readers_mutex_;
  std::map<std::string, std::shared_ptr<ReaderBase>> readers_;

  std::unique_ptr<NodeChannelImpl> node_channel_impl_ = nullptr;
  std::unique_ptr<NodeServiceImpl> node_service_impl_ = nullptr;
};
```
Node 类定义了2个友元类和两个友元函数，并且将构造函数私有化，这表明只能通过 Component 、TimerComponent 类以及 Init、CreateNode 两个函数来构造新的 Node。
public 部分提供了2个创建 writer 和3个创建 reader 的接口，1个创建 service 、1个创建 client 的接口
Name() 提供了访问 node 名字的接口
Observe() 提供了观察所有 reader 数据的接口，ClearData() 提供了清除所有 reader 数据的接口
GetReader() 提供了根据 channel name 获得对应 reader 的共享指针的接口
private 部分：
Node 构造函数需要提供 node_name 和 name_space ，其中 name_space 有默认值
定义了一个 readers_mutex_ 的锁 ， 一个 `std::map<std::string, std::shared_ptr<ReaderBase>>` 的readers
一个 NodeChannelImpl 的 unique_ptr 和一个 NodeServiceImpl 的 unique_ptr 

再看一下他们的实现：
```c++
template <typename MessageT>
auto Node::CreateWriter(const proto::RoleAttributes& role_attr)
    -> std::shared_ptr<Writer<MessageT>> {
  return node_channel_impl_->template CreateWriter<MessageT>(role_attr);
}

template <typename MessageT>
auto Node::CreateWriter(const std::string& channel_name)
    -> std::shared_ptr<Writer<MessageT>> {
  return node_channel_impl_->template CreateWriter<MessageT>(channel_name);
}

template <typename MessageT>
auto Node::CreateReader(const proto::RoleAttributes& role_attr,
                        const CallbackFunc<MessageT>& reader_func)
    -> std::shared_ptr<Reader<MessageT>> {
  std::lock_guard<std::mutex> lg(readers_mutex_);
  if (readers_.find(role_attr.channel_name()) != readers_.end()) {
    return nullptr;
  }
  auto reader = node_channel_impl_->template CreateReader<MessageT>(
      role_attr, reader_func);
  if (reader != nullptr) {
    readers_.emplace(std::make_pair(role_attr.channel_name(), reader));
  }
  return reader;
}

template <typename MessageT>
auto Node::CreateReader(const ReaderConfig& config,
                        const CallbackFunc<MessageT>& reader_func)
    -> std::shared_ptr<cyber::Reader<MessageT>> {
  std::lock_guard<std::mutex> lg(readers_mutex_);
  if (readers_.find(config.channel_name) != readers_.end()) {
    return nullptr;
  }
  auto reader =
      node_channel_impl_->template CreateReader<MessageT>(config, reader_func);
  if (reader != nullptr) {
    readers_.emplace(std::make_pair(config.channel_name, reader));
  }
  return reader;
}

template <typename MessageT>
auto Node::CreateReader(const std::string& channel_name,
                        const CallbackFunc<MessageT>& reader_func)
    -> std::shared_ptr<Reader<MessageT>> {
  std::lock_guard<std::mutex> lg(readers_mutex_);
  if (readers_.find(channel_name) != readers_.end()) {
    return nullptr;
  }
  auto reader = node_channel_impl_->template CreateReader<MessageT>(
      channel_name, reader_func);
  if (reader != nullptr) {
    readers_.emplace(std::make_pair(channel_name, reader));
  }
  return reader;
}

template <typename Request, typename Response>
auto Node::CreateService(
    const std::string& service_name,
    const typename Service<Request, Response>::ServiceCallback&
        service_callback) -> std::shared_ptr<Service<Request, Response>> {
  return node_service_impl_->template CreateService<Request, Response>(
      service_name, service_callback);
}

template <typename Request, typename Response>
auto Node::CreateClient(const std::string& service_name)
    -> std::shared_ptr<Client<Request, Response>> {
  return node_service_impl_->template CreateClient<Request, Response>(
      service_name);
}

template <typename MessageT>
auto Node::GetReader(const std::string& name)
    -> std::shared_ptr<Reader<MessageT>> {
  std::lock_guard<std::mutex> lg(readers_mutex_);
  auto it = readers_.find(name);
  if (it != readers_.end()) {
    return std::dynamic_pointer_cast<Reader<MessageT>>(it->second);
  }
  return nullptr;
}
```
从代码可以看到 node 的 reader 和 writer 实际上是通过调用 node_channel_impl_ 中的接口去创建 reader 和 writer， service 和 client 是通过 node_service_impl_ 去创建 service 和 client ，接下来先看 node_channel_impl 是如何创建 reader 和writer 的

## node_channel_impl.h

首先定义了一个 ReaderConfig的结构体，代码如下：
```c++
struct ReaderConfig {  
  ReaderConfig() {
    qos_profile.set_history(proto::QosHistoryPolicy::HISTORY_KEEP_LAST);
    qos_profile.set_depth(1);
    qos_profile.set_mps(0);
    qos_profile.set_reliability(
        proto::QosReliabilityPolicy::RELIABILITY_RELIABLE);
    qos_profile.set_durability(proto::QosDurabilityPolicy::DURABILITY_VOLATILE);

    pending_queue_size = DEFAULT_PENDING_QUEUE_SIZE;
  }
  ReaderConfig(const ReaderConfig& other)
      : channel_name(other.channel_name),
        qos_profile(other.qos_profile),
        pending_queue_size(other.pending_queue_size) {}

  std::string channel_name;
  proto::QosProfile qos_profile;
 
  uint32_t pending_queue_size;
};
```
Config 里主要是封装了 qos_profile 的 proto，以及 channel name 和 pending_queue_size ，这里的 Config 定义了 ChannelBuffer 对数据的处理方式， 默认情况下当老的数据没有来得及被消费又有新数据到来时会丢弃原数据，qos_profile proto 的定义为：
```c++
syntax = "proto2";

package apollo.cyber.proto;

enum QosHistoryPolicy {
  HISTORY_SYSTEM_DEFAULT = 0;
  HISTORY_KEEP_LAST = 1;
  HISTORY_KEEP_ALL = 2;
};

enum QosReliabilityPolicy {
  RELIABILITY_SYSTEM_DEFAULT = 0;
  RELIABILITY_RELIABLE = 1;
  RELIABILITY_BEST_EFFORT = 2;
};

enum QosDurabilityPolicy {
  DURABILITY_SYSTEM_DEFAULT = 0;
  DURABILITY_TRANSIENT_LOCAL = 1;
  DURABILITY_VOLATILE = 2;
};

message QosProfile {
  optional QosHistoryPolicy history = 1 [default = HISTORY_KEEP_LAST];
  optional uint32 depth = 2 [default = 1];  // capacity of history
  optional uint32 mps = 3 [default = 0];    // messages per second
  optional QosReliabilityPolicy reliability = 4
      [default = RELIABILITY_RELIABLE];
  optional QosDurabilityPolicy durability = 5 [default = DURABILITY_VOLATILE];
};
```

再看 NodeChannelImpl 类的定义：
```c++
class NodeChannelImpl {
  friend class Node;

 public:
  using NodeManagerPtr = std::shared_ptr<service_discovery::NodeManager>;

  explicit NodeChannelImpl(const std::string& node_name)
      : is_reality_mode_(true), node_name_(node_name) {
    node_attr_.set_host_name(common::GlobalData::Instance()->HostName());
    node_attr_.set_host_ip(common::GlobalData::Instance()->HostIp());
    node_attr_.set_process_id(common::GlobalData::Instance()->ProcessId());
    node_attr_.set_node_name(node_name);
    uint64_t node_id = common::GlobalData::RegisterNode(node_name);
    node_attr_.set_node_id(node_id);

    is_reality_mode_ = common::GlobalData::Instance()->IsRealityMode();

    if (is_reality_mode_) {
      node_manager_ =
          service_discovery::TopologyManager::Instance()->node_manager();
      node_manager_->Join(node_attr_, RoleType::ROLE_NODE);
    }
  }

  virtual ~NodeChannelImpl() {
    if (is_reality_mode_) {
      node_manager_->Leave(node_attr_, RoleType::ROLE_NODE);
      node_manager_ = nullptr;
    }
  }

  const std::string& NodeName() const { return node_name_; }

 private:
  template <typename MessageT>
  auto CreateWriter(const proto::RoleAttributes& role_attr)
      -> std::shared_ptr<Writer<MessageT>>;

  template <typename MessageT>
  auto CreateWriter(const std::string& channel_name)
      -> std::shared_ptr<Writer<MessageT>>;

  template <typename MessageT>
  auto CreateReader(const std::string& channel_name,
                    const CallbackFunc<MessageT>& reader_func)
      -> std::shared_ptr<Reader<MessageT>>;

  template <typename MessageT>
  auto CreateReader(const ReaderConfig& config,
                    const CallbackFunc<MessageT>& reader_func)
      -> std::shared_ptr<Reader<MessageT>>;

  template <typename MessageT>
  auto CreateReader(const proto::RoleAttributes& role_attr,
                    const CallbackFunc<MessageT>& reader_func,
                    uint32_t pending_queue_size = DEFAULT_PENDING_QUEUE_SIZE)
      -> std::shared_ptr<Reader<MessageT>>;

  template <typename MessageT>
  auto CreateReader(const proto::RoleAttributes& role_attr)
      -> std::shared_ptr<Reader<MessageT>>;

  template <typename MessageT>
  void FillInAttr(proto::RoleAttributes* attr);

  bool is_reality_mode_;
  std::string node_name_;
  proto::RoleAttributes node_attr_;
  NodeManagerPtr node_manager_ = nullptr;
};
```
先看 public： 
- 定义了一个显示构造函数 `explicit NodeChannelImpl(const std::string& node_name)` ，其作用为根据 GlobalData 去设置 node_attr，然后根据是否为实车模式决定是否将 node_attr_ 加入到 node_manager中
- 虚析构函数，如何是实车模式，将 node_attr_ 从 node_manager_ 中撤出，并将 node_manager_ 置为 nullptr
- NodeName 函数，返回 node_name_ 

再看 private：
- 私有变量 is_reality_mode_, node_name_, node_attr_, node_manager_
- 2个创建 writer 的接口
- 4个创建 reader 的接口
- 1个 FillInAttr 的接口

具体实现为（去掉了冗余LOG代码）：
```c++
template <typename MessageT>
auto NodeChannelImpl::CreateWriter(const proto::RoleAttributes& role_attr)
    -> std::shared_ptr<Writer<MessageT>> {
  if (!role_attr.has_channel_name() || role_attr.channel_name().empty()) {
    return nullptr;
  }
  proto::RoleAttributes new_attr(role_attr);
  FillInAttr<MessageT>(&new_attr);

  std::shared_ptr<Writer<MessageT>> writer_ptr = nullptr;
  if (!is_reality_mode_) {
    writer_ptr = std::make_shared<blocker::IntraWriter<MessageT>>(new_attr);
  } else {
    writer_ptr = std::make_shared<Writer<MessageT>>(new_attr);
  }

  RETURN_VAL_IF_NULL(writer_ptr, nullptr);
  RETURN_VAL_IF(!writer_ptr->Init(), nullptr);
  return writer_ptr;
}

template <typename MessageT>
auto NodeChannelImpl::CreateWriter(const std::string& channel_name)
    -> std::shared_ptr<Writer<MessageT>> {
  proto::RoleAttributes role_attr;
  role_attr.set_channel_name(channel_name);
  return this->CreateWriter<MessageT>(role_attr);
}

template <typename MessageT>
auto NodeChannelImpl::CreateReader(const std::string& channel_name,
                                   const CallbackFunc<MessageT>& reader_func)
    -> std::shared_ptr<Reader<MessageT>> {
  proto::RoleAttributes role_attr;
  role_attr.set_channel_name(channel_name);
  return this->template CreateReader<MessageT>(role_attr, reader_func);
}

template <typename MessageT>
auto NodeChannelImpl::CreateReader(const ReaderConfig& config,
                                   const CallbackFunc<MessageT>& reader_func)
    -> std::shared_ptr<Reader<MessageT>> {
  proto::RoleAttributes role_attr;
  role_attr.set_channel_name(config.channel_name);
  role_attr.mutable_qos_profile()->CopyFrom(config.qos_profile);
  return this->template CreateReader<MessageT>(role_attr, reader_func,
                                               config.pending_queue_size);
}

template <typename MessageT>
auto NodeChannelImpl::CreateReader(const proto::RoleAttributes& role_attr,
                                   const CallbackFunc<MessageT>& reader_func,
                                   uint32_t pending_queue_size)
    -> std::shared_ptr<Reader<MessageT>> {
  if (!role_attr.has_channel_name() || role_attr.channel_name().empty()) {
    return nullptr;
  }

  proto::RoleAttributes new_attr(role_attr);
  FillInAttr<MessageT>(&new_attr);

  std::shared_ptr<Reader<MessageT>> reader_ptr = nullptr;
  if (!is_reality_mode_) {
    reader_ptr =
        std::make_shared<blocker::IntraReader<MessageT>>(new_attr, reader_func);
  } else {
    reader_ptr = std::make_shared<Reader<MessageT>>(new_attr, reader_func,
                                                    pending_queue_size);
  }

  RETURN_VAL_IF_NULL(reader_ptr, nullptr);
  RETURN_VAL_IF(!reader_ptr->Init(), nullptr);
  return reader_ptr;
}

template <typename MessageT>
auto NodeChannelImpl::CreateReader(const proto::RoleAttributes& role_attr)
    -> std::shared_ptr<Reader<MessageT>> {
  return this->template CreateReader<MessageT>(role_attr, nullptr);
}

template <typename MessageT>
void NodeChannelImpl::FillInAttr(proto::RoleAttributes* attr) {
  attr->set_host_name(node_attr_.host_name());
  attr->set_host_ip(node_attr_.host_ip());
  attr->set_process_id(node_attr_.process_id());
  attr->set_node_name(node_attr_.node_name());
  attr->set_node_id(node_attr_.node_id());
  auto channel_id = GlobalData::RegisterChannel(attr->channel_name());
  attr->set_channel_id(channel_id);
  if (!attr->has_message_type()) {
    attr->set_message_type(message::MessageType<MessageT>());
  }
  if (!attr->has_proto_desc()) {
    std::string proto_desc("");
    message::GetDescriptorString<MessageT>(attr->message_type(), &proto_desc);
    attr->set_proto_desc(proto_desc);
  }
  if (!attr->has_qos_profile()) {
    attr->mutable_qos_profile()->CopyFrom(
        transport::QosProfileConf::QOS_PROFILE_DEFAULT);
  }
}
```
可以看到，真正创建 writer 的函数只有一个，即形参为 role_attr 的函数，这个函数做的事情为：
1. 根据 role_attr 拷贝构造一个新的 new_attr；
2. 调用 FillInAttr 给 new_attr 的字段赋值
3. 根据是否为实车模式创建两种不同的 writer ，实车为同级目录下的 Writer ，仿真为blocker下的 IntraWrite
4. 检查创建的 writer 是否为空指针以及 writer 是否已经初始化
5. 返回 writer 的 share_ptr

真正创建 reader 的函数也只有一个，即形参为 role_attr, reader_func, pending_queue_size 的函数，这个函数做的事情为：
1. 根据 role_attr 拷贝构造一个新的 new_attr；
2. 调用 FillInAttr 给 new_attr 的字段赋值
3. 根据是否为实车模式创建两种不同的 reader ，实车为同级目录下的 Reader ，仿真为blocker下的 IntraReader
4. 检查创建的 reader 是否为空指针以及 reader 是否已经初始化
5. 返回 reader 的 share_ptr

对于 FillInAttr 函数，其做了如下事情：
1. 将构造时候初始化的 host_name, host_ip, process_id, node_name, node_id 设置给传进来的新 attr 指针所指内容
2. 向 GlobalData 中注册 channel 得到一个 channel_id ，然后设置到 attr 中
3. 如果没有 attr 中没有 message_type, proto_desc, qos_profile 属性，给他们设置一个默认值**（具体什么作用暂时不清楚）**

再来看看实车模式下的 writer 和 reader 做了些什么事情：
## writer.h
writer 的定义为：
```c++
template <typename MessageT>
class Writer : public WriterBase {
 public:
  using TransmitterPtr = std::shared_ptr<transport::Transmitter<MessageT>>;
  using ChangeConnection =
      typename service_discovery::Manager::ChangeConnection;

  explicit Writer(const proto::RoleAttributes& role_attr);
  virtual ~Writer();

  bool Init() override;

  void Shutdown() override;

  virtual bool Write(const MessageT& msg);

  virtual bool Write(const std::shared_ptr<MessageT>& msg_ptr);

  bool HasReader() override;

  void GetReaders(std::vector<proto::RoleAttributes>* readers) override;

 private:
  void JoinTheTopology();
  void LeaveTheTopology();
  void OnChannelChange(const proto::ChangeMsg& change_msg);

  TransmitterPtr transmitter_;

  ChangeConnection change_conn_;
  service_discovery::ChannelManagerPtr channel_manager_;
};
```
先看 public 部分：
Writer 只能通过 role_attr 来显示构造， 重写了 Init, Shutdown, HasReader 以及 GetReaders 函数，定义了2个 write 函数，参数分别是 MessageT 和 MessageT 的 shared_ptr
再看 private 部分：
有3个函数 JoinTheTopology, LeaveTheTopology, OnChannelChange, 一个 Transmitter 类的共享指针 transmitter_, 一个 ChangeConnection 变量 change_conn_, 一个 管理 channel 的 channel_manager_ 

再看**各个函数具体做的事情**
Writer 的构造函数与析构函数：
```c++
template <typename MessageT>
Writer<MessageT>::Writer(const proto::RoleAttributes& role_attr)
    : WriterBase(role_attr), transmitter_(nullptr), channel_manager_(nullptr) {}

template <typename MessageT>
Writer<MessageT>::~Writer() {
  Shutdown();
}
```
构造函数先用 role_attr 去构造 WriterBase 类，将 transmitter_, channel_manager_ 设置成 nullptr ，析构函数调用 Shutdown 函数

**Init 函数**
```c++
template <typename MessageT>
bool Writer<MessageT>::Init() {
  {
    std::lock_guard<std::mutex> g(lock_);
    if (init_) {
      return true;
    }
    transmitter_ =
        transport::Transport::Instance()->CreateTransmitter<MessageT>(
            role_attr_);
    if (transmitter_ == nullptr) {
      return false;
    }
    init_ = true;
  }
  this->role_attr_.set_id(transmitter_->id().HashValue());
  channel_manager_ =
      service_discovery::TopologyManager::Instance()->channel_manager();
  JoinTheTopology();
  return true;
}
```
在 Init 函数中，首先获得基类 WriterBase 的锁 和 role_attr_ 去向 Transport 的单例创建一个 MessageT 的 transmitter_, 然后获得 TopologyManager 单例的 channel_manager 给 channel_manager_ ， 最后调用 JoinTheTopology 函数

**Shutdown函数**
```c++
template <typename MessageT>
void Writer<MessageT>::Shutdown() {
  {
    std::lock_guard<std::mutex> g(lock_);
    if (!init_) {
      return;
    }
    init_ = false;
  }
  LeaveTheTopology();
  transmitter_ = nullptr;
  channel_manager_ = nullptr;
}
```
在 Shutdown 函数中，首先获得基类 WriterBase 的锁，将 init 变量设为 false, 然后调用 LeaveTheTopology 函数，最后把 transmitter_ 和 channel_manager_ 设为空指针，释放其拥有的资源

**Write函数**

```c++
template <typename MessageT>
bool Writer<MessageT>::Write(const MessageT& msg) {
  RETURN_VAL_IF(!WriterBase::IsInit(), false);
  auto msg_ptr = std::make_shared<MessageT>(msg);
  return Write(msg_ptr);
}

template <typename MessageT>
bool Writer<MessageT>::Write(const std::shared_ptr<MessageT>& msg_ptr) {
  RETURN_VAL_IF(!WriterBase::IsInit(), false);
  return transmitter_->Transmit(msg_ptr);
}
```
这里可以看到 Write 是将 MessageT 的 msg 转化成 shared_ptr ，最终调用 transmitter_ 的 Transmit 函数，说明消息的发送最后是通过 transmitter_ 来实现的

**其他函数**
```c++
template <typename MessageT>
void Writer<MessageT>::JoinTheTopology() {
  // add listener
  change_conn_ = channel_manager_->AddChangeListener(std::bind(
      &Writer<MessageT>::OnChannelChange, this, std::placeholders::_1));

  // get peer readers
  const std::string& channel_name = this->role_attr_.channel_name();
  std::vector<proto::RoleAttributes> readers;
  channel_manager_->GetReadersOfChannel(channel_name, &readers);
  for (auto& reader : readers) {
    transmitter_->Enable(reader);
  }

  channel_manager_->Join(this->role_attr_, proto::RoleType::ROLE_WRITER,
                         message::HasSerializer<MessageT>::value);
}

template <typename MessageT>
void Writer<MessageT>::LeaveTheTopology() {
  channel_manager_->RemoveChangeListener(change_conn_);
  channel_manager_->Leave(this->role_attr_, proto::RoleType::ROLE_WRITER);
}

template <typename MessageT>
void Writer<MessageT>::OnChannelChange(const proto::ChangeMsg& change_msg) {
  if (change_msg.role_type() != proto::RoleType::ROLE_READER) {
    return;
  }

  auto& reader_attr = change_msg.role_attr();
  if (reader_attr.channel_name() != this->role_attr_.channel_name()) {
    return;
  }

  auto operate_type = change_msg.operate_type();
  if (operate_type == proto::OperateType::OPT_JOIN) {
    transmitter_->Enable(reader_attr);
  } else {
    transmitter_->Disable(reader_attr);
  }
}

template <typename MessageT>
bool Writer<MessageT>::HasReader() {
  RETURN_VAL_IF(!WriterBase::IsInit(), false);
  return channel_manager_->HasReader(role_attr_.channel_name());
}

template <typename MessageT>
void Writer<MessageT>::GetReaders(std::vector<proto::RoleAttributes>* readers) {
  if (readers == nullptr) {
    return;
  }

  if (!WriterBase::IsInit()) {
    return;
  }

  channel_manager_->GetReadersOfChannel(role_attr_.channel_name(), readers);
}
```
暂时不写作用

## writer_base.h
```c++
class WriterBase {
 public:
  explicit WriterBase(const proto::RoleAttributes& role_attr)
      : role_attr_(role_attr), init_(false) {}
  virtual ~WriterBase() {}

  virtual bool Init() = 0;

  virtual void Shutdown() = 0;

  virtual bool HasReader() { return false; }

  virtual void GetReaders(std::vector<proto::RoleAttributes>* readers) {}

  const std::string& GetChannelName() const {
    return role_attr_.channel_name();
  }

  bool IsInit() const {
    std::lock_guard<std::mutex> g(lock_);
    return init_;
  }

 protected:
  proto::RoleAttributes role_attr_;
  mutable std::mutex lock_;
  bool init_;
};

```
在 writer_base 中定义了一些接口和公用变量
## reader.h

reader 首先定义了回调函数：
```c++
template <typename M0>
using CallbackFunc = std::function<void(const std::shared_ptr<M0>&)>;
```
Reader 的定义如下：
```c++
template <typename MessageT>
class Reader : public ReaderBase {
 public:
  using BlockerPtr = std::unique_ptr<blocker::Blocker<MessageT>>;
  using ReceiverPtr = std::shared_ptr<transport::Receiver<MessageT>>;
  using ChangeConnection =
      typename service_discovery::Manager::ChangeConnection;
  using Iterator =
      typename std::list<std::shared_ptr<MessageT>>::const_iterator;
  explicit Reader(const proto::RoleAttributes& role_attr,
                  const CallbackFunc<MessageT>& reader_func = nullptr,
                  uint32_t pending_queue_size = DEFAULT_PENDING_QUEUE_SIZE);
  virtual ~Reader();
  bool Init() override;
  void Shutdown() override;
  void Observe() override;
  void ClearData() override;
  bool HasReceived() const override;
  bool Empty() const override;
  double GetDelaySec() const override;
  uint32_t PendingQueueSize() const override;
  virtual void Enqueue(const std::shared_ptr<MessageT>& msg);
  virtual void SetHistoryDepth(const uint32_t& depth);
  virtual uint32_t GetHistoryDepth() const;
  virtual std::shared_ptr<MessageT> GetLatestObserved() const;
  virtual std::shared_ptr<MessageT> GetOldestObserved() const;
  virtual Iterator Begin() const { return blocker_->ObservedBegin(); }
  virtual Iterator End() const { return blocker_->ObservedEnd(); }
  bool HasWriter() override;

  void GetWriters(std::vector<proto::RoleAttributes>* writers) override;

 protected:
  double latest_recv_time_sec_ = -1.0;
  double second_to_lastest_recv_time_sec_ = -1.0;
  uint32_t pending_queue_size_;

 private:
  void JoinTheTopology();
  void LeaveTheTopology();
  void OnChannelChange(const proto::ChangeMsg& change_msg);

  CallbackFunc<MessageT> reader_func_;
  ReceiverPtr receiver_ = nullptr;
  std::string croutine_name_;

  BlockerPtr blocker_ = nullptr;

  ChangeConnection change_conn_;
  service_discovery::ChannelManagerPtr channel_manager_ = nullptr;
};
```
Reader 的接口比较多，这里只写几个比较关键的接口及私有变量
- reader_func_, reader 的回调函数
- receiver_， transport::Receiver<MessageT> 的一个共享指针
- croutine_name_, 协程的名字
- blocker_, blocker::Blocker<MessageT>的 unique_ptr
有不少部分和 writer 的是一样的这里就不再赘述

**Reader构造函数**
```c++
template <typename MessageT>
Reader<MessageT>::Reader(const proto::RoleAttributes& role_attr,
                         const CallbackFunc<MessageT>& reader_func,
                         uint32_t pending_queue_size)
    : ReaderBase(role_attr),
      pending_queue_size_(pending_queue_size),
      reader_func_(reader_func) {
  blocker_.reset(new blocker::Blocker<MessageT>(blocker::BlockerAttr(
      role_attr.qos_profile().depth(), role_attr.channel_name())));
}
```
构造函数做了以下几件事：
- 根据 role_attr 去构造 ReaderBase
- 设置 pending_queue_size_
- 设置好回调函数 reader_func_
- new一个新的 Blocker 对象，并将其交给 blocker_ 管理

**Init 函数**
```c++
template <typename MessageT>
bool Reader<MessageT>::Init() {
  if (init_.exchange(true)) {
    return true;
  }
  std::function<void(const std::shared_ptr<MessageT>&)> func;
  if (reader_func_ != nullptr) {
    func = [this](const std::shared_ptr<MessageT>& msg) {
      this->Enqueue(msg);
      this->reader_func_(msg);
    };
  } else {
    func = [this](const std::shared_ptr<MessageT>& msg) { this->Enqueue(msg); };
  }
  auto sched = scheduler::Instance();
  croutine_name_ = role_attr_.node_name() + "_" + role_attr_.channel_name();
  auto dv = std::make_shared<data::DataVisitor<MessageT>>(
      role_attr_.channel_id(), pending_queue_size_);
  // Using factory to wrap templates.
  croutine::RoutineFactory factory =
      croutine::CreateRoutineFactory<MessageT>(std::move(func), dv);
  if (!sched->CreateTask(factory, croutine_name_)) {
    init_.store(false);
    return false;
  }

  receiver_ = ReceiverManager<MessageT>::Instance()->GetReceiver(role_attr_);
  this->role_attr_.set_id(receiver_->id().HashValue());
  channel_manager_ =
      service_discovery::TopologyManager::Instance()->channel_manager();
  JoinTheTopology();

  return true;
}
```
Init 函数做了以下几件事：
- 在构建 Reader 时候的回调函数的基础上封装一层 Enqueue 函数形成新的回调函数 func
- 将 node_name 与 channel_name 结合形成 croutine_name_
- 根据 channel_id 和 pending_queue_size_ 去构造一个 DataVisitor 的共享指针 dv
- 用 func 和 dv 去构建一个 RoutineFactory 的实例 factory
- 用 factory 和 croutine_name_ 去全局的 sched 创建一个 task
- 在 ReceiverManager 中根据 role_attr_ 获得 MessageT 类型的 receiver
- 在 TopologyManager 中获得 channel_manager_ 
- 调用 JoinTheTopology 函数

**Shutdown 函数**
```c++
template <typename MessageT>
void Reader<MessageT>::Shutdown() {
  if (!init_.exchange(false)) {
    return;
  }
  LeaveTheTopology();
  receiver_ = nullptr;
  channel_manager_ = nullptr;

  if (!croutine_name_.empty()) {
    scheduler::Instance()->RemoveTask(croutine_name_);
  }
}
```
Shutdown 做的事情为：
- 调用 LeaveTheTopology 函数
- 将 receiver_, channel_manager_ 重置为 nullptr
- 从 scheduler 中移除 task

**其他函数**
```c++
template <typename MessageT>
bool Reader<MessageT>::HasReceived() const {
  return !blocker_->IsPublishedEmpty();
}

template <typename MessageT>
bool Reader<MessageT>::Empty() const {
  return blocker_->IsObservedEmpty();
}

template <typename MessageT>
std::shared_ptr<MessageT> Reader<MessageT>::GetLatestObserved() const {
  return blocker_->GetLatestObservedPtr();
}

template <typename MessageT>
std::shared_ptr<MessageT> Reader<MessageT>::GetOldestObserved() const {
  return blocker_->GetOldestObservedPtr();
}

template <typename MessageT>
void Reader<MessageT>::ClearData() {
  blocker_->ClearPublished();
  blocker_->ClearObserved();
}

template <typename MessageT>
void Reader<MessageT>::SetHistoryDepth(const uint32_t& depth) {
  blocker_->set_capacity(depth);
}

template <typename MessageT>
uint32_t Reader<MessageT>::GetHistoryDepth() const {
  return static_cast<uint32_t>(blocker_->capacity());
}
```
可以看到 Reader 的很多接口实际上是调用 bloker_ 来实现的，后面再看 blocker 做了些什么事情

## reader_base.h
再来看 ReaderBase 的定义：
```c++
class ReaderBase {
 public:
  explicit ReaderBase(const proto::RoleAttributes& role_attr)
      : role_attr_(role_attr), init_(false) {}
  virtual ~ReaderBase() {}
  virtual bool Init() = 0;
  virtual void Shutdown() = 0;
  virtual void ClearData() = 0;
  virtual void Observe() = 0;
  virtual bool Empty() const = 0;
  virtual bool HasReceived() const = 0;
  virtual double GetDelaySec() const = 0;
  virtual uint32_t PendingQueueSize() const = 0;
  virtual bool HasWriter() { return false; }
  virtual void GetWriters(std::vector<proto::RoleAttributes>* writers) {}
  const std::string& GetChannelName() const {
    return role_attr_.channel_name();
  }
  uint64_t ChannelId() const { return role_attr_.channel_id(); }
  const proto::QosProfile& QosProfile() const {
    return role_attr_.qos_profile();
  }
  bool IsInit() const { return init_.load(); }

 protected:
  proto::RoleAttributes role_attr_;
  std::atomic<bool> init_;
};
```
这里就定义了一些 Reader必须实现的接口以及公共变量

关于 reader 和 writer 暂时写到这，可以看到他们均是调用其他类来实现具体的功能

最后再看 node_service_impl.h 如何去创建 service 和client
## node_service_impl.h
NodeServiceImpl的定义为：
```c++
class NodeServiceImpl {
 public:
  friend class Node;
  explicit NodeServiceImpl(const std::string& node_name)
      : node_name_(node_name) {
    attr_.set_host_name(common::GlobalData::Instance()->HostName());
    attr_.set_process_id(common::GlobalData::Instance()->ProcessId());
    attr_.set_node_name(node_name);
    auto node_id = common::GlobalData::RegisterNode(node_name);
    attr_.set_node_id(node_id);
  }
  NodeServiceImpl() = delete;
  ~NodeServiceImpl() {}

 private:
  template <typename Request, typename Response>
  auto CreateService(const std::string& service_name,
                     const typename Service<Request, Response>::ServiceCallback&
                         service_callback) ->
      typename std::shared_ptr<Service<Request, Response>>;

  template <typename Request, typename Response>
  auto CreateClient(const std::string& service_name) ->
      typename std::shared_ptr<Client<Request, Response>>;

  std::vector<std::weak_ptr<ServiceBase>> service_list_;
  std::vector<std::weak_ptr<ClientBase>> client_list_;
  std::string node_name_;
  proto::RoleAttributes attr_;
};
```
可以看到 CreateService 和 CreateClient 只能通过 Node 来操作，并且 NodeServiceImpl 只能通过有参构造，其内部还维持了一个 service_list_ 的 vector 和一个 client_list_ 的 vector, 再看一下是如何创建客户端和服务端的

```c++
template <typename Request, typename Response>
auto NodeServiceImpl::CreateService(
    const std::string& service_name,
    const typename Service<Request, Response>::ServiceCallback&
        service_callback) ->
    typename std::shared_ptr<Service<Request, Response>> {
  auto service_ptr = std::make_shared<Service<Request, Response>>(
      node_name_, service_name, service_callback);
  RETURN_VAL_IF(!service_ptr->Init(), nullptr);

  service_list_.emplace_back(service_ptr);
  attr_.set_service_name(service_name);
  auto service_id = common::GlobalData::RegisterService(service_name);
  attr_.set_service_id(service_id);
  service_discovery::TopologyManager::Instance()->service_manager()->Join(
      attr_, RoleType::ROLE_SERVER);
  return service_ptr;
}

template <typename Request, typename Response>
auto NodeServiceImpl::CreateClient(const std::string& service_name) ->
    typename std::shared_ptr<Client<Request, Response>> {
  auto client_ptr =
      std::make_shared<Client<Request, Response>>(node_name_, service_name);
  RETURN_VAL_IF(!client_ptr->Init(), nullptr);

  client_list_.emplace_back(client_ptr);
  attr_.set_service_name(service_name);
  auto service_id = common::GlobalData::RegisterService(service_name);
  attr_.set_service_id(service_id);
  service_discovery::TopologyManager::Instance()->service_manager()->Join(
      attr_, RoleType::ROLE_CLIENT);
  return client_ptr;
}
```
对于 service, 首先用 node_name, service_name, service_callback 构造出了 Service 对象的共享指针，然后将指针加入到 service_list_, 向 GlobalData 中注册 service_name ，最后加入到 TopologyManager 的 service_manager 中，返回 Service 的共享指针
对于 client, 其过程和 service 类似，只是其构造的是 client 对象
