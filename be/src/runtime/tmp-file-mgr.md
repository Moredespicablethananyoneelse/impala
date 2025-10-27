
```cpp

/// TmpFileMgr 提供了一个抽象层，用于管理文件系统上的临时（也称为草稿）文件及其 I/O 操作。
/// TmpFileMgr 管理跨多个设备的多个草稿目录，这些目录通过 --scratch_dirs 选项配置。
/// TmpFileMgr 管理对草稿文件的 I/O 操作，以抽象化文件分配的细节并从某些 I/O 错误中恢复。I/O 操作通过 DiskIoMgr 进行。
/// 如果通过 --disk_spill_encryption 命令行标志启用，TmpFileMgr 会加密写入磁盘的数据。
///
/// TmpFileGroup 管理跨多个设备的草稿空间。要写入草稿空间，首先创建一个 TmpFileGroup，然后调用 TmpFileGroup::Write() 以异步方式将内存缓冲区写入其中一个草稿文件。
/// TmpFileGroup::Write() 返回一个 TmpWriteHandle，用于标识该写入操作。
/// 当异步写入完成时，通过回调通知调用者，之后调用者可以使用 TmpWriteHandle 读取回数据。
///
/// 每个 TmpWriteHandle 由草稿文件中的数据范围支持。第一次调用 Write() 将在配置的临时设备上为 TmpFileGroup 创建具有唯一文件名的文件。
/// 每个设备最多使用一个目录（除非用于测试覆盖）。如果遇到写入错误，可以用不同的文件范围替换 TmpWriteHandle 的文件范围，并将数据写入不同的磁盘。
///
/// 空闲空间管理：
/// 空闲空间在 TmpFileGroup 内管理：一旦 TmpWriteHandle 被销毁，支持它的文件范围可以被回收以用于不同的 TmpWriteHandle。
/// 草稿文件范围按大小类分组，每个大小类对应 2 的幂字节数。每个大小类的空闲文件范围单独管理（即没有分割或合并范围）。
///
/// 资源管理：
/// TmpFileMgr 提供了一些基本的本地磁盘空间消耗管理支持。
/// 可以为 TmpFileGroup 创建总字节数限制，跨越所有文件。超出限制的写入将失败并返回错误状态。
/// TmpFileBufferPool 提供能力，让写入范围在单独的线程中等待本地缓冲区空间，用于溢出到远程文件系统，该池的目的是处理有限本地空间的缓冲文件分配竞争。
///
/// 锁机制：
/// 在溢出过程中，可能获取多个锁，为避免死锁，获取锁的顺序必须从较低编号的锁到较高编号的锁：
/// 1. BufferPool::Client::lock_
/// 2. BufferPool::Page::buffer_lock
/// 3. TmpFileGroup::lock_
/// 4. TmpFileBufferPool::lock_
/// 5. TmpWriteHandle::write_state_lock_
/// 6. RequestContext::lock_
/// 7. ScanRange::lock_
/// 8. DiskFile::physical_file_lock_
/// 特别地，在溢出到远程文件系统期间，可能需要同时获取两种 DiskFile 的锁，获取 DiskFile 锁的顺序必须从本地 DiskFile 到远程 DiskFile。
///
/// 除了上述锁之外，在溢出过程中可以使用终端锁，即在持有此锁时不应获取其他锁：
/// TmpFileBufferPool::tmp_files_avail_pool_lock_
/// TmpFileGroup::tmp_files_remote_ptrs_lock_
/// DiskQueue::lock_
/// DiskFile::status_lock_
///
/// TODO: IMPALA-4683: 我们可以实现更智能的故障处理，例如临时将显示 I/O 错误的设备列入黑名单。
/*
在[buffer-pool-internal](./bufferpool/buffer-pool-internal.h)中定义了其他锁的顺序。
/// Lock Ordering
/// =============
/// The lock acquisition order is:
/// 1. BufferPool::Client::lock_
/// 2. BufferPool::BufferAllocator::FreeBufferArena::lock_. If multiple arena locks are acquired, must be acquired in
///    ascending order.
/// 3. BufferPool::Page::buffer_lock
///
/// If a reference to a Page is acquired through a page list, the Page* reference only
/// remains valid so long as list's lock is held.
///
上面提到的两个锁的顺序，有部分锁是相同的。所以说，从整体上看，加锁顺序是个树形结构？

*/
/*
### Impala 配置选项解释

以下是您提供的这些 `DEFINE_` 语句的详细解释。这些选项主要用于 Impala 的临时文件管理（scratch space）和磁盘溢出（spill-to-disk）机制，用于处理查询执行过程中内存不足时将数据临时写入磁盘（或远程存储）的行为。Impala 使用 GFlags 库来定义这些命令行参数，默认值和描述基于 Impala 的配置系统。

我将逐一解释每个选项，包括：
- **选项名称**：变量名及其类型。
- **默认值**：预设值。
- **作用**：简要描述其功能。
- **详细说明**：基于描述的翻译和扩展解释。
- **适用场景**：何时调整此选项。

#### 1. `DEFINE_bool(disk_spill_encryption, true, "...")`
- **类型**：布尔值（bool），启用/禁用。
- **默认值**：`true`（启用）。
- **作用**：控制查询过程中溢出到磁盘的数据是否进行加密和完整性检查。
- **详细说明**：如果设置为 `true`，Impala 会对所有溢出到磁盘的数据进行加密（使用 AES 等算法），并在读取时进行完整性校验（例如哈希验证），以防止数据篡改或泄露。这增加了安全性，但会消耗额外的 CPU 和内存资源。加密密钥通常为每个写入操作动态生成。
- **适用场景**：在生产环境中处理敏感数据时启用；测试环境中可禁用以节省性能。

#### 2. `DEFINE_string(disk_spill_compression_codec, "", "...")`
- **类型**：字符串（string），指定压缩编解码器。
- **默认值**：空字符串（""，表示不压缩）。
- **作用**：指定溢出到磁盘前对数据进行压缩的算法。
- **详细说明**：这是一个高级选项。如果设置（如 "lz4"、"zstd" 或 "zstd:6"，其中数字表示压缩级别），Impala 会使用指定的压缩算法（如 LZ4、Zstd）压缩数据，以减少磁盘使用量（可显著降低 scratch 空间消耗）。语法与查询选项 `COMPRESSION_CODEC` 相同。但启用此选项时，必须同时启用 `--disk_spill_punch_holes`（打孔功能），以便在文件稀疏化时回收空间。压缩会增加 CPU 和内存开销。
- **适用场景**：磁盘空间紧张时使用；选择低 CPU 压缩如 LZ4 用于高吞吐场景，Zstd 用于平衡压缩率。

#### 3. `DEFINE_int64(disk_spill_compression_buffer_limit_bytes, 512L * 1024L * 1024L, "...")`
- **类型**：64 位整数（int64），单位为字节。
- **默认值**：512 MB（512 * 1024 * 1024 字节）。
- **作用**：限制所有查询中用于磁盘溢出压缩的缓冲区总字节数。
- **详细说明**：这是一个高级选项，控制压缩缓冲区的全局上限。如果超过此限制，部分数据将以未压缩形式直接溢出到磁盘，从而避免内存耗尽。这有助于在多查询并发时管理内存，但可能导致磁盘使用增加。
- **适用场景**：内存有限的集群中调整上限；如果压缩频繁失败，可增加到 1GB 或更高。

#### 4. `DEFINE_bool(disk_spill_punch_holes, false, "...")`
- **类型**：布尔值（bool），启用/禁用。
- **默认值**：`false`（禁用）。
- **作用**：改变 scratch 目录中创建文件的空闲空间管理策略，使用“打孔”（punch holes）机制释放未用空间。
- **详细说明**：这是一个高级选项。如果启用，当文件中的空间未使用时，Impala 会调用文件系统 API（如 `fallocate` 或 `ftruncate`）在文件中“打孔”，将稀疏区域标记为空闲，从而减少实际磁盘占用。特别与压缩结合使用时效果显著（压缩后数据更稀疏）。但要求 `--scratch_dirs` 中的目录文件系统支持打孔（如 ext4、XFS），否则无效。
- **适用场景**：启用压缩后使用，以优化磁盘利用率；不支持打孔的文件系统（如某些 NFS）需保持禁用。

#### 5. `DEFINE_string(scratch_dirs, "/tmp", "...")`
- **类型**：字符串（string），逗号分隔的目录列表。
- **默认值**：`"/tmp"`（使用 /tmp 目录）。
- **作用**：指定可写的 scratch（临时）目录列表，用于存储溢出数据。
- **详细说明**：这是一个逗号分隔的列表，每个目录可指定路径、字节限制（可选）和优先级（可选）。格式示例：
  - 路径 + 限制：`/dir1:10G,/dir2:5GB,/dir3`（/dir1 最多 10GB，/dir2 5GB，/dir3 无限制）。
  - 路径 + 限制 + 优先级：`/dir1:10G:0,/dir2:5GB:1,/dir3::1`（优先级数字越小优先级越高，低优先级目录用于轮询分配）。
  优先级基于 spilling 会先填充高优先级目录，然后轮询低优先级。限制防止单个目录过载。
- **适用场景**：多磁盘环境中指定多个目录以负载均衡；添加限制避免单盘满载。

#### 6. `DEFINE_bool(allow_multiple_scratch_dirs_per_device, true, "...")`
- **类型**：布尔值（bool），启用/禁用。
- **默认值**：`true`（允许）。
- **作用**：控制同一设备上是否允许多个 scratch 目录。
- **详细说明**：如果设置为 `false`，且 `--scratch_dirs` 中同一设备（如同一硬盘）有多个目录，则仅使用第一个可写目录。这简化管理，但可能降低并行性。
- **适用场景**：在单一设备上禁用多目录以避免 I/O 竞争；默认允许以充分利用空间。

#### 7. `DEFINE_string(remote_tmp_file_size, "16M", "...")`
- **类型**：字符串（string），文件大小（如 "16M"）。
- **默认值**：16 MB。
- **作用**：指定远程临时文件的默认大小（用于 HDFS/S3 等远程 scratch）。
- **详细说明**：上限 256 MB，下限为块大小。大小应为 2 的幂，并是块大小的整数倍。用于远程 spilling 时创建临时文件的大小。
- **适用场景**：远程存储（如 S3）中调整以匹配网络/存储优化；增大可减少文件数，但增加单文件开销。

#### 8. `DEFINE_string(remote_tmp_file_block_size, "1M", "...")`
- **类型**：字符串（string），块大小（如 "1M"）。
- **默认值**：1 MB。
- **作用**：指定远程临时文件上传/下载的块大小。
- **详细说明**：大小应为 2 的幂，且小于远程临时文件大小。用于分块传输，提高并行性和效率。
- **适用场景**：网络带宽高时增大块大小以减少元数据开销；低带宽时缩小。

#### 9. `DEFINE_string(remote_read_memory_buffer_size, "1G", "...")`
- **类型**：字符串（string），缓冲区大小（如 "1G"）。
- **默认值**：1 GB。
- **作用**：指定远程临时文件的读取内存缓冲区最大大小。
- **详细说明**：仅在 `--remote_batch_read` 为 `true` 时有效。用于批量读取远程文件时的内存分配上限，防止内存爆炸。
- **适用场景**：启用批量读取时，根据可用内存调整；集群内存大可增大到 2GB。

#### 10. `DEFINE_bool(remote_tmp_files_avail_pool_lifo, false, "...")`
- **类型**：布尔值（bool），选择驱逐算法。
- **默认值**：`false`（使用 FIFO）。
- **作用**：控制远程 spilling 时本地缓冲文件驱逐算法（LIFO vs FIFO）。

#### 11. `DEFINE_int32(wait_for_spill_buffer_timeout_s, 60, "...")`
- **类型**：32 位整数（int32），单位为秒。
- **默认值**：60 秒。
- **作用**：指定等待 spilling 缓冲区超时时间。
- **详细说明**：如果 spilling 操作在池中获取缓冲区超时，则操作失败并返回错误。用于防止无限等待。
- **适用场景**：网络/磁盘慢时增大超时；资源紧张时减小以快速失败。

#### 12. `DEFINE_bool(remote_batch_read, false, "...")`
- **类型**：布尔值（bool），启用/禁用。
- **默认值**：`false`（禁用）。
- **作用**：启用远程临时文件的批量读取。
- **详细说明**：允许在缓冲池尝试pin one page固定一页时异步读取整个块，提高 I/O 效率。但需配合 `--remote_read_memory_buffer_size` 使用。
- **适用场景**：远程存储 I/O 瓶颈时启用，以重叠读取和处理。

#### 13. `DEFINE_bool(remote_scratch_cleanup_on_start_stop, true, "...")`
- **类型**：布尔值（bool），启用/禁用。
- **默认值**：`true`（启用）。
- **作用**：控制 Impala 守护进程启动/关闭时是否清理远程 scratch 目录。
- **详细说明**：如果启用，在启动和关闭时清理主机级远程目录中的残留文件（假设每个主机只有一个 Impala 进程）。多进程主机需禁用以避免误删。
- **适用场景**：单实例部署时启用自动清理；多实例时手动管理。

这些选项主要影响查询的性能、安全性和资源利用率。在生产环境中，建议结合监控指标（如磁盘使用、CPU 负载）调优。如果需要更深入的 Impala 文档参考，可查阅官方手册（Cloudera 或 Apache Impala）。如果有特定选项的进一步疑问，请提供更多细节！
*/
class TmpFileMgr {
 public:
  /// DeviceId 是 TmpFileMgr 管理的临时设备的内部唯一标识符。DeviceId 在 [0, num tmp devices) 范围内任意分配。
  /// 需要公开以供 TmpFileMgrTest 使用。
  typedef int DeviceId;

  /// 与 io::WriteRange::WriteDoneCallback 相同的 typedef。
  typedef std::function<void(const Status&)> WriteDoneCallback;

  /// 远程临时目录控制参数的配置。
  /// 该结构体由 TmpFileMgr 使用，并与 TmpFileMgr 具有相同的生命周期。
  struct TmpDirRemoteCtrl {
    /// 计算远程溢出允许的最大读取缓冲区字节数。
    int64_t CalcMaxReadBufferBytes();

    /// 设置远程文件读取缓冲区参数的辅助函数。
    Status SetUpReadBufferParams() WARN_UNUSED_RESULT;

    /// 本地缓冲目录的高水位标记指标。
    AtomicInt64 local_buff_dir_bytes_high_water_mark_{0};

    /// 远程临时文件的默认大小。
    int64_t remote_tmp_file_size_;

    /// 远程临时文件读取缓冲区块的默认大小。
    int64_t read_buffer_block_size_;

    /// 远程文件每个文件的读取缓冲区块数，它来自 remote_tmp_file_size_/read_buffer_block_size_。
    int num_read_buffer_blocks_per_file_;

    /// 远程临时文件的默认块大小。该块用作上传和获取远程临时文件时的缓冲区。
    int64_t remote_tmp_block_size_;

    /// 远程溢出所有读取缓冲区的最大总大小。
    int64_t max_read_buffer_size_;

    /// 指定将临时文件入队到池的模式。
    /// 如果为 true，则文件将被放置在池中首先被弹出。
    /// 如果为 false，则文件将被放置在池的末尾。
    bool remote_tmp_files_avail_pool_lifo_;

    /// 表示是否为远程临时文件启用批量读取。
    bool remote_batch_read_enabled_;

    /// TmpFileMgr 管理的临时文件缓冲池，仅在注册远程草稿空间时激活。因此，如果 TmpFileMgr::HasRemoteDir() 为 true，则 tmp_file_pool_ 非空。否则，它为空。
    std::unique_ptr<TmpFileBufferPool> tmp_file_pool_;

    /// 包含 TmpFileMgr 创建的线程的线程组。
    ThreadGroup tmp_file_mgr_thread_group_;

    /// 等待缓冲区（us）的超时持续时间。
    int64_t wait_for_spill_buffer_timeout_us_;
  };

  TmpFileMgr();

  ~TmpFileMgr();

  /// 创建配置的临时目录。如果每个磁盘指定多个目录，则仅创建一个并使用。必须在 DiskInfo::Init() 之后调用。
  Status Init(MetricGroup* metrics) WARN_UNUSED_RESULT;

  /// 自定义初始化 - 使用提供的目录列表进行初始化。
  /// 如果 one_dir_per_device 为 true，则每个设备仅使用一个临时目录。
  /// 此接口用于测试目的。'tmp_dir_specifiers' 使用命令行语法，即 <path>[:<limit>]。第一个变体接受逗号分隔的列表，第二个接受向量。
  Status InitCustom(const std::string& tmp_dirs_spec, bool one_dir_per_device,
      const std::string& compression_codec, bool punch_holes,
      MetricGroup* metrics) WARN_UNUSED_RESULT;
  Status InitCustom(const std::vector<std::string>& tmp_dir_specifiers,
      bool one_dir_per_device, const std::string& compression_codec, bool punch_holes,
      MetricGroup* metrics) WARN_UNUSED_RESULT;
  // 为异步缓冲文件预留创建 TmpFile 缓冲池线程。
  Status CreateTmpFileBufferPoolThread(MetricGroup* metrics) WARN_UNUSED_RESULT;

  // 在关闭期间清理临时目录，如果需要。
  void CleanupAtShutdown();

  /// 尝试从本地缓冲目录预留缓冲文件空间。
  /// 如果 quick_return 为 true，则如果没有可用空间，函数不会等待。
  Status ReserveLocalBufferSpace(bool quick_return) WARN_UNUSED_RESULT;

  /// 返回设备的草稿目录路径。
  std::string GetTmpDirPath(DeviceId device_id) const;

  /// 返回远程临时文件大小。
  int64_t GetRemoteTmpFileSize() const {
    return tmp_dirs_remote_ctrl_.remote_tmp_file_size_;
  }

  /// 返回远程临时文件的读取缓冲区块大小。
  int64_t GetReadBufferBlockSize() const {
    return tmp_dirs_remote_ctrl_.read_buffer_block_size_;
  }

  /// 返回远程临时文件每个文件的读取缓冲区块数。
  int GetNumReadBuffersPerFile() const {
    return tmp_dirs_remote_ctrl_.num_read_buffer_blocks_per_file_;
  }

  /// 返回远程溢出所有读取缓冲区块的最大总大小。
  int64_t GetRemoteMaxTotalReadBufferSize() const {
    return tmp_dirs_remote_ctrl_.max_read_buffer_size_;
  }

  /// 返回远程临时块大小。
  int64_t GetRemoteTmpBlockSize() const {
    return tmp_dirs_remote_ctrl_.remote_tmp_block_size_;
  }

  /// 返回等待溢出缓冲区的超时持续时间。
  int64_t GetSpillBufferWaitTimeout() const {
    return tmp_dirs_remote_ctrl_.wait_for_spill_buffer_timeout_us_;
  }

  /// 返回是否以 LIFO 方式返回远程临时文件到池。
  /// 如果返回 false，则由 FIFO 设置。
  bool GetRemoteTmpFileBufferPoolLifo() {
    return tmp_dirs_remote_ctrl_.remote_tmp_files_avail_pool_lifo_;
  }

  // 返回是否为远程临时文件启用批量读取。
  bool IsRemoteBatchReadingEnabled() {
    return tmp_dirs_remote_ctrl_.remote_batch_read_enabled_;
  }

  /// 返回远程溢出的本地缓冲目录。
  TmpDir* GetLocalBufferDir() const;

  /// 本地文件系统中活动临时目录的设备总数。
  /// 每个设备有一个临时目录。
  int NumActiveTmpDevicesLocal();

  /// 活动临时目录的设备总数。每个设备有一个临时目录。
  int NumActiveTmpDevices();

  /// 返回所有正在积极使用的临时设备 ID 的向量。
  /// 即那些未被列入黑名单的设备。
  std::vector<DeviceId> ActiveTmpDevices();

  /// 将写入范围添加到 TmpFileBufferPool 以等待写入范围写入的 TmpFile 的缓冲区预留。池将在预留完成后将范围发送到磁盘队列。
  /// 如果缓冲区空间已预留，则返回错误，调用者应自行将范围入队到磁盘队列。
  Status AsyncWriteRange(io::WriteRange* write_range, TmpFile* file);

  /// 将虚拟临时文件入队到 TmpFileBufferPool 的辅助函数。
  void EnqueueTmpFilesPoolDummyFile();

  /// 当临时文件的本地缓冲区准备好被删除以释放缓冲区空间时，将临时文件入队到池。
  /// 如果 front 设置为 true，则文件被入队到池的前部并首先被弹出。
  void EnqueueTmpFilesPool(std::shared_ptr<TmpFile>& tmp_file, bool front);

  /// 从池中出队临时文件。成功从池中出队文件的调用者有权删除文件本地缓冲区并创建其缓冲区。
  /// 该函数由 TryEvictFile() 调用，以从池中获取空间，如果本地缓冲目录达到字节限制。如果池中没有文件，调用者需要等待直到文件被入队（通常在文件上传后）。
  /// 如果由于磁盘队列堵塞或网络缓慢而没有文件上传，可能需要长时间等待，因此不推荐在调用该函数时持有独占锁。
  /// 如果 quick_return 为 true，则如果池中没有文件则不会等待。否则，将等待直到文件出队。
  Status DequeueTmpFilesPool(std::shared_ptr<TmpFile>* tmp_file, bool quick_return);

  /// 该函数释放临时文件中所有读取缓冲区的内存。
  /// 调用者需要持有缓冲文件的唯一锁。
  void ReleaseTmpFileReadBuffer(
      const std::unique_lock<boost::shared_mutex>& lock, TmpFile* tmp_file);

  /// 尝试删除 TmpFile 的缓冲区以为其他缓冲区腾出空间。
  /// 如果在删除缓冲区期间发生错误，可能返回错误状态。
  Status TryEvictFile(TmpFile* tmp_file);

  /// 如果在 TmpFileMgr 中注册了远程草稿空间。
  bool HasRemoteDir() { return tmp_dirs_remote_ != nullptr; }

  /// 返回溢出到 S3 的默认 S3 选项。
  const vector<std::pair<string, string>>* s3a_options() { return &s3a_options_; }

  MemTracker* compressed_buffer_tracker() const {
    return compressed_buffer_tracker_.get();
  }

  /// 溢出到磁盘使用的溢出到磁盘压缩类型。
  THdfsCompression::type compression_codec() const { return compression_codec_; }
  bool compression_enabled() const {
    return compression_codec_ != THdfsCompression::NONE;
  }
  std::optional<int> compression_level() const { return compression_level_; }
  bool punch_holes() const { return punch_holes_; }

  /// 我们将尝试在草稿文件中打孔的最小孔大小。
  /// 这避免了无效的打孔，其中我们仅在块的一部分打孔而无法回收空间。4kb 是基于 Linux 文件系统通常使用 4kb 或更小的块选择的
  static constexpr int64_t HOLE_PUNCH_BLOCK_SIZE_BYTES = 4096;

 private:
  friend class TmpFile;
  friend class TmpFileRemote;
  friend class TmpFileGroup;
  friend class TmpFileMgrTest;
  friend class TmpDirLocal;
  friend class TmpDirHdfs;
  friend class TmpDirS3;

  /// 返回一个新的 TmpFile 句柄，其路径基于 file_group->unique_id。文件与 'file_group' 关联，文件路径位于指定设备 ID 的（单个）草稿目录内。
  /// 调用者拥有返回的句柄，并负责删除它。文件未创建 - 创建延迟到文件被写入时。
  void NewFile(TmpFileGroup* file_group, DeviceId device_id,
    std::unique_ptr<TmpFile>* new_file);

  /// 删除存储临时文件组临时文件的远程目录。
  void RemoveRemoteDirForQuery(TmpFileGroup* file_group);

  /// 删除存储整个主机临时文件的远程目录。
  /// 在 Impala 守护进程启动期间调用，以清理任何剩余的临时文件。
  /// 假设每个主机仅运行一个 Impala 守护进程。
  void RemoveRemoteDirForHost(const string& dir, hdfsFS hdfs_conn);

  bool initialized_ = false;

  /// 溢出到磁盘使用的溢出到磁盘压缩类型。NONE 表示不使用压缩。
  THdfsCompression::type compression_codec_ = THdfsCompression::NONE;

  /// 用于某些压缩编解码器（即 ZSTD、ZLIB、BZIP2）的压缩级别，其他情况下忽略。
  std::optional<int> compression_level_;

  /// 是否启用打孔。
  bool punch_holes_ = false;

  /// 是否每个设备一个本地草稿目录。
  bool one_dir_per_device_ = false;

  /// 创建的临时目录路径，用于溢出到本地文件系统。
  std::vector<std::unique_ptr<TmpDir>> tmp_dirs_;

  /// 远程目录路径，用于溢出到远程文件系统。
  std::unique_ptr<TmpDir> tmp_dirs_remote_;

  /// 远程临时目录的控制参数。
  TmpDirRemoteCtrl tmp_dirs_remote_ctrl_;

  /// 存储远程临时文件本地缓冲区的目录路径。
  std::unique_ptr<TmpDir> local_buff_dir_;

  /// 溢出到 S3 的默认 S3 选项。
  HdfsFsCache::HdfsConnOptions s3a_options_;

  /// HDFS 连接句柄的本地缓存。
  HdfsFsCache::HdfsFsMap hdfs_conns_;

  /// 用于跟踪压缩缓冲区的内存跟踪器。如果启用磁盘溢出压缩，则在 InitCustom() 中设置
  std::unique_ptr<MemTracker> compressed_buffer_tracker_;

  /// 用于跟踪活动草稿目录的指标。
  IntGauge* num_active_scratch_dirs_metric_ = nullptr;
  SetMetric<std::string>* active_scratch_dirs_metric_ = nullptr;

  /// 用于跟踪草稿空间 HWM 的指标。
  AtomicHighWaterMarkGauge* scratch_bytes_used_metric_ = nullptr;

  /// 用于跟踪读取内存缓冲区 HWM 的指标。
  AtomicHighWaterMarkGauge* scratch_read_memory_buffer_used_metric_ = nullptr;
};

/// 表示一组临时文件 - 每个具有草稿目录的磁盘一个文件。可以通过设置空间分配限制来绑定组的总分配字节数。
/// TmpFileGroup 对象的拥有者负责调用 Close() 方法以删除组中的所有文件。
///
/// TmpFileGroup 和 TmpWriteHandle 的公共方法可以从多个线程并发调用，只要提供不同的 TmpWriteHandle 参数。
class TmpFileGroup {
 public:
  /// 初始化一个新的文件组，它将使用 'tmp_file_mgr' 创建文件，并使用 'io_mgr' 执行 I/O 操作。
  /// 将计数器添加到 'profile' 以跟踪使用的草稿空间。'unique_id' 是一个唯一 ID，用于前缀任何草稿文件名。
  /// 使用相同的 'unique_id' 创建多个 TmpFileGroup 是错误的。
  /// 'bytes_limit' 是总文件空间分配的限制。
  TmpFileGroup(TmpFileMgr* tmp_file_mgr, io::DiskIoMgr* io_mgr, RuntimeProfile* profile,
      const TUniqueId& unique_id, int64_t bytes_limit = -1);

  ~TmpFileGroup();

  /// 异步将 'buffer' 写入此文件组的临时文件。如果有多个草稿文件，这可以写入其中任何一个，并将尝试从一个文件的 I/O 错误中恢复，通过写入不同的文件。
  /// 'buffer' 引用的内存必须在写入完成前保持有效。被调用者可能就地重写 'buffer' 中的数据（例如进行就地加密或压缩）。
  /// 调用者不应在写入完成或取消前修改 'buffer' 中的数据，否则可能写入无效数据到磁盘。
  ///
  /// 写入可能需要一段时间完成。它可能排队在其他 I/O 操作后面。如果启用远程草稿，它可能还需要等待其他查询进展并释放本地缓冲目录中的空间。
  ///
  /// 如果无法分配草稿空间或无法启动写入，则返回错误。否则设置 'handle'，并且 'cb' 将在写入成功完成、不成功或被取消时从不同的线程异步调用。
  /// 如果非空，则 'counters' 中的计数器将更新写入信息。
  ///
  /// 'handle' 必须通过传递给 DestroyWriteHandle() 或 RestoreData() 来销毁。
  /*
    |-------------------------------------------------|
    ---------------------------------------------------
    |                    TmpFileGroup                 |
    ---------------------------------------------------
    |                  TmpWriteHandle                 |
    ---------------------------------------------------
    |          WriteRange     |        ScanRange      |

    1：创建TmpWriteHandle对象。将TmpFileGroup::Write委托给TmpWriteHandle：：Write接口.
    2：TmpWriteHandle保存了用户的cb(callback)和所属的TmpFileGroup。
    3：除了用户指定的cb(callback),TmWriteHandle：：Write也创建了自己用的callback。
       用户的cb会在TmpWriteHandle::Write自己的callback中调用。
    4：TmpFileGroup::Write是异步函数。但是没有像TmpFileGroup：：ReadAsync那样清晰的在名字里体现(这两个函数都用于创建并启动异步操作）。
       补充一句：TmpFileGroup：：Read是同步接口。
    5：TmpFileGroup::Write并没有提供TmpFileGroup这个层面的Wait操作
       TmpFileGroup::ReadAsync提供了TmpFileGroup这个层面的Wait操作：即TmpFileGroup::WaitForAsyncRead(TmpWriteHandle* handle, MemRange buffer...);
       这是因为WriteRange提供的同步机制是回调函数。没有提供条件变量阻塞等待写操作完成。
       ReadRange提供的同步机制是GetNext。ReadRange::GetNext是阻塞接口。利用的是scan-range-buffer,相当于使用了条件变量阻塞等待操作完成。
       [ScanRange和WriteRange](/be/src/runtime/io/scan-range.md)

    6：异步函数的设计：
        6.1：通常对外提供两个函数接口：
          - 启动异步操作。
          - 等待异步操作完成。
        6.2：可对外同步接口
             - 即将6.1步骤的两个函数封装在一个函数中
        6.3：异步实现诗依托的底层函数可分为两种：
            6.3.1：底层函数是同步的：
                如hdfs和本地文件的读/写接口都是同步的。这是为了给用户提供异步接口。需要DiskIOMgr这个中间层提供独立的IO线程完成读/写操作。由DiskIOMgr的IO线程等待读写操作的完成，而用户线程无需等待。从用户线程的角度看，读写操作变成了异步操作。

            6.3.2：底层函数是异步的：
                 TmpFileGroup异步接口依托的ScanRange和WriteRange本身是异步的。

    5：我发现impala函数的一种设计模式：
        5.1:void  BufferPool::Client::DestroyPageInternal(PageHandle* handle,  
              BufferHandle* out_buffer = NULL)
        5.2:Status BufferPool::Pin(ClientHandle* client, PageHandle* handle) 
              WARN_UNUSED_RESULT;
        5.3:void BufferPool::Unpin(ClientHandle* client, PageHandle* handle);
        5.4:void BufferPool::DestroyPage(ClientHandle* client, PageHandle* handle);
        5.5:Status BufferPool::ExtractBuffer(ClientHandle* client, PageHandle*      
            page_handle,BufferHandle* buffer_handle) WARN_UNUSED_RESULT;
        5.6：BufferPool的方法接收他的操作对象ClientHandle和PageHandle。
            BufferPool::Client的方法接收他的操作对象PageHandle。
            TmpFileGroup：：Write也接收他的操作对象TmpWriteHandle。
            因为BufferPool管理很多BufferPool::Client.
            因为BufferPool::Client管理很多Page，
            因为TmpFileGroup管理很多TmpWriteHandle。
            BufferPool作为BufferPool::Client与BufferPool::BufferAllocator的交互的中介。也作为对外（用户）提供的接口（门面模式），如RegisterClient/UnregisterClient/CreatePinnedPage/Pin/UnPin/AllocateBuffer/AllocateUnReserveBuffer/FreeBuffer。
            TmpFileGroup也作为TmpWriteHandle与DiskIOMgr交互的中介。TmpFileGroup也作为提供给用户的接口，如TmpFileGroup::write,TmpFileGroup::Read


  */
  Status Write(MemRange buffer, TmpFileMgr::WriteDoneCallback cb,
      std::unique_ptr<TmpWriteHandle>* handle,
      const BufferPoolClientCounters* counters = nullptr);

  /// 同步从临时文件读取 'handle' 引用的数据到 'buffer'。buffer.len() 必须与 handle->len() 相同。
  /// 仅可在写入成功完成后再调用。不应在异步读取飞行时调用。等价于调用 ReadAsync() 然后 WaitForAsyncRead()。

  Status Read(TmpWriteHandle* handle, MemRange buffer) WARN_UNUSED_RESULT;

  /// 异步从临时文件读取 'handle' 引用的数据到 'buffer'。buffer.len() 必须与 handle->len() 相同。
  /// 仅可在写入成功完成后再调用。在缓冲区数据有效前必须调用 WaitForAsyncRead()。
  /// 不应在异步读取已飞行时调用。
    /*
    1:会根据TmpWriteHandle::write_range_,创建TmpWriteHandle::read_range_.
    2:指定TmpWriteHandle::read_range_使用的buffer，读取的偏移，长度，所在的磁盘，是否使用缓存等等。
    3：提交到DiskIOMgr执行。
    4：TmpFileGrouop::ReadAysnc虽然是异步提交读取请求，但是不需要指定回调函数。TmpFileGroup自己也没有用于同步的条件变量。与之相比，TmpFileGroup：：Write也没有自己的条件变量。

    5：TmpFileGroup::Write异步启动某个TmpWriteHandle::Write写操作后。没有提供同步写操作完成的接口。
    在TmpWriteHandle层面提供了TmpWriteHandle::Wait等待接口
       
  */
  Status ReadAsync(TmpWriteHandle* handle, MemRange buffer) WARN_UNUSED_RESULT;

  /// 等待由 ReadAsync() 为 'handle' 启动的读取完成。'buffer' 应是传递给 ReadAsync() 的相同缓冲区。
  /// 如果读取失败，则返回错误。允许通过再次调用 ReadAsync() 重试失败的读取。
  /// 如果非空，则 'counters' 中的计数器将更新读取信息。
  Status WaitForAsyncRead(TmpWriteHandle* handle, MemRange buffer,
      const BufferPoolClientCounters* counters = nullptr) WARN_UNUSED_RESULT;

  /// 恢复传递给 Write() 的 'buffer' 中的原始数据，必要时解密。如果恢复数据失败，则返回错误。
  /// 写入不得飞行 - 调用者负责等待写入完成。
  /// 如果非空，则 'counters' 中的计数器将更新读取信息。
  Status RestoreData(std::unique_ptr<TmpWriteHandle> handle, MemRange buffer,
      const BufferPoolClientCounters* counters = nullptr) WARN_UNUSED_RESULT;

  /// 等待飞行中的 I/O 完成并销毁与 'handle' 关联的资源。
  void DestroyWriteHandle(std::unique_ptr<TmpWriteHandle> handle);

  /// 对组中的所有文件调用 Remove() 并删除它们。
  void Close();

  /// 从本地或远程草稿空间关闭文件的函数模板。
  template <typename T>
  void CloseInternal(vector<T>& tmp_files);

  /// 在分配新草稿空间后更新草稿空间的相应指标。
  void UpdateScratchSpaceMetrics(int64_t num_bytes, bool is_remote = false);

  /// 组装并返回新路径。
  std::string GenerateNewPath(const string& dir, const string& unique_name);

  std::string DebugString();

  const TUniqueId& unique_id() const { return unique_id_; }

  TmpFileMgr* tmp_file_mgr() const { return tmp_file_mgr_; }

  void SetDebugAction(const std::string& debug_action) { debug_action_ = debug_action; }

  /// 返回 true 如果溢出到磁盘由于本地故障磁盘失败。
  bool IsSpillingDiskFaulty();

  /// 返回远程 TmpFile 的 shared_ptr。
  std::shared_ptr<TmpFile>& FindTmpFileSharedPtr(TmpFile* tmp_file);

 private:
  friend class TmpFile;
  friend class TmpFileLocal;
  friend class TmpFileRemote;
  friend class TmpFileMgrTest;
  friend class TmpWriteHandle;
  friend class io::WriteRange;
  friend class io::ScanRange;
  friend class io::RemoteOperRange;

  /// 使用每个具有草稿目录的磁盘上的一个临时文件初始化文件组。如果至少成功创建了一个临时文件，则返回 OK。
  /// 如果没有成功创建临时文件，则返回错误。仅可调用一次。必须在 'lock_' 持有时调用。
  Status CreateFiles() WARN_UNUSED_RESULT;

  /// 在临时文件中分配 'num_bytes' 字节。如果发生错误，则尝试多个磁盘。
  /// 仅当没有可用临时文件或超出草稿限制时返回错误。必须在不持有 'lock_' 时调用。
  Status AllocateSpace(
      int64_t num_bytes, TmpFile** tmp_file, int64_t* file_offset) WARN_UNUSED_RESULT;

  /// 尝试从本地草稿空间分配 'num_bytes' 字节。由 AllocateSpace() 调用。
  /// alloc_full 返回 true，如果所有目录都已满。
  Status AllocateLocalSpace(int64_t num_bytes, TmpFile** tmp_file, int64_t* file_offset,
      vector<int>* at_capacity_dirs, bool* alloc_full) WARN_UNUSED_RESULT;

  /// 当本地草稿空间没有空间剩余时，尝试从远程草稿空间分配 'num_bytes' 字节。由 AllocateSpace() 调用。
  Status AllocateRemoteSpace(int64_t num_bytes, TmpFile** tmp_file, int64_t* file_offset,
      vector<int>* at_capacity_dirs) WARN_UNUSED_RESULT;

  /// 回收草稿文件中的字节范围并销毁 'handle'。当范围不再用于 'handle' 时调用。
  /// 一旦调用此函数，关联的磁盘空间可以被回收，要么通过添加到 'free_ranges_' 以回收，要么通过在文件中打孔。
  /// 必须在不持有 'lock_' 时调用。
  void RecycleFileRange(std::unique_ptr<TmpWriteHandle> handle);

  /// 当 DiskIoMgr 写入为 'handle' 完成时调用。在错误情况下，将尝试重试写入。在成功或无法重试写入时，调用 handle->WriteComplete()。
  void WriteComplete(TmpWriteHandle* handle, const Status& write_status);

  /// 处理写入错误。记录写入错误，如果原因是 I/O 错误，则为此文件组将设备列入黑名单。
  /// 黑名单限制写入重试次数，因为每个设备仅尝试一次。如果成功重新发出写入，则返回 OK。
  /// 如果原始错误不可恢复或在重新发出写入时遇到不可恢复错误，则返回错误状态。错误状态将包含所有先前的 I/O 错误在其细节中。
  Status RecoverWriteError(
      TmpWriteHandle* handle, const Status& write_status) WARN_UNUSED_RESULT;

  /// 返回带有适当信息的 SCRATCH_ALLOCATION_FAILED 错误，包括草稿目录、分配的草稿量和导致此失败的先前错误。
  /// 如果某些目录已满但未遇到错误，则 tmp_file_mgr_->tmp_dir_ 中这些目录的索引应包含在 'at_capacity_dirs' 中。
  /// 调用者必须持有 'lock_'。
  Status ScratchAllocationFailedStatus(const std::vector<int>& at_capacity_dirs);

  /// 查询的调试操作。
  std::string debug_action_;

  /// 它关联的 TmpFileMgr。
  TmpFileMgr* const tmp_file_mgr_;

  /// 用于所有临时文件 I/O 的 DiskIoMgr。
  io::DiskIoMgr* const io_mgr_;

  /// 用于所有读取和写入的 I/O 上下文。在构造函数中注册。
  std::unique_ptr<io::RequestContext> io_ctx_;

  /// 在 Read() 中分配的扫描范围存储。需要是因为 ScanRange 对象即使在扫描完成后也可能被 DiskIoMgr 触及。
  /// TODO: IMPALA-4249: 一旦 ScanRange 对象的生命周期更好地定义，则移除。
  ObjectPool scan_range_pool_;

  /// 在所有 TmpFileGroup 中唯一。用于前缀文件名。
  const TUniqueId unique_id_;

  /// 允许的最大写入空间（-1 表示无限制）。
  const int64_t bytes_limit_;

  /// 写入操作数（包括已启动但尚未完成的写入）。
  RuntimeProfile::Counter* const write_counter_;

  /// 写入磁盘的字节数（包括已启动但尚未完成的写入）。
  RuntimeProfile::Counter* const bytes_written_counter_;

  /// 压缩前写入磁盘的字节数（包括已启动但尚未完成的写入）。
  RuntimeProfile::Counter* const uncompressed_bytes_written_counter_;

  /// 读取操作数（包括已启动但尚未完成的读取）。
  RuntimeProfile::Counter* const read_counter_;

  /// 从磁盘读取的字节数（包括已启动但尚未完成的读取）。
  RuntimeProfile::Counter* const bytes_read_counter_;

  /// 使用内存缓冲区的读取操作数。
  RuntimeProfile::Counter* const read_use_mem_counter_;

  /// 从内存缓冲区读取的字节数。
  RuntimeProfile::Counter* const bytes_read_use_mem_counter_;

  /// 使用本地磁盘缓冲区的读取操作数。
  RuntimeProfile::Counter* const read_use_local_disk_counter_;

  /// 从本地磁盘缓冲区读取的字节数。
  RuntimeProfile::Counter* const bytes_read_use_local_disk_counter_;

  /// 以字节为单位分配的草稿空间量。
  RuntimeProfile::Counter* const scratch_space_bytes_used_counter_;

  /// 等待磁盘读取的时间。
  RuntimeProfile::Counter* const disk_read_timer_;

  /// 在磁盘溢出加密、解密和完整性检查中花费的时间。
  RuntimeProfile::Counter* encryption_timer_;

  /// 在磁盘溢出压缩和解压缩中花费的时间。如果未启用压缩，则为 nullptr。
  RuntimeProfile::Counter* compression_timer_;

  /// 保护 tmp_files_remote_ptrs_。
  SpinLock tmp_files_remote_ptrs_lock_;

  /// 远程 TmpFile 的原始指针及其 shared_ptr 的映射。
  std::unordered_map<TmpFile*, std::shared_ptr<TmpFile>> tmp_files_remote_ptrs_;

  /// 保护以下成员。
  SpinLock lock_;

  /// 表示 TmpFileGroup 的文件列表。文件按相关 TmpDir 的优先级排序。
  std::vector<std::unique_ptr<TmpFile>> tmp_files_;

  /// TmpFileGroup 中已列入黑名单的文件数。
  int num_blacklisted_files_;

  /// 设置为 true 以指示由于故障磁盘导致的溢出到磁盘失败。它设置为 true 如果此 TmpFileGroup 中的所有临时（也称为草稿）文件都被列入黑名单，或读取临时文件时获取磁盘错误。
  bool spilling_disk_faulty_;

  /// 表示 TmpFileGroup 的远程文件列表。
  std::vector<std::shared_ptr<TmpFile>> tmp_files_remote_;

  /// 'tmp_files' 中的索引范围。用于跟踪对应给定优先级的索引范围。
  struct TmpFileIndexRange {
    TmpFileIndexRange(int start, int end)
      : start(start), end(end) {}
    // 范围的起始索引。
    const int start;
    // 范围的结束索引。
    const int end;
  };
  /// 存储 'tmp_files' 中索引范围的映射，对应草稿目录的优先级。
  std::map<int, TmpFileIndexRange> tmp_files_index_range_;

  /// 此组文件中分配的总空间。
  AtomicInt64 current_bytes_allocated_;

  /// 此组文件中远程分配的总空间。
  AtomicInt64 current_bytes_allocated_remote_;

  /// 'tmp_files' 中的索引，表示下一个临时文件范围应从给定优先级的哪个文件分配，用于实现从临时文件的轮询分配。
  std::unordered_map<int, int> next_allocation_index_;

  /// free_ranges_[i] 中的每个向量是长度为 2^i 字节的空闲草稿范围的 File/offset 对向量。有 64 个条目，以便每个 int64_t 长度都有一个有效的列表关联。
  /// 仅在 --disk_spill_punch_holes 为 false 时使用。
  std::vector<std::vector<std::pair<TmpFile*, int64_t>>> free_ranges_;

  /// 创建/写入草稿文件时遇到的错误。我们存储历史记录，以便如果我们用尽写入设备，可以报告草稿错误的原始原因。
  std::vector<Status> scratch_errors_;
};

/// 写入操作的句柄，由临时文件范围支持。该操作正在飞行中或已完成。如果它以无错误完成且未被取消，则数据在文件中并可以读取回。
///
/// TmpWriteHandle 从 TmpFileGroup::Write() 返回。在写入完成后的句柄可以传递给 TmpFileGroup::Read() 以零次或多次读取回数据。
/// 可以随时调用 TmpFileGroup::DestroyWriteHandle() 以销毁句柄并允许重用写入的草稿文件范围。或者，可以调用 TmpFileGroup::RestoreData() 以逆转 TmpFileGroup::Write() 的效果，通过销毁句柄并恢复原始数据到缓冲区，只要调用者未修改缓冲区中的数据。
///
/// TmpWriteHandle 的公共方法可以从多个线程并发调用。
class TmpWriteHandle {
 public:
  /// 写入必须通过传递给 TmpFileGroup 来销毁 - 在写入完成前销毁它是错误。
  ~TmpWriteHandle();

  /// 同步取消任何飞行中的读取。
  void CancelRead();

  /// 支持块的临时文件路径。用于测试目的。
  /// 如果未分配支持文件，则返回空字符串。
  std::string TmpFilePath() const;

  /// 支持块的临时文件缓冲路径。用于测试目的。
  /// 如果未分配支持文件或临时文件不在远程，则返回空字符串。
  std::string TmpFileBufferPath() const;

  /// 写入磁盘的内存数据长度（字节），在任何压缩前。
  int64_t data_len() const { return data_len_; }

  /// 磁盘上数据的大小（压缩后）（字节）。仅在 Write() 成功时有效调用。
  int64_t on_disk_len() const;

  bool is_compressed() const { return compressed_len_ >= 0; }

  std::string DebugString();

 private:
  friend class TmpFileGroup;
  friend class TmpFileMgrTest;

  TmpWriteHandle(TmpFileGroup* const parent, TmpFileMgr::WriteDoneCallback cb);

  /// 启动写入。此方法在文件中分配空间，必要时压缩和加密。在调用前 'write_in_flight_' 必须为 false。返回后，在成功时 'write_in_flight_' 为 true，在失败时为 false，并且 'is_cancelled_' 在失败时设置为 true。
  /// 如果数据被压缩，则 'compressed_len_' 将为非负，并且 'compressed_' 将是用于存储压缩数据的临时缓冲区。
  /// 如果非空，则 'counters' 中的计数器将更新读取信息。
  Status Write(io::RequestContext* io_ctx, MemRange buffer,
      TmpFileMgr::WriteDoneCallback callback,
      const BufferPoolClientCounters* counters = nullptr);

  /// 尝试压缩 'buffer'。成功时，返回 true，并且 'compressed_' 和 'compressed_len_' 分别包含使用的缓冲区（长度反映分配大小）和压缩数据的长度。
  /// 失败时，返回 false，'compressed_' 将为空缓冲区，并且 'compressed_len_' 将为 -1。压缩失败的原因可能被记录。
  /// 如果非空，则 'counters' 中的计数器将更新压缩时间。
  bool TryCompress(MemRange buffer, const BufferPoolClientCounters* counters);

  /// 在初始写入以错误失败后重试写入，而是写入 'file' 的 'offset'。在调用前 'write_in_flight_' 必须为 true。
  /// 返回后，在成功时 'write_in_flight_' 为 true，在失败时为 false。
  Status RetryWrite(io::RequestContext* io_ctx, TmpFile* file,
      int64_t offset) WARN_UNUSED_RESULT;

  /// 当写入成功或不成功完成时调用。设置 'write_in_flight_' 然后调用 'cb_'。
  void WriteComplete(const Status& write_status);

  /// 当上传成功或不成功完成时调用。
  /// 如果上传成功，则将文件入队到可用池。
  void UploadComplete(TmpFile* file, const Status& write_status);

  /// 取消任何飞行中的写入或读取。读取同步取消，写入异步取消。调用 Cancel() 后，写入不重试。
  /// 写入回调可能以 CANCELLED_INTERNALLY 状态调用（除非它先成功或遇到不同错误）。
  void Cancel();

  /// 阻塞直到写入成功或不成功完成。
  /// 可能在写入回调调用前返回。
  void WaitForWrite();

  /// 就地加密 'buffer' 中的数据并计算 'hash_'。
  /// 如果非空，则 'counters' 中的计数器将更新压缩时间。
  Status EncryptAndHash(MemRange buffer, const BufferPoolClientCounters* counters);

  /// 验证完整性哈希并就地解密 'buffer' 的内容。
  /// 如果非空，则 'counters' 中的计数器将更新压缩时间。
  Status CheckHashAndDecrypt(MemRange buffer, const BufferPoolClientCounters* counters);

  /// 释放 'compressed_' 并更新内存记账。如果 'compressed_' 为空，则无操作。
  void FreeCompressedBuffer();

  TmpFileGroup* const parent_;

  /// 当写入完成时调用的回调。
  TmpFileMgr::WriteDoneCallback cb_;

  /// 写入磁盘的内存数据缓冲区长度。如果使用压缩，这是未压缩大小。在 Write() 中设置。
  int64_t data_len_ = -1;

  /// 此写入的 DiskIoMgr 写入范围。
  boost::scoped_ptr<io::WriteRange> write_range_;

  /// 正在写入的临时文件。
  TmpFile* file_ = nullptr;

  /// 如果 --disk_spill_encryption 开启，则为 AES 256 位密钥和初始化向量。
  /// 为每个写入重新生成。
  EncryptionKey key_;

  /// 如果 --disk_spill_encryption 开启，则为正在写入数据的哈希。在写入时填充；在读取时验证。这是计算在加密之后。
  IntegrityHash hash_;

  /// 当前飞行中的读取的扫描范围。无读取飞行时为 NULL。
  io::ScanRange* read_range_ = nullptr;

  /// 在 'write_in_flight_' 为 true 时保护以下所有字段。在其他时间，禁止从多个线程并发调用 WriteRange/TmpFileGroup 方法，因此无需锁。
  /// 在调用 'cb_' 时不应持有锁以避免死锁。
  std::mutex write_state_lock_;

  /// 如果写入已被取消（但不一定完成），则为 true。
  bool is_cancelled_ = false;

  /// 如果写入正在飞行中，则为 true。
  bool write_in_flight_ = false;

  /// 用于存储压缩数据的缓冲区。在读取或写入压缩范围时分配缓冲区。
  /// TODO: ScopedBuffer 是一种次优的内存分配方法。我们最好更直接地与缓冲池集成，使用其缓冲区分配器，并使压缩缓冲区某种可驱逐。
  ScopedBuffer compressed_;

  /// 如果此范围中的数据被压缩，则设置为非负。在这种情况下，'compressed_' 是用于存储数据的缓冲区，并且 'compressed_len_' 是缓冲区中有效数据的量。
  int64_t compressed_len_ = -1;

  /// 当写入完成且 'write_in_flight_' 变为 false 时发出信号，在调用 'cb_' 前。
  ConditionVariable write_complete_cv_;
};
}
```
******************************************************************************
Status TmpFileGroup::CreateFiles() {
  lock_.DCheckLocked();
  DCHECK(tmp_files_.empty());
  vector<DeviceId> tmp_devices = tmp_file_mgr_->ActiveTmpDevices();
  DCHECK(tmp_file_mgr_->NumActiveTmpDevicesLocal() <= tmp_devices.size());
  int files_allocated = 0;
  // Initialize the tmp files and the initial file to use.
  for (int i = 0; i < tmp_file_mgr_->NumActiveTmpDevicesLocal(); ++i) {
    DeviceId device_id = tmp_devices[i];
    unique_ptr<TmpFile> tmp_file;
    tmp_file_mgr_->NewFile(this, device_id, &tmp_file);
    tmp_files_.emplace_back(std::move(tmp_file));
    ++files_allocated;
  }
  DCHECK_EQ(tmp_file_mgr_->NumActiveTmpDevicesLocal(), files_allocated);
  DCHECK_EQ(tmp_file_mgr_->NumActiveTmpDevicesLocal(), tmp_files_.size());
  if (tmp_files_.size() == 0) return ScratchAllocationFailedStatus({});
  // Initialize the next allocation index for each priority.
  for (const auto& entry: tmp_files_index_range_) {
    const int priority = entry.first;
    const int start = entry.second.start;
    const int end = entry.second.end;
    // Start allocating on a random device to avoid overloading the first device.
    next_allocation_index_.emplace(priority, start + rand() % (end - start + 1));
  }
  return Status::OK();
}

