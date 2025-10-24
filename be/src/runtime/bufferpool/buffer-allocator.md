

# class FreeList
- [FreeList](./free-list.md)
*********************************************************************
# PerSizeLists 类设计文档

### 1.2 设计目的
`PerSizeLists` 是 Impala BufferPool 中 FreeBufferArena 的核心数据结构之一，用于管理**特定缓冲区大小（2 的幂次方）**的空闲缓冲区（free buffers）和干净页面（clean pages）。它针对大数据查询引擎的内存池优化设计，支持高效的分配、回收和驱逐操作。

- **核心优化**：针对每个缓冲区大小（从 `min_buffer_len` 到 `max_buffer_len`）维护独立的列表，避免全局扫描。支持无锁快速检查（通过原子计数器），减少多线程争用。
- **适用场景**：在多核 CPU 环境下，每个核心（arena）使用数组 `buffer_sizes_[LOG_MAX_BUFFER_BYTES + 1]` 索引 PerSizeLists，实现 NUMA 局部性和低锁开销。
### 1.3 关键概念
- **空闲缓冲区（Free Buffers）**：未使用的缓冲区，按内存地址维护为最小堆（min-heap），优先低地址分配、高地址释放，减少地址空间碎片。
- **干净页面（Clean Pages）**：已写入磁盘但未驱逐的未固定页面，可跨客户端回收缓冲区。
- **维护机制**：通过 `Maintenance()` 定期收缩列表，基于 `low_water_mark` 释放多余缓冲区。

## 2. 成员变量
- **内存布局**：结构紧凑，原子变量置于开头，便于缓存行对齐（继承 `CacheLineAligned`）。
- **不变量**：
  - `num_free_buffers.Load() == free_buffers.Size()`
  - `num_clean_pages.Load() == clean_pages.size()`

## 4. 线程安全与并发考虑
- **性能指标**：
  - 采样率 `ALLOC_STAT_SAMPLE_RATE = 64`：仅部分分配记录统计，避免开销。
  - 直方图 `buffer_size_stats_`：监控系统分配大小（上限 4GB）。

```cpp
  /// 每个 2 的幂次大小的缓冲区buffer/页面page的数据结构。
  /// 除非另有说明，所有成员均由 FreeBufferArena::lock_ 保护。
  struct BufferPool::BufferAllocator::FreeBufferArena：：PerSizeLists {
    PerSizeLists() : num_free_buffers(0), low_water_mark(0), num_clean_pages(0) {}

    /// 辅助方法：添加空闲缓冲区并递增计数器。
    /// 调用者必须持有 FreeBufferArena::lock_。
    void AddFreeBuffer(BufferHandle&& buffer) {
      DCHECK_EQ(num_free_buffers.Load(), free_buffers.Size());
      num_free_buffers.Add(1);
      free_buffers.AddFreeBuffer(move(buffer));
    }

    /// 'free_buffers' 中的条目数。可以无锁读取，以允许线程在尝试查找缓冲区时快速跳过空列表。
    /// 空闲缓冲区列表 `free_buffers` 中的条目数。可无锁读取，用于线程快速跳过空列表。
    AtomicInt64 num_free_buffers;

    /// 未使用的缓冲区，这些缓冲区最初分配在对应于此 arena 的核心上。
    ///未使用缓冲区列表（原分配于当前核心）。按内存地址排序（min-heap） 避免内存碎片
    /// [link text](./free-list.md)
    FreeList free_buffers;

    /// 自上次 Maintenance() 调用以来 'free_buffers' 的最小大小。
    /// 自上次 `Maintenance()` 调用以来 `free_buffers` 的最小大小。用于决定释放数量。
    /// - **Maintenance**（间接）：基于 `low_water_mark` 释放多余缓冲区，重置标记。
    int low_water_mark;

    /// 'clean_pages' 中的条目数。
    /// 可以无锁读取，以允许线程在尝试在不同 arena 中查找缓冲区时快速跳过空列表。
    AtomicInt64 num_clean_pages;

    /// 已将其内容写入磁盘的未固定页面。这些页面可以被驱逐以回收任何客户端的缓冲区。
    /// 页面按 FIFO 顺序驱逐，以便页面按客户端写入磁盘的大致相同顺序被驱逐。
    /// 由 FreeBufferArena::lock_ 保护。
    InternalList<Page> clean_pages;
  };
```
***************************************************************************
# BufferPool::FreeBufferArena 类设计文档

