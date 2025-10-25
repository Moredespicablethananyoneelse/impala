

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
- **关键参数**： `buffer_sizes_`    的LOG_MAX_BUFFER_BYTES为48，也就是最大可分配281TB的buffer
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
    - 从后向前遍历当前BufferPool::FreeBufferArena的类型为PerSizeLists数组的buffer_sizes_（即按照buffer/page从大到小大小搜索）；
    - 首先释放最大的PerSizeLists中的各个buffer，page。如果释放的还不够，继续释放稍小的PerSizeLists的buffer和page。
    - 同一个PerSizeList中先看free_buffers够不够，如果不够，将clean_pages的各个page的buffer先转移一部分到free_buffers,然后统一通过free_buffers释放。
    - (最终由system_allocator_->Free(move(buffer))释放各个buffers)
    返回 (释放字节, 声明字节)。多余字节加回 system_bytes_remaining_。
    - 如果 arena_lock 非空，通过传出参数arena_lock转移锁所有权给调用者。
    - 解释函数中涉及的几个变量：
       - bytes_freed：该函数运行期间累计已经释放的内存。
       - buffer_len:当前遍历的PerSizeLists中free_buffers每个buffer的大小。
       - buffers_to_free:当前遍历的PerSizeLists中的free_buffers需要释放buffer的个数。
       - buffer_bytes_to_free：准备好释放的各个buffer的大小的和(包括从page中提取的buffer)。
       - num_pages_evicted:在该PerSizeLists中的free_buffers不够的情况下，需要驱逐的page的个数。

    - 如果释放的实际释放的内存bytes_freed > target_bytes_to_claim,
    则只将bytes_freed - target_bytes_to_claim返回给所属BufferAllocator的system_bytes_remaininig_;



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

#### `pair<int64_t, int64_t> BufferPool::FreeBufferArena::FreeSystemMemory(int64_t target_bytes_to_free, int64_t target_bytes_to_claim,std::unique_lock<SpinLock>* arena_lock)`

### 5.2 集成点
- **父级**：BufferAllocator 的 per_core_arenas_ 向量（unique_ptr），每个核心一个。
- **子结构**：buffer_sizes_ 数组，每个 PerSizeLists 管理一种大小。
- **指标**：注册到 MetricGroup，支持 RuntimeProfile 监控。
- **测试**：friend BufferPoolTest；GetFreeListSize 用于验证。
- **全局不变量**：确保总内存平衡（DCHECK 在 ~BufferAllocator）。

此文档基于提供的实现，如需代码示例或 UML，请进一步指定！
******************************************************************************
# BufferPool：：BufferAllocator 类
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
**************************************************************************
# BufferPool::BufferAllocator 类设计文档（开发指南）

## 1. 概述

### 1.2 设计目的
`BufferAllocator` 是 Impala BufferPool 的内部内存分配器，用于管理 2 的幂次大小的缓冲区分配。它在 `SystemAllocator`（基于 mmap/malloc + TCMalloc）基础上添加缓存层，支持空闲缓冲区（free buffers）和干净页面（clean pages）的复用，确保在 `system_bytes_limit` 内分配成功，同时最小化系统调用开销。

- **核心优化**：
  - **分层缓存**：per-core arenas（`per_core_arenas_`） + per-size lists（via `FreeBufferArena::PerSizeLists`），优先本地/NUMA 局部分配。
  - **渐进分配路径**：快速路径（本地 arena 复用） → NUMA 搜索/驱逐 → scavenge（回收） → 系统分配。

### 1.3 关键概念

- **Scavenge**：回收策略，3 次尝试（MAX_SCAVENGE_ATTEMPTS=3）：前 2 次 opportunistic（无全锁），最后 1 次 locked（全 arena 锁，保证成功）。
- **NUMA 优化**：优先当前核心 → 同节点 arena → 同节点驱逐 → 全局 scavenge。


## 2. 成员变量

