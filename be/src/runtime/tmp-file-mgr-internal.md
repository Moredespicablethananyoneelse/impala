```cpp
/// TmpFile 是临时目录中物理文件的句柄。可以使用 AllocateSpace() 分配文件空间，并使用 Remove() 删除文件。由 TmpFileMgr 内部使用。
///
/// 物理文件在文件系统中的创建被推迟，直到由 DiskIoMgr 写入该文件为止。
///
/// TmpFile 的方法不是线程安全的。
class TmpFile {
 public:
  TmpFile(TmpFileGroup* file_group, TmpFileMgr::DeviceId device_id,
          const std::string& path, bool expected_local = true);
  virtual ~TmpFile() {}

  /// 如果临时目录中有空闲容量，则为该文件中的新数据块分配 'num_bytes' 字节。
  /// 如果容量不足，则返回 false。否则，更新状态并返回 true。
  /// 此函数实际上不执行任何文件操作。
  /// 成功时，将 'offset' 设置为分配范围内第一个字节的文件偏移量。
  virtual bool AllocateSpace(int64_t num_bytes, int64_t* offset) = 0;

  /// 当遇到该文件的 IO 错误时调用。记录错误并将文件列入黑名单。
  /// 如果文件刚刚被列入黑名单，则返回 true。
  bool Blacklist(const ErrorMsg& msg);

  /// 删除物理文件，包括磁盘上的缓冲区（如果已创建）。
  /// 调用 Remove() 后，不得再读取或写入该文件。
  virtual Status Remove() = 0;

  /// 获取用于 IO 管理器队列的磁盘 ID。
  int AssignDiskQueue(bool is_local_buffer = false) const;

  /// 尝试在文件中的 'offset' 处打孔，大小为 'len'。
  Status PunchHole(int64_t offset, int64_t len);

  /// 返回用于写入的确切文件句柄。对于远程溢出，默认返回本地缓冲区文件而不是远程文件。
  virtual io::DiskFile* GetWriteFile() = 0;

  /// 返回 TmpFile 的路径。
  /// 如果是远程临时文件，则路径应为远程暂存空间路径。
  /// 类似地，如果是本地文件，则路径应为本地路径。
  const std::string& path() const { return path_; }

  /// 调用者必须持有 TmpFileMgr::FileGroup::lock_。
  bool is_blacklisted() const { return blacklisted_; }

  /// 返回文件的当前长度。
  int64_t len() const { return allocation_offset_; }

  /// 返回临时文件的磁盘 ID。
  virtual int disk_id(bool is_file_op = false) const {
    // 文件操作的磁盘 ID 仅在 TmpFileRemote 中支持。
    DCHECK(!is_file_op);
    return disk_id_;
  }

  /// 返回临时文件是否在本地文件系统中。
  bool is_local() { return expected_local_; }

  /// 返回 TmpFile 的本地缓冲区路径。
  const string& LocalBuffPath() { return local_buffer_path_; }

  std::string DebugString();

  /// 帮助获取与该文件关联的 TmpDir。
  TmpDir* GetDir();

  /// 帮助获取与该文件关联的 TmpFileGroup。
  TmpFileGroup* FileGroup() const { return file_group_; }

  /// 返回磁盘文件的指针。
  io::DiskFile* DiskFile() { return disk_file_.get(); }

  /// 返回文件所在磁盘类型。
  io::DiskFileType disk_type() { return disk_type_; }

 private:
  friend class TmpFileMgrTest;
  friend class TmpFileLocal;
  friend class TmpFileRemote;
  friend class TmpFileDummy;
  friend class TmpWriteHandle;

  /// Impala 在每个配置的暂存目录中创建的子目录名称。
  const static std::string TMP_SUB_DIR_NAME;

  /// 暂存目录中理想情况下必须可用于写入的空间（以 MB 为单位）。
  /// 如果可用空间少于此阈值，则发出警告。
  const static uint64_t AVAILABLE_SPACE_THRESHOLD_MB;

  /// 所属的 TmpFileGroup。不能为空。
  TmpFileGroup* const file_group_;

  /// 文件系统中的物理文件路径。
  const std::string path_;

  /// 该文件存储的临时设备。
  const TmpFileMgr::DeviceId device_id_;

  /// 物理文件所在的磁盘 ID。
  int disk_id_;

  // 如果文件预期在本地文件系统中。
  bool expected_local_;

  /// 通过 AllocateSpace() 分配的文件总字节数。请注意，
  /// 这些字节可能并未实际使用文件系统上的空间，要么是因为数据尚未写入，要么是因为已打孔。
  /// 由 AllocateSpace() 修改。
  /*
    记录文件的逻辑大小，而不是物理大小。
    因为punch hole并不修改文件的逻辑大小。
    allocattion_offset_ - bytes_reclaimed才是真正的文件占用的磁盘的大小
  */
  int64_t allocation_offset_ = 0;

  /// 通过打孔回收的字节。
  AtomicInt64 bytes_reclaimed_{0};

  /// 设置为 true 以指示不应在此文件中分配更多空间。
  /// 由 TmpFileMgr::FileGroup::lock_ 保护。
  bool blacklisted_;

  /// 指定文件所在磁盘的类型。
  io::DiskFileType disk_type_;

  /// TmpFile 的本地缓冲区文件路径。
  string local_buffer_path_;

  /// 临时文件的磁盘文件，是用于管理状态和操作物理文件的句柄。
  std::unique_ptr<io::DiskFile> disk_file_;
};

/// TmpFileLocal 是 TmpFile 的派生类，用于提供处理本地文件系统临时目录中物理文件的方法。
class TmpFileLocal : public TmpFile {
 public:
  TmpFileLocal(TmpFileGroup* file_group, TmpFileMgr::DeviceId device_id,
               const std::string& path, bool expected_local = true);

  bool AllocateSpace(int64_t num_bytes, int64_t* offset) override;

  io::DiskFile* GetWriteFile() override;

  Status Remove() override;
};

/// TmpFileRemote 是 TmpFile 的派生类，用于提供处理远程文件系统临时目录中物理文件的方法。
///
/// 锁定：
/// 对于远程临时文件，应用锁定机制以保证对文件的写入、读取或上传的安全。
/// 远程临时文件可以有两个 DiskFile：本地缓冲区和远程文件。
/// 每个 DiskFile 拥有两种类型的锁：文件锁和状态锁。
/// DiskFile::physical_file_lock_ -- 文件锁
/// DiskFile::status_lock_ -- 状态锁
/// 对于文件删除操作，需要唯一的文件锁。对于文件上的其他操作（如读取或写入），需要共享文件锁以保护文件免于删除。
/// 对于状态转换，线程需要持有状态锁，详细信息可在 TmpFileRemote 的头部找到。
/// 如果操作需要来自两个 DiskFile 的锁，例如上传操作，则获取锁的顺序必须从本地文件到远程文件，以避免死锁，并且文件锁需要在状态锁之前获取。
///
/// 默认情况下，有两个 DiskFile：本地缓冲区和远程文件，用于管理临时文件的状态。对于每个 DiskFile，有三种状态：InWriting/Persisted/Deleted，详细信息可在 DiskFile 的头部找到。
/// 假设：
/// 本地缓冲区：InWriting A, Persisted B, Deleted C
/// 远程文件：InWriting D, Persisted E, Deleted F
/// 远程临时文件的正常状态转换过程应为：
/// AD ---> BD 本地缓冲区文件关闭
/// BD ---> BE 上传完成
/// BE ---> CE 本地缓冲区被驱逐
/// 任何状态 ---> CF 临时文件销毁
/// 大多数状态转换在 DiskIoMgr 中完成，当 IO 操作完成时。
/// 对于读取或更改状态，应获取特定 DiskFile 的状态锁。
class TmpFileRemote : public TmpFile {
 public:
  TmpFileRemote(TmpFileGroup* file_group, TmpFileMgr::DeviceId device_id,
                const std::string& path, const std::string& local_buffer_path,
                bool expected_local = false, const char* url = nullptr);
  ~TmpFileRemote();

  bool AllocateSpace(int64_t num_bytes, int64_t* offset) override;

  io::DiskFile* GetWriteFile() override;

  TmpDir* GetLocalBufferDir() const;

  Status Remove() override;

  /// 返回用于读取的缓冲区文件句柄。
  /// 如果本地文件未被驱逐，则立即返回。
  /// 如果本地文件被驱逐且启用批量读取，则可能异步发送请求从远程获取块到内存。
  io::DiskFile* GetReadBufferFile(int64_t offset);

  /// 向磁盘队列发送请求，从远程文件系统异步获取块。
  /// 如果内容在缓冲区块中，则将 "fetched" 设置为 true。否则，
  /// 调用者应从远程文件系统获取页面。
  void AsyncFetchReadBufferBlock(io::DiskFile* read_buffer_file,
                                 io::MemBlock* read_buffer_block, int buffer_idx,
                                 bool* fetched);

  /// 从文件偏移量获取读取缓冲区块索引。
  int GetReadBufferIndex(int64_t offset);

  /// 增加已从缓冲区块读取的页面计数器。
  /// 如果块的所有页面都已被读取，则返回 true。
  bool IncrementReadPageCount(int buffer_idx);

  /// 尝试删除缓冲区块并释放预留。
  template <typename T>
  void TryDeleteReadBuffer(const T& lock, int buffer_idx);

  /// 使用独占锁尝试删除缓冲区块并释放预留。
  void TryDeleteReadBufferExcl(int buffer_idx) {
    std::unique_lock<boost::shared_mutex> lock(*(disk_buffer_file_->GetFileLock()));
    TryDeleteReadBuffer(lock, buffer_idx);
  }

  /// 使用共享锁尝试删除缓冲区块并释放预留。
  /// 除非确定在删除期间无人访问特定读取缓冲区块且场景需要高性能，否则使用独占锁。
  void TryDeleteMemReadBufferShared(int buffer_idx) {
    boost::shared_lock<boost::shared_mutex> lock(*(disk_buffer_file_->GetFileLock()));
    TryDeleteReadBuffer(lock, buffer_idx);
  }

  /// 返回文件大小。
  int64_t file_size() const { return file_size_; }

  /// 返回磁盘缓冲区文件指针。
  io::DiskFile* DiskBufferFile() { return disk_buffer_file_.get(); }

  /// 将 at_capacity_ 指示器设置为 true。
  void SetAtCapacity() {
    DCHECK(!at_capacity_);
    at_capacity_ = true;
  }

  /// 设置文件是否已入队。
  /// 该函数是线程安全的。
  void SetEnqueued(bool is_enqueued) {
    std::lock_guard<SpinLock> l(lock_);
    DCHECK(is_enqueued != enqueued_);
    enqueued_ = is_enqueued;
  }

  /// 设置文件的缓冲区已返回到池中。
  /// 该函数是线程安全的。
  void SetBufferReturned() {
    std::lock_guard<SpinLock> l(lock_);
    DCHECK(!buffer_returned_);
    buffer_returned_ = true;
  }

  /// 返回文件是否已入队到临时文件可用池中。
  /// 该函数是线程安全的。
  bool is_enqueued() {
    std::lock_guard<SpinLock> l(lock_);
    return enqueued_;
  }

  /// 返回缓冲区是否已返回到临时文件可用池中。
  /// 该函数是线程安全的。
  bool is_buffer_returned() {
    std::lock_guard<SpinLock> l(lock_);
    return buffer_returned_;
  }

  /// 设置标志以指示文件即将被删除。
  void SetToDeleteFlag(bool to_delete = true) {
    disk_buffer_file_->SetToDeleteFlag(to_delete);
    disk_file_->SetToDeleteFlag(to_delete);
  }

  /// 返回临时文件的磁盘 ID。
  /// 如果 is_file_op 为 true，则返回专门用于文件操作的磁盘 ID。
  int disk_id(bool is_file_op = false) const override {
    if (!is_file_op) return disk_id_;
    return disk_id_file_op_;
  }

 private:
  friend class TmpWriteHandle;
  friend class TmpFileMgr;
  friend class TmpFileGroup;
  friend class TmpFileBufferPool;
  friend class TmpFileMgrTest;

  /// 临时文件的默认文件大小，但如果写入文件的最后一个页面的尺寸超过剩余空间，则实际文件大小可能略大。
  int64_t file_size_ = 0;

  /// 远程文件读取缓冲区块的默认大小。
  /*
     如果文件小于16M，就取值文件大小。否则最大16M，
  */
  int64_t read_buffer_block_size_ = 0;

  /// 用于文件操作的磁盘 ID。
  int disk_id_file_op_ = 0;

  /// HDFS 文件的 mtime 的伪值。
  const int64_t mtime_{100000};

  // 磁盘缓冲区文件的指针，即远程磁盘文件的本地缓冲区。该缓冲区用于写入。
  std::unique_ptr<io::DiskFile> disk_buffer_file_;

  /// 用于连接到远程暂存路径的 HDFS 连接。
  hdfsFS hdfs_conn_;

  /// 如果分配的空间等于或刚好超过默认文件大小时，无法再分配空间，则 at_capacity_ 设置为 true。
  bool at_capacity_ = false;

  /// 用于文件上传的范围。
  std::unique_ptr<io::RemoteOperRange> upload_range_;

  /// 用于从远程文件系统执行获取操作的范围。
  std::vector<std::unique_ptr<io::RemoteOperRange>> fetch_ranges_;

  /// 保护以下成员。
  SpinLock lock_;

  /// 指示文件是否已入队到池中。用于调试。
  bool enqueued_ = false;

  /// 如果文件的缓冲区已返回到池中，则为 true。我们假设缓冲区仅返回一次，并且仅在预留缓冲区空间时需要返回。
  bool buffer_returned_ = false;

  // 每个读取缓冲区的已读取页面数。
  std::unique_ptr<int64_t[]> disk_read_page_cnts_;

  // 返回读取缓冲区块的起始偏移量。
  int64_t GetReadBuffStartOffset(int buffer_idx) {
    DCHECK(disk_buffer_file_ != nullptr);
    return disk_buffer_file_->GetReadBuffStartOffset(buffer_idx);
  }

  // 返回读取缓冲区块的页面数。
  int64_t GetReadBuffPageCount(int buffer_idx) {
    DCHECK(disk_buffer_file_ != nullptr);
    return disk_buffer_file_->GetReadBuffPageCount(buffer_idx);
  }

  /// 缓冲区索引的内部 DCHECK。
  void DCheckReadBufferIdx(int buffer_idx) {
    DCHECK_LT(buffer_idx, file_group_->tmp_file_mgr()->GetNumReadBuffersPerFile());
    DCHECK_GE(buffer_idx, 0);
  }
};

/// TmpFileDummy 是 TmpFile 的派生类，用于虚拟分配，仅在 TmpFileBufferPool 中使用。
class TmpFileDummy : public TmpFile {
 public:
  TmpFileDummy() : TmpFile(nullptr, -1, "") { disk_type_ = io::DiskFileType::DUMMY; }

  bool AllocateSpace(int64_t num_bytes, int64_t* offset) override { return true; }

  io::DiskFile* GetWriteFile() override { return nullptr; }

  Status Remove() override { return Status::OK(); }
};

/// 配置的临时目录，TmpFileMgr 在其中分配文件。
class TmpDir {
 public:
  TmpDir(const std::string& raw_path) : raw_path_(raw_path) {}

  virtual ~TmpDir() {}

  /// 解析原始路径并识别暂存目录选项。
  Status Parse();

  /// 验证暂存路径并创建目录。
  virtual Status VerifyAndCreate(MetricGroup* metrics, vector<bool>* is_tmp_dir_on_disk,
                                 bool need_local_buffer_dir,
                                 TmpFileMgr* tmp_mgr) = 0;

  /// 获取目录路径的连接。仅用于远程目录。
  virtual Status GetConnection(TmpFileMgr* tmp_mgr, hdfsFS* conn) = 0;

  int64_t bytes_limit() { return bytes_limit_; }

  int priority() { return priority_; }

  const string& path() { return path_; }

  IntGauge* bytes_used_metric() const { return bytes_used_metric_; }

  virtual bool is_local() { return false; }

 private:
  friend class TmpFileMgr;
  friend class TmpDirHdfs;
  friend class TmpDirS3;
  friend class TmpDirLocal;

  /// 临时目录的原始路径。
  const std::string raw_path_;

  /// 临时目录的解析后的原始路径，例如，已修剪。
  std::string parsed_raw_path_;

  /// 临时目录的完整路径。
  std::string path_;

  /// 应写入该路径的字节限制。如果没有限制，则设置为 int64_t 的最大值。
  /*
    TmpDirLocal和TmpDirS3和TmpDirHdfs都有各自的限制。
  */
  int64_t bytes_limit_ = numeric_limits<int64_t>::max();

  /// 暂存目录优先级。
  int priority_ = numeric_limits<int>::max();

  /// 该临时目录当前使用的暂存字节。
  /*
    记录的是该Dir对象下的文件占用的物理磁盘大小。
    punch hole时会减少该值。
  */
  IntGauge* bytes_used_metric_;

  /// Parse() 的辅助函数，用于解析原始路径并生成暂存目录的完整路径。
  /// "Tokens" 将包含从原始路径解析的 {path, [bytes_limit, [priority]]} 令牌。
  virtual Status ParsePathTokens(std::vector<string>& tokens) = 0;

  /// Parse() 的辅助函数，用于解析暂存目录的字节限制。
  Status ParseByteLimit(const string& byte_limit);

  /// Parse() 的辅助函数，用于解析暂存目录的优先级。
  Status ParsePriority(const string& priority);
};

class TmpDirLocal : public TmpDir {
 public:
  TmpDirLocal(const std::string& path) : TmpDir(path) {}

  Status VerifyAndCreate(MetricGroup* metrics, vector<bool>* is_tmp_dir_on_disk,
                         bool need_local_buffer_dir, TmpFileMgr* tmp_mgr) override;

  Status GetConnection(TmpFileMgr* tmp_mgr, hdfsFS* conn) override {
    DCHECK(false) << "GetConnection() is not supported for a local temporary dir";
    return Status("GetConnection() is not supported for a local temporary dir");
  }

  bool is_local() override { return true; }

 private:
  /// VerifyAndCreate() 的辅助函数，用于创建本地暂存目录。
  Status CreateLocalDirectory(MetricGroup* metrics, vector<bool>* is_tmp_dir_on_disk,
                              bool need_local_buffer_dir, int disk_id,
                              TmpFileMgr* tmp_mgr);

  /// CreateLocalDirectory() 的辅助函数，用于记录配置的本地暂存目录。
  void LogScratchLocalDirectoryInfo(bool is_local_buffer_dir, int disk_id);

  Status ParsePathTokens(std::vector<string>& tokens) override;
};
/*
   1:remote dir:
      remote-dirbase/hostname/backend-id/query-id/unique_id (参见ConstructRemoteDirPath)
   2:new_file_path(即remote file)
      remote-dirbase/hostname/backend-id/query-id/unique_id/impala-scratc-unique-name-xxx
  3：local_buffer_dir:
     dir1/
  4：new_file_path_local:
     dir1/impala-scratc-unique-name-xxx
     
*/
class TmpDirS3 : public TmpDir {
 public:
  TmpDirS3(const std::string& path) : TmpDir(path) {}

  Status VerifyAndCreate(MetricGroup* metrics, vector<bool>* is_tmp_dir_on_disk,
                         bool need_local_buffer_dir, TmpFileMgr* tmp_mgr) override;

  Status GetConnection(TmpFileMgr* tmp_mgr, hdfsFS* conn) override;

 private:
  Status ParsePathTokens(std::vector<string>& tokens) override;
};

class TmpDirHdfs : public TmpDir {
 public:
  TmpDirHdfs(const std::string& path) : TmpDir(path) {}

  Status VerifyAndCreate(MetricGroup* metrics, vector<bool>* is_tmp_dir_on_disk,
                         bool need_local_buffer_dir, TmpFileMgr* tmp_mgr) override;

  Status GetConnection(TmpFileMgr* tmp_mgr, hdfsFS* conn) override;

 private:
  Status ParsePathTokens(std::vector<string>& tokens) override;
};

/// 临时文件缓冲池允许临时文件将其缓冲区返回到池中，并可以被驱逐以为其他文件腾出空间。
/// 该池还为写入范围提供异步方式，在发送到磁盘队列进行写入之前等待可用空间预留。
class TmpFileBufferPool {
 public:
  TmpFileBufferPool(TmpFileMgr* tmp_file_mgr);
  ~TmpFileBufferPool();

  // 循环为放入队列 write_ranges_ 的范围预留空间，
  // 空间通过调用 DequeueTmpFilesPool() 从 tmp_files_avail_pool_ 获取。
  // 一旦预留了空间，属于同一文件的写入范围将被发送到磁盘队列进行写入。
  // 如果在发送到磁盘队列期间发生错误，则将使用状态调用写入范围的回调函数。
  void TmpFileSpaceReserveThreadLoop();

  /// 将写入范围入队等待缓冲区空间预留。同一文件的所有范围将被放入 write_ranges_to_add_ 映射中，
  /// 等待预留完成，然后发送到磁盘队列。特别是，文件的第一写入范围（偏移量为 0）用于执行预留。
  Status EnqueueWriteRange(io::WriteRange* range, TmpFile* file);

  /// 当 "io_ctx" 所属的 TmpFileGroup 关闭时，调用该函数移除属于 "io_ctx" 的所有入队写入范围。
  void RemoveWriteRanges(io::RequestContext* io_ctx);

  // 将其缓冲区文件可用被驱逐的临时文件入队。
  void EnqueueTmpFilesPool(std::shared_ptr<TmpFile>& tmp_file, bool front);

  // 从可用池中出队一个临时文件，其缓冲区应可用被驱逐，为其他文件的缓冲区腾出空间。
  Status DequeueTmpFilesPool(std::shared_ptr<TmpFile>* tmp_file, bool quick_return);

  // 在销毁前关闭池。
  void ShutDown();

 private:
  friend class TmpFileMgr;
  friend class TmpFileMgrTest;

  /// TmpFileBufferPool 所属的 TmpFileMgr。
  TmpFileMgr* tmp_file_mgr_ = nullptr;

  /// 存储远程临时文件指针的池，其缓冲区可用被驱逐。
  /// 文件在以下情况下入队：
  /// 1. 文件已上传到远程目录。
  /// 2. 文件在不卸载的情况下被删除。
  /// 文件通过调用 EnqueueTmpFilesPool() 入队，并通过 DequeueTmpFilesPool() 出队。
  std::list<std::shared_ptr<TmpFile>> tmp_files_avail_pool_;

  /// 临时文件可用池的条件变量，用于等待可用临时文件被驱逐。
  /// DequeueTmpFilesPool() 中的 Wait() 和 EnqueueTmpFilesPool() 中的 NotifyOne()。
  ConditionVariable tmp_files_available_cv_;

  /// 保护临时文件可用池成员。
  std::mutex tmp_files_avail_pool_lock_;

  /// 保护以下成员的锁。
  std::mutex lock_;

  /// 条件变量，用于向空间预留线程发出信号，指示有工作要做或线程应关闭。
  /// 当写入范围添加到队列 write_ranges_ 时，将唤醒空间预留线程。
  ConditionVariable work_available_;

  /// 等待缓冲区空间预留的写入范围。
  /// 键是范围写入的 DiskFile 指针。
  std::unordered_map<io::DiskFile*, std::vector<io::WriteRange*>> write_ranges_to_add_;

  /// 记录 io_ctx 和磁盘文件的关系，在添加等待预留的写入范围时插入数据，
  /// 在 TmpFileGroup 关闭时请求上下文析构时移除。
  std::unordered_map<io::RequestContext*, std::unordered_set<io::DiskFile*>>
      io_ctx_to_file_set_map_;

  /// 等待缓冲区空间预留的写入范围（仅文件的第一范围）。
  std::list<io::WriteRange*> write_ranges_;

  /// 索引，用于将写入范围绑定到 write_ranges_ 中的迭代器及其 TmpFile。
  std::unordered_map<io::WriteRange*, std::pair<std::list<io::WriteRange*>::const_iterator, TmpFile*>>
      write_ranges_iterator_;

  /// 当前等待预留的写入范围。
  io::WriteRange* cur_write_range_ = nullptr;

  /// 当前等待预留的写入范围关联的临时文件。
  std::shared_ptr<TmpFile> cur_tmp_file_;

  /// 用于在没有 TmpFile 的情况下将空间返回到池的虚拟 TmpFile。
  std::shared_ptr<TmpFile> tmp_file_dummy_;

  /// 如果 TmpFileMgr 正在销毁，则为 true。
  bool shut_down_ = false;

  /// 用于记录出队等待时间的计时器指标。
  HistogramMetric* dequeue_timer_metric_ = nullptr;

  /// 辅助函数，用于向池中添加或移除与同一磁盘文件相关的所有写入范围。
  /// 当缓冲区文件的预留完成后调用该函数时，它将文件的所有范围添加到磁盘队列。
  /// 如果任何添加失败，则将范围的其他写回函数放入向量 "write_callbacks"，供调用者使用错误状态回调。
  /// 文件的所有范围都将从池中移除。
  /// 当写入任务被取消时调用该函数（发生在 TmpFileGroup 关闭时），则 is_cancelled 设置为 true，
  /// 该函数用于移除池中属于特定 disk_file 的所有范围。
  /// 调用者应持有 lock_。
  Status MoveWriteRangesHelper(io::DiskFile* disk_file,
                               std::vector<TmpFileMgr::WriteDoneCallback>* write_callbacks,
                               bool is_cancelled);

  /// RemoveWriteRanges() 的内部函数。
  void RemoveWriteRangesInternal(io::RequestContext* io_ctx,
                                 std::vector<TmpFileMgr::WriteDoneCallback>* write_callbacks);
};

}  // namespace impala

#endif
```

