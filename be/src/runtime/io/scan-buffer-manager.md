```cpp




namespace impala {
namespace io {
// 前向声明
class ScanRange;
class BufferDescriptor;
class ScanRangeLockStore;

// ScanRange 的缓冲区管理实现。每个 ScanRange 包含一个就绪缓冲区队列和一个未使用缓冲区队列。对于每个 ScanRange，只有一个生产者和消费者线程，即只有一个磁盘线程会在任何时间向扫描范围scan range推送，并且只有一个线程会从队列中移除。这是为了保证缓冲区按文件顺序入队和读取。
//
// 缓冲区生命周期：
// 磁盘线程将使用来自未使用缓冲区队列unused buffer queue的缓冲区来读取数据。一旦数据被读入缓冲区，它将被放入就绪缓冲区中，消费者线程将使用 ScanRange::GetNext() 读取它。一旦读取完成，消费者将使用 ScanRange::ReturnBuffer() 返回该缓冲区以供重用，它将被添加到未使用缓冲区unused buffer queue中。一旦 ScanRange 完成数据读取或被取消，两队列中的所有剩余缓冲区将被释放。
class ScanBufferManager {
 public:

  /// 与使用此缓冲区管理器的扫描范围关联的缓冲区标签。
  /// 有 3 个标签，每个标签标识用于读取的不同类型的缓冲区：
  /// a) CLIENT_BUFFER：客户端分配的缓冲区，大小足够容纳整个扫描范围的数据，由调用者在构建扫描范围时提供。
  ///    此缓冲区是此缓冲区管理器的外部缓冲区，不由其管理
  ///    即，不分配、释放或不维护在内部队列中。
  /// b) CACHED_BUFFER：如果扫描范围从 HDFS 缓存读取，则使用缓存的 HDFS 缓冲区。又像 CLIENT_BUFFER 一样，这是此缓冲区管理器的外部缓冲区。
  /// c) INTERNAL_BUFFER：表示由此缓冲区管理器分配和管理缓冲区。IoMgr 通过 AllocateBuffersForRange() 分配缓冲区，此
  ///    管理器在内部队列中维护它们。
  enum class BufferTag { CLIENT_BUFFER, CACHED_BUFFER, INTERNAL_BUFFER };

  /// 为 ScanRange 创建 ScanBufferManager。它将负责相应范围respective range的缓冲区
  /// 管理。
  ScanBufferManager(ScanRange* range);

  ~ScanBufferManager();

  void Init();

  /// 将 'buffers' 添加到用于读取数据的缓冲区中。缓冲区被添加到 'unused_iomgr_buffers_' 中。
  /// 在调用此函数之前，需要使用此缓冲区管理器的扫描范围scan range锁获取锁。
  /// 如果 'returned' 为 true，则这些是从 ScanRange::GetNext() 返回并通过 ScanRange::ReturnBuffer() 回收的缓冲区。否则，这些是新分配的要添加的缓冲区。
  /// 如果至少一个缓冲区被添加到 'unused_iomgr_buffers_' 中，则返回 'true'。
  bool AddUnusedBuffers(const std::unique_lock<std::mutex>& scan_range_lock,
    std::vector<std::unique_ptr<BufferDescriptor>>&& buffers, bool returned);

  /// 从 'unused_iomgr_buffers_' 中移除一个缓冲区并更新
  /// 'unused_iomgr_buffer_bytes_'。如果 'unused_iomgr_buffers_' 为空，则返回 nullptr。
  /// 调用者必须通过 'scan_range_lock' 持有 'scan_range_->lock_'。
  std::unique_ptr<BufferDescriptor> GetUnusedBuffer(
      const std::unique_lock<std::mutex>& scan_range_lock);

  /// 将带有有效数据的缓冲区入队到 'ready_buffer_' 中。
  /// 调用者将缓冲区的所有权传递给缓冲区管理器，此调用之后访问缓冲区无效。在调用此之前，需要获取使用此缓冲区管理器的 ScanRange 的锁。如果 'scan_range_' 已被取消，
  /// 'buffer' 将被清理而不是入队到 'ready_buffer_' 中。
  void EnqueueReadyBuffer(const std::unique_lock<std::mutex>& scan_range_lock,
      std::unique_ptr<BufferDescriptor> buffer);

  /// 为最多 'max_bytes' 个缓冲区分配内存并将其添加到 'buffers' 中。
  /// 从 ScanRange::AllocateBuffersForRange 调用
  ///
  /// 缓冲区大小基于 'scan_range_->len()' 选择。'max_bytes' 必须 >=
  /// 'min_buffer_size'，以便至少分配一个缓冲区。调用者
  /// 必须确保 'bp_client' 至少有 'max_bytes' 未使用的预留额度。
  /// 如果缓冲区成功分配，则返回 ok。
  Status AllocateBuffersForRange(
      BufferPool::ClientHandle* bp_client, int64_t max_bytes,
      std::vector<std::unique_ptr<BufferDescriptor>>& buffers, int64_t min_buffer_size,
      int64_t max_buffer_size);

  /// 清理不被回收或由客户端返回的缓冲区。
  /// 调用者必须通过 'scan_range_lock' 持有 'scan_range_->lock_'。
  /// 此函数可能获取 'scan_range_->file_reader_->lock()'
  void CleanUpBuffer(const std::unique_lock<std::mutex>& scan_range_lock,
      const std::unique_ptr<BufferDescriptor> buffer);

  /// 与 CleanUpBuffer() 相同，但清理多个缓冲区。调用者必须
  /// 通过 'scan_range_lock' 持有 'scan_range_->lock_'。
  void CleanUpBuffers(const std::unique_lock<std::mutex>& scan_range_lock,
      std::vector<std::unique_ptr<BufferDescriptor>>&& buffers);

  /// 清理 'unused_iomgr_buffers_' 中的所有缓冲区。只有在扫描
  /// 范围被取消或到达 eos 时才有效调用。调用者必须通过
  /// 'scan_range_lock' 持有 'scan_range_->lock_'。
  void CleanUpUnusedBuffers(const std::unique_lock<std::mutex>& scan_range_lock);

  /// 清理 'ready_buffer_' 中的所有缓冲区。调用者必须通过 'scan_range_lock' 持有 'scan_range_->lock_'。只有在使用此缓冲区管理器的扫描范围
  /// 被取消时才有效调用。
  void CleanUpReadyBuffers(const std::unique_lock<std::mutex>& scan_range_lock);

  std::string DebugString() const {
    std::stringstream ss;
    ss << " buffer_queue=" << ready_buffers_.size()
       << " num_buffers_in_readers=" << num_buffers_in_reader_.Load()
       << " unused_iomgr_buffers=" << unused_iomgr_buffers_.size()
       << " unused_iomgr_buffer_bytes=" << unused_iomgr_buffer_bytes_;
    return ss.str();
  }

  /// 从 'ready_buffer_' 中移除第一个缓冲区并将其分配给 '*buffer'。
  /// 如果 'ready_buffer_' 为空且 '*buffer' 无法分配，则返回 'false'，
  /// 否则返回 'true'。
  /// 在调用此方法之前需要持有 'scan_range_->lock_'。
  bool PopFirstReadyBuffer(const std::unique_lock<std::mutex>& scan_range_lock,
      std::unique_ptr<BufferDescriptor>* buffer);

  /// 验证缓冲区状态。验证基于使用此管理器的
  /// ScanRange 的状态进行。
  /// 在调用此方法之前，需要通过 'scan_range_lock' 持有 'scan_range_->lock_'。
  bool Validate(const std::unique_lock<std::mutex>& scan_range_lock);

  void set_cached_buffer() {
    buffer_tag_ = BufferTag::CACHED_BUFFER;
  }

  void set_client_buffer() {
    buffer_tag_ = BufferTag::CLIENT_BUFFER;
  }

  void set_internal_buffer() {
    buffer_tag_ = BufferTag::INTERNAL_BUFFER;
  }

  bool is_cached() const {
    return buffer_tag_ == BufferTag::CACHED_BUFFER;
  }

  bool is_client_buffer() const {
    return buffer_tag_ == BufferTag::CLIENT_BUFFER;
  }

  bool is_internal_buffer() const {
    return buffer_tag_ == BufferTag::INTERNAL_BUFFER;
  }

  BufferTag buffer_tag() const { return buffer_tag_; }

  bool is_readybuffer_empty() const { return ready_buffers_.empty(); }

  int num_buffers_in_reader() const { return num_buffers_in_reader_.Load(); }

  void add_buffers_in_reader(int inc) { num_buffers_in_reader_.Add(inc); }

  void add_iomgr_buffer_cumulative_bytes_used(int inc) {
    iomgr_buffer_cumulative_bytes_used_ += inc;
  }

 private:

  /// 使用此缓冲区管理器的扫描范围。
  ScanRange* const scan_range_;

  /// 已通过 GetNext() 返回给客户端但尚未返回的缓冲区数量。
  AtomicInt32 num_buffers_in_reader_{0};

  /// 用于读取的缓冲区，如果 'buffer_tag_' 为 INTERNAL_BUFFER。
  /// 这些缓冲区最初在客户端调用 AllocateBuffersForRange() 时填充
  /// 并用于读取扫描数据。每次读取时从此向量中取出缓冲区，并在使用后添加回去。
  std::vector<std::unique_ptr<BufferDescriptor>> unused_iomgr_buffers_;

  /// 'unused_iomgr_buffers_' 中缓冲区的总字节数。
  int64_t unused_iomgr_buffer_bytes_ = 0;

  /// 由 DoRead() 从 'unused_iomgr_buffers_' 取出的 I/O mgr 缓冲区的累积字节数。
  /// 用于推断读取扫描范围剩余部分需要保留多少字节的缓冲区。
  int64_t iomgr_buffer_cumulative_bytes_used_ = 0;

  /// 为此扫描范围排队的 I/O 缓冲区。当调用 Cancel() 时，
  /// 此队列由取消线程清空。即如果
  /// 'cancel_status_' 不是 OK，则此队列始终为空。
  std::deque<std::unique_ptr<BufferDescriptor>> ready_buffers_;

  /// 表示用于读取扫描数据的缓冲区类型的标签。
  /// 请参阅枚举 'BufferTag' 的注释以获取更多详细信息。
  BufferTag buffer_tag_;

  /// 为长度为 'scan_range_len' 的扫描范围数据选择缓冲区大小。
  /// 'min_buffer_size' 和 'max_buffer_size' 是可以
  /// 分配的最小和最大缓冲区大小。此外，缓冲区大小的累积和不应
  /// 超过 'max_bytes'。
  static std::vector<int64_t> ChooseBufferSizes(int64_t scan_range_len,
      int64_t max_bytes, int64_t min_buffer_size, int64_t max_buffer_size) {
    DCHECK_GE(max_bytes, min_buffer_size);
    std::vector<int64_t> buffer_sizes;
    int64_t bytes_allocated = 0;
    while (bytes_allocated < scan_range_len) {
      int64_t bytes_remaining = scan_range_len - bytes_allocated;
      // 要么分配一个最大大小的缓冲区，要么分配一个较小的缓冲区以适应范围的剩余部分。
      int64_t next_buffer_size;
      if (bytes_remaining >= max_buffer_size) {
        next_buffer_size = max_buffer_size;
      } else {
        next_buffer_size =
            std::max(min_buffer_size, BitUtil::RoundUpToPowerOfTwo(bytes_remaining));
      }
      if (next_buffer_size + bytes_allocated > max_bytes) {
        // 无法分配所需的缓冲区大小。确保至少分配一个
        // 缓冲区。
        if (bytes_allocated > 0) break;
        next_buffer_size = BitUtil::RoundDownToPowerOfTwo(max_bytes);
      }
      DCHECK(BitUtil::IsPowerOf2(next_buffer_size)) << next_buffer_size;
      buffer_sizes.push_back(next_buffer_size);
      bytes_allocated += next_buffer_size;
    }
    return buffer_sizes;
  }
};
}
}
```
-----------------------------------------------------------------------------------
讲述下这个类的设计思路，我作为文档保存，必要时，结合代码讲述