-----

`TmpFileGroup::CreateFiles()` 函数是 Impala 临时文件管理中的一个核心部分，它负责**为一个新的临时文件组初始化和创建其所需的所有临时文件对象**。请注意，这里的“创建文件”通常指的是在内存中创建代表这些临时文件的 C++ 对象，而不是立即在磁盘上创建物理文件。物理文件的创建通常会延迟到实际写入数据时。

我们来一步步拆解它的功能和逻辑。

-----

## 函数功能与设计意图

`TmpFileGroup::CreateFiles()` 的主要目标是：

1.  **为文件组分配逻辑文件对象：** 根据 `TmpFileMgr` 中配置的活跃临时设备（磁盘），为当前 `TmpFileGroup` 创建相应数量的 `TmpFile` 对象。
2.  **初始化内部状态：** 设置文件组内部用于管理这些文件的索引和分配策略（例如，轮询分配的起始点）。
3.  **错误检查：** 确保至少有一个临时文件被成功创建，否则返回错误状态。

-----

## 代码逐行解释

```cpp
Status TmpFileGroup::CreateFiles() {
  lock_.DCheckLocked();
  // 确保在调用此函数时，TmpFileGroup 的内部锁已经被持有。
  // 这是一个调试断言 (DCheck)，在生产环境中通常会被优化掉，
  // 但在开发和测试阶段用于捕获编程错误。

  DCHECK(tmp_files_.empty());
  // 调试断言：确保 tmp_files_ 列表在开始创建文件时是空的。
  // 这个函数应该只被调用一次，以初始化文件组。

  vector<DeviceId> tmp_devices = tmp_file_mgr_->ActiveTmpDevices();
  // 从 TmpFileMgr 获取所有当前活跃的临时设备（磁盘）的 ID 列表。
  // 这些是 TmpFileMgr 认为可以用于暂存的设备。

  DCHECK(tmp_file_mgr_->NumActiveTmpDevicesLocal() <= tmp_devices.size());
  // 调试断言：确保本地活跃的临时设备数量不超过总的活跃设备数量。
  // (NumActiveTmpDevicesLocal() 应该只统计本地文件系统的设备)

  int files_allocated = 0;
  // 用于计数实际分配的 TmpFile 对象的数量。

  // 初始化临时文件和要使用的初始文件。
  for (int i = 0; i < tmp_file_mgr_->NumActiveTmpDevicesLocal(); ++i) {
    // 遍历所有本地活跃的临时设备。
    // (Impala可能区分本地磁盘和远程存储，这里只处理本地部分)

    DeviceId device_id = tmp_devices[i];
    // 获取当前设备的 ID。

    unique_ptr<TmpFile> tmp_file;
    // 声明一个智能指针，用于接收新创建的 TmpFile 对象。

    tmp_file_mgr_->NewFile(this, device_id, &tmp_file);
    // 调用 TmpFileMgr 的 NewFile 方法来创建一个新的 TmpFile 对象。
    // 'this' 指针传递给 TmpFileMgr，以便新文件知道它属于哪个 TmpFileGroup。
    // device_id 标识该文件将位于哪个物理设备上。
    // NewFile 会填充 'tmp_file' 这个 unique_ptr。
    // 重要：NewFile 只是创建 TmpFile 对象，并为其生成一个路径，但不会在磁盘上实际创建文件。

    tmp_files_.emplace_back(std::move(tmp_file));
    // 将新创建的 TmpFile 对象（通过移动语义）添加到当前文件组的 tmp_files_ 列表中。
    // unique_ptr 确保了内存的自动管理。

    ++files_allocated;
    // 增加已分配文件对象的计数。
  }

  DCHECK_EQ(tmp_file_mgr_->NumActiveTmpDevicesLocal(), files_allocated);
  // 调试断言：确认实际分配的文件对象数量与本地活跃设备数量相符。

  DCHECK_EQ(tmp_file_mgr_->NumActiveTmpDevicesLocal(), tmp_files_.size());
  // 调试断言：再次确认 tmp_files_ 列表的大小与本地活跃设备数量相符。

  if (tmp_files_.size() == 0) return ScratchAllocationFailedStatus({});
  // 如果没有任何临时文件被成功分配（例如，没有活跃的临时设备），
  // 则返回一个表示“暂存分配失败”的错误状态。这是唯一可能的错误返回路径。

  // 初始化每个优先级的下一个分配索引。
  for (const auto& entry: tmp_files_index_range_) {
    // 遍历 tmp_files_index_range_，这是一个映射，
    // 可能根据设备的优先级或其他策略对文件进行分组。

    const int priority = entry.first;
    // 获取当前文件组的优先级。

    const int start = entry.second.start;
    // 获取此优先级对应的文件在 tmp_files_ 列表中的起始索引。

    const int end = entry.second.end;
    // 获取此优先级对应的文件在 tmp_files_ 列表中的结束索引。

    // Start allocating on a random device to avoid overloading the first device.
    next_allocation_index_.emplace(priority, start + rand() % (end - start + 1));
    // 为每个优先级初始化一个“下一个分配索引”。
    // 这个索引决定了下次分配空间时，应该从哪个文件开始进行轮询。
    // 关键在于 `rand() % (end - start + 1)`，它在当前优先级的文件范围内生成一个随机的起始索引。
    // 这样做的目的是为了避免所有查询或所有文件组都倾向于首先使用第一个设备/文件，
    // 从而有助于在多个设备之间**负载均衡**，避免 I/O 热点。
  }

  return Status::OK();
  // 所有临时文件对象都已成功创建和初始化，返回 OK 状态。
}
```

