---
title: Apollo-Cyber-学习笔记（三）
date: 2022-09-08 11:41:23
tags:
---
在 reader 中，我们看到其通过 receiver 接收消息，在 writer 中我们看到其 transmitter 来发消息，他们都位于 transport 目录下，这篇博客来学习 transport

<!-- more -->
目录结构如下：
```shell
├── BUILD
├── common
├── dispatcher
├── integration_test
├── message
├── qos
├── receiver
├── rtps
├── shm
├── transmitter
├── transport.cc
├── transport.h
└── transport_test.cc
```
从目录结构可以看出这里是一个比较大的目录

# transport类
## transport.h
我们先看 Transport 类的定义：
```c++
class Transport {
 public:
  virtual ~Transport();

  void Shutdown();

  template <typename M>
  auto CreateTransmitter(const RoleAttributes& attr,
                         const OptionalMode& mode = OptionalMode::HYBRID) ->
      typename std::shared_ptr<Transmitter<M>>;

  template <typename M>
  auto CreateReceiver(const RoleAttributes& attr,
                      const typename Receiver<M>::MessageListener& msg_listener,
                      const OptionalMode& mode = OptionalMode::HYBRID) ->
      typename std::shared_ptr<Receiver<M>>;

  ParticipantPtr participant() const { return participant_; }

 private:
  void CreateParticipant();

  std::atomic<bool> is_shutdown_ = {false};
  ParticipantPtr participant_ = nullptr;
  NotifierPtr notifier_ = nullptr;
  IntraDispatcherPtr intra_dispatcher_ = nullptr;
  ShmDispatcherPtr shm_dispatcher_ = nullptr;
  RtpsDispatcherPtr rtps_dispatcher_ = nullptr;

  DECLARE_SINGLETON(Transport)
};
```
Transport 类是一个单例，通过 cyber/common/macros.h 的 DECLARE_SINGLETON 宏实现，其拥有 intra, shm, rtps 三种 dispatcher 的指针, 还拥有 Participant, Notifier 的指针，私有接口为创建 Participant， 公有接口为 Shutdown, CreateTransmitter, CreateReceiver

```c++
template <typename M>
auto Transport::CreateTransmitter(const RoleAttributes& attr,
                                  const OptionalMode& mode) ->
    typename std::shared_ptr<Transmitter<M>> {
  if (is_shutdown_.load()) {
    return nullptr;
  }

  std::shared_ptr<Transmitter<M>> transmitter = nullptr;
  RoleAttributes modified_attr = attr;
  if (!modified_attr.has_qos_profile()) {
    modified_attr.mutable_qos_profile()->CopyFrom(
        QosProfileConf::QOS_PROFILE_DEFAULT);
  }

  switch (mode) {
    case OptionalMode::INTRA:
      transmitter = std::make_shared<IntraTransmitter<M>>(modified_attr);
      break;

    case OptionalMode::SHM:
      transmitter = std::make_shared<ShmTransmitter<M>>(modified_attr);
      break;

    case OptionalMode::RTPS:
      transmitter =
          std::make_shared<RtpsTransmitter<M>>(modified_attr, participant());
      break;

    default:
      transmitter =
          std::make_shared<HybridTransmitter<M>>(modified_attr, participant());
      break;
  }

  RETURN_VAL_IF_NULL(transmitter, nullptr);
  if (mode != OptionalMode::HYBRID) {
    transmitter->Enable();
  }
  return transmitter;
}
```
在 CreateTransmitter 的具体实现中, 可以根据传参 mode 来决定用哪种 Transmitter 类, 支持 Intra, Shm, Rtps, 默认是 Hybrid, 涉及到 Rtps 时还需要提供 participant, 当不为 Hybrid 时还会调用对应 transmitter 的 Enable 函数

```c++
template <typename M>
auto Transport::CreateReceiver(
    const RoleAttributes& attr,
    const typename Receiver<M>::MessageListener& msg_listener,
    const OptionalMode& mode) -> typename std::shared_ptr<Receiver<M>> {
  if (is_shutdown_.load()) {
    return nullptr;
  }

  std::shared_ptr<Receiver<M>> receiver = nullptr;
  RoleAttributes modified_attr = attr;
  if (!modified_attr.has_qos_profile()) {
    modified_attr.mutable_qos_profile()->CopyFrom(
        QosProfileConf::QOS_PROFILE_DEFAULT);
  }

  switch (mode) {
    case OptionalMode::INTRA:
      receiver =
          std::make_shared<IntraReceiver<M>>(modified_attr, msg_listener);
      break;

    case OptionalMode::SHM:
      receiver = std::make_shared<ShmReceiver<M>>(modified_attr, msg_listener);
      break;

    case OptionalMode::RTPS:
      receiver = std::make_shared<RtpsReceiver<M>>(modified_attr, msg_listener);
      break;

    default:
      receiver = std::make_shared<HybridReceiver<M>>(
          modified_attr, msg_listener, participant());
      break;
  }

  RETURN_VAL_IF_NULL(receiver, nullptr);
  if (mode != OptionalMode::HYBRID) {
    receiver->Enable();
  }
  return receiver;
}
```
Receiver 的实现和 Transmitter 类似，只是多了一个 msg_listener 参数

# common
## identity.h
```c++
constexpr uint8_t ID_SIZE = 8;
class Identity {
 public:
  explicit Identity(bool need_generate = true);
  Identity(const Identity& another);
  virtual ~Identity();

  Identity& operator=(const Identity& another);
  bool operator==(const Identity& another) const;
  bool operator!=(const Identity& another) const;

  std::string ToString() const;
  size_t Length() const;
  uint64_t HashValue() const;

  const char* data() const { return data_; }
  void set_data(const char* data) {
    if (data == nullptr) {
      return;
    }
    std::memcpy(data_, data, sizeof(data_));
    Update();
  }

 private:
  void Update();

  char data_[ID_SIZE];
  uint64_t hash_value_;
};
```
Identity 类主要是用来标识，除了有一个8字节的字符数组外还有一个 hash_value_ 
再看看具体实现：
```c++
Identity::Identity(bool need_generate) : hash_value_(0) {
  std::memset(data_, 0, ID_SIZE);
  if (need_generate) {
    uuid_t uuid;
    uuid_generate(uuid);
    std::memcpy(data_, uuid, ID_SIZE);
    Update();
  }
}

Identity::Identity(const Identity& rhs) {
  std::memcpy(data_, rhs.data_, ID_SIZE);
  hash_value_ = rhs.hash_value_;
}
```
构造函数中会去生成一个 uuid, 并设置到 data_ 中,然后调用 Update 函数去生成哈希值（若构造函数传参 false, 那么不会去设置 hash_value_）
```c++
std::string Identity::ToString() const { return std::to_string(hash_value_); }
```
Tostring 函数将 hash_value_ 构造成string
```c++
void Identity::Update() {
  hash_value_ = common::Hash(std::string(data_, ID_SIZE));
}
```
Update 函数负责调用 common 中的 Hash 函数将 data_ 转化为哈希值