# ScanBufferManager 类设计思路文档


## 一、类核心定位
`ScanBufferManager` 是 Apache Impala 中 **扫描范围（`ScanRange`）的专属缓冲区管理器**，负责单个 `ScanRange` 的缓冲区全生命周期管理——包括缓冲区的分配、复用、流转、清理，以及与 `ScanRange` 状态（如取消、完成）的联动。其核心目标是**高效管理内存缓冲区，确保扫描任务（如读取 HDFS/S3 文件）按序、安全地处理数据，同时避免内存泄漏或资源浪费**。


## 二、核心设计前提与约束
在理解具体设计前，需明确两个关键前提，这是所有逻辑的基础：
1. **单生产者-单消费者模型**：每个 `ScanRange` 仅对应一个“生产者线程”（如磁盘 IO 线程，负责将数据读入缓冲区）和一个“消费者线程”（如查询计算线程，负责从缓冲区取数据）。这保证了缓冲区的“入队”和“出队”严格按文件顺序执行，无需复杂的多线程同步逻辑。
2. **缓冲区类型区分**：通过 `BufferTag` 枚举明确缓冲区来源，不同类型的缓冲区管理策略完全不同（避免混淆外部/内部缓冲区的责任边界）：
   - `CLIENT_BUFFER`：调用方（如上层查询模块）自带的缓冲区，管理器仅“借用”，不负责分配/释放；
   - `CACHED_BUFFER`：HDFS 缓存中的缓冲区，同样属于“外部资源”，管理器不干预其生命周期；
   - `INTERNAL_BUFFER`：管理器自行分配、维护的缓冲区（核心管理对象），需处理分配、复用、清理全流程。