---

好的，`TmpFileBufferPool` 是 Impala 中一个比较复杂但非常重要的组件，尤其在处理**远程临时文件（如 HDFS 或 S3）的溢出写**时。它的主要作用是**优化临时文件的写入性能和资源管理**，尤其是在涉及到缓冲区（buffer）的分配和回收、以及异步写入操作的协调时。

---

## TmpFileBufferPool 的核心作用

`TmpFileBufferPool` 的主要作用可以概括为以下几点：

1.  **管理临时文件写缓冲区：** 尤其针对远程文件，写操作通常会先写入一个本地缓冲区（local buffer），然后再异步上传到远程存储。`TmpFileBufferPool` 负责管理这些本地缓冲区的可用性。
2.  **实现异步空间预留：** 在写入数据到临时文件之前，需要先“预留”或“分配”一块缓冲区空间。当系统资源紧张时，可能无法立即获得空间。`TmpFileBufferPool` 提供一个机制，让写入请求可以**排队等待可用的缓冲区空间**，而不是立即失败或阻塞。
3.  **负载均衡和故障处理：** 通过池化和调度，它可以更好地在多个临时文件之间分配写入请求，并处理因磁盘空间不足或 I/O 错误导致的缓冲区回收和重试。
4.  **避免死锁和资源竞争：** 在复杂的异步 I/O 场景中，正确处理锁和竞争条件至关重要。`TmpFileBufferPool` 的设计考虑了这些问题，通过特定的锁顺序和状态管理来避免死锁。