-----

## 核心概念和设计模式

1.  **延迟物理文件创建：** 这个函数仅仅是在内存中准备好 `TmpFile` 对象，这些对象包含了文件路径、所属设备 ID 等信息。实际的 `.tmp` 文件直到第一次有数据写入时才会在磁盘上被创建。这种延迟创建策略可以节省资源，避免为可能不会被使用的临时文件占用实际磁盘空间。
2.  **内存池/管理器模式：** `TmpFileMgr` 扮演了更高级别的资源管理器角色，它了解系统中有哪些可用的临时设备，并负责创建 `TmpFile` 这样的“资源”实例。`TmpFileGroup` 则是对这些资源的逻辑分组，用于管理特定任务（例如一个查询的溢出）所需的临时文件。
3.  **负载均衡：** 通过 `rand() % (end - start + 1)` 来随机化每个优先级文件组的起始分配索引，`TmpFileGroup` 尝试将写入操作分散到不同的临时文件（以及底层的物理设备）上，从而避免单个磁盘成为性能瓶颈。这对于 I/O 密集型的大数据工作负载至关重要。
4.  **智能指针：** 使用 `unique_ptr` 管理 `TmpFile` 对象，确保了当 `TmpFileGroup` 对象销毁时，其拥有的所有 `TmpFile` 对象也能被正确销毁，避免内存泄漏。