## 三、核心成员变量设计
成员变量的设计围绕“缓冲区状态跟踪”和“`ScanRange` 关联”展开，每个变量都有明确的职责，避免状态混乱：

| 成员变量                  | 类型                          | 核心作用                                                                 |
|---------------------------|-------------------------------|--------------------------------------------------------------------------|
| `scan_range_`             | `ScanRange* const`            | 绑定的扫描范围，所有缓冲区操作均为该 `ScanRange` 服务（一对一关联）。     |
| `num_buffers_in_reader_`  | `AtomicInt32`                 | 跟踪“已被消费者取走但未归还”的缓冲区数量（避免重复释放或内存泄漏）。     |
| `unused_iomgr_buffers_`   | `vector<unique_ptr<BufferDescriptor>>` | 未使用的内部缓冲区列表（空缓冲区池），供生产者线程读取数据时取用。       |
| `unused_iomgr_buffer_bytes_` | `int64_t`                   | 快速记录 `unused_iomgr_buffers_` 中所有缓冲区的总字节数（避免频繁遍历）。|
| `iomgr_buffer_cumulative_bytes_used_` | `int64_t`             | 累计已从“未使用列表”中取出的缓冲区总字节数（用于判断是否需补充缓冲区）。 |
| `ready_buffers_`          | `deque<unique_ptr<BufferDescriptor>>` | 已读好数据的缓冲区队列（就绪池），供消费者线程取用（FIFO 顺序）。        |
| `buffer_tag_`             | `BufferTag`                   | 标记当前 `ScanRange` 使用的缓冲区类型（决定管理器的行为模式）。           |


## 四、核心功能模块设计（结合代码）
`ScanBufferManager` 的功能按“缓冲区生命周期”可分为 **分配、流转、清理、校验** 四大模块，每个模块的代码逻辑均围绕“安全、高效”展开。


### 模块1：缓冲区分配（`AllocateBuffersForRange`）
#### 功能目标
为 `INTERNAL_BUFFER` 类型的 `ScanRange` 分配符合规则的缓冲区，确保：
- 单个缓冲区大小在 `min_buffer_size` ~ `max_buffer_size` 之间；
- 所有缓冲区总字节数不超过 `max_bytes`；
- 缓冲区大小为 2 的幂（便于内存对齐，提升操作系统处理效率）。

#### 代码逻辑拆解
```cpp
Status ScanBufferManager::AllocateBuffersForRange(
    BufferPool::ClientHandle* bp_client, int64_t max_bytes,
    vector<unique_ptr<BufferDescriptor>>& buffers,
    int64_t min_buffer_size, int64_t max_buffer_size) {
  // 1. 前置校验：确保参数合法（如总内存至少能装下一个最小缓冲区）
  DCHECK_GE(max_bytes, min_buffer_size);
  DCHECK(buffers.empty());
  DCHECK(buffer_tag_ == BufferTag::INTERNAL_BUFFER)  // 仅内部缓冲区需分配
      << "invalid to allocate buffers when already reading into an external buffer";

  // 2. 计算缓冲区大小列表（调用静态方法 ChooseBufferSizes）
  BufferPool* bp = ExecEnv::GetInstance()->buffer_pool();
  Status status;
  vector<int64_t> buffer_sizes = ChooseBufferSizes(
      scan_range_->bytes_to_read(),  // 扫描任务需读的总数据量
      max_bytes, min_buffer_size, max_buffer_size);

  // 3. 逐个从 BufferPool 分配缓冲区，封装为 BufferDescriptor
  for (int64_t buffer_size : buffer_sizes) {
    BufferPool::BufferHandle handle;
    status = bp->AllocateBuffer(bp_client, buffer_size, &handle);  // 从内存池拿内存
    if (!status.ok()) return status;  // 分配失败直接返回错误
    // 封装为 BufferDescriptor（关联当前 ScanRange 和内存句柄）
    buffers.emplace_back(new BufferDescriptor(scan_range_, bp_client, move(handle)));
  }
  return Status::OK();
}
```

#### 关键辅助：`ChooseBufferSizes` 静态方法
该方法是“分配逻辑的核心”，通过循环计算每个缓冲区的大小，平衡“效率”（用大缓冲区减少数量）和“规则”（不超内存上限、满足大小约束）：
1. 优先用 `max_buffer_size` 分配（减少缓冲区数量，降低管理开销）；
2. 剩余数据不足 `max_buffer_size` 时，按“剩余数据量向上取 2 的幂”分配，但不小于 `min_buffer_size`；
3. 若加当前缓冲区会超 `max_bytes`：
   - 若已分配过缓冲区，直接停止（见好就收）；
   - 若未分配过，将缓冲区大小改为“`max_bytes` 向下取 2 的幂”（兜底，确保至少有一个缓冲区）。


### 模块2：缓冲区流转（分配→使用→归还→复用）
缓冲区流转是 `ScanBufferManager` 最核心的逻辑，覆盖“生产者取空缓冲→读数据→入就绪队列→消费者取就绪缓冲→归还空缓冲→复用”全流程，涉及 4 个关键方法：

#### 2.1 生产者取空缓冲：`GetUnusedBuffer`
生产者（磁盘线程）读取数据前，从“未使用列表”中取一个空缓冲区：
```cpp
unique_ptr<BufferDescriptor> ScanBufferManager::GetUnusedBuffer(
    const unique_lock<mutex>& scan_range_lock) {
  DCHECK(scan_range_->is_locked(scan_range_lock));  // 必须加锁，确保线程安全
  if (unused_iomgr_buffers_.empty()) return nullptr;  // 无空缓冲，返回空（需等待）
  
  // 从列表尾部取缓冲（vector 尾部操作效率高）
  unique_ptr<BufferDescriptor> result = move(unused_iomgr_buffers_.back());
  unused_iomgr_buffers_.pop_back();
  unused_iomgr_buffer_bytes_ -= result->buffer_len();  // 更新总字节数
  return result;
}
```