---

## TmpFileBufferPool 的设计与工作原理

### 1. 缓冲区的概念

在理解 `TmpFileBufferPool` 之前，需要先理解 Impala 临时文件写入的两种主要模式：

* **本地文件 (TmpFileLocal)：** 直接写入本地磁盘。分配空间时，通常直接在逻辑上增加文件长度，实际磁盘分配由操作系统处理。
* **远程文件 (TmpFileRemote)：** 这是 `TmpFileBufferPool` 主要关注的场景。为了提高性能和可靠性，写入远程文件（如 HDFS、S3）通常不是直接进行的。
    * **写入流程：** 数据首先写入一个**本地缓冲区文件 (`disk_buffer_file_`)**，当本地缓冲区达到一定大小或写入完成后，数据会从本地缓冲区**异步上传 (`upload_range_`)**到真正的远程文件 (`disk_file_`)。
    * **读取流程：** 类似地，从远程文件读取时，数据可能先被**异步预取 (`fetch_ranges_`)**到本地读取缓冲区，然后再供消费。

`TmpFileBufferPool` 主要管理的就是这些**本地缓冲区文件**（尤其是用于写入的缓冲区）。

### 2. 关键数据结构和成员

* **`tmp_files_avail_pool_` (`std::list<std::shared_ptr<TmpFile>>`)**:
    * 这是一个**可用的临时文件缓冲区池**。当一个 `TmpFileRemote` 的本地缓冲区完成数据上传，或者因为某些原因不再需要时（例如文件被删除），它的本地缓冲区就可以被“回收”并添加到这个池中。
    * 这个池中的文件（`TmpFile` 对象）的本地缓冲区被认为是可用于其他写入请求的。
    * 由 `tmp_files_available_cv_` (条件变量) 和 `tmp_files_avail_pool_lock_` (互斥锁) 保护。