-----

## 为什么这么做？

Impala 作为一个高性能的分析型数据库，经常需要处理比可用内存更大的数据。这时，它会将中间结果\*\*溢出（spill）\*\*到磁盘上的临时文件。`TmpFileGroup::CreateFiles()` 函数是这个溢出机制的第一步。

通过这种设计，Impala 能够：

  * **高效管理：** 集中管理所有临时文件，无需每个查询或操作都手动处理文件路径和句柄。
  * **资源抽象：** 向上层提供一个抽象接口，隐藏了底层是哪个磁盘、哪个目录，以及如何实现对齐和负载均衡等细节。
  * **弹性：** 通过 `TmpFileMgr` 跟踪活跃设备，它可以在某些设备故障时（尽管这个函数本身不处理故障设备，而是依赖 `TmpFileMgr` 的 `ActiveTmpDevices`）依然能够分配文件。

总而言之，`TmpFileGroup::CreateFiles()` 是 Impala 临时文件管理系统中的一个初始化函数，它巧妙地平衡了内存效率、性能（通过负载均衡）和系统健壮性，为后续的实际数据溢出操作打下基础。
*****************************************
我想弄清楚tmp_files_index_range_的含义，我补充如下的代码
好的，结合 TmpFileGroup 的构造函数和 CreateFiles 函数，我们可以更清晰地理解 tmp_files_index_range_ 的含义和作用。