| 成员变量                  | 类型                          | 描述                                                                 | 线程安全 | 初始化 |
|---------------------------|-------------------------------|----------------------------------------------------------------------|----------|--------|
| `pool_`                   | `BufferPool* const`           | 关联的 BufferPool。                                                  | 是（const） | 构造函数传入 |
| `system_allocator_`       | `boost::scoped_ptr<SystemAllocator>` | 底层系统分配器（mmap/malloc + huge pages）。                         | 是（scoped） | new SystemAllocator(min_buffer_len_) |
| `min_buffer_len_`         | `const int64_t`               | 最小缓冲区大小（2 的幂次）。                                        | 是（const） | 参数 |
| `max_buffer_len_`         | `const int64_t`               | 最大缓冲区大小（CalcMaxBufferLen 计算，确保 >= min）。               | 是（const） | CalcMaxBufferLen |
| `log_min_buffer_len_`     | `const int`                   | min_buffer_len_ 的 log2（BitUtil::Log2Ceiling64）。                  | 是（const） | 计算 |
| `log_max_buffer_len_`     | `const int`                   | max_buffer_len_ 的 log2。                                            | 是（const） | 计算 |
| `system_bytes_limit_`     | `const int64_t`               | 系统总内存上限。                                                     | 是（const） | 参数 |
| `system_bytes_remaining_` | `AtomicInt64`                 | 剩余可分配字节（原子更新，CAS 循环）。                               | 是（原子） | system_bytes_limit_ |
| `clean_page_bytes_limit_` | `const int64_t`               | 干净页面总缓存上限（跨所有 arena）。                                 | 是（const） | 参数 |
| `clean_page_bytes_remaining_` | `AtomicInt64`              | 干净页面剩余配额（添加前 Increment，回收后 Decrement）。             | 是（原子） | clean_page_bytes_limit_ |
| `per_core_arenas_`        | `std::vector<std::unique_ptr<FreeBufferArena>>` | per-core arenas（大小 = CpuInfo::GetMaxNumCores()）。                | 否（需外部同步） | 构造函数循环 new |
| `max_scavenge_attempts_`  | `int`                         | Scavenge 尝试次数（默认 3，可测试覆盖）。                            | 否（测试专用） | MAX_SCAVENGE_ATTEMPTS |

- **常量**：ALLOC_STAT_SAMPLE_RATE=64（采样率）；STATS_MAX_BUFFER_SIZE=4GB（直方图上限）。