* **`write_ranges_to_add_` (`std::unordered_map<io::DiskFile*, std::vector<io::WriteRange*>>`)**:
    * 存储所有**等待缓冲区空间预留**的写入范围 (`io::WriteRange`)。
    * `DiskFile*` 作为键，表示这些写入范围属于哪个物理文件。一个文件可能有多个等待写入的 `WriteRange`。
* **`write_ranges_` (`std::list<io::WriteRange*>`)**:
    * 这是一个更精简的队列，只包含**每个文件第一个等待预留空间的 `WriteRange`**。
    * 这个队列的目的是为了驱动 `TmpFileSpaceReserveThreadLoop()` 线程的工作，当有新的文件需要缓冲区时，其第一个 `WriteRange` 会被放入此队列。
* **`io_ctx_to_file_set_map_` (`std::unordered_map<io::RequestContext*, std::unordered_set<io::DiskFile*>>`)**:
    * 记录了 `io::RequestContext`（通常代表一个查询或操作）与它所关联的 `DiskFile` 之间的关系。
    * 用于在取消写入任务（例如查询结束）时，快速找到并移除所有相关的等待中的 `WriteRange`。
* **`work_available_` (`ConditionVariable`) 和 `lock_` (`std::mutex`)**:
    * `work_available_` 是一个条件变量，用于唤醒**空间预留线程** (`TmpFileSpaceReserveThreadLoop`)。当有新的 `WriteRange` 被 `EnqueueWriteRange()` 时，会通知这个条件变量。
    * `lock_` 保护 `write_ranges_to_add_` 和 `write_ranges_` 等队列的并发访问。
* **`TmpFileSpaceReserveThreadLoop()`**:
    * 这是一个**后台线程**，是 `TmpFileBufferPool` 的核心执行者。它会不断循环：
        1.  从 `write_ranges_` 队列中取出**等待预留空间**的 `WriteRange`（通常是某个文件的第一个写入请求）。
        2.  尝试从 `tmp_files_avail_pool_` 中**获取一个可用的 `TmpFile` 缓冲区**。如果池中没有，它会等待（通过 `tmp_files_available_cv_`）。
        3.  一旦获得缓冲区，它会将属于同一个文件的所有等待中的 `WriteRange` 从 `write_ranges_to_add_` 中取出，并将其提交给底层的 `DiskIoMgr` 进行实际的异步写入。

### 3. 工作流程概览

1.  **写入请求（`EnqueueWriteRange()`）:**
    * 当 Impala 需要将数据溢出到远程临时文件时，它会生成一个或多个 `io::WriteRange` 对象。
    * 这些 `WriteRange` 会被添加到 `TmpFileBufferPool` 的 `write_ranges_to_add_` 映射中，等待缓冲区空间。
    * 如果这是该文件的第一个写入请求，它还会被添加到 `write_ranges_` 队列中。
    * `EnqueueWriteRange()` 会通知 `work_available_` 条件变量，唤醒后台的空间预留线程。

2.  **空间预留线程（`TmpFileSpaceReserveThreadLoop()`）:**
    * 线程被唤醒后，会从 `write_ranges_` 队列中取出一个 `WriteRange`。
    * 它会尝试从 `tmp_files_avail_pool_` 中**“借用”**一个 `TmpFile`（实际上是它的本地缓冲区）。如果池中没有可用的缓冲区，线程会在这里等待。
    * 一旦获得缓冲区，线程会调用 `MoveWriteRangesHelper()`。
    * `MoveWriteRangesHelper()` 会把所有属于当前文件的 `WriteRange` (从 `write_ranges_to_add_` 获取) 提交给 `DiskIoMgr` 的磁盘队列，进行真正的异步写入操作。

3.  **缓冲区回收（`EnqueueTmpFilesPool()`）：**
    * 当 `DiskIoMgr` 完成了对 `TmpFileRemote` 本地缓冲区的数据写入，并成功异步上传到远程文件后，它会通知 `TmpFileMgr`。
    * `TmpFileMgr` 继而会调用 `TmpFileBufferPool::EnqueueTmpFilesPool()`，将这个 `TmpFile`（及其本地缓冲区）**重新放回 `tmp_files_avail_pool_`**，使其可以被其他等待写入的任务复用。
    * `EnqueueTmpFilesPool()` 会通知 `tmp_files_available_cv_` 条件变量，唤醒任何正在等待可用缓冲区的线程。

4.  **缓冲区获取（`DequeueTmpFilesPool()`）：**
    * 当空间预留线程需要一个缓冲区时，它会调用 `DequeueTmpFilesPool()`。
    * 如果池中有可用的 `TmpFile`，它会立即返回。
    * 如果池中没有，它会等待 `tmp_files_available_cv_`，直到有缓冲区可用。

5.  **取消与清理（`RemoveWriteRanges()`）：**
    * 当查询结束或被取消时，`TmpFileGroup` 会调用 `RemoveWriteRanges()`。
    * 这个函数会遍历 `io_ctx_to_file_set_map_`，找到所有属于该 `io_ctx` 的 `DiskFile`，并移除所有相关的等待中的 `WriteRange`，防止它们继续阻塞或被处理。这确保了资源的及时释放和避免僵尸任务。

### 4. 锁机制

`TmpFileRemote` 内部使用了复杂的锁机制来保证并发安全：

* **`DiskFile::physical_file_lock_` (文件锁):** 用于保护对文件本身的物理操作（如删除）。写入和读取通常需要共享锁，删除需要排他锁。
* **`DiskFile::status_lock_` (状态锁):** 用于保护文件状态的转换（如 `InWriting`, `Persisted`, `Deleted`）。
* **锁顺序：** 为了避免死锁，规定了严格的锁获取顺序：**本地文件锁 -> 远程文件锁**，并且**文件锁 -> 状态锁**。

`TmpFileBufferPool` 自己的成员 (`tmp_files_avail_pool_`, `write_ranges_to_add_` 等) 也通过 `std::mutex` 和 `ConditionVariable` 进行保护。

---

## 为什么需要 `TmpFileBufferPool`？

在没有 `TmpFileBufferPool` 的情况下，处理远程临时文件溢出可能会面临以下挑战：

* **性能瓶颈：** 直接对远程文件进行大量小写入可能效率低下。使用本地缓冲区可以聚合写入，减少网络往返。
* **资源竞争：** 多个查询或任务可能同时竞争有限的本地缓冲区资源。如果没有一个集中的管理器，容易导致资源耗尽或饥饿。
* **阻塞查询：** 如果没有可用的缓冲区，写入操作可能会阻塞整个查询，降低并发性。异步预留机制允许查询在等待缓冲区的同时继续执行其他任务。
* **复杂的状态管理：** 远程文件的写入、上传、本地缓冲区回收等是一个多阶段的异步过程，错误处理和状态转换非常复杂。`TmpFileBufferPool` 将这些复杂性封装起来。
* **故障恢复：** 当磁盘空间不足或远程存储出现问题时，`TmpFileBufferPool` 可以通过其调度和回收机制，更好地处理这些异常情况，提供一定的弹性。

总之，`TmpFileBufferPool` 是 Impala 实现高效、健壮的**远程临时文件溢出**写入能力的关键组件。它通过池化、异步预留、负载均衡和精细的锁管理，确保了在复杂并发环境下临时文件操作的性能和稳定性。

---
*********************************************
我补充下实现：

TmpFileBufferPool::TmpFileBufferPool(TmpFileMgr* tmp_file_mgr)

  : tmp_file_mgr_(tmp_file_mgr) {

  tmp_file_dummy_.reset(new TmpFileDummy());

}



TmpFileBufferPool::~TmpFileBufferPool() {

  DCHECK(shut_down_);

}



void TmpFileBufferPool::ShutDown() {

  {

    unique_lock<mutex> l(lock_);

    shut_down_ = true;

  }

  // Wake up the waiting thread.

  work_available_.NotifyAll();

}