tmp_files_index_range_ 的含义
tmp_files_index_range_ 是 TmpFileGroup 类的一个成员变量，其定义是：

C++

std::map<int, TmpFileIndexRange> tmp_files_index_range_;
其中 TmpFileIndexRange 是一个结构体，包含 start 和 end 两个 int 成员。

核心含义：

tmp_files_index_range_ 是一个映射 (map)，它将暂存目录的优先级 (int priority) 映射到该优先级下对应的 TmpFile 对象在 tmp_files_ 向量中的一个连续索引范围 (TmpFileIndexRange)。

换句话说，它告诉我们：对于某个特定的优先级，哪些 TmpFile 对象属于这个优先级，以及它们在 tmp_files_ 向量中占据了从哪里到哪里的位置。

为什么需要这个？
在 Impala 中，不同的暂存目录（TmpDir）可能具有不同的优先级。例如，用户可能配置了 SSD 上的目录优先级高于 HDD 上的目录。Impala 希望优先使用高优先级的设备进行溢出，以获得更好的性能。

当 TmpFileGroup 创建其内部的 tmp_files_ 向量时（在 CreateFiles 中），它会按照 TmpFileMgr 返回的活跃设备的顺序填充这个向量。这个顺序通常会隐含或明确地考虑到设备的优先级。