#### 2.2 生产者将数据缓冲入就绪队列：`EnqueueReadyBuffer`
生产者读完数据后，将缓冲区放入“就绪队列”，供消费者取用；若 `ScanRange` 已取消，直接清理缓冲区（避免无效数据）：
```cpp
void ScanBufferManager::EnqueueReadyBuffer(
    const std::unique_lock<std::mutex>& scan_range_lock,
    unique_ptr<BufferDescriptor> buffer) {
  DCHECK(scan_range_->is_locked(scan_range_lock));
  DCHECK(buffer->buffer_ != nullptr) << "Cannot enqueue freed buffer";  // 防止入队已释放缓冲

  if (scan_range_->is_cancelled()) {
    CleanUpBuffer(scan_range_lock, move(buffer));  // 已取消，直接清理
  } else {
    // 若缓冲区标记“数据已读完（eosr）”，清理剩余未使用缓冲（避免浪费）
    if (buffer->eosr()) CleanUpUnusedBuffers(scan_range_lock);
    ready_buffers_.emplace_back(move(buffer));  // 入队（deque 头部入队效率高）
  }
}
```

#### 2.3 消费者取就绪缓冲：`PopFirstReadyBuffer`
消费者（计算线程）从“就绪队列”头部取缓冲（FIFO 顺序，确保数据按文件顺序读取）：
```cpp
bool ScanBufferManager::PopFirstReadyBuffer(
    const unique_lock<mutex>& scan_range_lock,
    unique_ptr<BufferDescriptor>* buffer) {
  DCHECK(scan_range_->is_locked(scan_range_lock));
  if (ready_buffers_.empty()) return false;  // 无就绪缓冲，返回 false（需等待）
  
  *buffer = move(ready_buffers_.front());  // 取队列头部缓冲
  ready_buffers_.pop_front();
  // 校验：若缓冲标记“数据已读完”，未使用列表必须为空（避免残留缓冲）
  DCHECK(!(*buffer)->eosr() || unused_iomgr_buffers_.empty()) << DebugString();
  return true;
}
```

#### 2.4 消费者归还空缓冲：`AddUnusedBuffers`
消费者用完缓冲后，通过该方法将缓冲归还到“未使用列表”，供生产者复用；同时处理“缓冲是否需清理”的判断（如 `ScanRange` 已取消/完成，则直接清理，不复用）：
```cpp
bool ScanBufferManager::AddUnusedBuffers(
    const unique_lock<mutex>& scan_range_lock,
    vector<unique_ptr<BufferDescriptor>>&& buffers, bool returned) {
  DCHECK(scan_range_->is_locked(scan_range_lock));
  if (returned) {
    // 若为“消费者归还的缓冲”，更新“已取未还”计数
    num_buffers_in_reader_.Add(-buffers.size());
  }

  bool buffer_added = false;
  for (unique_ptr<BufferDescriptor>& buffer : buffers) {
    // 以下情况不复用，直接清理缓冲：
    // 1. 不是内部缓冲区（外部缓冲无需管理器维护）；
    // 2. ScanRange 已取消/已完成；
    // 3. 未使用缓冲总字节数已足够读取剩余数据（无需更多缓冲）
    if (buffer_tag_ != BufferTag::INTERNAL_BUFFER
        || scan_range_->is_cancelled()
        || scan_range_->is_eosr_queued()
        || unused_iomgr_buffer_bytes_ >= scan_range_->len() - iomgr_buffer_cumulative_bytes_used_) {
      CleanUpBuffer(scan_range_lock, move(buffer));
    } else {
      // 复用：加入未使用列表，更新总字节数
      unused_iomgr_buffer_bytes_ += buffer->buffer_len();
      unused_iomgr_buffers_.emplace_back(move(buffer));
      buffer_added = true;
    }
  }
  return buffer_added;  // 返回是否有缓冲被加入（用于判断是否需唤醒阻塞的生产者）
}
```


### 模块3：缓冲区清理（避免内存泄漏）
清理逻辑是“防止内存泄漏”的关键，针对不同场景提供 4 个清理方法，核心是调用 `CleanUpBuffer` 释放缓冲区内存，并联动关闭 `ScanRange` 的文件读取器（避免文件句柄泄漏）。

#### 3.1 单个缓冲清理：`CleanUpBuffer`
```cpp
void ScanBufferManager::CleanUpBuffer(
    const unique_lock<mutex>& scan_range_lock,
    const unique_ptr<BufferDescriptor> buffer_desc) {
  DCHECK(scan_range_->is_locked(scan_range_lock));
  DCHECK(buffer_desc != nullptr);
  DCHECK_EQ(buffer_desc->scan_range(), scan_range_);  // 确保缓冲属于当前 ScanRange

  buffer_desc->Free();  // 调用 BufferDescriptor 的 Free 方法，释放内存（归还 BufferPool）
  // 若“已取未还”缓冲数为 0 或后续无缓冲需归还，关闭文件读取器（释放文件句柄）
  scan_range_->CloseReader(scan_range_lock);
}
```

#### 3.2 批量缓冲清理：`CleanUpBuffers`
对多个缓冲批量调用 `CleanUpBuffer`，避免重复代码：
```cpp
void ScanBufferManager::CleanUpBuffers(
    const unique_lock<mutex>& scan_range_lock,
    vector<unique_ptr<BufferDescriptor>>&& buffers) {
  for (unique_ptr<BufferDescriptor>& buffer : buffers) {
    CleanUpBuffer(scan_range_lock, move(buffer));
  }
}
```

#### 3.3 未使用缓冲清理：`CleanUpUnusedBuffers`
当 `ScanRange` 完成（eosr）或取消时，清理“未使用列表”中所有空缓冲：
```cpp
void ScanBufferManager::CleanUpUnusedBuffers(const unique_lock<mutex>& scan_range_lock) {
  DCHECK(scan_range_->is_locked(scan_range_lock));
  // 循环从“未使用列表”取缓冲，调用 CleanUpBuffer 释放
  while (!unused_iomgr_buffers_.empty()) {
    CleanUpBuffer(scan_range_lock, GetUnusedBuffer(scan_range_lock));
  }
}
```