此文档作为开发指南，聚焦实现细节。如需伪代码或特定方法扩展，请补充！
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
  // 调用Status BufferPool::BufferAllocator::AllocateInternal( BufferPool::Client* client, int64_t len, BufferHandle* buffer)
  /* 1：首先获取当前线程所在的core
        然后找到对应该core的FreeBufferArena
        然后从该FreeBufferArena中找到对应len的PerSizeLists
        然后从free_buffers的队列中（实际为堆，堆顶的buffer地址低）出队一个buffer。
        这个是最快fast path的分配路径。

   2：如果BufferAllocator::system_bytes_remaining_>len
      直接从system_allocator_->Allocate(len, buffer);
      这个是次优快速路径Fast-ish path

   3: 获取当前core所在的NUMA的socket上的所有cores
      上述cores在NUMA架构上位于同一个node上，

      3.1：首先尝试从隶属与同一个NUMA node的“其他”core获取buffer（i=1开始所以不包含当前core）。
        获取过程见FreeBufferArena::PopFreeBuffer

      3.2: 再次尝试从隶属于同一个NUMA node的所有core（包括当前core，因为遍历i=0开始）的clean_pages获取内存。
        方法是FreeBufferArena：：EvictCleanPage(len,buffer)

      3.3:从当前core或者从其他core驱逐
         scavenge 会回收所有core（不限于同一个NUMA nodes上的core）不同大小的缓冲区和页面（不限于 len），目的是腾出足够空间让系统分配器（SystemAllocator）能分配 len 字节。

         ScavengeBuffers有两种工作模式：
         - 非锁定模式（前 2 次）：
            start_core = current_core，从当前开始core，释放该core的free_buffers和clean_pages（见FreeBufferArena::freeSystemMemory)，
            如果还不够，继续遍历其他core。
         - 锁定模式（最后 1 次）：
            start_core = 0，从 0 开始（尊重锁序：低 ID 先锁），依次释放每个core的ree_buffers和clean_pages.
          
         - 以上是腾出空间的过程。腾出空间后
            然后调用Status status = system_allocator_->Allocate(len, buffer);分配内存

   */
  Status Allocate(ClientHandle* client, int64_t len,
      BufferPool::BufferHandle* buffer) WARN_UNUSED_RESULT;

  /// Frees 'buffer', which must be open before calling. Closes 'buffer' and updates
  /// internal state but does not release to any reservation.
  /*
       将buffer回收到对应core的FreeBufferArena，即FreeBufferArena对应大小的PerSizeLists的free_buffers列表
  */
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
  /*
    尝试释放最少bytes_to_free大小的内存给systemAllocator
    会从当前core开始逐个遍历所有core的FreeBufferArena。
    调用每个FreeBuferArena::freeSystemMemory,
    直到释放buffer大小达到bytes_to_free结束
   */
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
  [见后文专门讨论](#scavengebuffers)
  int64_t ScavengeBuffers(bool slow_but_sure, int current_core, int64_t target_bytes);


  /// Helper to free a list of buffers to the system. Returns the number of bytes freed.
  /*
    通过system_allocator_->Free(move(buffer));释放输入参数buffers的每个buffer。每个buffer的区域取消毒化
  */
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
# ScavengeBuffers
### 这段代码是否只清理当前核心？

**不是**。这段 slow path（scavenge 回收）代码**会遍历并清理所有核心（arenas）的空闲缓冲区和干净页面**，而非仅限于当前核心（current_core）。它通过调用 `ScavengeBuffers` 实现全局回收，确保在必要时从整个系统范围内释放内存。下面我一步步解释为什么，以及实现细节，帮助你理解。

#### 1. 代码片段的作用回顾
- 这段是 `AllocateInternal` 的**慢路径**（Slow path）：当快速/相对快速路径（本地复用、NUMA 搜索/驱逐）失败后，进入回收模式。
- 核心逻辑：循环 `max_scavenge_attempts_` 次（默认 3 次），每次调用 `ScavengeBuffers(final_attempt, current_core, len - delta)` 回收 `len - delta` 字节。
  - `final_attempt`：最后一次（attempt == 2）为 true，表示“锁定模式”（slow_but_sure）。
  - 如果总回收 `delta < len`，回滚 `system_bytes_remaining_.Add(delta)` 并返回 INTERNAL_ERROR（表示会计 bug：预留应保证足够内存）。
- **目的**：通过 scavenge 释放内存空间（从 free lists 和 clean pages），为后续系统分配腾出 `system_bytes_remaining_`。

#### 2. ScavengeBuffers 的实现（关键：全局遍历）
这段代码**不直接清理**，而是委托给 `ScavengeBuffers` 函数（私有方法）。查看其实现（从完整代码）：
```cpp
int64_t BufferPool::BufferAllocator::ScavengeBuffers(
    bool slow_but_sure, int current_core, int64_t target_bytes) {
  DCHECK_GT(target_bytes, 0);
  // 先尝试从剩余空间扣减（不需全额）
  int64_t bytes_found = DecreaseBytesRemaining(target_bytes, false, &system_bytes_remaining_);
  if (bytes_found == target_bytes) return bytes_found;

  // 确定起始核心：锁定模式从 0 开始（锁序），否则从当前核心（局部性）
  int start_core = slow_but_sure ? 0 : current_core;
  vector<std::unique_lock<SpinLock>> arena_locks;  // 锁定模式下预分配锁
  if (slow_but_sure) arena_locks.resize(per_core_arenas_.size());

  // **核心：遍历所有 arenas（所有核心）**
  for (int i = 0; i < per_core_arenas_.size(); ++i) {  // i 从 0 到 核心数-1
    int core_to_check = (start_core + i) % per_core_arenas_.size();  // 轮询所有核心
    FreeBufferArena* arena = per_core_arenas_[core_to_check].get();
    int64_t bytes_needed = target_bytes - bytes_found;
    bytes_found += arena->FreeSystemMemory(bytes_needed, bytes_needed,
         slow_but_sure ? &arena_locks[i] : nullptr).second;  // 每个 arena 释放内存
    if (bytes_found == target_bytes) break;  // 够了就停
  }
  DCHECK_LE(bytes_found, target_bytes);

  // 锁定模式下，持锁再扣减剩余空间（防竞态）
  if (slow_but_sure && bytes_found < target_bytes) {
    bytes_found += DecreaseBytesRemaining(target_bytes - bytes_found, true, &system_bytes_remaining_);
    DCHECK_EQ(bytes_found, target_bytes) << DebugString();  // 保证成功
  }
  return bytes_found;
}
```
- **遍历所有核心**：
  - `for (int i = 0; i < per_core_arenas_.size(); ++i)`：循环**所有 arenas**（per_core_arenas_.size() = 最大核心数，如 64）。
  - `core_to_check = (start_core + i) % size`：从 start_core 开始轮询（% 确保循环覆盖所有），**不是只当前核心**。
    - 非锁定模式（前 2 次）：start_core = current_core，从当前开始，优先局部，但仍遍历全。
    - 锁定模式（最后 1 次）：start_core = 0，从 0 开始（尊重锁序：低 ID 先锁），全锁 vector<unique_lock> 防其他线程抢内存。
- **每个 arena 的清理**：调用 `arena->FreeSystemMemory`，它：
  - 从大到小缓冲区大小遍历 PerSizeLists。
  - 释放 free_buffers（GetBuffersToFree，高地址优先防碎片）。
  - 驱逐 clean_pages（Dequeue FIFO，移动缓冲区到 free list，再释放）。
  - 返回 (释放字节, 声明字节)；声明字节用于 caller 分配。

#### 3. 为什么不只清理当前核心？
- **全局保证**：预留系统（ReservationTracker）是**全局**的，所有客户端共享 `system_bytes_limit_`。当前核心可能无足够内存，但其他核心有（e.g., 其他查询释放了）。只当前核心会违反“保证成功”原则，导致 INTERNAL_ERROR。
- **公平与负载均衡**：Impala 是多租户查询引擎，核心间负载不均。只当前会饿死本地，轮询确保公平回收。
- **性能权衡**：
  - **局部优先**：start_core = current_core（非锁定），第一轮试当前/邻近。
  - **全遍历必要**：如果同 NUMA 驱逐失败，仍需全局（慢路径），否则 OOM。
  - **锁定模式**：最后一次全锁所有 arenas（expensive but guaranteed），从 0 开始避死锁（锁序低→高）。
- **与前路径区别**：
  - Fast-ish Pop：i=1 排除当前（当前已试 PopFreeBuffer）。
  - Evict：i=0 包括当前（当前未试 Evict）。
  - Scavenge：全遍历（回收更广，跨大小/核心）。

#### 4. 示例模拟
假设 4 核心，current_core=1，target_bytes=1MB。
- **非锁定尝试**（attempt=0）：start_core=1，遍历：核心1（当前）→2→3→0。每个 FreeSystemMemory 释放部分。
- **锁定尝试**（attempt=2）：start_core=0，遍历：0→1→2→3，全锁 vector<arena_locks>，每个释放 + 最终 Decrement remaining。
- 如果总 bytes_found < 1MB：错误（但设计上不应发生）。

#### 5. 开发/优化建议
- **为什么这样设计**：平衡局部性（start_core）和完整性（全遍历）。高负载下，监控 `num_scavenges_`（>0 表示热点，调大 clean_page_bytes_limit_）。
- **潜在改进**：加随机 start_core（TODO IMPALA-4703）；或优先高水位 arena。
- **测试**：模拟多核心，验证 bytes_found == target（锁定模式）；检查锁序（无死锁）。

如果这是性能瓶颈，或想看 ScavengeBuffers 伪代码，我可以深入！
***********************************************
什么时候才会走到int64_t BufferPool::BufferAllocator::ScavengeBuffers(
    bool slow_but_sure, int current_core, int64_t target_bytes)，我觉得是BufferAllocator::AllocateInternal准备分配的Buffer大小，所有核心（arenas）都没有这个大小的Buffer和Page缓存
### ScavengeBuffers 函数的触发条件分析

你的理解基本正确，但需要稍作细化：`ScavengeBuffers` 确实是在 `AllocateInternal` 的**慢路径（slow path）**中被调用的，当系统剩余空间不足（`system_bytes_remaining_ < len`），且快速/相对快速路径（本地 + NUMA 复用/驱逐）都失败时，才会走到这里。但**不是严格“所有核心都没有这个大小（len）的 Buffer 和 Page 缓存”**——相反，scavenge 会**回收不同大小的缓冲区和页面**（不限于 len），目的是腾出足够空间让系统分配器（SystemAllocator）能分配 len 字节。它更像是“全局内存压力大，需要强制回收”的场景。下面我一步步解释触发逻辑、条件和实现意图，帮助你彻底澄清。

#### 1. 整体上下文：AllocateInternal 的分层路径
`AllocateInternal` 的设计是**渐进式**的：从 O(1) 快速复用到全局回收，确保分配成功（在 limit 内）。ScavengeBuffers 是最后的“重型武器”，只在前面路径全失败时触发。关键变量：
- `delta`：已回收/腾出的空间字节（初始 0）。
- 目标：腾出至少 `len` 字节，让 `system_bytes_remaining_ >= len`，然后系统分配。

路径流程（简化）：
1. **Fast path**：当前核心 PopFreeBuffer(len) 成功？返回（复用 len 大小空闲缓冲区）。
2. **Fast-ish path: Decrement remaining**：原子扣减 `system_bytes_remaining_` len 字节（全额或 0）。成功？直接系统分配（delta = len）。
3. **Fast-ish path: NUMA PopFreeBuffer**：遍历同 NUMA 其他核心 PopFreeBuffer(len) 成功？返回（复用 len）。
4. **Fast-ish path: NUMA EvictCleanPage**：遍历同 NUMA 全核心（包括当前）EvictCleanPage(len) 成功？返回（驱逐 len 大小干净页面，回收缓冲区）。
5. **Slow path: ScavengeBuffers**：如果以上全失败（delta == 0 < len），进入 while 循环，多次调用 ScavengeBuffers 回收空间，直到 delta >= len 或失败。
6. **系统分配**：delta >= len 后，调用 SystemAllocator::Allocate(len)。

**触发 ScavengeBuffers 的精确条件**：
- `DecreaseBytesRemaining(len, true, &system_bytes_remaining_)` 返回 0（无足够剩余空间，全额扣减失败）。
- **且** 当前核心 + 同 NUMA 核心的 len 大小空闲缓冲区（PopFreeBuffer）和干净页面（EvictCleanPage）都不可用（无或不足）。
- 此时，内存压力高：快速复用/驱逐失败，需要“更广、更深”的回收。

你的猜测“所有核心都没有这个大小的 Buffer 和 Page 缓存”接近，但不完全准确：
- **不是所有核心**：只检查当前 + 同 NUMA（~4-16 核心），全局其他节点未查（scavenge 会补上）。
- **不是严格‘没有’**：可能是“有但不足 len”（e.g., 小缓冲区多，但无 len 大小）；或有但锁争用失败。
- **ScavengeBuffers 的特殊性**：它回收**任意大小**的缓冲区/页面（从大到小遍历 PerSizeLists），不限于 len——目的是腾空间，而不是精确匹配 len。

#### 2. 代码触发点详解
从提供的代码片段：
```cpp
// ... Fast path 失败后 ...

// Fast-ish path: allocate a new buffer if there is room in 'system_bytes_remaining_'.
int64_t delta = DecreaseBytesRemaining(len, true, &system_bytes_remaining_);  // 尝试全扣 len
bool sample_sys_alloc_stats = false;
if (delta == len) {
  // 直接系统分配（有空间）
  // ...
} else {
  DCHECK_EQ(0, delta);  // 失败，delta=0
  // ... NUMA PopFreeBuffer (i=1, 排除当前) 失败 ...
  // ... NUMA EvictCleanPage (i=0, 包括当前) 失败 ...

  // Slow path: scavenge ...  # 这里触发
  int attempt = 0;
  int64_t count = current_core_arena->num_scavenges()->Increment(1);  // +1 scavenge 计数
  sample_sys_alloc_stats = count % ALLOC_STAT_SAMPLE_RATE == 0;  // 采样
  while (attempt < max_scavenge_attempts_ && delta < len) {  // delta=0 < len，进入
    bool final_attempt = attempt == max_scavenge_attempts_ - 1;  // 最后一次 true
    if (final_attempt) current_core_arena->num_final_scavenges()->Increment(1);
    delta += ScavengeBuffers(final_attempt, current_core, len - delta);  // 回收 len-delta
    ++attempt;
  }
  if (delta < len) {  // 回收不足，回滚 + 错误
    system_bytes_remaining_.Add(delta);
    return Status(TErrorCode::INTERNAL_ERROR, ...);  // bug: 预留应保证成功
  }
}
```
- **进入 while 的条件**：`delta < len`（初始 delta=0），且 attempt < 3。
- **每次调用**：`ScavengeBuffers(final_attempt, current_core, len - delta)`，target_bytes = len - delta（剩余需求）。
- **TODO IMPALA-4703**：暗示未来可随机变尝试次数（压力测试）。

#### 3. ScavengeBuffers 内部：为什么全局清理？
如前述，ScavengeBuffers 遍历**所有 arenas**（per_core_arenas_.size()）：
- 先 Decrement system_bytes_remaining_（非全额，false）。
- 然后 for 循环所有核心：`arena->FreeSystemMemory(target_bytes - bytes_found, ...)`。
  - FreeSystemMemory：从大到小大小，释放 free_buffers + 驱逐 clean_pages（FIFO），高地址优先（防碎片）。
- 锁定模式（final_attempt=true）：全锁 arenas（vector<unique_lock>），从核心 0 开始（锁序），+ 最终全扣 remaining（true）。
- **全局性**：即使 target_bytes 只为当前分配，它会从任意核心回收，更新全局 remaining——因为内存是共享的。

#### 4. 触发场景示例
- **典型触发**：高并发查询，system_bytes_remaining_ 接近 0；当前/NUMA 无 len=1MB 空闲/干净页面，但其他核心有小缓冲区（e.g., 512KB），scavenge 回收它们腾空间。
- **不触发**：有剩余空间（delta=len，直接系统分配）；或 NUMA Evict 成功（回收 len 页面）。
- **为什么设计这样**：预留（ReservationTracker）是全局的，确保“总预留 <= limit”，但局部可能紧——scavenge 强制均衡。

#### 5. 开发/调试提示
- **监控**：`num_scavenges_` 高 → 增加 clean_page_bytes_limit_ 或优化查询并行度。
- **测试**：模拟 delta=0，验证 3 次尝试后 delta >= len（锁定保证）。
- **潜在 bug**：如果 scavenge 失败（delta < len），是会计错误（e.g., 预留超限）——检查 ReservationTracker。

如果这个解释还不够清晰，或想分析 ScavengeBuffers 内部，我可以继续深挖！
*************************************************************