void TmpFileBufferPool::TmpFileSpaceReserveThreadLoop() {

  while (true) {

    {

      unique_lock<mutex> l(lock_);

      while (!shut_down_ && write_ranges_.empty()) {

        // Wait if there are no ranges in the queue.

        work_available_.Wait(l);

      }

      if (shut_down_) return;

      DCHECK(!write_ranges_.empty());

      cur_write_range_ = write_ranges_.front();

      write_ranges_.pop_front();

      DCHECK(cur_write_range_ != nullptr);



      // Find out the TmpFile which the current range is associated with, and store the

      // shared_ptr of the file to cur_tmp_file_ in case it is deconstructed while waiting

      // for reservation.

      auto it = write_ranges_iterator_.find(cur_write_range_);

      DCHECK(it != write_ranges_iterator_.end());

      TmpFile* tmp_file = it->second.second;

      cur_tmp_file_ = tmp_file->FileGroup()->FindTmpFileSharedPtr(tmp_file);

      DCHECK(cur_tmp_file_ != nullptr);

      DCHECK_EQ(cur_write_range_->disk_file(), cur_tmp_file_->GetWriteFile());

      write_ranges_iterator_.erase(it);

    }



    // Reserve space from the tmp_files_avail_pool_. The process could need a long wait.

    Status status = tmp_file_mgr_->ReserveLocalBufferSpace(false);

    vector<TmpFileMgr::WriteDoneCallback> write_callbacks;

    {

      unique_lock<mutex> lock(lock_);

      if (status.ok()) {

        DCHECK(cur_tmp_file_ != nullptr);

        cur_tmp_file_->GetWriteFile()->SetSpaceReserved();

        if (cur_write_range_ != nullptr) {

          // Send all of the writes of the same disk file to the disk queue.

          status = MoveWriteRangesHelper(

              cur_write_range_->disk_file(), &write_callbacks, false);

        } else {

          // If the current range becomes a nullptr, it must be set by

          // RemoveWriteRanges(). In this case, the io_ctx which the range belongs to is

          // cancelled, and all the writes using that io_ctx are already cancelled. So, we

          // are safe to return the TmpFile to the pool to recycle the buffer space.

          EnqueueTmpFilesPool(cur_tmp_file_, true);

        }

      } else if (!status.ok() && cur_write_range_ != nullptr) {

        // Cancel the spilling if fails to reserve the buffer.

        RemoveWriteRangesInternal(cur_write_range_->io_ctx(), &write_callbacks);

        status = Status::CancelledInternal(

            Substitute("TmpFileBufferPool because: $0", status.GetDetail()).c_str());

      }

      cur_write_range_ = nullptr;

      cur_tmp_file_.reset();

    }

    for (const TmpFileMgr::WriteDoneCallback& write_callback : write_callbacks) {

      write_callback(status);

    }

  }

}



Status TmpFileBufferPool::MoveWriteRangesHelper(DiskFile* disk_file,

    vector<TmpFileMgr::WriteDoneCallback>* write_callbacks, bool is_cancelled) {

  Status status = Status::OK();

  auto write_ranges_it = write_ranges_to_add_.find(disk_file);

  if (write_ranges_it != write_ranges_to_add_.end()) {

    auto write_range_it = write_ranges_it->second.begin();

    while (write_range_it != write_ranges_it->second.end()) {

      auto range = *write_range_it;

      DCHECK(range != nullptr);

      if (status.ok() && !is_cancelled) {

        status = range->io_ctx()->AddWriteRange(range);

      } else {

        write_callbacks->push_back(range->callback());

        if (is_cancelled && range->offset() == 0) {

          // If is_cancelled is set, try to remove the range from the write_ranges list.

          // If the range hasn't been popped, it must still be in the write_ranges list.

          if (cur_write_range_ != range) {

            auto key_range_it = write_ranges_iterator_.find(range);

            DCHECK(key_range_it != write_ranges_iterator_.end());

            DCHECK_EQ(*(key_range_it->second.first), range);

            write_ranges_.erase(key_range_it->second.first);

            write_ranges_iterator_.erase(key_range_it);

          }

        }

      }

      write_range_it = write_ranges_it->second.erase(write_range_it);

    }

    write_ranges_to_add_.erase(write_ranges_it);

  }

  return status;

}



Status TmpFileBufferPool::EnqueueWriteRange(io::WriteRange* range, TmpFile* tmp_file) {

  Status status = Status::OK();

  {

    unique_lock<mutex> write_range_list_lock(lock_);

    DCHECK(range != nullptr);

    DCHECK(range->disk_file() != nullptr);

    DCHECK(range->io_ctx() != nullptr);

    if (range->disk_file()->IsSpaceReserved()) {

      // If the space is reserved, send the range to the DiskQueue.

      return range->io_ctx()->AddWriteRange(range);

    } else if (range->io_ctx()->IsCancelled()) {

      // If the io_ctx is cancelled, nofity the caller to cancel the query.

      return TMP_FILE_BUFFER_POOL_CONTEXT_CANCELLED;

    } else {

      io_ctx_to_file_set_map_[range->io_ctx()].insert(range->disk_file());

      write_ranges_to_add_[range->disk_file()].emplace_back(range);

    }

    // Put the first range of a file to the queue for waiting for the available space,

    // the ranges in the queue would be popped one by one, when the space is reserved,

    // all ranges of the file are added to the DiskQueue by io_ctx.

    if (range->offset() == 0) {

      write_ranges_.emplace_back(range);

      DCHECK(tmp_file != nullptr);

      write_ranges_iterator_[range] =

          std::make_pair(prev(write_ranges_.cend()), tmp_file);

    }

  }

  work_available_.NotifyAll();

  return status;

}



void TmpFileBufferPool::RemoveWriteRangesInternal(

    RequestContext* io_ctx, vector<TmpFileMgr::WriteDoneCallback>* write_callbacks) {

  auto file_set_it = io_ctx_to_file_set_map_.find(io_ctx);

  if (file_set_it != io_ctx_to_file_set_map_.end()) {

    auto file_it = file_set_it->second.begin();

    while (file_it != file_set_it->second.end()) {

      DCHECK(*file_it != nullptr);

      // Remove all the ranges belonging to the file, and fetch the callback

      // functions of the ranges.

      Status status = MoveWriteRangesHelper(*file_it, write_callbacks, true);

      DCHECK_OK(status);

      if (cur_write_range_ != nullptr && *file_it == cur_write_range_->disk_file()) {

        // Set the current write range to nullptr if the TmpFileGroup is closing to

        // notify the reservation thread (it is waiting for the reservation) that the

        // space is no longer needed for the write range.

        cur_write_range_ = nullptr;

      }

      file_it = file_set_it->second.erase(file_it);

    }

    io_ctx_to_file_set_map_.erase(file_set_it);

  }

}



void TmpFileBufferPool::RemoveWriteRanges(RequestContext* io_ctx) {

  DCHECK(io_ctx != nullptr);

  vector<TmpFileMgr::WriteDoneCallback> write_callbacks;

  {

    unique_lock<mutex> lock(lock_);

    RemoveWriteRangesInternal(io_ctx, &write_callbacks);

  }

  for (const TmpFileMgr::WriteDoneCallback& write_callback : write_callbacks) {

    write_callback(TMP_FILE_BUFFER_POOL_CONTEXT_CANCELLED);

  }

}



void TmpFileBufferPool::EnqueueTmpFilesPool(shared_ptr<TmpFile>& tmp_file, bool front) {

  DCHECK(tmp_file != nullptr);

  {

    unique_lock<mutex> buffer_lock(tmp_files_avail_pool_lock_);

    if (tmp_file->disk_type() != io::DiskFileType::DUMMY) {

      TmpFileRemote* tmp_file_remote = static_cast<TmpFileRemote*>(tmp_file.get());

      if (tmp_file_remote->is_enqueued()) return;

      tmp_file_remote->SetEnqueued(true);

      tmp_file_remote->SetBufferReturned();

    }

    if (front) {

      tmp_files_avail_pool_.push_front(tmp_file);

    } else {

      tmp_files_avail_pool_.push_back(tmp_file);

    }

    tmp_file_mgr_->GetLocalBufferDir()->bytes_used_metric()->Increment(

        -1 * tmp_file_mgr_->GetRemoteTmpFileSize());

  }

  tmp_files_available_cv_.NotifyOne();

}



Status TmpFileBufferPool::DequeueTmpFilesPool(

    shared_ptr<TmpFile>* tmp_file, bool quick_return) {

  DCHECK(tmp_file != nullptr);

  DCHECK(dequeue_timer_metric_ != nullptr);

  ScopedHistogramTimer wait_timer(dequeue_timer_metric_);

  unique_lock<mutex> buffer_lock(tmp_files_avail_pool_lock_);

  // If quick return is set and no buffer is available, return immediately.

  if (quick_return && tmp_files_avail_pool_.empty()) {

    return TMP_FILE_MGR_NO_AVAILABLE_FILE_TO_EVICT;

  }

  while (tmp_files_avail_pool_.empty()) {

    // Wait if there is no temporary file on the queue.

    // If timeout, return immediately.

    if (!tmp_files_available_cv_.WaitFor(

            buffer_lock, tmp_file_mgr_->GetSpillBufferWaitTimeout())) {

      return Status(Substitute("Timeout waiting for a local buffer in $0 seconds",

          tmp_file_mgr_->GetSpillBufferWaitTimeout() / MICROS_PER_SEC));

    };

  }

  DCHECK(!tmp_files_avail_pool_.empty());

  *tmp_file = tmp_files_avail_pool_.front();

  tmp_files_avail_pool_.pop_front();

  DCHECK(*tmp_file != nullptr);

  if ((*tmp_file)->disk_type() != io::DiskFileType::DUMMY) {

    TmpFileRemote* tmp_file_remote = static_cast<TmpFileRemote*>(tmp_file->get());

    // Assert the default size remains the same in case the object is corrupted.

    DCHECK_EQ(tmp_file_remote->file_size_, tmp_file_mgr_->GetRemoteTmpFileSize());

    DCHECK(tmp_file_remote->is_enqueued());

    tmp_file_remote->SetEnqueued(false);

  }

  tmp_file_mgr_->GetLocalBufferDir()->bytes_used_metric()->Increment(

      tmp_file_mgr_->GetRemoteTmpFileSize());

  return Status::OK();

}
-----