## endpoint.h
```c++
class Endpoint;
using EndpointPtr = std::shared_ptr<Endpoint>;
using proto::RoleAttributes;

class Endpoint {
 public:
  explicit Endpoint(const RoleAttributes& attr);
  virtual ~Endpoint();

  const Identity& id() const { return id_; }
  const RoleAttributes& attributes() const { return attr_; }

 protected:
  bool enabled_;
  Identity id_;
  RoleAttributes attr_;
};
```
Endpoint 拥有一个 Identity 对象 id, 一个 enable_ 布尔变量, 一个 RoleAttributes 的变量 attr_
再看构造函数
```c++
Endpoint::Endpoint(const RoleAttributes& attr)
    : enabled_(false), id_(), attr_(attr) {
  if (!attr_.has_host_name()) {
    attr_.set_host_name(common::GlobalData::Instance()->HostName());
  }

  if (!attr_.has_process_id()) {
    attr_.set_process_id(common::GlobalData::Instance()->ProcessId());
  }

  if (!attr_.has_id()) {
    attr_.set_id(id_.HashValue());
  }
}
```
从构造函数可以看到, 如果构建 Endpoint 对象时的 attr 没有设置 host_name, process_id, id 会给它设置上正确的值，同时默认的 enabled_ 为 false

# transmitter
## transmitter.h
```c++
template <typename M>
class Transmitter : public Endpoint {
 public:
  using MessagePtr = std::shared_ptr<M>;

  explicit Transmitter(const RoleAttributes& attr);
  virtual ~Transmitter();

  virtual void Enable() = 0;
  virtual void Disable() = 0;

  virtual void Enable(const RoleAttributes& opposite_attr);
  virtual void Disable(const RoleAttributes& opposite_attr);

  virtual bool Transmit(const MessagePtr& msg);
  virtual bool Transmit(const MessagePtr& msg, const MessageInfo& msg_info) = 0;

  uint64_t NextSeqNum() { return ++seq_num_; }

  uint64_t seq_num() const { return seq_num_; }

 protected:
  uint64_t seq_num_;
  MessageInfo msg_info_;
};
```
这里我们可以看到 Transmitter 是一个虚基类，主要是定义了 Transmitter 对外该提供的接口, 私有变量中有一个消息序列 seq_num_ 和一个 MessageInfo 的变量
```c++
template <typename M>
Transmitter<M>::Transmitter(const RoleAttributes& attr)
    : Endpoint(attr), seq_num_(0) {
  msg_info_.set_sender_id(this->id_);
  msg_info_.set_seq_num(this->seq_num_);
}

template <typename M>
Transmitter<M>::~Transmitter() {}

template <typename M>
bool Transmitter<M>::Transmit(const MessagePtr& msg) {
  msg_info_.set_seq_num(NextSeqNum());
  return Transmit(msg, msg_info_);
}

template <typename M>
void Transmitter<M>::Enable(const RoleAttributes& opposite_attr) {
  (void)opposite_attr; //这里是为了防止编译器报错加了 (void)，没有实际意义
  Enable();
}

template <typename M>
void Transmitter<M>::Disable(const RoleAttributes& opposite_attr) {
  (void)opposite_attr;
  Disable();
}
```
在 Transmitter 的构造函数中会设置 msg_info_ 中的 sender_id 和 seq_num, Transmit 会设置 seq_num 然后调用派生类的 Transmit 接口, Enable, Disable均是转掉派生类对应的接口

## shm_transmitter.h
```c++
template <typename M>
class ShmTransmitter : public Transmitter<M> {
 public:
  using MessagePtr = std::shared_ptr<M>;

  explicit ShmTransmitter(const RoleAttributes& attr);
  virtual ~ShmTransmitter();

  void Enable() override;
  void Disable() override;

  bool Transmit(const MessagePtr& msg, const MessageInfo& msg_info) override;

 private:
  bool Transmit(const M& msg, const MessageInfo& msg_info);

  SegmentPtr segment_;
  uint64_t channel_id_;
  uint64_t host_id_;
  NotifierPtr notifier_;
};
```
ShmTransmitter 中的接口基本就是 override 基类的接口，同时拥有 segment_, channel_id_, host_id_, notifier_ 四个私有变量
```c++
template <typename M>
ShmTransmitter<M>::ShmTransmitter(const RoleAttributes& attr)
    : Transmitter<M>(attr),
      segment_(nullptr),
      channel_id_(attr.channel_id()),
      notifier_(nullptr) {
  host_id_ = common::Hash(attr.host_ip());
}

template <typename M>
ShmTransmitter<M>::~ShmTransmitter() {
  Disable();
}
```
构造函数中会构造基类，设置 channel_id_ 和 host_id_ , segment_ 和 notifier_ 在构造时设置为空指针， 析构函数调用了 Disable 函数
```c++
template <typename M>
void ShmTransmitter<M>::Enable() {
  if (this->enabled_) {
    return;
  }

  segment_ = SegmentFactory::CreateSegment(channel_id_);
  notifier_ = NotifierFactory::CreateNotifier();
  this->enabled_ = true;
}

template <typename M>
void ShmTransmitter<M>::Disable() {
  if (this->enabled_) {
    segment_ = nullptr;
    notifier_ = nullptr;
    this->enabled_ = false;
  }
}
```
Enable 函数中会去 SegmentFactory 和 NotifierFactory 中去创建新的 Segment 和 Notifier, 并将标志位设为 true
Disable 会将 segment_, 和 notifier_ 设为空指针，并将标志位设为 false
```c++
template <typename M>
bool ShmTransmitter<M>::Transmit(const MessagePtr& msg,
                                 const MessageInfo& msg_info) {
  return Transmit(*msg, msg_info);
}

template <typename M>
bool ShmTransmitter<M>::Transmit(const M& msg, const MessageInfo& msg_info) {
  if (!this->enabled_) {
    return false;
  }

  WritableBlock wb;
  std::size_t msg_size = message::ByteSize(msg);
  if (!segment_->AcquireBlockToWrite(msg_size, &wb)) {
    return false;
  }

  if (!message::SerializeToArray(msg, wb.buf, static_cast<int>(msg_size))) {
    segment_->ReleaseWrittenBlock(wb);
    return false;
  }
  wb.block->set_msg_size(msg_size);

  char* msg_info_addr = reinterpret_cast<char*>(wb.buf) + msg_size;
  if (!msg_info.SerializeTo(msg_info_addr, MessageInfo::kSize)) {
    segment_->ReleaseWrittenBlock(wb);
    return false;
  }
  wb.block->set_msg_info_size(MessageInfo::kSize);
  segment_->ReleaseWrittenBlock(wb);

  ReadableInfo readable_info(host_id_, wb.index, channel_id_);

  return notifier_->Notify(readable_info);
}
```
最后看 ShmTransmitter 类中最核心的 Transmit函数, 共享指针做参数的 Transmit 函数是转调引用类型的 Transmit 函数， Transmit做的事情为：
- 声明一个 WritableBlock 对象 wb
- 计算需要传输的 msg 的大小，从 segment_ 中去获得对应大小的 block
- 通过 message 中的 SerializeToArray 将 msg 序列化，并设置 wb 中的 msg_size
- 将 msg_info 序列化在 msg 之后
- 在 segment_ 中调用 ReleaseWrittenBlock，然后构造 ReadableInfo 对象
- 调用 notifier_ 的 Notify 去通知