#### 3.4 就绪缓冲清理：`CleanUpReadyBuffers`
仅当 `ScanRange` 取消时调用，清理“就绪队列”中所有已读数据的缓冲（避免无效数据占用内存）：
```cpp
void ScanBufferManager::CleanUpReadyBuffers(const unique_lock<mutex>& scan_range_lock) {
  DCHECK(scan_range_->is_locked(scan_range_lock));
  // 循环从“就绪队列”取缓冲，调用 CleanUpBuffer 释放
  while (!ready_buffers_.empty()) {
    CleanUpBuffer(scan_range_lock, move(ready_buffers_.front()));
    ready_buffers_.pop_front();
  }
}
```


### 模块4：状态校验（`Validate`）
为避免逻辑漏洞导致的状态错乱（如“已取消但就绪队列非空”“缺缓冲阻塞但有未使用缓冲”），提供 `Validate` 方法做“健康检查”，通常在调试或关键节点调用：
```cpp
bool ScanBufferManager::Validate(const std::unique_lock<std::mutex>& scan_range_lock) {
  DCHECK(scan_range_->is_locked(scan_range_lock));
  // 先获取 ScanRange 的关键状态（取消、完成、阻塞）
  bool range_cancelled = scan_range_->is_cancelled();
  bool eosr_queued = scan_range_->is_eosr_queued();
  bool blocked_on_buffer = scan_range_->is_blocked_on_buffer();

  // 校验1：已取消的 ScanRange，就绪队列必须空
  if (range_cancelled && !ready_buffers_.empty()) {
    LOG(ERROR) << "Cancelled range should not have queued buffers";
    return false;
  }

  // 校验2：未使用缓冲的“记录总字节数”与“实际总和”必须一致
  int64_t unused_iomgr_buffer_bytes = 0;
  for (auto& buffer : unused_iomgr_buffers_) {
    unused_iomgr_buffer_bytes += buffer->buffer_len();
  }
  if (unused_iomgr_buffer_bytes != unused_iomgr_buffer_bytes_) {
    LOG(ERROR) << "unused_iomgr_buffer_bytes_ incorrect actual: "
               << unused_iomgr_buffer_bytes_ << " vs. expected: " << unused_iomgr_buffer_bytes;
    return false;
  }

  // 校验3：已完成/取消的 ScanRange，未使用列表必须空
  bool is_finished = range_cancelled || eosr_queued;
  if (is_finished && !unused_iomgr_buffers_.empty()) {
    LOG(ERROR) << "Held onto too many buffers " << unused_iomgr_buffers_.size();
    return false;
  }

  // 校验4：未完成但因缺缓冲阻塞的 ScanRange，未使用列表必须空（否则阻塞逻辑有误）
  if (!is_finished && blocked_on_buffer && !unused_iomgr_buffers_.empty()) {
    LOG(ERROR) << "ScanRange is Blocked despite having buffers";
    return false;
  }

  return true;  // 所有校验通过，状态正常
}
```