tmp_files_index_range_ 的作用就是建立优先级与 tmp_files_ 向量中实际文件对象位置的关联，从而允许 TmpFileGroup 在分配空间时：

按优先级选择文件： 优先从高优先级的 TmpFile 中分配空间。

在同一优先级内进行负载均衡： 在选择了一个优先级后，再利用 next_allocation_index_ 在该优先级对应的文件范围内进行轮询分配，避免过度使用某个文件。
*********************************************
-----

好的，结合 `TmpFileGroup` 的构造函数和 `CreateFiles` 函数，我们可以更清晰地理解 `tmp_files_index_range_` 的含义和作用。

## `tmp_files_index_range_` 的含义

`tmp_files_index_range_` 是 `TmpFileGroup` 类的一个成员变量，其定义是：

```cpp
std::map<int, TmpFileIndexRange> tmp_files_index_range_;
```

其中 `TmpFileIndexRange` 是一个结构体，包含 `start` 和 `end` 两个 `int` 成员。

**核心含义：**

`tmp_files_index_range_` 是一个**映射 (map)**，它将**暂存目录的优先级 (`int priority`) 映射到该优先级下对应的 `TmpFile` 对象在 `tmp_files_` 向量中的一个连续索引范围 (`TmpFileIndexRange`)。**