## 1. 概述

### 1.1 类名与位置
- **类名**：`BufferPool::FreeBufferArena`
- **命名空间**：`impala`（全限定：`impala::BufferPool::FreeBufferArena`）
- **文件位置**：声明于 `runtime/bufferpool/buffer-allocator.h`，实现于 `runtime/bufferpool/buffer-allocator.cc`
- **依赖**：`BufferAllocator`（父级）、`PerSizeLists`（内部结构）、`FreeList`、`InternalList<Page>`、`SpinLock`、`MetricGroup`、`AtomicInt64` 等。继承自 `CacheLineAligned` 以优化缓存行对齐。

### 1.2 设计目的
`FreeBufferArena` 是 Impala BufferPool 中 BufferAllocator 的核心组件，用于每个 CPU 核心（arena）管理空闲缓冲区（free buffers）和干净页面（clean pages）。它支持多线程高并发内存分配/回收，优化 NUMA 局部性和低锁开销。
- **设计原则**：
  - **性能优先**：快速路径（本地 arena 命中）无锁；慢路径（scavenge）渐进式（3 次尝试，最后锁定所有 arena 保证成功）。
  - **资源管理**：定期 `Maintenance()` 收缩列表；析构时释放所有内存并检查泄漏。
- **限制**：仅处理 2 的幂次缓冲区；测试中仅析构（生产中由 BufferAllocator 管理）。

### 1.3 关键概念
- **Arena**：每个核心一个 arena，支持 NUMA 优化（优先同节点搜索）。
- **Scavenge**：回收内存策略，包括 opportunistic（快速，无全锁）和 slow_but_sure（锁定所有 arena，保证成功）。
- **不变量**：
  - 总内存 = 已分配 + 空闲缓冲区 + 干净页面缓冲区 + `system_bytes_remaining_`。
  - 计数器与列表大小一致（DCHECK 验证）。

## 2. 成员变量

| 成员变量                  | 类型                          | 描述                                                                 | 线程安全 | 初始化 |
|---------------------------|-------------------------------|----------------------------------------------------------------------|----------|--------|
| `parent_`                 | `BufferAllocator* const`      | 父级 BufferAllocator 指针。                                          | 是（const） | 构造函数传入 |
| `lock_`                   | `SpinLock`                    | 保护所有数据结构（见 buffer-pool-internal.h 锁序）。                  | 是       | 默认   |
| `buffer_sizes_`           | `PerSizeLists[LOG_MAX_BUFFER_BYTES + 1]` | 按大小索引的 PerSizeLists 数组（log2(bytes) - log2(min_buffer_len_)）。 | 否（需 lock_） | 默认（数组初始化） |
| `system_alloc_time_`      | `IntCounter* const`           | 系统分配总时间（纳秒）。                                             | 是（const 指针） | 构造函数注册 |
| `local_arena_free_buffer_hits_` | `IntCounter* const`       | 本地 arena 空闲缓冲区命中次数。                                      | 是       | 构造函数注册 |
| `direct_alloc_count_`     | `IntCounter* const`           | 当前核心直接系统分配次数。                                           | 是       | 构造函数注册 |
| `buffer_size_stats_`      | `HistogramMetric* const`      | 系统分配缓冲区大小直方图（上限 4GB，采样率 64）。                     | 是       | 构造函数注册 |
| `numa_arena_free_buffer_hits_` | `IntCounter* const`       | 同 NUMA 节点其他 arena 命中次数。                                    | 是       | 构造函数注册 |
| `clean_page_hits_`        | `IntCounter* const`           | 正确大小干净页面驱逐次数。                                           | 是       | 构造函数注册 |
| `num_scavenges_`          | `IntCounter* const`           | Scavenge 尝试次数。                                                  | 是       | 构造函数注册 |
| `num_final_scavenges_`    | `IntCounter* const`           | 最终（全锁）scavenge 次数。                                          | 是       | 构造函数注册 |