好的，有了 `TmpFileBufferPool` 的具体实现代码，我们可以更深入地了解它的工作机制。这些代码展示了如何处理并发、等待、调度和资源管理。

-----

## TmpFileBufferPool 构造函数与析构函数

```cpp
TmpFileBufferPool::TmpFileBufferPool(TmpFileMgr* tmp_file_mgr)
  : tmp_file_mgr_(tmp_file_mgr) {
  tmp_file_dummy_.reset(new TmpFileDummy());
}

TmpFileBufferPool::~TmpFileBufferPool() {
  DCHECK(shut_down_);
}
```

  * **构造函数：**
      * 接收一个 `TmpFileMgr` 指针，这是它的顶层管理器。
      * 初始化 `tmp_file_dummy_`。`TmpFileDummy` 是一个空的 `TmpFile` 实现，它的 `AllocateSpace` 和 `Remove` 方法都是空操作。它可能用于某些特殊场景，例如当需要一个 `TmpFile` 对象来填充某个结构但不需要实际文件操作时。
  * **析构函数：**
      * `DCHECK(shut_down_);` 强调了一个重要设计：在析构 `TmpFileBufferPool` 之前，必须先调用 `ShutDown()` 方法来停止后台线程。这是一种确保资源正确释放和避免竞态条件的好实践。

## ShutDown() 方法

```cpp
void TmpFileBufferPool::ShutDown() {
  {
    unique_lock<mutex> l(lock_);
    shut_down_ = true;
  }
  // Wake up the waiting thread.
  work_available_.NotifyAll();
}
```

  * 这是一个\*\*优雅停机（graceful shutdown）\*\*的机制。
  * 它首先获取 `lock_`（保护 `write_ranges_` 队列的锁），将 `shut_down_` 标志设置为 `true`。
  * 然后，它调用 `work_available_.NotifyAll()` 来**唤醒所有正在等待 `work_available_` 条件变量的线程**。这包括 `TmpFileSpaceReserveThreadLoop()` 线程。一旦线程被唤醒，它会检查 `shut_down_` 标志，发现为 `true` 后就会退出循环并结束。
  * 这是确保后台线程在程序退出时能够安全退出的标准模式。

## TmpFileSpaceReserveThreadLoop() 核心线程逻辑

```cpp
void TmpFileBufferPool::TmpFileSpaceReserveThreadLoop() {
  while (true) {
    {
      unique_lock<mutex> l(lock_);
      while (!shut_down_ && write_ranges_.empty()) {
        // Wait if there are no ranges in the queue.
        work_available_.Wait(l); // 线程等待，直到有工作或被关闭
      }
      if (shut_down_) return; // 线程退出
      DCHECK(!write_ranges_.empty());

      // 从队列头部取出当前需要处理的 WriteRange
      cur_write_range_ = write_ranges_.front();
      write_ranges_.pop_front();
      DCHECK(cur_write_range_ != nullptr);

      // 找到与当前 WriteRange 关联的 TmpFile 的共享指针
      auto it = write_ranges_iterator_.find(cur_write_range_);
      DCHECK(it != write_ranges_iterator_.end());
      TmpFile* tmp_file = it->second.second;
      // 从 TmpFileGroup 获取 TmpFile 的 shared_ptr，确保在等待期间文件不被析构
      cur_tmp_file_ = tmp_file->FileGroup()->FindTmpFileSharedPtr(tmp_file);
      DCHECK(cur_tmp_file_ != nullptr);
      DCHECK_EQ(cur_write_range_->disk_file(), cur_tmp_file_->GetWriteFile());
      write_ranges_iterator_.erase(it); // 移除映射条目
    }

    // 在没有锁的情况下执行可能长时间阻塞的操作：预留本地缓冲区空间
    Status status = tmp_file_mgr_->ReserveLocalBufferSpace(false);
    vector<TmpFileMgr::WriteDoneCallback> write_callbacks;
    {
      unique_lock<mutex> lock(lock_); // 重新获取锁以更新内部状态和调度写入

      if (status.ok()) {
        // 成功预留空间
        DCHECK(cur_tmp_file_ != nullptr);
        cur_tmp_file_->GetWriteFile()->SetSpaceReserved(); // 标记空间已预留

        if (cur_write_range_ != nullptr) {
          // 将同一个文件所有等待中的 WriteRange 发送到 DiskQueue
          status = MoveWriteRangesHelper(
              cur_write_range_->disk_file(), &write_callbacks, false);
        } else {
          // 如果 cur_write_range_ 为 nullptr，说明它已被 RemoveWriteRanges() 取消
          // 此时，需要将当前文件（的缓冲区）返回到可用池中
          // 这确保了即使写入请求被取消，其占用的缓冲区也能被回收
          EnqueueTmpFilesPool(cur_tmp_file_, true);
        }
      } else if (!status.ok() && cur_write_range_ != nullptr) {
        // 预留空间失败
        // 取消与该 WriteRange 关联的整个 RequestContext 中的所有写入请求
        RemoveWriteRangesInternal(cur_write_range_->io_ctx(), &write_callbacks);
        status = Status::CancelledInternal(
            Substitute("TmpFileBufferPool because: $0", status.GetDetail()).c_str());
      }
      cur_write_range_ = nullptr; // 重置当前处理的 WriteRange 和 TmpFile
      cur_tmp_file_.reset();
    }
    // 在释放锁后，异步回调所有需要回调的 WriteDoneCallback
    // 这样做是为了避免在持有锁的情况下执行用户回调，防止死锁或长时间阻塞
    for (const TmpFileMgr::WriteDoneCallback& write_callback : write_callbacks) {
      write_callback(status);
    }
  }
}
```

  * **循环等待工作：** 线程在一个 `while(true)` 循环中运行。
      * 它首先获取 `lock_`。
      * 使用 `work_available_.Wait(l)` 在 `while (!shut_down_ && write_ranges_.empty())` 条件下等待。这意味着：
          * 如果 `shut_down_` 是 `true`，线程会退出。
          * 如果没有 `write_ranges_`（没有等待的写入请求），线程就会休眠，直到 `work_available_.NotifyAll()` 被调用（通常在新的 `WriteRange` 被 `EnqueueWriteRange` 时）。
  * **取出工作：** 一旦被唤醒，如果 `shut_down_` 是 `false` 且 `write_ranges_` 不为空，它会从 `write_ranges_` 的**头部**取出一个 `WriteRange` 来处理。
  * **`TmpFile` 共享指针：**
      * `cur_tmp_file_ = tmp_file->FileGroup()->FindTmpFileSharedPtr(tmp_file);` 这一步非常重要。
      * `cur_write_range_` 只是一个原始指针，不能保证 `TmpFile` 对象在线程等待 `ReserveLocalBufferSpace` 期间不会被析构。
      * 通过从 `TmpFileGroup` 获取其 `shared_ptr`，可以增加 `TmpFile` 的引用计数，确保其生命周期在预留空间操作完成前保持有效。
  * **`ReserveLocalBufferSpace()`：**
      * 这一行 `Status status = tmp_file_mgr_->ReserveLocalBufferSpace(false);` 是关键。这个调用是**阻塞的**。
      * 它会尝试从 `tmp_files_avail_pool_` 中获取一个可用的 `TmpFile`（实际上是它的本地缓冲区）。
      * 如果池中没有可用的缓冲区，这个函数会**等待**，直到有缓冲区可用（或者超时）。
      * 这个操作是在**不持有 `TmpFileBufferPool` 的 `lock_`** 的情况下进行的，这是避免死锁和提高并发性的重要设计。否则，如果它在持有 `lock_` 的情况下等待，其他线程将无法访问 `write_ranges_` 等数据结构。
  * **处理预留结果：**
      * **成功 (`status.ok()`)：**
          * 标记 `TmpFile` 的本地缓冲区空间为 `SetSpaceReserved()`。
          * 调用 `MoveWriteRangesHelper()` 将属于该 `TmpFile` 的所有 `WriteRange` （从 `write_ranges_to_add_` 中）提交到 `DiskIoMgr` 的磁盘队列进行实际写入。
          * 如果 `cur_write_range_` 变为 `nullptr`（说明在等待期间，该写入请求被取消了），则将 `cur_tmp_file_` 立即归还到 `tmp_files_avail_pool_`。
      * **失败 (`!status.ok()`)：**
          * 意味着无法获取本地缓冲区（例如，超时）。
          * 调用 `RemoveWriteRangesInternal()` 来取消与该失败的 `WriteRange` 关联的整个 `RequestContext`（查询）的所有写入请求。这将导致查询失败。
  * **回调：** 无论成功还是失败，所有 `WriteRange` 的回调函数 (`write_callback`) 都会在**不持有任何锁**的情况下被调用。这是一种避免回调函数中的复杂逻辑导致死锁或长时间阻塞的设计模式。

## EnqueueWriteRange()