换句话说，它告诉我们：**对于某个特定的优先级，哪些 `TmpFile` 对象属于这个优先级，以及它们在 `tmp_files_` 向量中占据了从哪里到哪里的位置。**

### 为什么需要这个？

在 Impala 中，不同的暂存目录（`TmpDir`）可能具有不同的**优先级**。例如，用户可能配置了 SSD 上的目录优先级高于 HDD 上的目录。Impala 希望优先使用高优先级的设备进行溢出，以获得更好的性能。

当 `TmpFileGroup` 创建其内部的 `tmp_files_` 向量时（在 `CreateFiles` 中），它会按照 `TmpFileMgr` 返回的活跃设备的顺序填充这个向量。这个顺序通常会隐含或明确地考虑到设备的优先级。

`tmp_files_index_range_` 的作用就是**建立优先级与 `tmp_files_` 向量中实际文件对象位置的关联**，从而允许 `TmpFileGroup` 在分配空间时：

1.  **按优先级选择文件：** 优先从高优先级的 `TmpFile` 中分配空间。
2.  **在同一优先级内进行负载均衡：** 在选择了一个优先级后，再利用 `next_allocation_index_` 在该优先级对应的文件范围内进行轮询分配，避免过度使用某个文件。

-----

## `TmpFileGroup` 构造函数中 `tmp_files_index_range_` 的填充逻辑解释