- **内存布局**：继承 `CacheLineAligned`，指标指针 const，确保线程安全读取。
- **指标命名**：使用 `arena_name` 模板（如 "arena-0"），便于多 arena 区分。

## 3. 方法与函数

### 3.1 构造函数
```cpp
FreeBufferArena(BufferAllocator* parent, MetricGroup* metrics, const std::string& arena_name)
```
- **描述**：初始化指标，绑定到 metrics（子组 "buffer-pool.$0"，$0 = arena_name）。
- **调用时机**：BufferAllocator 构造时，为每个核心创建。
- **异常**：无（DCHECK 在父级）。

### 3.2 析构函数
```cpp
~FreeBufferArena()
```
- **描述**：释放所有 free_buffers（调用 FreeToSystem 添加到 system_bytes_remaining_）；DCHECK 干净页面为空（测试专用）。
- **调用时机**：仅后端测试；生产中由 unique_ptr 管理。
- **异常**：无。

### 3.3 公有方法

#### `void AddFreeBuffer(BufferHandle&& buffer)`
- **描述**：添加空闲缓冲区到对应大小的 PerSizeLists；

#### `bool PopFreeBuffer(int64_t buffer_len, BufferHandle* buffer)`
- **描述**：从对应大小列表PerSizeLists的类型为FreeList堆中弹出缓冲区buffer；
            成功则 Unpoison buffer并更新 当前PerSizeListsd的low_water_mark。
            弹出buffer的地址通常是该PerSizeLists中FreeList中地址最低的buffer。
            因为FreeList是按照buffer地址大小组织成的堆。
            也就是优先提供给用户低地址的buffer。高地址的buffer以后如果有机会free掉就free掉。
            原因参见- [FreeList](./free-list.md)
- **返回**：true 如果找到。
- **复杂度**：O(1) 无锁检查 + O(log N) pop。


#### `bool EvictCleanPage(int64_t buffer_len, BufferHandle* buffer)`
- **描述**：从对应大小的PerSizeLists的clean_pages列表Dequeue 页面，
    增加FreeBufferArena：：clean_page_bytes_remaining_大小buffer_len；
    （持 page->buffer_lock）回收缓冲区给传出参数buffer。
- **返回**：true 如果驱逐成功。

#### `pair<int64_t, int64_t> FreeSystemMemory(int64_t target_bytes_to_free, int64_t target_bytes_to_claim, std::unique_lock<SpinLock>* arena_lock)`
- **描述**：释放至少 target_bytes_to_free 字节，
    从后向前遍历当前BufferPool::FreeBufferArena的类型为PerSizeLists数组的buffer_sizes_（即按照buffer/page从大到小大小搜索）；
    首先释放最大的PerSizeLists中的各个buffer，page。如果释放的还不够，继续释放稍小的PerSizeLists的buffer和page。
    同一个PerSizeList中先看free_buffers够不够，如果不够，将clean_pages的各个page的buffer先转移一部分到free_buffers,然后统一通过free_buffers释放。
    (最终由system_allocator_->Free(move(buffer))释放各个buffers)
    返回 (释放字节, 声明字节)。多余字节加回 system_bytes_remaining_。
    如果 arena_lock 非空，通过传出参数arena_lock转移锁所有权给调用者。