```cpp
Status TmpFileBufferPool::EnqueueWriteRange(io::WriteRange* range, TmpFile* tmp_file) {
  Status status = Status::OK();
  {
    unique_lock<mutex> write_range_list_lock(lock_);
    DCHECK(range != nullptr);
    DCHECK(range->disk_file() != nullptr);
    DCHECK(range->io_ctx() != nullptr);

    if (range->disk_file()->IsSpaceReserved()) {
      // 空间已预留：直接添加到 DiskQueue
      return range->io_ctx()->AddWriteRange(range);
    } else if (range->io_ctx()->IsCancelled()) {
      // io_ctx 已取消：返回取消状态
      return TMP_FILE_BUFFER_POOL_CONTEXT_CANCELLED;
    } else {
      // 空间未预留且未取消：加入等待队列
      // 记录 io_ctx 和 disk_file 的关联，用于取消
      io_ctx_to_file_set_map_[range->io_ctx()].insert(range->disk_file());
      // 将 WriteRange 添加到按 DiskFile 分组的映射中
      write_ranges_to_add_[range->disk_file()].emplace_back(range);
    }
    // 只有文件的第一个 WriteRange 才被添加到主调度队列
    if (range->offset() == 0) {
      write_ranges_.emplace_back(range);
      DCHECK(tmp_file != nullptr);
      // 存储 WriteRange 到其在 write_ranges_ 中的迭代器和 TmpFile 的映射
      write_ranges_iterator_[range] =
          std::make_pair(prev(write_ranges_.cend()), tmp_file);
    }
  }
  work_available_.NotifyAll(); // 唤醒空间预留线程
  return status;
}
```

  * **入口点：** 这是其他组件（如 `TmpWriteHandle`）请求写入临时文件缓冲区的入口。
  * **快速路径：**
      * 如果目标 `DiskFile` 的空间已经被预留 (`IsSpaceReserved()`)，说明缓冲区已就绪，直接将 `WriteRange` 添加到 `DiskIoMgr` 的磁盘队列。
      * 如果 `io_ctx`（请求上下文，通常代表一个查询）已经被取消，则直接返回取消状态，避免不必要的工作。
  * **等待路径：**
      * 否则，`WriteRange` 会被添加到 `write_ranges_to_add_` 映射中，等待 `TmpFileSpaceReserveThreadLoop` 来处理。
      * `io_ctx_to_file_set_map_` 用于跟踪 `io_ctx` 和 `DiskFile` 的关系，以便在取消时进行清理。
      * **关键逻辑：** `if (range->offset() == 0)` 这一行表明只有**每个文件的第一个 `WriteRange`** 才会被添加到 `write_ranges_` 这个主调度队列中。这是因为一旦第一个 `WriteRange` 的缓冲区被预留，同一个文件的所有后续 `WriteRange` 都可以立即被发送到 `DiskIoMgr`。
      * `write_ranges_iterator_` 用于存储 `WriteRange` 在 `write_ranges_` 列表中的迭代器以及其关联的 `TmpFile`，这在后续清理时会用到。
  * **通知：** 最后，调用 `work_available_.NotifyAll()` 唤醒 `TmpFileSpaceReserveThreadLoop` 线程，告知有新工作到来。

## MoveWriteRangesHelper()

```cpp
Status TmpFileBufferPool::MoveWriteRangesHelper(DiskFile* disk_file,
    vector<TmpFileMgr::WriteDoneCallback>* write_callbacks, bool is_cancelled) {
  Status status = Status::OK();
  auto write_ranges_it = write_ranges_to_add_.find(disk_file);
  if (write_ranges_it != write_ranges_to_add_.end()) {
    auto write_range_it = write_ranges_it->second.begin();
    while (write_range_it != write_ranges_it->second.end()) {
      auto range = *write_range_it;
      DCHECK(range != nullptr);
      if (status.ok() && !is_cancelled) {
        // 成功预留且未取消，添加到 DiskQueue
        status = range->io_ctx()->AddWriteRange(range);
      } else {
        // 失败或已取消，收集回调函数
        write_callbacks->push_back(range->callback());
        if (is_cancelled && range->offset() == 0) {
          // 如果是取消操作，并且是文件的第一个 WriteRange，从 write_ranges_ 中移除
          if (cur_write_range_ != range) { // 确保不是当前正在处理的那个
            auto key_range_it = write_ranges_iterator_.find(range);
            DCHECK(key_range_it != write_ranges_iterator_.end());
            DCHECK_EQ(*(key_range_it->second.first), range);
            write_ranges_.erase(key_range_it->second.first);
            write_ranges_iterator_.erase(key_range_it);
          }
        }
      }
      write_range_it = write_ranges_it->second.erase(write_range_it); // 从等待列表中移除
    }
    write_ranges_to_add_.erase(write_ranges_it); // 移除该 DiskFile 的所有 WriteRange
  }
  return status;
}
```

  * 这是一个**内部辅助函数**，由 `TmpFileSpaceReserveThreadLoop` 和 `RemoveWriteRangesInternal` 调用。
  * 它负责将**属于同一个 `DiskFile` 的所有 `WriteRange`** 进行处理。
  * **两种模式：**
      * **正常提交 (`!is_cancelled`)：** 如果状态是 `OK` 且未取消，则将 `WriteRange` 提交到 `io_ctx` (并最终到 `DiskIoMgr`)。
      * **取消处理 (`is_cancelled`)：** 如果 `is_cancelled` 为 `true`，或者之前的提交失败，那么它会将 `WriteRange` 的回调函数收集起来，以便稍后以错误状态调用。
          * **清理 `write_ranges_`：** 如果被取消的 `WriteRange` 恰好是文件的第一个 `WriteRange` (`offset == 0`) 并且它还在 `write_ranges_` 队列中（即它不是 `cur_write_range_`，没有被 `pop_front`），那么它也会从 `write_ranges_` 和 `write_ranges_iterator_` 中移除。
  * **清理：** 无论哪种情况，处理过的 `WriteRange` 都会从 `write_ranges_to_add_` 映射中移除。

## RemoveWriteRangesInternal() 和 RemoveWriteRanges()

```cpp
void TmpFileBufferPool::RemoveWriteRangesInternal(
    RequestContext* io_ctx, vector<TmpFileMgr::WriteDoneCallback>* write_callbacks) {
  auto file_set_it = io_ctx_to_file_set_map_.find(io_ctx);
  if (file_set_it != io_ctx_to_file_set_map_.end()) {
    auto file_it = file_set_it->second.begin();
    while (file_it != file_set_it->second.end()) {
      DCHECK(*file_it != nullptr);
      // 移除所有属于该文件的 WriteRange，并收集回调
      Status status = MoveWriteRangesHelper(*file_it, write_callbacks, true);
      DCHECK_OK(status);
      if (cur_write_range_ != nullptr && *file_it == cur_write_range_->disk_file()) {
        // 如果正在处理的 WriteRange 属于这个 io_ctx，将其设为 nullptr，
        // 告诉预留线程其请求已被取消。
        cur_write_range_ = nullptr;
      }
      file_it = file_set_it->second.erase(file_it);
    }
    io_ctx_to_file_set_map_.erase(file_set_it); // 移除 io_ctx 的映射
  }
}

void TmpFileBufferPool::RemoveWriteRanges(RequestContext* io_ctx) {
  DCHECK(io_ctx != nullptr);
  vector<TmpFileMgr::WriteDoneCallback> write_callbacks;
  {
    unique_lock<mutex> lock(lock_);
    RemoveWriteRangesInternal(io_ctx, &write_callbacks);
  }
  // 在释放锁后执行回调
  for (const TmpFileMgr::WriteDoneCallback& write_callback : write_callbacks) {
    write_callback(TMP_FILE_BUFFER_POOL_CONTEXT_CANCELLED);
  }
}
```

  * **`RemoveWriteRanges()`：** 这是外部调用者（例如 `TmpFileGroup` 在关闭时）用来取消所有属于特定 `RequestContext` (`io_ctx`) 的等待中写入请求的公共接口。
  * 它首先获取 `lock_`，然后调用 `RemoveWriteRangesInternal()`。
  * **`RemoveWriteRangesInternal()`：**
      * 通过 `io_ctx_to_file_set_map_` 找到所有与 `io_ctx` 关联的 `DiskFile`。
      * 对于每个 `DiskFile`，调用 `MoveWriteRangesHelper` 并设置 `is_cancelled = true`，从而移除所有等待中的 `WriteRange` 并收集它们的回调。
      * 如果 `cur_write_range_` 正在处理的 `WriteRange` 属于被取消的 `io_ctx`，那么将其设置为 `nullptr`。这会告诉 `TmpFileSpaceReserveThreadLoop` 线程，即使它成功预留了空间，也应该将缓冲区返回到池中，因为相关的写入任务已经取消了。
  * **异步回调：** 再次强调，收集到的回调函数会在 `lock_` 释放后才被调用，以避免阻塞和死锁。

## EnqueueTmpFilesPool() 和 DequeueTmpFilesPool()