## 五、生命周期管理
`ScanBufferManager` 与 `ScanRange` 生命周期强绑定：
1. **创建**：`ScanRange` 初始化时，创建对应的 `ScanBufferManager`（构造函数绑定 `scan_range_`，默认缓冲区类型为 `INTERNAL_BUFFER`）；
2. **初始化**：调用 `Init()` 重置缓冲区统计变量（如 `unused_iomgr_buffer_bytes_`）；
3. **运行**：通过上述“分配→流转→清理”逻辑管理缓冲区；
4. **销毁**：`ScanRange` 销毁时，`ScanBufferManager` 析构函数触发，此时需确保：
   - `ready_buffers_` 为空（无就绪缓冲残留）；
   - `num_buffers_in_reader

   *****************************************************************************************************



#pragma once

#include <algorithm>
#include <vector>

#include "util/condition-variable.h"
#include "util/bit-util.h"

namespace impala {
namespace io {
// 这是给ScanRange做缓冲区管理的具体实现代码。每个ScanRange里都有两个队列：
// 一个装“就绪缓冲区”（里面的缓冲区已经存好数据，等着用），另一个装“未使用缓冲区”（里面的缓冲区是空的，还没用来存数据）。
// 对每个ScanRange来说，只有一个“生产者线程”和一个“消费者线程”在干活——简单说就是，
// 任何时候都只有一个磁盘线程往队列里塞缓冲区，也只有一个线程从队列里拿缓冲区。这么设计是为了保证：
// 缓冲区按文件里数据的顺序排队，也按这个顺序被读取（不会乱序）。


// 缓冲区的生命周期是这样的：
// 1. 磁盘线程先从“未使用缓冲区”队列里拿一个空缓冲区，用它来读数据；
// 2. 数据读完、存进缓冲区后，这个缓冲区就会被放进“就绪缓冲区”队列；
// 3. 然后“消费者线程”会调用ScanRange::GetNext()方法，从“就绪缓冲区”里拿这个缓冲区来用；
// 4. 消费者用完之后，会调用ScanRange::ReturnBuffer()方法把缓冲区还回来，
// 这个缓冲区又会被加回“未使用缓冲区”队列，等着下次再用；
// 5. 等ScanRange把所有数据都读完了，或者这个ScanRange被取消了，
// 那两个队列里剩下的所有缓冲区都会被释放掉（把内存还回去，不浪费）。
class ScanBufferManager {
 public:

  /// Tag for the buffer associated with scan range using this buffer manager.
  /// There are 3 tags and each identify different types of buffer used for read:
  /// a) CLIENT_BUFFER: A client allocated buffer, large enough to fit the whole scan
  ///    range's data, that is provided by the caller when constructing the scan range.
  ///    This buffer is external to this buffer manager and is not managed by it
  ///    i.e., not allocated, freed or maintained in internal queues.
  /// b) CACHED_BUFFER: Cached HDFS buffers if the scan range was read from the HDFS
  ///    cache. Again like CLIENT_BUFFER this is external to this buffer manager.
  /// c) INTERNAL_BUFFER: It represents buffers allocated and managed by this buffer
  ///    manager. IoMgr allocates the buffer via AllocateBuffersForRange() and this
  ///    manager maintains them in their internal queues.
  // 给“当前缓冲区管理器负责的扫描任务”所用的缓冲区，贴的“身份标签”。
// 一共3种标签，每种标签对应一种读取数据时会用到的缓冲区类型，具体说明如下：
// a) CLIENT_BUFFER（调用方自带缓冲区）：这是调用方（比如用这个扫描功能的人/模块）自己分配的缓冲区，
//    大小得够装下当前扫描任务的所有数据，而且要在创建扫描任务的时候就提供给程序。
//    这个缓冲区不归当前缓冲区管理器管，是“外部的”——也就是说，管理器不会帮它分配内存、不会帮它释放内存，
//    也不会把它放进自己的内部队列里存着。
// b) CACHED_BUFFER（HDFS缓存缓冲区）：如果当前扫描任务的数据是从HDFS缓存（HDFS提前存好的高速数据区）里读的，
//    用的就是这种缓冲区。跟上面的调用方自带缓冲区一样，它也不归当前缓冲区管理器管，是“外部的”。
// c) INTERNAL_BUFFER（管理器内部缓冲区）：这种是当前缓冲区管理器自己分配、自己管理的缓冲区。
//    由IO管理器（IoMgr）通过AllocateBuffersForRange()这个方法分配内存，然后由当前管理器把它放进自己的内部队列里维护（比如存着备用、标记使用状态等）。

enum class BufferTag { CLIENT_BUFFER, CACHED_BUFFER, INTERNAL_BUFFER };
  enum class BufferTag { CLIENT_BUFFER, CACHED_BUFFER, INTERNAL_BUFFER };

  /// Creates ScanBufferManager for a ScanRange. It will take care of buffer
  /// management for the respective range.
  /// 为一个 ScanRange（扫描范围）创建对应的 ScanBufferManager（扫描缓冲区管理器）实例。
  /// 该 ScanBufferManager 实例将负责 “对应扫描范围”（即构造函数参数 range 指向的 ScanRange）的所有缓冲区管理工作。
  ScanBufferManager(ScanRange* range);

  ~ScanBufferManager();

  void Init();

  /// Add 'buffers' to read data into. Buffer is added to 'unused_iomgr_buffers_' .
  /// Need to take lock on scan range using this buffer manager before invoking this
  /// function.
  /// If 'returned' is true, the buffers returned from ScanRange::GetNext() that are
  /// being recycled via ScanRange::ReturnBuffer(). Otherwise the buffers are newly
  /// allocated buffers to be added.
  /// Returns 'true' if at least one buffer gets added to 'unused_iomgr_buffers_'.
  bool AddUnusedBuffers(const std::unique_lock<std::mutex>& scan_range_lock,
    std::vector<std::unique_ptr<BufferDescriptor>>&& buffers, bool returned);

  /// Remove a buffer from 'unused_iomgr_buffers_' and update
  /// 'unused_iomgr_buffer_bytes_'. If 'unused_iomgr_buffers_' is empty, return nullptr.
  /// 'scan_range_->lock_' must be held by the caller via 'scan_range_lock'.
  std::unique_ptr<BufferDescriptor> GetUnusedBuffer(
      const std::unique_lock<std::mutex>& scan_range_lock);

  /// Enqueues into 'ready_buffer_' with valid data.
  /// The caller passes ownership of buffer to buffer manager and it is not valid
  /// to access buffer after this call. It needs lock taken upon ScanRange using
  /// this buffer manager before invoking this. If 'scan_range_' is already cancelled,
  /// 'buffer' will be cleaned up instead of enqueing into 'ready_buffer_'.
  // 把“存好有效数据的缓冲区”放进名叫ready_buffer_的队列里（简单说就是：让缓冲区排队等着被用）。
// 调用这个方法后，缓冲区的“所有权”就交给缓冲区管理器了——之后调用方就不能再碰这个缓冲区了，再访问就无效了。
// 调用这个方法之前，必须先给“当前缓冲区管理器对应的ScanRange（扫描任务）”加锁（保证操作安全，避免乱改）。
// 如果这个ScanRange（扫描任务）已经被取消了，那这个缓冲区就不会放进ready_buffer_队列了，而是直接被清理掉（释放内存）。

void EnqueueReadyBuffer(const std::unique_lock<std::mutex>& scan_range_lock,
    std::unique_ptr<BufferDescriptor> buffer);
  void EnqueueReadyBuffer(const std::unique_lock<std::mutex>& scan_range_lock,
      std::unique_ptr<BufferDescriptor> buffer);

  /// Allocates up to 'max_bytes' buffers and adds it to 'buffers'.
  /// Called from ScanRange::AllocateBuffersForRange
  ///
  /// The buffer sizes are chosen based on 'scan_range_->len()'. 'max_bytes' must be >=
  /// 'min_buffer_size' so that at least one buffer can be allocated. The caller
  /// must ensure that 'bp_client' has at least 'max_bytes' unused reservation.
  /// Returns ok if the buffers were successfully allocated.
  Status AllocateBuffersForRange(
      BufferPool::ClientHandle* bp_client, int64_t max_bytes,
      std::vector<std::unique_ptr<BufferDescriptor>>& buffers, int64_t min_buffer_size,
      int64_t max_buffer_size);

  /// Cleans up a buffer that is not being recycled or returned by client.
  /// The caller must hold 'scan_range_->lock_' via 'scan_range_lock'.
  /// This function may acquire 'scan_range_->file_reader_->lock()'
  void CleanUpBuffer(const std::unique_lock<std::mutex>& scan_range_lock,
      const std::unique_ptr<BufferDescriptor> buffer);

  /// Same as CleanUpBuffer() except cleans up multiple buffers. Caller must
  /// hold 'scan_range_->lock_' via 'scan_range_lock'.
  void CleanUpBuffers(const std::unique_lock<std::mutex>& scan_range_lock,
      std::vector<std::unique_ptr<BufferDescriptor>>&& buffers);

  /// Clean up all buffers in 'unused_iomgr_buffers_'. Only valid to call when the scan
  /// range is cancelled or at eos. The caller must hold 'scan_range_->lock_' via
  /// 'scan_range_lock'.
  void CleanUpUnusedBuffers(const std::unique_lock<std::mutex>& scan_range_lock);

  /// Clean up all buffers in 'ready_buffer_'. Caller must hold 'scan_range_->lock_' via
  /// 'scan_range_lock'. Only valid to call when scan range using this buffer manager
  /// is cancelled.
  void CleanUpReadyBuffers(const std::unique_lock<std::mutex>& scan_range_lock);

  std::string DebugString() const {
    std::stringstream ss;
    ss << " buffer_queue=" << ready_buffers_.size()
       << " num_buffers_in_readers=" << num_buffers_in_reader_.Load()
       << " unused_iomgr_buffers=" << unused_iomgr_buffers_.size()
       << " unused_iomgr_buffer_bytes=" << unused_iomgr_buffer_bytes_;
    return ss.str();
  }

  /// Remove the first buffer from 'ready_buffer_' and assign it to '*buffer'.
  /// Returns 'false', if 'ready_buffer_' is empty and '*buffer' cannot be assigned,
  /// 'false' otherwise.
  /// 'scan_range_->lock_' needs to be held before invoking this method.
  bool PopFirstReadyBuffer(const std::unique_lock<std::mutex>& scan_range_lock,
      std::unique_ptr<BufferDescriptor>* buffer);
  /*
  这个 `Validate` 函数的核心作用是 **“给缓冲区管理器做‘健康检查’”**——它会根据绑定的扫描任务（`ScanRange`）的当前状态（比如是否已取消、是否已读完所有数据），检查缓冲区管理器的内部状态（比如就绪队列里有没有缓冲、未使用缓冲区的字节数对不对）是否符合预期规则。如果发现状态错乱（比如“任务已取消但还有缓冲没清理”），就打日志报错并返回“无效”，反之返回“有效”。

下面用通俗的话拆解它的实现思路，一步一步讲清楚“查什么、怎么查、为什么要查”：


### 第一步：先做“锁检查”，确保操作安全
函数第一行 `DCHECK(scan_range_->is_locked(scan_range_lock))` 是个“安全门禁”：  
它强制要求调用这个函数前，必须先给 `scan_range_`（当前绑定的扫描任务）加锁（通过 `scan_range_lock` 参数传入）。  
为什么要这样？因为缓冲区管理器的状态（比如 `ready_buffers_`、`unused_iomgr_buffers_`）可能被多个线程修改（比如磁盘线程加缓冲、客户端线程拿缓冲），加锁能防止“检查到一半，状态突然被改了”，保证检查结果是准确的。


### 第二步：先“抄录”扫描任务的关键状态，方便后续对比
接下来先把 `scan_range_` 的3个核心状态存到局部变量里，避免后续反复访问 `scan_range_`（相当于“抄一份快照”，提高效率）：
- `range_cancelled`：扫描任务是否已被取消（比如用户中途停掉任务）；
- `eosr_queued`：扫描任务是否已“读完所有数据”（`eosr` 是“扫描结束”的意思，即所有数据都已通过缓冲区返回）；
- `blocked_on_buffer`：扫描任务是否因“缺缓冲区”而阻塞（比如没空闲缓冲，没法继续读数据）。


### 第三步：开始“逐项检查”，每一项对应一个“状态规则”
函数的核心是4个“检查项”，每个检查项对应一个“正常逻辑下必须满足的规则”，只要有一项不满足，就返回 `false`（表示状态无效）。


#### 检查项1：“任务已取消” → “就绪队列必须空”
**规则**：如果扫描任务已经被取消（`range_cancelled` 为真），那么 `ready_buffers_`（存“已读好数据、等着用”的缓冲队列）必须是空的。  
**代码逻辑**：`if (range_cancelled && !ready_buffers_.empty())` → 要是“已取消但队列非空”，就打错误日志（比如“已取消的任务不该有排队的缓冲”），返回 `false`。  
**为什么要查**：任务取消后，所有缓冲都该被清理（不能再用了），如果就绪队列还有缓冲，说明清理逻辑有bug，可能导致内存泄漏或错乱。


#### 检查项2：“未使用缓冲的字节数” → “记录值必须等于实际值”
**规则**：`unused_iomgr_buffer_bytes_`（记录“未使用缓冲总字节数”的变量）必须等于 `unused_iomgr_buffers_`（存“空缓冲”的列表）里所有缓冲的字节数总和。  
**代码逻辑**：
1. 先遍历 `unused_iomgr_buffers_`，把每个缓冲的 `buffer_len()`（缓冲大小）加起来，算一个“实际总字节数”；
2. 对比“实际总字节数”和 `unused_iomgr_buffer_bytes_`：如果不相等，就打日志（比如“记录的字节数不对，实际xxx vs 预期xxx”），返回 `false`。  
**为什么要查**：`unused_iomgr_buffer_bytes_` 是个“快捷记录值”，方便快速获取总大小（不用每次都遍历列表）。如果它和实际总和对不上，说明之前修改这个值的逻辑有bug（比如加缓冲时没更该值、删缓冲时没减该值），后续用这个值做判断（比如“还剩多少内存”）会出错。


#### 检查项3：“任务已结束（取消或读完）” → “未使用缓冲列表必须空”
**规则**：如果扫描任务已经“结束”（要么被取消，要么已读完所有数据，即 `is_finished = range_cancelled || eosr_queued` 为真），那么 `unused_iomgr_buffers_`（空缓冲列表）必须是空的。  
**代码逻辑**：`if (is_finished && !unused_iomgr_buffers_.empty())` → 要是“已结束但列表非空”，就打日志（比如“任务结束了还拿着多余的缓冲”），返回 `false`。  
**为什么要查**：任务结束后，所有未使用的缓冲都该被释放（还给内存池），如果还有剩，就是“内存泄漏”——占着内存不用，浪费资源。


#### 检查项4：“任务没结束但缺缓冲阻塞” → “未使用缓冲列表必须空”
**规则**：如果扫描任务“没结束”（还在读取中），且因“缺缓冲”阻塞（`blocked_on_buffer` 为真），那么 `unused_iomgr_buffers_` 必须是空的。  
**代码逻辑**：`if (!is_finished && blocked_on_buffer && !unused_iomgr_buffers_.empty())` → 要是“没结束、阻塞了，但有空缓冲”，就打日志（比如“有缓冲还阻塞，这不正常”），返回 `false`。  
**为什么要查**：“阻塞”的原因是“没缓冲可用”，如果此时还有空缓冲，说明阻塞逻辑有bug（比如没正确把空缓冲分配出去），导致任务“明明有资源却闲着”，属于逻辑矛盾。


### 第四步：所有检查通过 → 返回“有效”
如果4个检查项都满足，说明缓冲区管理器的状态完全符合正常逻辑，返回 `true`（表示状态有效）。


### 总结：实现思路的核心
这个函数本质是一个“逻辑校验器”，它的设计思路可以概括为：  
1. **先保障检查环境安全**（加锁检查）；  
2. **再获取对比基准**（抄录扫描任务的状态）；  
3. **最后逐项验证“状态一致性”**（每个检查项对应一个“正常逻辑规则”，确保缓冲区状态和扫描任务状态匹配）；  
4. **发现异常就报错**（帮助开发者定位bug，比如内存泄漏、状态错乱）。  

它不修改任何状态，只做“只读检查”，是一种典型的“防御性编程”实现——通过定期或关键节点调用这个函数，能及时发现缓冲区管理中的逻辑漏洞，避免问题扩散。
  */
  /// Validates the buffer state. Validation is done based on the state of
  /// ScanRange using this manager.
  /// 'scan_range_->lock_' needs to be held via 'scan_range_lock' before invoking
  /// this method.
  bool Validate(const std::unique_lock<std::mutex>& scan_range_lock);

  void set_cached_buffer() {
    buffer_tag_ = BufferTag::CACHED_BUFFER;
  }

  void set_client_buffer() {
    buffer_tag_ = BufferTag::CLIENT_BUFFER;
  }

  void set_internal_buffer() {
    buffer_tag_ = BufferTag::INTERNAL_BUFFER;
  }

  bool is_cached() const {
    return buffer_tag_ == BufferTag::CACHED_BUFFER;
  }

  bool is_client_buffer() const {
    return buffer_tag_ == BufferTag::CLIENT_BUFFER;
  }

  bool is_internal_buffer() const {
    return buffer_tag_ == BufferTag::INTERNAL_BUFFER;
  }

  BufferTag buffer_tag() const { return buffer_tag_; }

  bool is_readybuffer_empty() const { return ready_buffers_.empty(); }

  int num_buffers_in_reader() const { return num_buffers_in_reader_.Load(); }

  void add_buffers_in_reader(int inc) { num_buffers_in_reader_.Add(inc); }

  void add_iomgr_buffer_cumulative_bytes_used(int inc) {
    iomgr_buffer_cumulative_bytes_used_ += inc;
  }

 private:

  /// Scan range that uses this buffer manager.
  ScanRange* const scan_range_;

  /// The number of buffers that have been returned to a client via GetNext() that have
  /// not yet been returned.
  AtomicInt32 num_buffers_in_reader_{0};

  /// Buffers to read into, used if the 'buffer_tag_' is INTERNAL_BUFFER.
  /// These are initially populated when the client calls AllocateBuffersForRange()
  /// and are used to read scanned data into. Buffers are taken from this vector for
  /// every read and added back after use.
  std::vector<std::unique_ptr<BufferDescriptor>> unused_iomgr_buffers_;

  /// Total number of bytes of buffers in 'unused_iomgr_buffers_'.
  int64_t unused_iomgr_buffer_bytes_ = 0;

  /// Cumulative bytes of I/O mgr buffers taken from 'unused_iomgr_buffers_' by DoRead().
  /// Used to infer how many bytes of buffers need to be held onto to read the rest of
  /// the scan range.
  int64_t iomgr_buffer_cumulative_bytes_used_ = 0;

  /// IO buffers that are queued for this scan range. When Cancel() is called
  /// this is drained by the cancelling thread. I.e. this is always empty if
  /// 'cancel_status_' is not OK.
  std::deque<std::unique_ptr<BufferDescriptor>> ready_buffers_;

  /// Tag that represents the kind of buffers used to read scan data.
  /// Please see comments for enum 'BufferTag' for more details.
  BufferTag buffer_tag_;




/*小步骤 2：按 “剩余数据” 和 “最大缓冲区大小”，初步定一个缓冲区大小
这一步是为了 “尽量用大缓冲区（减少缓冲区数量，提高效率），但不超过单个缓冲区的上限”：
如果 “剩余数据” 比 “最大缓冲区大小” 还多（比如剩 744 字节，最大缓冲区 256 字节）：
那下一个缓冲区就用最大的 256 字节（能多装一点是一点，减少循环次数）；
如果 “剩余数据” 比 “最大缓冲区大小” 少（比如剩 128 字节，最大缓冲区 256 字节）：
那先按 “剩余数据” 来，但要满足两个小条件：
不能小于 “最小缓冲区大小”（比如剩 32 字节，但最小要 64 字节，那就按 64 字节算）；
大小要改成 “2 的幂”（比如剩 123 字节，2 的幂里比 123 大的最小数是 128，那就按 128 字节算）
—— 这里用 BitUtil::RoundUpToPowerOfTwo 工具函数实现，本质是方便内存对齐
（操作系统 / 内存管理器对 2 的幂大小的内存块处理更高效）。

其实这两种情况的本质，是在 “总内存不够装所有数据” 时的两种应对策略：
情况 A（已有部分缓冲区）：不贪多，有多少用多少，先分配已有的缓冲区去读数据，后续不够再想办法（比如后续再申请内存）；
情况 B（还没任何缓冲区）：必须兜底，哪怕把缓冲区改小（改成 “总内存上限以内的最大 2 的幂”），也要至少分配一个，不然扫描任务根本没法启动。
*/

  /// Choose buffer sizes to read scan range's data of length 'scan_range_len'.
  /// 'min_buffer_size' and 'max_buffer_size' are minimum and maximum buffer size
  /// that can be allocated. Additionally, cumulative sum of buffer sizes should not
  /// exceed 'max_bytes'.
  static std::vector<int64_t> ChooseBufferSizes(int64_t scan_range_len,
      int64_t max_bytes, int64_t min_buffer_size, int64_t max_buffer_size) {
    DCHECK_GE(max_bytes, min_buffer_size);
    std::vector<int64_t> buffer_sizes;
    int64_t bytes_allocated = 0;
    while (bytes_allocated < scan_range_len) {
      int64_t bytes_remaining = scan_range_len - bytes_allocated;
      // Either allocate a max-sized buffer or a smaller buffer to fit the rest of the
      // range.
      int64_t next_buffer_size;
      if (bytes_remaining >= max_buffer_size) {
        next_buffer_size = max_buffer_size;
      } else {
        next_buffer_size =
            std::max(min_buffer_size, BitUtil::RoundUpToPowerOfTwo(bytes_remaining));
      }
      if (next_buffer_size + bytes_allocated > max_bytes) {
        // Can't allocate the desired buffer size. Make sure to allocate at least one
        // buffer.
        if (bytes_allocated > 0) break;
        next_buffer_size = BitUtil::RoundDownToPowerOfTwo(max_bytes);
      }
      DCHECK(BitUtil::IsPowerOf2(next_buffer_size)) << next_buffer_size;
      buffer_sizes.push_back(next_buffer_size);
      bytes_allocated += next_buffer_size;
    }
    return buffer_sizes;
  }
};
}
}