## rtps_transmitter.h
> 注：RTPS (Real-Time Publish-Subscribe, 实时发布订阅)
> 在Cyber中使用了Fast-RTPS，Fast-RTPS是eprosima对于RTPS的C++实现，是一个免费开源软件，遵循Apache License 2.0，开源地址如下
> [Fast-RTPS开源地址](https://github.com/eProsima/Fast-DDS/tree/master/include/fastrtps)


```c++
template <typename M>
class RtpsTransmitter : public Transmitter<M> {
 public:
  using MessagePtr = std::shared_ptr<M>;

  RtpsTransmitter(const RoleAttributes& attr,
                  const ParticipantPtr& participant);
  virtual ~RtpsTransmitter();

  void Enable() override;
  void Disable() override;
  bool Transmit(const MessagePtr& msg, const MessageInfo& msg_info) override;

 private:
  bool Transmit(const M& msg, const MessageInfo& msg_info);

  ParticipantPtr participant_;
  eprosima::fastrtps::Publisher* publisher_;
};
```
RtpsTransmitter 也是继承自 Transmitter，这里与 shm_transmitter 的不同是没有了 segment_ 和 notifier_，多了 participant_ 和 publisher_
```c++
template <typename M>
void RtpsTransmitter<M>::Enable() {
  if (this->enabled_) {
    return;
  }

  RETURN_IF_NULL(participant_);

  eprosima::fastrtps::PublisherAttributes pub_attr;
  RETURN_IF(!AttributesFiller::FillInPubAttr(
      this->attr_.channel_name(), this->attr_.qos_profile(), &pub_attr));
  publisher_ = eprosima::fastrtps::Domain::createPublisher(
      participant_->fastrtps_participant(), pub_attr);
  RETURN_IF_NULL(publisher_);
  this->enabled_ = true;
}
```
在 Enable 函数中首先会构造 PublisherAttributes 对象 pub_attr，并填充对应的字段，然后根据 participant_ 和 pub_attr 去创建新的 Publisher

```c++
template <typename M>
bool RtpsTransmitter<M>::Transmit(const M& msg, const MessageInfo& msg_info) {
  if (!this->enabled_) {
    return false;
  }

  UnderlayMessage m;
  RETURN_VAL_IF(!message::SerializeToString(msg, &m.data()), false);

  eprosima::fastrtps::rtps::WriteParams wparams;

  char* ptr =
      reinterpret_cast<char*>(&wparams.related_sample_identity().writer_guid());

  memcpy(ptr, msg_info.sender_id().data(), ID_SIZE);
  memcpy(ptr + ID_SIZE, msg_info.spare_id().data(), ID_SIZE);

  wparams.related_sample_identity().sequence_number().high =
      (int32_t)((msg_info.seq_num() & 0xFFFFFFFF00000000) >> 32);
  wparams.related_sample_identity().sequence_number().low =
      (int32_t)(msg_info.seq_num() & 0xFFFFFFFF);

  if (participant_->is_shutdown()) {
    return false;
  }
  return publisher_->write(reinterpret_cast<void*>(&m), wparams);
}
```
Transmit 函数主要做了以下事情：
- 构造 UnderlayMessage 对象 m, 将 msg 消息序列化到 m 中
- 构造 eprosima::fastrtps::rtps::WriteParams 对象 wparams
- 设置 msg_info 中的一些信息到 wparams 中
- 通过 publisher_ 来 write 最终的消息

## intra_transmitter.h
```C++
template <typename M>
class IntraTransmitter : public Transmitter<M> {
 public:
  using MessagePtr = std::shared_ptr<M>;

  explicit IntraTransmitter(const RoleAttributes& attr);
  virtual ~IntraTransmitter();

  void Enable() override;
  void Disable() override;

  bool Transmit(const MessagePtr& msg, const MessageInfo& msg_info) override;

 private:
  uint64_t channel_id_;
  IntraDispatcherPtr dispatcher_;
};
```
intra_transmitter 主要是有个 IntraDispatcherPtr

```c++
template <typename M>
void IntraTransmitter<M>::Enable() {
  if (!this->enabled_) {
    dispatcher_ = IntraDispatcher::Instance();
    this->enabled_ = true;
  }
}

template <typename M>
void IntraTransmitter<M>::Disable() {
  if (this->enabled_) {
    dispatcher_ = nullptr;
    this->enabled_ = false;
  }
}

template <typename M>
bool IntraTransmitter<M>::Transmit(const MessagePtr& msg,
                                   const MessageInfo& msg_info) {
  if (!this->enabled_) {
    ADEBUG << "not enable.";
    return false;
  }

  dispatcher_->OnMessage(channel_id_, msg, msg_info);
  return true;
}
```
可以看到 IntraTransmitter 的核心就是 dispatcher_
Transmit 函数直接调用 dispatcher_->OnMessage 

> **总结一下3种 transmitter 的区别**
> intra transmitter 是内部调用的方式来通信
> shm transmitter 是通过将 msg 写到共享内存，读取方去共享内存读取
> rtps transmitter 是将 msg 序列化，然后通过 rtps(本质是UDP) 来进行通信

## hybrid_transmitter.h
```c++
using apollo::cyber::proto::OptionalMode;
using apollo::cyber::proto::QosDurabilityPolicy;
using apollo::cyber::proto::RoleAttributes;

template <typename M>
class HybridTransmitter : public Transmitter<M> {
 public:
  using MessagePtr = std::shared_ptr<M>;
  using HistoryPtr = std::shared_ptr<History<M>>;
  using TransmitterPtr = std::shared_ptr<Transmitter<M>>;
  using TransmitterMap =
      std::unordered_map<OptionalMode, TransmitterPtr, std::hash<int>>;
  using ReceiverMap =
      std::unordered_map<OptionalMode, std::set<uint64_t>, std::hash<int>>;
  using CommunicationModePtr = std::shared_ptr<proto::CommunicationMode>;
  using MappingTable =
      std::unordered_map<Relation, OptionalMode, std::hash<int>>;

  HybridTransmitter(const RoleAttributes& attr,
                    const ParticipantPtr& participant);
  virtual ~HybridTransmitter();

  void Enable() override;
  void Disable() override;
  void Enable(const RoleAttributes& opposite_attr) override;
  void Disable(const RoleAttributes& opposite_attr) override;

  bool Transmit(const MessagePtr& msg, const MessageInfo& msg_info) override;

 private:
  void InitMode();
  void ObtainConfig();
  void InitHistory();
  void InitTransmitters();
  void ClearTransmitters();
  void InitReceivers();
  void ClearReceivers();
  void TransmitHistoryMsg(const RoleAttributes& opposite_attr);
  void ThreadFunc(const RoleAttributes& opposite_attr,
                  const std::vector<typename History<M>::CachedMessage>& msgs);
  Relation GetRelation(const RoleAttributes& opposite_attr);

  HistoryPtr history_;
  TransmitterMap transmitters_;
  ReceiverMap receivers_;
  std::mutex mutex_;

  CommunicationModePtr mode_;
  MappingTable mapping_table_;

  ParticipantPtr participant_;
};
```
hybrid transmitter 是一种混合的 transmitter, 从定义上共有接口都是 override 基类的接口, 多了很多私有变量和接口
```c++
template <typename M>
HybridTransmitter<M>::HybridTransmitter(const RoleAttributes& attr,
                                        const ParticipantPtr& participant)
    : Transmitter<M>(attr),
      history_(nullptr),
      mode_(nullptr),
      participant_(participant) {
  InitMode();
  ObtainConfig();
  InitHistory();
  InitTransmitters();
  InitReceivers();
}

template <typename M>
HybridTransmitter<M>::~HybridTransmitter() {
  ClearReceivers();
  ClearTransmitters();
}
```
在构造和析构函数中调用了很多私有接口，我们看下这些接口做了什么事情：
```c++
//注 Relation 在 common 的 types.h中定义
enum Relation : std::uint8_t {
  NO_RELATION = 0,
  DIFF_HOST,  // different host
  DIFF_PROC,  // same host, but different process
  SAME_PROC,  // same process
};

template <typename M>
void HybridTransmitter<M>::InitMode() {
  mode_ = std::make_shared<proto::CommunicationMode>();
  mapping_table_[SAME_PROC] = mode_->same_proc();
  mapping_table_[DIFF_PROC] = mode_->diff_proc();
  mapping_table_[DIFF_HOST] = mode_->diff_host();
}
```
InitMode 做的事情主要是建立 Relation 与 proto中OptionalMode 的映射关系 mapping_table_, 这里有三种 Mode, 一是相同进程，二是同一主机的不同进程，三是不同主机的进程 
```c++
template <typename M>
void HybridTransmitter<M>::ObtainConfig() {
  auto& global_conf = common::GlobalData::Instance()->Config();
  if (!global_conf.has_transport_conf()) {
    return;
  }
  if (!global_conf.transport_conf().has_communication_mode()) {
    return;
  }
  mode_->CopyFrom(global_conf.transport_conf().communication_mode());

  mapping_table_[SAME_PROC] = mode_->same_proc();
  mapping_table_[DIFF_PROC] = mode_->diff_proc();
  mapping_table_[DIFF_HOST] = mode_->diff_host();
}
```
ObtainConfig 做的事情主要是用 GlobalData 中获得 communication_mode
```c++
template <typename M>
void HybridTransmitter<M>::InitHistory() {
  HistoryAttributes history_attr(this->attr_.qos_profile().history(),
                                 this->attr_.qos_profile().depth());
  history_ = std::make_shared<History<M>>(history_attr);
  if (this->attr_.qos_profile().durability() ==
      QosDurabilityPolicy::DURABILITY_TRANSIENT_LOCAL) {
    history_->Enable();
  }
}
```
InitHistory 主要是用来初始化 history_
```c++
template <typename M>
void HybridTransmitter<M>::InitTransmitters() {
  std::set<OptionalMode> modes;
  modes.insert(mode_->same_proc());
  modes.insert(mode_->diff_proc());
  modes.insert(mode_->diff_host());
  for (auto& mode : modes) {
    switch (mode) {
      case OptionalMode::INTRA:
        transmitters_[mode] =
            std::make_shared<IntraTransmitter<M>>(this->attr_);
        break;
      case OptionalMode::SHM:
        transmitters_[mode] = std::make_shared<ShmTransmitter<M>>(this->attr_);
        break;
      default:
        transmitters_[mode] =
            std::make_shared<RtpsTransmitter<M>>(this->attr_, participant_);
        break;
    }
  }
}
```
InitTransmitters 是为 mode 创建对应的 Transmitter, 并存到 map 中
```c++
template <typename M>
void HybridTransmitter<M>::InitReceivers() {
  std::set<uint64_t> empty;
  for (auto& item : transmitters_) {
    receivers_[item.first] = empty;
  }
}
```
InitReceivers 作用是初始化 receivers_, receivers_ 是一个 OptionalMode 与 set<64_t> 的一个 unordered_map
```c++
template <typename M>
void HybridTransmitter<M>::ClearTransmitters() {
  for (auto& item : transmitters_) {
    item.second->Disable();
  }
  transmitters_.clear();
}
template <typename M>
void HybridTransmitter<M>::ClearReceivers() {
  receivers_.clear();
}
```
ClearTransmitters 首先调用每个 transmitter 的 Disable函数, 再清空 map , ClearReceivers 直接清空 map
```c++
template <typename M>
void HybridTransmitter<M>::TransmitHistoryMsg(
    const RoleAttributes& opposite_attr) {
  // check qos
  if (this->attr_.qos_profile().durability() !=
      QosDurabilityPolicy::DURABILITY_TRANSIENT_LOCAL) {
    return;
  }

  // get unsent messages
  std::vector<typename History<M>::CachedMessage> unsent_msgs;
  history_->GetCachedMessage(&unsent_msgs);
  if (unsent_msgs.empty()) {
    return;
  }

  auto attr = opposite_attr;
  cyber::Async(&HybridTransmitter<M>::ThreadFunc, this, attr, unsent_msgs);
}
```
TransmitHistoryMsg 函数先去 history_ 获得没有发送的消息，再用 ThreadFunc 去发送 msg
```c++
template <typename M>
void HybridTransmitter<M>::ThreadFunc(
    const RoleAttributes& opposite_attr,
    const std::vector<typename History<M>::CachedMessage>& msgs) {
  // create transmitter to transmit msgs
  RoleAttributes new_attr;
  new_attr.CopyFrom(this->attr_);
  std::string new_channel_name =
      std::to_string(this->attr_.id()) + std::to_string(opposite_attr.id());
  uint64_t channel_id = common::GlobalData::RegisterChannel(new_channel_name);
  new_attr.set_channel_name(new_channel_name);
  new_attr.set_channel_id(channel_id);
  auto new_transmitter =
      std::make_shared<RtpsTransmitter<M>>(new_attr, participant_);
  new_transmitter->Enable();

  for (auto& item : msgs) {
    new_transmitter->Transmit(item.msg, item.msg_info);
    cyber::USleep(1000);
  }
  new_transmitter->Disable();
}
```
ThreadFunc 首先根据 RoleAttributes 去注册一个 channel, 然后去创建一个新的 RtpsTransmitter 去发送所有的消息，最后 Disable 掉 transmitter
```c++
template <typename M>
Relation HybridTransmitter<M>::GetRelation(
    const RoleAttributes& opposite_attr) {
  if (opposite_attr.channel_name() != this->attr_.channel_name()) {
    return NO_RELATION;
  }
  if (opposite_attr.host_ip() != this->attr_.host_ip()) {
    return DIFF_HOST;
  }
  if (opposite_attr.process_id() != this->attr_.process_id()) {
    return DIFF_PROC;
  }
  return SAME_PROC;
}
```
GetRelation 根据 RoleAttributes 去判断 Relation

# receiver
tranmmiter 是消息的发送方, 而 receiver 是消息的接收方，我们来看 receiver 做了什么事：
## receiver.h
```c++
template <typename M>
class Receiver : public Endpoint {
 public:
  using MessagePtr = std::shared_ptr<M>;
  using MessageListener = std::function<void(
      const MessagePtr&, const MessageInfo&, const RoleAttributes&)>;

  Receiver(const RoleAttributes& attr, const MessageListener& msg_listener);
  virtual ~Receiver();

  virtual void Enable() = 0;
  virtual void Disable() = 0;
  virtual void Enable(const RoleAttributes& opposite_attr) = 0;
  virtual void Disable(const RoleAttributes& opposite_attr) = 0;

 protected:
  void OnNewMessage(const MessagePtr& msg, const MessageInfo& msg_info);

  MessageListener msg_listener_;
};
```
receiver 中封装了回调函数 MessageListener, 一个 receiver 基类只有一个回调函数 msg_listener_，其定义了 Enable 和 Disable 接口
```c++
template <typename M>
Receiver<M>::Receiver(const RoleAttributes& attr,
                      const MessageListener& msg_listener)
    : Endpoint(attr), msg_listener_(msg_listener) {}

template <typename M>
Receiver<M>::~Receiver() {}

template <typename M>
void Receiver<M>::OnNewMessage(const MessagePtr& msg,
                               const MessageInfo& msg_info) {
  if (msg_listener_ != nullptr) {
    msg_listener_(msg, msg_info, attr_);
  }
}
```
OnNewMessage 就是去调用 msg_listener_ 回调函数

## intra_receiver.h
```c++
template <typename M>
class IntraReceiver : public Receiver<M> {
 public:
  IntraReceiver(const RoleAttributes& attr,
                const typename Receiver<M>::MessageListener& msg_listener);
  virtual ~IntraReceiver();

  void Enable() override;
  void Disable() override;

  void Enable(const RoleAttributes& opposite_attr) override;
  void Disable(const RoleAttributes& opposite_attr) override;

 private:
  IntraDispatcherPtr dispatcher_;
};
```
intra_receiver 也包含一个 IntraDispatcherPtr 对象 dispatcher_, 接口均是实现基类定义的接口
```c++
template <typename M>
IntraReceiver<M>::IntraReceiver(
    const RoleAttributes& attr,
    const typename Receiver<M>::MessageListener& msg_listener)
    : Receiver<M>(attr, msg_listener) {
  dispatcher_ = IntraDispatcher::Instance();
}

template <typename M>
IntraReceiver<M>::~IntraReceiver() {
  Disable();
}
```
在构造函数中给 dispatcher_ 初始化，析构函数调用 Disable
```c++
template <typename M>
void IntraReceiver<M>::Enable() {
  if (this->enabled_) {
    return;
  }

  dispatcher_->AddListener<M>(
      this->attr_, std::bind(&IntraReceiver<M>::OnNewMessage, this,
                             std::placeholders::_1, std::placeholders::_2));
  this->enabled_ = true;
}
template <typename M>
void IntraReceiver<M>::Enable(const RoleAttributes& opposite_attr) {
  dispatcher_->AddListener<M>(
      this->attr_, opposite_attr,
      std::bind(&IntraReceiver<M>::OnNewMessage, this, std::placeholders::_1,
                std::placeholders::_2));
}
```
两个 Enable 函数都是在 dispatcher_ 中 AddListenser
```c++
template <typename M>
void IntraReceiver<M>::Disable() {
  if (!this->enabled_) {
    return;
  }

  dispatcher_->RemoveListener<M>(this->attr_);
  this->enabled_ = false;
}

template <typename M>
void IntraReceiver<M>::Disable(const RoleAttributes& opposite_attr) {
  dispatcher_->RemoveListener<M>(this->attr_, opposite_attr);
}
```
Disable 函数在 dispatcher_ 中 RemoveListener

## shm_receiver.h
```c++
template <typename M>
class ShmReceiver : public Receiver<M> {
 public:
  ShmReceiver(const RoleAttributes& attr,
              const typename Receiver<M>::MessageListener& msg_listener);
  virtual ~ShmReceiver();

  void Enable() override;
  void Disable() override;

  void Enable(const RoleAttributes& opposite_attr) override;
  void Disable(const RoleAttributes& opposite_attr) override;

 private:
  ShmDispatcherPtr dispatcher_;
};
```
shm_receiver 中包含一个 ShmDispatcherPtr 对象 dispatcher_
```c++
template <typename M>
ShmReceiver<M>::ShmReceiver(
    const RoleAttributes& attr,
    const typename Receiver<M>::MessageListener& msg_listener)
    : Receiver<M>(attr, msg_listener) {
  dispatcher_ = ShmDispatcher::Instance();
}

template <typename M>
ShmReceiver<M>::~ShmReceiver() {
  Disable();
}

template <typename M>
void ShmReceiver<M>::Enable() {
  if (this->enabled_) {
    return;
  }

  dispatcher_->AddListener<M>(
      this->attr_, std::bind(&ShmReceiver<M>::OnNewMessage, this,
                             std::placeholders::_1, std::placeholders::_2));
  this->enabled_ = true;
}

template <typename M>
void ShmReceiver<M>::Disable() {
  if (!this->enabled_) {
    return;
  }

  dispatcher_->RemoveListener<M>(this->attr_);
  this->enabled_ = false;
}

template <typename M>
void ShmReceiver<M>::Enable(const RoleAttributes& opposite_attr) {
  dispatcher_->AddListener<M>(
      this->attr_, opposite_attr,
      std::bind(&ShmReceiver<M>::OnNewMessage, this, std::placeholders::_1,
                std::placeholders::_2));
}

template <typename M>
void ShmReceiver<M>::Disable(const RoleAttributes& opposite_attr) {
  dispatcher_->RemoveListener<M>(this->attr_, opposite_attr);
}
```
实现上和 intra_receiver 基本类似，不再赘述
## rtps_receiver.h
```c++
template <typename M>
class RtpsReceiver : public Receiver<M> {
 public:
  RtpsReceiver(const RoleAttributes& attr,
               const typename Receiver<M>::MessageListener& msg_listener);
  virtual ~RtpsReceiver();

  void Enable() override;
  void Disable() override;

  void Enable(const RoleAttributes& opposite_attr) override;
  void Disable(const RoleAttributes& opposite_attr) override;

 private:
  RtpsDispatcherPtr dispatcher_;
};
```
不同的 receiver 都拥有自己的 DispatcherPtr，其余实现基本一样
# hybrid_receiver.h
```c++
template <typename M>
class HybridReceiver : public Receiver<M> {
 public:
  using HistoryPtr = std::shared_ptr<History<M>>;
  using ReceiverPtr = std::shared_ptr<Receiver<M>>;
  using ReceiverContainer =
      std::unordered_map<OptionalMode, ReceiverPtr, std::hash<int>>;
  using TransmitterContainer =
      std::unordered_map<OptionalMode,
                         std::unordered_map<uint64_t, RoleAttributes>,
                         std::hash<int>>;
  using CommunicationModePtr = std::shared_ptr<proto::CommunicationMode>;
  using MappingTable =
      std::unordered_map<Relation, OptionalMode, std::hash<int>>;

  HybridReceiver(const RoleAttributes& attr,
                 const typename Receiver<M>::MessageListener& msg_listener,
                 const ParticipantPtr& participant);
  virtual ~HybridReceiver();

  void Enable() override;
  void Disable() override;

  void Enable(const RoleAttributes& opposite_attr) override;
  void Disable(const RoleAttributes& opposite_attr) override;

 private:
  void InitMode();
  void ObtainConfig();
  void InitHistory();
  void InitReceivers();
  void ClearReceivers();
  void InitTransmitters();
  void ClearTransmitters();
  void ReceiveHistoryMsg(const RoleAttributes& opposite_attr);
  void ThreadFunc(const RoleAttributes& opposite_attr);
  Relation GetRelation(const RoleAttributes& opposite_attr);

  HistoryPtr history_;
  ReceiverContainer receivers_;
  TransmitterContainer transmitters_;
  std::mutex mutex_;

  CommunicationModePtr mode_;
  MappingTable mapping_table_;

  ParticipantPtr participant_;
};
```
从定义上看 HybridReceiver 与 HybridTransmitter 的定义基本一样, 其中 ObtainConfig, InitMode, InitHistory, GetRelation与 HybridTransmitter中的实现一样
```c++
template <typename M>
void HybridReceiver<M>::Enable() {
  std::lock_guard<std::mutex> lock(mutex_);
  for (auto& item : receivers_) {
    item.second->Enable();
  }
}

template <typename M>
void HybridReceiver<M>::Disable() {
  std::lock_guard<std::mutex> lock(mutex_);
  for (auto& item : receivers_) {
    item.second->Disable();
  }
}

template <typename M>
void HybridReceiver<M>::Enable(const RoleAttributes& opposite_attr) {
  auto relation = GetRelation(opposite_attr);
  RETURN_IF(relation == NO_RELATION);

  uint64_t id = opposite_attr.id();
  std::lock_guard<std::mutex> lock(mutex_);
  if (transmitters_[mapping_table_[relation]].count(id) == 0) {
    transmitters_[mapping_table_[relation]].insert(
        std::make_pair(id, opposite_attr));
    receivers_[mapping_table_[relation]]->Enable(opposite_attr);
    ReceiveHistoryMsg(opposite_attr);
  }
}

template <typename M>
void HybridReceiver<M>::Disable(const RoleAttributes& opposite_attr) {
  auto relation = GetRelation(opposite_attr);
  RETURN_IF(relation == NO_RELATION);

  uint64_t id = opposite_attr.id();
  std::lock_guard<std::mutex> lock(mutex_);
  if (transmitters_[mapping_table_[relation]].count(id) > 0) {
    transmitters_[mapping_table_[relation]].erase(id);
    receivers_[mapping_table_[relation]]->Disable(opposite_attr);
  }
}
```
带参的 Enable 函数会去看 transmitters_ 中是否存在 opposite_attr 所对应的 transmitter, 如果没有就创建对应的 transmitter 并调用对应 reveiver 的 Enable 函数，最后调用 ReceiveHistoryMsg 函数，无参的 Enable 函数会调用每个 reveiver 的 Enable 函数
同理，有参的 Disable 会先在 transmitters_ 删除 opposite_attr 所对应的 transmitter 然后调用对应的 reveiver 的 Disable 函数
```c++
template <typename M>
void HybridReceiver<M>::InitReceivers() {
  std::set<OptionalMode> modes;
  modes.insert(mode_->same_proc());
  modes.insert(mode_->diff_proc());
  modes.insert(mode_->diff_host());
  auto listener = std::bind(&HybridReceiver<M>::OnNewMessage, this,
                            std::placeholders::_1, std::placeholders::_2);
  for (auto& mode : modes) {
    switch (mode) {
      case OptionalMode::INTRA:
        receivers_[mode] =
            std::make_shared<IntraReceiver<M>>(this->attr_, listener);
        break;
      case OptionalMode::SHM:
        receivers_[mode] =
            std::make_shared<ShmReceiver<M>>(this->attr_, listener);
        break;
      default:
        receivers_[mode] =
            std::make_shared<RtpsReceiver<M>>(this->attr_, listener);
        break;
    }
  }
}

template <typename M>
void HybridReceiver<M>::ClearReceivers() {
  receivers_.clear();
}
```
InitReceivers 中会根据 mode_ 去创建每种模式的 receiver, 回调函数为 OnNewMessage（基类定义实现）, ClearReceivers 函数作用是清空 receivers_
```c++
template <typename M>
void HybridReceiver<M>::InitTransmitters() {
  std::unordered_map<uint64_t, RoleAttributes> empty;
  for (auto& item : receivers_) {
    transmitters_[item.first] = empty;
  }
}

template <typename M>
void HybridReceiver<M>::ClearTransmitters() {
  for (auto& item : transmitters_) {
    for (auto& upper_reach : item.second) {
      receivers_[item.first]->Disable(upper_reach.second);
    }
  }
  transmitters_.clear();
}
```
InitTransmitters 将每个 transmitters_ 设置为空 map, ClearTransmitters 会先将 transmitter 对应的 receiver 给 Disable, 然后清空所有 transmitters_
```c++
template <typename M>
void HybridReceiver<M>::ReceiveHistoryMsg(const RoleAttributes& opposite_attr) {
  // check qos
  if (opposite_attr.qos_profile().durability() !=
      QosDurabilityPolicy::DURABILITY_TRANSIENT_LOCAL) {
    return;
  }

  auto attr = opposite_attr;
  cyber::Async(&HybridReceiver<M>::ThreadFunc, this, attr);
}

template <typename M>
void HybridReceiver<M>::ThreadFunc(const RoleAttributes& opposite_attr) {
  std::string channel_name =
      std::to_string(opposite_attr.id()) + std::to_string(this->attr_.id());
  uint64_t channel_id = common::GlobalData::RegisterChannel(channel_name);

  RoleAttributes attr(this->attr_);
  attr.set_channel_name(channel_name);
  attr.set_channel_id(channel_id);
  attr.mutable_qos_profile()->CopyFrom(opposite_attr.qos_profile());

  volatile bool is_msg_arrived = false;
  auto listener = [&](const std::shared_ptr<M>& msg,
                      const MessageInfo& msg_info, const RoleAttributes& attr) {
    is_msg_arrived = true;
    this->OnNewMessage(msg, msg_info);
  };

  auto receiver = std::make_shared<RtpsReceiver<M>>(attr, listener);
  receiver->Enable();

  do {
    if (is_msg_arrived) {
      is_msg_arrived = false;
    }
    cyber::USleep(1000000);
  } while (is_msg_arrived);

  receiver->Disable();
}
```
ReceiveHistoryMsg 其实就是转调用 ThreadFunc, 这里是放到协程中去执行
ThreadFunc 中先注册一个 channel, 然后创建一个 RtpsReceiver 的 receiver 去接收消息

# dispatcher
receiver 的很多操作是调用 dispatcher 完成的，接下来看 dispathcer 做了什么事情
## dispatcher.h
```c++
using apollo::cyber::base::AtomicHashMap;
using apollo::cyber::base::AtomicRWLock;
using apollo::cyber::base::ReadLockGuard;
using apollo::cyber::base::WriteLockGuard;
using apollo::cyber::common::GlobalData;
using cyber::proto::RoleAttributes;
class Dispatcher;
using DispatcherPtr = std::shared_ptr<Dispatcher>;

template <typename MessageT>
using MessageListener =
    std::function<void(const std::shared_ptr<MessageT>&, const MessageInfo&)>;

class Dispatcher {
 public:
  Dispatcher();
  virtual ~Dispatcher();

  virtual void Shutdown();

  template <typename MessageT>
  void AddListener(const RoleAttributes& self_attr,
                   const MessageListener<MessageT>& listener);

  template <typename MessageT>
  void AddListener(const RoleAttributes& self_attr,
                   const RoleAttributes& opposite_attr,
                   const MessageListener<MessageT>& listener);

  template <typename MessageT>
  void RemoveListener(const RoleAttributes& self_attr);

  template <typename MessageT>
  void RemoveListener(const RoleAttributes& self_attr,
                      const RoleAttributes& opposite_attr);

  bool HasChannel(uint64_t channel_id);

 protected:
  std::atomic<bool> is_shutdown_;
  // key: channel_id of message
  AtomicHashMap<uint64_t, ListenerHandlerBasePtr> msg_listeners_;
  base::AtomicRWLock rw_lock_;
};
```
dispatcher 中有一个无锁map msg_listeners_, 一个读写锁，核心功能就是添加和删除 listener 回调函数
```c++
template <typename MessageT>
void Dispatcher::AddListener(const RoleAttributes& self_attr,
                             const MessageListener<MessageT>& listener) {
  if (is_shutdown_.load()) {
    return;
  }
  uint64_t channel_id = self_attr.channel_id();

  std::shared_ptr<ListenerHandler<MessageT>> handler;
  ListenerHandlerBasePtr* handler_base = nullptr;
  if (msg_listeners_.Get(channel_id, &handler_base)) {
    handler =
        std::dynamic_pointer_cast<ListenerHandler<MessageT>>(*handler_base);
    if (handler == nullptr) {
      return;
    }
  } else {
    handler.reset(new ListenerHandler<MessageT>());
    msg_listeners_.Set(channel_id, handler);
  }
  handler->Connect(self_attr.id(), listener);
}

template <typename MessageT>
void Dispatcher::AddListener(const RoleAttributes& self_attr,
                             const RoleAttributes& opposite_attr,
                             const MessageListener<MessageT>& listener) {
  if (is_shutdown_.load()) {
    return;
  }
  uint64_t channel_id = self_attr.channel_id();

  std::shared_ptr<ListenerHandler<MessageT>> handler;
  ListenerHandlerBasePtr* handler_base = nullptr;
  if (msg_listeners_.Get(channel_id, &handler_base)) {
    handler =
        std::dynamic_pointer_cast<ListenerHandler<MessageT>>(*handler_base);
    if (handler == nullptr) {
      return;
    }
  } else {
    handler.reset(new ListenerHandler<MessageT>());
    msg_listeners_.Set(channel_id, handler);
  }
  handler->Connect(self_attr.id(), opposite_attr.id(), listener);
}
```
添加 listener 的流程为：
- 获得 channel_id
- 根据 channel_id 去 msg_listeners_ 中查找对应的 ListenerHandlerBasePtr, 如果有，则将 handler 转化成 ListenerHandler 类指针 
- 如果在 msg_listeners 中没有找到，则重新 new 一个 ListenerHandler 指针，并将 <channel_id, handler> 键值对插入到 msg_listeners_ 中
- 调用 handler 的 Connect 函数

两个 AddListener 接口只是在最后一步的 Connect 函数调的不同的接口

```c++
template <typename MessageT>
void Dispatcher::RemoveListener(const RoleAttributes& self_attr) {
  if (is_shutdown_.load()) {
    return;
  }
  uint64_t channel_id = self_attr.channel_id();

  ListenerHandlerBasePtr* handler_base = nullptr;
  if (msg_listeners_.Get(channel_id, &handler_base)) {
    (*handler_base)->Disconnect(self_attr.id());
  }
}

template <typename MessageT>
void Dispatcher::RemoveListener(const RoleAttributes& self_attr,
                                const RoleAttributes& opposite_attr) {
  if (is_shutdown_.load()) {
    return;
  }
  uint64_t channel_id = self_attr.channel_id();

  ListenerHandlerBasePtr* handler_base = nullptr;
  if (msg_listeners_.Get(channel_id, &handler_base)) {
    (*handler_base)->Disconnect(self_attr.id(), opposite_attr.id());
  }
}
```
删除 listener 的流程为：
- 获得 channel_id
- 去 msg_listeners_ 查找 channel_id 所对应的 handler
- 如果 handler 存在，调用其 Disconnect 函数

```c++
Dispatcher::Dispatcher() : is_shutdown_(false) {}

Dispatcher::~Dispatcher() { Shutdown(); }

void Dispatcher::Shutdown() {
  is_shutdown_.store(true);
}

bool Dispatcher::HasChannel(uint64_t channel_id) {
  return msg_listeners_.Has(channel_id);
}
```
其他一些接口的实现

## intra_dispatcher.h
```c++
class IntraDispatcher : public Dispatcher {
 public:
  virtual ~IntraDispatcher();

  template <typename MessageT>
  void OnMessage(uint64_t channel_id, const std::shared_ptr<MessageT>& message,
                 const MessageInfo& message_info);

  template <typename MessageT>
  void AddListener(const RoleAttributes& self_attr,
                   const MessageListener<MessageT>& listener);

  template <typename MessageT>
  void AddListener(const RoleAttributes& self_attr,
                   const RoleAttributes& opposite_attr,
                   const MessageListener<MessageT>& listener);

  template <typename MessageT>
  void RemoveListener(const RoleAttributes& self_attr);

  template <typename MessageT>
  void RemoveListener(const RoleAttributes& self_attr,
                      const RoleAttributes& opposite_attr);

  DECLARE_SINGLETON(IntraDispatcher)

 private:
  template <typename MessageT>
  std::shared_ptr<ListenerHandler<MessageT>> GetHandler(uint64_t channel_id);

  ChannelChainPtr chain_;
};
```
IntraDispatcher 被声明为一个单例，同时有一个 OnMessage 接口, 同时拥有一个 ChannelChainPtr
```c++
template <typename MessageT>
void IntraDispatcher::OnMessage(uint64_t channel_id,
                                const std::shared_ptr<MessageT>& message,
                                const MessageInfo& message_info) {
  if (is_shutdown_.load()) {
    return;
  }
  ListenerHandlerBasePtr* handler_base = nullptr;
  if (msg_listeners_.Get(channel_id, &handler_base)) {
    auto handler =
        std::dynamic_pointer_cast<ListenerHandler<MessageT>>(*handler_base);
    if (handler) {
      handler->Run(message, message_info);
    } else {
      auto msg_size = message::FullByteSize(*message);
      if (msg_size < 0) {
        return;
      }
      std::string msg;
      msg.resize(msg_size);
      if (message::SerializeToHC(*message, const_cast<char*>(msg.data()),
                                 msg_size)) {
        (*handler_base)->RunFromString(msg, message_info);
      } else {
      }
    }
  }
}
```
Onmessage 做的事情为：
- 根据 channel_id 去 msg_listeners_ 中查找对应的 handler, 并将其转化成 ListenerHandler
- 如果 handler 不为空指针，直接调用 handler 的 Run 接口
- 如果 handler 为空，先将 msg 序列化成 string， 然后调用 handler_base 的 RunFromString 接口

```c++
template <typename MessageT>
std::shared_ptr<ListenerHandler<MessageT>> IntraDispatcher::GetHandler(
    uint64_t channel_id) {
  std::shared_ptr<ListenerHandler<MessageT>> handler;
  ListenerHandlerBasePtr* handler_base = nullptr;

  if (msg_listeners_.Get(channel_id, &handler_base)) {
    handler =
        std::dynamic_pointer_cast<ListenerHandler<MessageT>>(*handler_base);
  } else {
    handler.reset(new ListenerHandler<MessageT>());
    msg_listeners_.Set(channel_id, handler);
  }

  return handler;
}
```
GetHandler 是将 dispatcher 中的 AddListener 的部分代码封装成了一个函数，主要作用就是获得 handler
```c++
template <typename MessageT>
void IntraDispatcher::AddListener(const RoleAttributes& self_attr,
                                  const MessageListener<MessageT>& listener) {
  if (is_shutdown_.load()) {
    return;
  }

  auto channel_id = self_attr.channel_id();
  std::string message_type = message::GetMessageName<MessageT>();
  uint64_t self_id = self_attr.id();

  bool created =
      chain_->AddListener(self_id, channel_id, message_type, listener);

  auto handler = GetHandler<MessageT>(self_attr.channel_id());
  if (handler && created) {
    auto listener_wrapper = [this, self_id, channel_id, message_type](
                                const std::shared_ptr<MessageT>& message,
                                const MessageInfo& message_info) {
      this->chain_->Run<MessageT>(self_id, channel_id, message_type, message,
                                  message_info);
    };
    handler->Connect(self_id, listener_wrapper);
  }
}

template <typename MessageT>
void IntraDispatcher::AddListener(const RoleAttributes& self_attr,
                                  const RoleAttributes& opposite_attr,
                                  const MessageListener<MessageT>& listener) {
  if (is_shutdown_.load()) {
    return;
  }

  auto channel_id = self_attr.channel_id();
  std::string message_type = message::GetMessageName<MessageT>();
  uint64_t self_id = self_attr.id();
  uint64_t oppo_id = opposite_attr.id();

  bool created =
      chain_->AddListener(self_id, oppo_id, channel_id, message_type, listener);

  auto handler = GetHandler<MessageT>(self_attr.channel_id());
  if (handler && created) {
    auto listener_wrapper = [this, self_id, oppo_id, channel_id, message_type](
                                const std::shared_ptr<MessageT>& message,
                                const MessageInfo& message_info) {
      this->chain_->Run<MessageT>(self_id, oppo_id, channel_id, message_type,
                                  message, message_info);
    };
    handler->Connect(self_id, oppo_id, listener_wrapper);
  }
}
```
AddListener 做的事情为：
- 获得 channel_id, message_type, self_id
- 调用 chain_ 的 AddListener 接口
- 将 chain_ 的 Run 接口进行 wrap, 然后调用 handler 的 Connect 接口

另一个版本做的事情基本一样，只是多了一个传参
```c++
template <typename MessageT>
void IntraDispatcher::RemoveListener(const RoleAttributes& self_attr) {
  if (is_shutdown_.load()) {
    return;
  }
  Dispatcher::RemoveListener<MessageT>(self_attr);
  chain_->RemoveListener<MessageT>(self_attr.id(), self_attr.channel_id(),
                                   message::GetMessageName<MessageT>());
}

template <typename MessageT>
void IntraDispatcher::RemoveListener(const RoleAttributes& self_attr,
                                     const RoleAttributes& opposite_attr) {
  if (is_shutdown_.load()) {
    return;
  }
  Dispatcher::RemoveListener<MessageT>(self_attr, opposite_attr);
  chain_->RemoveListener<MessageT>(self_attr.id(), opposite_attr.id(),
                                   self_attr.channel_id(),
                                   message::GetMessageName<MessageT>());
}
```
RemoveListener 先调用 Dispatcher 类的 RemoveListener 接口，再调用 chain_ 的 RemoveListener 接口
```c++
class ChannelChain {
  using BaseHandlersType =
      std::map<uint64_t, std::map<std::string, ListenerHandlerBasePtr>>;

 public:
  template <typename MessageT>
  bool AddListener(uint64_t self_id, uint64_t channel_id,
                   const std::string& message_type,
                   const MessageListener<MessageT>& listener) {
    WriteLockGuard<base::AtomicRWLock> lg(rw_lock_);
    auto ret = GetHandler<MessageT>(channel_id, message_type, &handlers_);
    auto handler = ret.first;
    if (handler == nullptr) {
      return ret.second;
    }
    handler->Connect(self_id, listener);
    return ret.second;
  }

  template <typename MessageT>
  bool AddListener(uint64_t self_id, uint64_t oppo_id, uint64_t channel_id,
                   const std::string& message_type,
                   const MessageListener<MessageT>& listener) {
    WriteLockGuard<base::AtomicRWLock> lg(oppo_rw_lock_);
    if (oppo_handlers_.find(oppo_id) == oppo_handlers_.end()) {
      oppo_handlers_[oppo_id] = BaseHandlersType();
    }
    BaseHandlersType& handlers = oppo_handlers_[oppo_id];
    auto ret = GetHandler<MessageT>(channel_id, message_type, &handlers);
    auto handler = ret.first;
    if (handler == nullptr) {
      return ret.second;
    }
    handler->Connect(self_id, oppo_id, listener);
    return ret.second;
  }

  template <typename MessageT>
  void RemoveListener(uint64_t self_id, uint64_t channel_id,
                      const std::string& message_type) {
    WriteLockGuard<base::AtomicRWLock> lg(rw_lock_);
    auto handler = RemoveHandler(channel_id, message_type, &handlers_);
    if (handler) {
      handler->Disconnect(self_id);
    }
  }

  template <typename MessageT>
  void RemoveListener(uint64_t self_id, uint64_t oppo_id, uint64_t channel_id,
                      const std::string& message_type) {
    WriteLockGuard<base::AtomicRWLock> lg(oppo_rw_lock_);
    if (oppo_handlers_.find(oppo_id) == oppo_handlers_.end()) {
      return;
    }
    BaseHandlersType& handlers = oppo_handlers_[oppo_id];
    auto handler = RemoveHandler(channel_id, message_type, &handlers);
    if (oppo_handlers_[oppo_id].empty()) {
      oppo_handlers_.erase(oppo_id);
    }
    if (handler) {
      handler->Disconnect(self_id, oppo_id);
    }
  }

  template <typename MessageT>
  void Run(uint64_t self_id, uint64_t channel_id,
           const std::string& message_type,
           const std::shared_ptr<MessageT>& message,
           const MessageInfo& message_info) {
    ReadLockGuard<base::AtomicRWLock> lg(rw_lock_);
    Run(channel_id, message_type, handlers_, message, message_info);
  }

  template <typename MessageT>
  void Run(uint64_t self_id, uint64_t oppo_id, uint64_t channel_id,
           const std::string& message_type,
           const std::shared_ptr<MessageT>& message,
           const MessageInfo& message_info) {
    ReadLockGuard<base::AtomicRWLock> lg(oppo_rw_lock_);
    if (oppo_handlers_.find(oppo_id) == oppo_handlers_.end()) {
      return;
    }
    BaseHandlersType& handlers = oppo_handlers_[oppo_id];
    Run(channel_id, message_type, handlers, message, message_info);
  }

 private:
  // NOTE: lock hold
  template <typename MessageT>
  std::pair<std::shared_ptr<ListenerHandler<MessageT>>, bool> GetHandler(
      uint64_t channel_id, const std::string& message_type,
      BaseHandlersType* handlers) {
    std::shared_ptr<ListenerHandler<MessageT>> handler;
    bool created = false;  // if the handler is created

    if (handlers->find(channel_id) == handlers->end()) {
      (*handlers)[channel_id] = std::map<std::string, ListenerHandlerBasePtr>();
    }

    if ((*handlers)[channel_id].find(message_type) ==
        (*handlers)[channel_id].end()) {
      handler.reset(new ListenerHandler<MessageT>());
      (*handlers)[channel_id][message_type] = handler;
      created = true;
    } else {
      handler = std::dynamic_pointer_cast<ListenerHandler<MessageT>>(
          (*handlers)[channel_id][message_type]);
    }

    return std::make_pair(handler, created);
  }

  // NOTE: Lock hold
  ListenerHandlerBasePtr RemoveHandler(int64_t channel_id,
                                       const std::string message_type,
                                       BaseHandlersType* handlers) {
    ListenerHandlerBasePtr handler_base;
    if (handlers->find(channel_id) != handlers->end()) {
      if ((*handlers)[channel_id].find(message_type) !=
          (*handlers)[channel_id].end()) {
        handler_base = (*handlers)[channel_id][message_type];
        (*handlers)[channel_id].erase(message_type);
      }
      if ((*handlers)[channel_id].empty()) {
        (*handlers).erase(channel_id);
      }
    }
    return handler_base;
  }

  template <typename MessageT>
  void Run(const uint64_t channel_id, const std::string& message_type,
           const BaseHandlersType& handlers,
           const std::shared_ptr<MessageT>& message,
           const MessageInfo& message_info) {
    const auto channel_handlers_itr = handlers.find(channel_id);
    if (channel_handlers_itr == handlers.end()) {
      return;
    }
    const auto& channel_handlers = channel_handlers_itr->second;

    std::string msg;
    for (const auto& ele : channel_handlers) {
      auto handler_base = ele.second;
      if (message_type == ele.first) {
        auto handler =
            std::static_pointer_cast<ListenerHandler<MessageT>>(handler_base);
        if (handler == nullptr) {
          continue;
        }
        handler->Run(message, message_info);
      } else {
        if (msg.empty()) {
          auto msg_size = message::FullByteSize(*message);
          if (msg_size < 0) {
            continue;
          }
          msg.resize(msg_size);
          if (!message::SerializeToHC(*message, const_cast<char*>(msg.data()),
                                      msg_size)) {
            msg.clear();
          }
        }
        if (!msg.empty()) {
          (handler_base)->RunFromString(msg, message_info);
        }
      }
    }
  }

  BaseHandlersType handlers_;
  base::AtomicRWLock rw_lock_;
  std::map<uint64_t, BaseHandlersType> oppo_handlers_;
  base::AtomicRWLock oppo_rw_lock_;
};
```
ChannelChain 类接口在类内实现