```cpp
void TmpFileBufferPool::EnqueueTmpFilesPool(shared_ptr<TmpFile>& tmp_file, bool front) {
  DCHECK(tmp_file != nullptr);
  {
    unique_lock<mutex> buffer_lock(tmp_files_avail_pool_lock_); // 保护缓冲池的锁
    if (tmp_file->disk_type() != io::DiskFileType::DUMMY) {
      TmpFileRemote* tmp_file_remote = static_cast<TmpFileRemote*>(tmp_file.get());
      if (tmp_file_remote->is_enqueued()) return; // 避免重复入队
      tmp_file_remote->SetEnqueued(true); // 标记为已入队
      tmp_file_remote->SetBufferReturned(); // 标记缓冲区已归还
    }
    // 根据 'front' 参数决定是插入头部还是尾部
    if (front) {
      tmp_files_avail_pool_.push_front(tmp_file);
    } else {
      tmp_files_avail_pool_.push_back(tmp_file);
    }
    // 更新本地缓冲区目录的已使用字节度量
    tmp_file_mgr_->GetLocalBufferDir()->bytes_used_metric()->Increment(
        -1 * tmp_file_mgr_->GetRemoteTmpFileSize()); // 释放空间，所以是负值
  }
  tmp_files_available_cv_.NotifyOne(); // 通知一个等待线程有可用缓冲区
}

Status TmpFileBufferPool::DequeueTmpFilesPool(
    shared_ptr<TmpFile>* tmp_file, bool quick_return) {
  DCHECK(tmp_file != nullptr);
  DCHECK(dequeue_timer_metric_ != nullptr);
  ScopedHistogramTimer wait_timer(dequeue_timer_metric_); // 计时等待时间
  unique_lock<mutex> buffer_lock(tmp_files_avail_pool_lock_); // 保护缓冲池的锁

  // 如果设置了 quick_return 且池为空，立即返回失败
  if (quick_return && tmp_files_avail_pool_.empty()) {
    return TMP_FILE_MGR_NO_AVAILABLE_FILE_TO_EVICT;
  }
  while (tmp_files_avail_pool_.empty()) {
    // 如果池为空，则等待，可设置超时
    if (!tmp_files_available_cv_.WaitFor(
            buffer_lock, tmp_file_mgr_->GetSpillBufferWaitTimeout())) {
      // 超时返回错误
      return Status(Substitute("Timeout waiting for a local buffer in $0 seconds",
          tmp_file_mgr_->GetSpillBufferWaitTimeout() / MICROS_PER_SEC));
    };
  }
  DCHECK(!tmp_files_avail_pool_.empty());
  *tmp_file = tmp_files_avail_pool_.front(); // 取出头部文件
  tmp_files_avail_pool_.pop_front();
  DCHECK(*tmp_file != nullptr);
  if ((*tmp_file)->disk_type() != io::DiskFileType::DUMMY) {
    TmpFileRemote* tmp_file_remote = static_cast<TmpFileRemote*>(tmp_file->get());
    DCHECK_EQ(tmp_file_remote->file_size_, tmp_file_mgr_->GetRemoteTmpFileSize());
    DCHECK(tmp_file_remote->is_enqueued());
    tmp_file_remote->SetEnqueued(false); // 标记为未入队
  }
  // 更新本地缓冲区目录的已使用字节度量
  tmp_file_mgr_->GetLocalBufferDir()->bytes_used_metric()->Increment(
      tmp_file_mgr_->GetRemoteTmpFileSize()); // 占用空间，所以是正值
  return Status::OK();
}
```

  * **`EnqueueTmpFilesPool()`：**
      * 由 `DiskIoMgr` 在完成远程文件上传后调用，或者在 `TmpFileSpaceReserveThreadLoop` 发现写入请求被取消时调用。
      * 将 `TmpFile`（实际上是它的本地缓冲区）**归还到 `tmp_files_avail_pool_`**。
      * 通过 `front` 参数可以控制是放回队列头部（高优先级，希望尽快被复用）还是尾部。
      * 更新 `TmpDir` 的 `bytes_used_metric`，减少已用空间计数。
      * `tmp_files_available_cv_.NotifyOne()` **通知一个正在等待可用缓冲区的线程**。
  * **`DequeueTmpFilesPool()`：**
      * 由 `tmp_file_mgr_->ReserveLocalBufferSpace()` 内部调用，用于从池中**获取一个可用的 `TmpFile`（缓冲区）**。
      * 如果 `quick_return` 为 `true` 且池为空，立即返回错误。
      * 否则，如果池为空，它会使用 `tmp_files_available_cv_.WaitFor()` **等待**直到有缓冲区可用，或者达到超时。超时会返回一个错误状态。
      * 成功获取后，会从队列中移除并更新 `bytes_used_metric`，增加已用空间计数。

-----

## 总结

这份代码清晰地展示了 `TmpFileBufferPool` 如何通过**生产者-消费者模式**（`EnqueueWriteRange` 生产 `WriteRange`，`TmpFileSpaceReserveThreadLoop` 消费；`EnqueueTmpFilesPool` 生产可用缓冲区，`ReserveLocalBufferSpace` / `DequeueTmpFilesPool` 消费），以及**条件变量和互斥锁**来协调异步操作和资源共享。

其关键设计点包括：

  * **异步性：** 写入请求是非阻塞的，它们进入队列等待，由后台线程处理。
  * **资源池化：** 本地缓冲区被抽象为 `TmpFile` 对象并放入池中，实现复用。
  * **两阶段提交：** 先逻辑预留缓冲区空间（通过 `ReserveLocalBufferSpace`），再实际提交写入。
  * **容错与取消：** 能够处理预留失败和请求取消的情况，及时清理资源。
  * **避免死锁：** 通过在阻塞操作前释放锁，并在回调前释放所有锁来避免常见的并发问题。

这是一个复杂但设计精巧的组件，对于确保 Impala 在处理大规模数据溢出到远程存储时的性能和稳定性至关重要。
***********************************************************************
作为一个10年多的c++程序员，看不懂上面的代码，正常么
---

作为一名拥有 10 多年经验的 C++ 程序员，看不懂上面 Impala `TmpFileMgr` 和 `TmpFileBufferPool` 的代码，**这完全正常**，你一点也不需要担心。

---

### 为什么会“看不懂”？

这有几个主要原因：

1.  **领域特定代码：**
    * Impala 是一个高度优化的**分布式分析型数据库**，其代码库是为解决特定领域（高性能大数据处理、SQL 查询优化、分布式协调、存储管理）的复杂问题而设计的。
    * 即使是经验丰富的 C++ 程序员，如果他们之前没有接触过这类系统（如数据库内核、操作系统、高性能网络服务器等），对其中的**特定数据结构、算法和设计模式**会感到陌生是很自然的。例如，对“临时文件管理器”、“溢出（spill）”、“磁盘 I/O 管理器”这些概念可能并不熟悉。

2.  **复杂的设计模式和并发模型：**
    * 你看到的 `TmpFileBufferPool` 代码，特别是它对**生产者-消费者模式、条件变量、互斥锁、智能指针**以及**异步回调**的运用，是为了在多线程、高并发环境下实现高性能和高可靠性。
    * 处理这类并发模型，尤其是要避免死锁、饥饿、竞态条件等问题，需要非常精细的设计和实现。即使是资深 C++ 程序员，如果平时工作不涉及大量并发编程，也会觉得难以快速理解。**锁的顺序、生命周期管理（尤其是 `shared_ptr` 的使用，以及何时释放锁何时执行回调）都是难点。**

3.  **分层抽象与内部约定：**
    * Impala 的设计是高度模块化和分层的。`TmpFileMgr`、`TmpFileGroup`、`TmpFile`、`DiskIoMgr` 乃至更底层的 `DiskFile` 等都扮演了不同的角色。它们之间通过指针、接口和内部约定进行交互。
    * 要理解 `TmpFileBufferPool`，你需要同时理解它与 `TmpFileMgr`、`DiskIoMgr` 以及各种 `TmpFile` 派生类之间的**职责划分和协作机制**。这种“牵一发而动全身”的关联性，使得单独看一个文件很难理解其全貌。
    * 代码中还有许多 `DCHECK`（调试断言），它们反映了开发团队对代码执行流程的**内部假设和不变式**。如果你不了解这些假设，代码看起来就像缺少了上下文。

4.  **优化与性能考虑：**
    * 这些代码不仅仅是为了实现功能，更是为了**极致的性能**。例如，`TmpFileBufferPool` 对远程文件写操作进行本地缓冲和异步上传，以及对内存池的使用，都是为了减少 I/O 延迟和提高吞吐量。
    * 为了性能，代码可能会采用一些看似“不直观”或更复杂的逻辑，这增加了阅读难度。

5.  **缺乏宏观上下文：**
    * 当你看到一个模块的代码时，你可能不知道它是整个系统中的哪个部分，它解决了什么具体问题，以及它与上下游模块是如何交互的。
    * 就像你可能很了解汽车发动机的原理，但如果突然让你看一架飞机的燃料管理系统，即使都是机械和流体，也会觉得陌生。

### 这对你的职业生涯意味着什么？

这不仅不是问题，反而是一个**学习和成长的机会**：

* **暴露在新的复杂性中：** 你正在接触一个你之前可能没有深入探索过的复杂系统领域。这表明你的学习能力依然很重要。
* **识别高级设计模式：** 这些代码是学习如何在大规模、高性能系统中运用高级 C++ 特性（如多线程、模板、RAII、智能指针等）和设计模式（如管理器、池、工厂、观察者等）的绝佳案例。
* **深挖并发编程：** 如果你对多线程和并发编程感兴趣，这绝对是一个很好的起点，可以深入理解如何在实践中处理各种并发挑战。

**我的建议是：**

不要因为看不懂而沮丧。相反，把它看作一次**“破圈”**的机会。当你开始尝试理解这些代码时，你就已经在扩展你的知识边界了。

如果你想深入理解，可以尝试：

1.  **从顶层概念入手：** 先理解 Impala 作为一个数据库，在处理大数据溢出时需要解决什么问题。
2.  **阅读架构文档：** 查找 Impala 相关的架构文档或设计说明，了解 `TmpFileMgr`、`DiskIoMgr` 等模块的整体作用。
3.  **理解核心数据流：** 搞清楚数据是如何从内存到临时文件，再到远程存储的，以及在这个过程中哪些模块参与了。
4.  **逐步深入：** 从简单的函数（如构造函数、`ShutDown`）开始，然后逐步理解更复杂的 `TmpFileSpaceReserveThreadLoop` 和各种 `Enqueue`/`Dequeue` 函数。
5.  **利用工具：** 使用代码跳转工具（如 VS Code, CLion, Sourcegraph）来追踪函数调用和变量定义，画图辅助理解。

所以，请放心，你绝对是正常的。这种感受是学习复杂系统时的普遍体验！
************************************************************