让我们关注构造函数中填充 `tmp_files_index_range_` 的部分：

```cpp
TmpFileGroup::TmpFileGroup(...)
  // ... 其他初始化列表成员 ...
  // Populate the priority based index ranges.
  const std::vector<std::unique_ptr<TmpDir>>& tmp_dirs = tmp_file_mgr_->tmp_dirs_;
  // 获取 TmpFileMgr 管理的所有 TmpDir 列表。
  // 注意：tmp_dirs_ 是 TmpFileMgr 的成员，它包含了所有（包括活跃和非活跃的）
  // 暂存目录，并且通常是按照优先级排序的。
  // CreateFiles 函数会根据 ActiveTmpDevices() 来实际创建 tmp_files_ 向量，
  // 但这里的 tmp_dirs_ 已经包含了优先级信息。

  if (tmp_dirs.size() > 0) {
    int start_index = 0; // 当前优先级文件范围的起始索引
    int priority = tmp_dirs[0]->priority(); // 第一个目录的优先级

    // 遍历 TmpDirs 列表，查找优先级发生变化的点
    for (int i = 0; i < tmp_dirs.size() - 1; ++i) {
      priority = tmp_dirs[i]->priority(); // 当前目录的优先级
      const int next_priority = tmp_dirs[i + 1]->priority(); // 下一个目录的优先级

      // 如果下一个目录的优先级与当前目录不同，说明我们遇到了一个优先级边界
      if (next_priority != priority) {
        // 将当前优先级的文件范围插入到 tmp_files_index_range_ 中
        // 范围是从 start_index 到当前的 i
        tmp_files_index_range_.emplace(priority, TmpFileIndexRange(start_index, i));
        start_index = i + 1; // 更新下一个优先级范围的起始索引
        priority = next_priority; // 更新当前优先级为下一个优先级
      }
    }
    // 处理最后一个优先级的文件范围，因为循环在倒数第二个元素处结束
    tmp_files_index_range_.emplace(priority,
      TmpFileIndexRange(start_index, tmp_dirs.size() - 1));
  }
}
```

### 逐步分析填充逻辑

假设 `tmp_file_mgr_->tmp_dirs_` 包含以下目录（及其优先级）：

| 索引 | 目录路径 | 优先级 |
| :--- | :------- | :----- |
| 0 | /ssd/scratch0 | 0 |
| 1 | /ssd/scratch1 | 0 |
| 2 | /hdd/scratch0 | 1 |
| 3 | /hdd/scratch1 | 1 |
| 4 | /cold/scratch0 | 2 |

1.  **`start_index = 0`**, **`priority = tmp_dirs[0]->priority()` (0)**.

2.  **循环 `i = 0`**:

      * `priority = tmp_dirs[0]->priority()` (0)
      * `next_priority = tmp_dirs[1]->priority()` (0)
      * `next_priority == priority` (0 == 0)，条件不满足，继续。

3.  **循环 `i = 1`**:

      * `priority = tmp_dirs[1]->priority()` (0)
      * `next_priority = tmp_dirs[2]->priority()` (1)
      * `next_priority != priority` (1 \!= 0)，条件满足：
          * **`tmp_files_index_range_.emplace(0, TmpFileIndexRange(0, 1))`** (插入优先级 0 的范围，从索引 0 到 1)
          * `start_index = 1 + 1 = 2`
          * `priority = 1`

4.  **循环 `i = 2`**:

      * `priority = tmp_dirs[2]->priority()` (1)
      * `next_priority = tmp_dirs[3]->priority()` (1)
      * `next_priority == priority` (1 == 1)，条件不满足，继续。

5.  **循环 `i = 3`**:

      * `priority = tmp_dirs[3]->priority()` (1)
      * `next_priority = tmp_dirs[4]->priority()` (2)
      * `next_priority != priority` (2 \!= 1)，条件满足：
          * **`tmp_files_index_range_.emplace(1, TmpFileIndexRange(2, 3))`** (插入优先级 1 的范围，从索引 2 到 3)
          * `start_index = 3 + 1 = 4`
          * `priority = 2`

6.  **循环结束** (因为 `i` 到达 `tmp_dirs.size() - 1`，即 4)。

7.  **循环后处理最后一部分：**

      * **`tmp_files_index_range_.emplace(2, TmpFileIndexRange(4, 4))`** (插入优先级 2 的范围，从索引 4 到 4)

最终，`tmp_files_index_range_` 会是：

| 优先级 | `TmpFileIndexRange` (`start`, `end`) | 对应的 `TmpFile` 索引 |
| :----- | :----------------------------------- | :-------------------- |
| 0 | (0, 1) | `tmp_files_[0]`, `tmp_files_[1]` |
| 1 | (2, 3) | `tmp_files_[2]`, `tmp_files_[3]` |
| 2 | (4, 4) | `tmp_files_[4]` |

## `CreateFiles()` 中对 `tmp_files_index_range_` 的使用

在 `CreateFiles()` 函数的最后部分：

```cpp
  // Initialize the next allocation index for each priority.
  for (const auto& entry: tmp_files_index_range_) {
    const int priority = entry.first;
    const int start = entry.second.start;
    const int end = entry.second.end;
    // Start allocating on a random device to avoid overloading the first device.
    next_allocation_index_.emplace(priority, start + rand() % (end - start + 1));
  }
```

这里就利用了 `tmp_files_index_range_` 中存储的优先级和索引范围信息。对于每个优先级组：

  * 它获取该优先级组的 `start` 和 `end` 索引。
  * 然后，它在 `[start, end]` 这个范围内**随机选择一个起始索引**，并将其存储在 `next_allocation_index_` 中。

这样做的目的如前面所解释，是为了**负载均衡**：当 Impala 需要分配暂存空间时，它会首先尝试最高优先级的目录。如果该优先级有多个文件，它会从 `next_allocation_index_` 中记录的那个随机起始文件开始，然后以轮询（round-robin）的方式在属于该优先级的文件中分配，从而避免总是从第一个文件开始分配导致 I/O 热点。

-----

## 总结

`tmp_files_index_range_` 是 `TmpFileGroup` 用来**按优先级对它所管理的所有临时文件进行分组和索引**的关键数据结构。它记录了每个优先级的文件在 `tmp_files_` 向量中的起止位置。这使得 `TmpFileGroup` 能够在进行内存分配时，有效地结合**优先级选择**和**同一优先级内的负载均衡**策略，从而优化磁盘 I/O 性能。
****************************************************8
核心概念和设计模式
1.	延迟物理文件创建： 这个函数仅仅是在内存中准备好 TmpFile 对象，这些对象包含了文件路径、所属设备 ID 等信息。实际的 .tmp 文件直到第一次有数据写入时才会在磁盘上被创建。这种延迟创建策略可以节省资源，避免为可能不会被使用的临时文件占用实际磁盘空间。
2.	内存池/管理器模式： TmpFileMgr 扮演了更高级别的资源管理器角色，它了解系统中有哪些可用的临时设备，并负责创建 TmpFile 这样的“资源”实例。TmpFileGroup 则是对这些资源的逻辑分组，用于管理特定任务（例如一个查询的溢出）所需的临时文件。
3.	负载均衡： 通过 rand() % (end - start + 1) 来随机化每个优先级文件组的起始分配索引，TmpFileGroup 尝试将写入操作分散到不同的临时文件（以及底层的物理设备）上，从而避免单个磁盘成为性能瓶颈。这对于 I/O 密集型的大数据工作负载至关重要。
4.	智能指针： 使用 unique_ptr 管理 TmpFile 对象，确保了当 TmpFileGroup 对象销毁时，其拥有的所有 TmpFile 对象也能被正确销毁，避免内存泄漏。


你在上面的回答中曾经提到过：“内存池/管理器模式： TmpFileMgr 扮演了更高级别的资源管理器角色，它了解系统中有哪些可用的临时设备，并负责创建 TmpFile 这样的“资源”实例。TmpFileGroup 则是对这些资源的逻辑分组，用于管理特定任务（例如一个查询的溢出）所需的临时文件。” 。请问这种模式有什么特点
2. 资源统一管理：
•	集中控制： 所有的资源分配都通过管理器进行，这使得系统可以集中监控和控制资源的使用情况。例如，可以轻松地跟踪总共分配了多少内存、峰值使用了多少、哪些任务正在消耗大量资源等。
分层管理： 模式可以分层实现。TmpFileMgr 是顶层管理器，了解全局的磁盘信息和策略。TmpFileGroup 则是在此基础上为特定任务提供逻辑分组管理，它从 TmpFileMgr 获取资源，并在此基础上进行更细粒度的分配和回收。

*************************************************8