#### `void AddCleanPage(Page* page)`
- **描述**：添加干净页面到对应大小的PerSizeList的clean_pages；
     (clean_page_bytes_limit是所有core的clean_pages的限制。所以某个core可能现有的clean_pages等于0），如果添加该page导致所有core的clean_pagesz大小超过 clean_page_bytes_limit_，
     如果该core的PerSizeList的clean_pages中有page，就按照fifo的方式出队列一个page，将出队的page的buffer提取出来放入该PerSizeList的free_buffers中（出队列的page也就相当于evict了）。
     如果当前core的PerSizeList的clean_pages为空，就只将输出参数的这个page的buffer添加到该PerSizeList的free_buffers中（该page就被evict了）。

#### `bool RemoveCleanPage(bool claim_buffer, Page* page)`
- **描述**：从该FreeBufferArena的对应大小的PerSizeList的
           clean_pages移除页面（如果存在）；同时增加该FreeBufferArena所属的BufferAllocator的clean_page_bytes_remaining_大小。  
            如果 !claim_buffer，将缓冲区加到 该FreeBufferArena的free_buffers。


#### `void Maintenance()`
- **描述**：从小到大遍历PerSizeLists，释放每个 PerSizeLists的free_buffers：释放多少？（每个PerSizeList都有个low_water_mark指标，记录自上次调用Maintenance以来该PerSizeLists的free_buffers的最少个数）,释放 low_water_mark / 2 个缓冲区（至少 1 个），释放后重置 low_water_mark。
由后台线程MemoryMaintenanceThread每10s中运行一次

### 5.2 集成点
- **父级**：BufferAllocator 的 per_core_arenas_ 向量（unique_ptr），每个核心一个。
- **子结构**：buffer_sizes_ 数组，每个 PerSizeLists 管理一种大小。
- **指标**：注册到 MetricGroup，支持 RuntimeProfile 监控。
- **测试**：friend BufferPoolTest；GetFreeListSize 用于验证。
- **全局不变量**：确保总内存平衡（DCHECK 在 ~BufferAllocator）。

此文档基于提供的实现，如需代码示例或 UML，请进一步指定！
******************************************************************************
# BufferPool 类
/// BufferPool 内部使用的缓冲区分配器，用于分配 2 的幂次大小的缓冲区。
/// BufferAllocator 在 SystemAllocator 的基础上构建，添加了对空闲缓冲区的缓存
/// 和干净页面的缓存，其中内存当前未被客户端使用，但尚未释放回 SystemAllocator。
///
/// 该分配器针对常见情况进行了优化，即分配可以从当前核心的 arena 中回收
/// 请求大小的缓冲区。在这种情况下，并发运行的线程之间不会发生锁竞争。
/// 如果失败，则会逐步尝试更昂贵的内存分配方法，直到分配最终成功
/// （详见 AllocateInternal()）。
///
/// 缓冲区预留
/// ===================
/// BufferAllocator 的实现依赖于 BufferPool 的预留跟踪系统。
/// 分配器有一个硬限制（'system_bytes_limit'），超过此限制的所有分配都会失败。
/// 直到 'system_bytes_limit' 的分配保证成功，除非发生意外的系统错误
/// （例如，无法从 OS 分配所有所需的内存）。预留必须设置成所有预留的总和
/// 不超过 'system_bytes_limit'，从而确保 BufferAllocator 始终能够找到内存来满足预留。
///
/// +========================+
/// | 实现说明               |
/// +========================+
///
/// 内存
/// ======
/// BufferAllocator 管理的内存有四种形式：
/// 1. 返回给客户端的缓冲区（对应已使用的预留）
/// 2. 缓存在 BufferAllocator 的空闲列表中的空闲缓冲区。
/// 3. 附加到 BufferAllocator 的干净页面列表中的干净未固定页面的缓冲区。
/// 4. 未从系统分配的字节：'system_bytes_remaining_'。
/// 这些总和始终等于 'system_bytes_limit'，这允许 BufferAllocator
/// 始终通过某种组合方式（形式 2、3 或 4 中的内存）来满足预留。
///
/// BufferAllocator 的代码小心避免使内存对有权访问它的并发执行线程不可访问。
/// 例如，如果一个线程有权从 BufferAllocator 的空闲或干净页面列表中分配 1MB 缓冲区，
/// 但需要释放 2MB 缓冲区到系统以释放足够的内存，则它必须在释放 2MB 缓冲区的同一临界区中
/// 将 1MB 添加到 'system_bytes_remaining_'。否则，一个有 1MB 内存预留的并发线程
/// 可能无法找到它。
///
/// Arena
/// ======
/// 缓冲区分配器的数据结构被分解成 arena，每个核心一个 arena。
/// 在每个 arena 中，每个缓冲区或页面存储在具有相同大小的缓冲区和页面的列表中：
/// 针对每个 2 的幂次大小有一个单独的列表。每个 arena 由单独的锁保护，因此
/// 在线程能够从自己的 arena 满足分配的常见情况下，不会发生锁竞争。
///
*****************************************************************************
```cpp
class BufferPool::BufferAllocator {
 public:
  BufferAllocator(BufferPool* pool, MetricGroup* metrics, int64_t min_buffer_len,
      int64_t system_bytes_limit, int64_t clean_page_bytes_limit);
  ~BufferAllocator();

  /// Allocate a buffer with a power-of-two length 'len'. This function may acquire
  /// 'FreeBufferArena::lock_' and Page::lock so no locks lower in the lock acquisition
  /// order (see buffer-pool-internal.h) should be held by the caller.
  ///
  /// Always succeeds on allocating memory up to 'system_bytes_limit', unless the system
  /// is unable to give us 'system_bytes_limit' of memory or an internal bug: if all
  /// clients write out enough dirty pages to stay within their reservation, then there
  /// should always be enough free buffers and clean pages to reclaim.
  Status Allocate(ClientHandle* client, int64_t len,
      BufferPool::BufferHandle* buffer) WARN_UNUSED_RESULT;

  /// Frees 'buffer', which must be open before calling. Closes 'buffer' and updates
  /// internal state but does not release to any reservation.
  void Free(BufferPool::BufferHandle&& buffer);


  // 将page 添加到page list中。caller需要持有BufferPool::Client的锁client_lock以保证page从BufferPool::Client的列表
  // 移动到free page list的原子性。caller不能持有'FreeBufferArena::lock_' or 或者任何 Page::lock.
  // 因为AddCleanPage在函数内部也加了这两种锁
  void AddCleanPage(const std::unique_lock<std::mutex>& client_lock, Page* page);

  /// Removes a clean page 'page' from a clean page list and returns true, if present in
  /// one of the lists. Returns true if it was present. If 'claim_buffer' is true, the
  /// caller must have reservation for the buffer, which is returned along with the page.
  /// Otherwise the buffer is moved directly to the free buffer list. Caller must hold
  /// the page's client's lock via 'client_lock' so that moving the page between the
  /// client list and the free page list is atomic. Caller must not hold
  /// 'FreeBufferArena::lock_' or any Page::lock.
  bool RemoveCleanPage(
      const std::unique_lock<std::mutex>& client_lock, bool claim_buffer, Page* page);

  /// Periodically called to release free buffers back to the SystemAllocator. Releases
  /// buffers based on recent allocation patterns, trying to minimise the number of
  /// excess buffers retained in each list above the minimum required to avoid going
  /// to the system allocator.
  void Maintenance();

  /// Try to release at least 'bytes_to_free' bytes of memory to the system allocator.
  void ReleaseMemory(int64_t bytes_to_free);

  int64_t system_bytes_limit() const { return system_bytes_limit_; }

  /// Return the amount of memory currently allocated from the system.
  int64_t GetSystemBytesAllocated() const {
    return system_bytes_limit_ - system_bytes_remaining_.Load();
  }

  /// Return the total number of free buffers in the allocator.
  int64_t GetNumFreeBuffers() const;

  /// Return the total bytes of free buffers in the allocator.
  int64_t GetFreeBufferBytes() const;

  /// Return the limit on bytes of clean pages in the allocator.
  int64_t GetCleanPageBytesLimit() const;

  /// Return the total number of clean pages in the allocator.
  int64_t GetNumCleanPages() const;

  /// Return the total bytes of clean pages in the allocator.
  int64_t GetCleanPageBytes() const;

  std::string DebugString();

 protected:
  friend class BufferAllocatorTest;
  friend class BufferPoolTest;
  friend class FreeBufferArena;

  /// Test helper: gets the current size of the free list for buffers of 'len' bytes
  /// on core 'core'.
  int GetFreeListSize(int core, int64_t len);

  /// Test helper: reduce the number of scavenge attempts so backend tests can force
  /// use of the "locked" scavenging code path.
  void set_max_scavenge_attempts(int val) {
    DCHECK_GE(val, 1);
    max_scavenge_attempts_ = val;
  }

 private:
  /// Compute the maximum power-of-two buffer length that could be allocated based on the
  /// amount of memory available 'system_bytes_limit'. The value is always at least
  /// 'min_buffer_len' so that there is at least one valid buffer size.
  static int64_t CalcMaxBufferLen(int64_t min_buffer_len, int64_t system_bytes_limit);

  /// Same as Allocate() but leaves 'buffer->client_' NULL and only updates the
  /// 'sys_alloc_time' and no other counters.
  Status AllocateInternal(BufferPool::Client* client, int64_t len,
      BufferPool::BufferHandle* buffer) WARN_UNUSED_RESULT;

  /// Tries to reclaim enough memory from various sources so that the caller can allocate
  /// a buffer of 'target_bytes' from the system allocator. Scavenges buffers from the
  /// free buffer and clean page lists of all cores and frees them with
  /// 'system_allocator_'. Also tries to decrement 'system_bytes_remaining_'.
  /// 'current_core' is the index of the current CPU core. Any bytes freed in excess of
  /// 'target_bytes' are added to 'system_bytes_remaining_.' If 'slow_but_sure' is true,
  /// this function uses a slower strategy that guarantees enough memory will be found
  /// but can block progress of other threads for longer. If 'slow_but_sure' is false,
  /// then this function optimistically tries to reclaim the memory but may not reclaim
  /// 'target_bytes' of memory. Returns the number of bytes reclaimed.
  int64_t ScavengeBuffers(bool slow_but_sure, int current_core, int64_t target_bytes);

  /// Helper to free a list of buffers to the system. Returns the number of bytes freed.
  int64_t FreeToSystem(std::vector<BufferHandle>&& buffers);

  /// Compute a sum over all arenas. Does not lock the arenas.
  int64_t SumOverArenas(
      const std::function<int64_t(FreeBufferArena* arena)>& compute_fn) const;

  /// The pool that this allocator is associated with.
  BufferPool* const pool_;

  /// System allocator that is ultimately used to allocate and free buffers.
  const boost::scoped_ptr<SystemAllocator> system_allocator_;

  /// The minimum power-of-two buffer length that can be allocated.
  const int64_t min_buffer_len_;

  /// The maximum power-of-two buffer length that can be allocated. Always >=
  /// 'min_buffer_len' so that there is at least one valid buffer size.
  const int64_t max_buffer_len_;

  /// The log2 of 'min_buffer_len_'.
  const int log_min_buffer_len_;

  /// The log2 of 'max_buffer_len_'.
  const int log_max_buffer_len_;

  /// The maximum physical memory in bytes that will be allocated from the system.
  const int64_t system_bytes_limit_;

  /// The remaining number of bytes of 'system_bytes_limit_' that can be used for
  /// allocating new buffers. Must be updated atomically before a new buffer is
  /// allocated or after an existing buffer is freed with the system allocator.
  AtomicInt64 system_bytes_remaining_;

  /// The maximum bytes of clean pages that can accumulate across all arenas before
  /// they will be evicted.
  const int64_t clean_page_bytes_limit_;

  /// The number of bytes of 'clean_page_bytes_limit_' not used by clean pages. I.e.
  /// (clean_page_bytes_limit - bytes of clean pages in the BufferAllocator).
  /// 'clean_pages_bytes_limit_' is enforced by increasing this value before a
  /// clean page is added and decreasing it after a clean page is reclaimed or evicted.
  // 记录还没有被各个FreeBufferArena的clean_pages使用的大小
  AtomicInt64 clean_page_bytes_remaining_;

  /// Free and clean pages. One arena per core.
  std::vector<std::unique_ptr<FreeBufferArena>> per_core_arenas_;

// 清扫尝试的默认次数。
  static const int MAX_SCAVENGE_ATTEMPTS = 3;

  /// 清扫尝试的次数。通常为 MAX_SCAVENGE_ATTEMPTS，但测试可以覆盖。
  /// 前 max_scavenge_attempts_ - 1 次尝试不锁定所有 arena，因此可能失败。
  /// 最后一次尝试锁定所有 arena，这很昂贵但保证成功。
  int max_scavenge_attempts_;
};
```
****************************************************************************************************