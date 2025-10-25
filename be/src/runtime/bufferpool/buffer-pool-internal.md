# BufferAllocator
[Class BuferAllocator](./buffer-allocator.md)
-------------------------------------------

# BufferPool

```cpp
// 此文件包含缓冲池内部使用的类定义。
//
/// +========================+
/// | 实现说明               |
/// +========================+
///
/// 锁获取顺序
/// =============
/// 锁获取顺序为：
/// 1. Client::lock_
/// 2. FreeBufferArena::lock_。如果获取多个 arena 锁，必须按升序获取。
/// 3. Page::lock,保护BufferHandle的访问，参见Page类注释
///
/// 如果通过页面列表获取 Page 引用，则 Page* 引用仅在持有列表锁时有效。
///
/// 页面状态
/// ===========
/// 每个 Page 对象在任何给定时刻最多属于一个 InternalList<Page>。
/// 每个页面要么被固定（pinned），要么未固定（unpinned）。未固定有多个子状态，
/// 由 Client/BufferPool 中的哪个列表包含该页面决定。
/// * 固定（Pinned）：当 'pin_count' > 0 时始终处于此状态。页面有一个缓冲区，并在
///     Client::pinned_pages_ 中。'pin_in_flight' 决定页面的子状态：
///   -> 当 pin_in_flight=false 时，缓冲区包含页面的数据，客户端可以
///      读写缓冲区。
///   -> 当 pin_in_flight=true 时，页面的数据正在从 scratch 磁盘读取到
///      缓冲区中。如果客户端尝试访问缓冲区，将阻塞在读取 I/O 上。
/// * 未固定 - 脏（Unpinned - Dirty）：当未固定页面的 scratch 写入尚未启动时。
///     页面在 Client::dirty_unpinned_pages_ 中。
/// * 未固定 - 写入进行中（Unpinned - Write in flight）：当脏未固定页面的 scratch 写入已启动但未
///     完成时。页面在
///     Client::write_in_flight_pages_ 中。就计费目的而言，这被视为脏页面。
/// * 未固定 - 干净（Unpinned - Clean）：当 scratch 写入完成但页面未被驱逐时。
///     页面在 BufferAllocator arena 中的干净页面列表中。
/// * 未固定 - 驱逐（Unpinned - Evicted）：在干净页面的缓冲区被回收后。页面
///     不在任何列表中。
///
/// 页面驱逐策略
/// ====================
/// 页面驱逐策略设计为，仅运行内存中（即不 unpin 页面）的客户端永不阻塞 I/O。
/// 为实现此目的，我们必须能够
/// 通过分配缓冲区或驱逐干净页面来满足预留。假设
/// 预留未过度提交（不应如此），此全局不变量可以通过为每个客户端强制执行本地不变量来维护：
///
///   reservation >= 返回给客户端的 BufferHandles
//                   + 固定页面 + 脏页面（脏未固定或写入进行中）
///
/// 本地不变量通过在任何分配新缓冲区或从干净页面回收缓冲区的操作的第一步写入页面到磁盘来维护。即
/// “脏页面”必须在不变量右侧的其他值增加之前减少。操作会阻塞等待足够的写入完成
/// 以满足不变量。

#pragma once

#include <iosfwd>
#include <memory>
#include <mutex>

#include "runtime/bufferpool/buffer-pool-counters.h"
#include "runtime/bufferpool/buffer-pool.h"
#include "runtime/bufferpool/reservation-tracker.h"
#include "util/condition-variable.h"
#include "util/internal-queue.h"
#include "util/spinlock.h"

// 确保在发布构建中移除 DCheckConsistency() 函数调用。
#ifndef NDEBUG
#define DCHECK_CONSISTENCY() DCheckConsistency()
#else
#define DCHECK_CONSISTENCY()
#endif

namespace impala {

class TmpFileGroup;
class TmpWriteHandle;

/// 页面的内部表示，可以被固定或未固定。参见
/// 类注释以了解不同页面状态的解释。
struct BufferPool::Page : public InternalList<Page>::Node {
  Page(Client* client, int64_t len);
  ~Page();

  std::string DebugString();

  // BufferPool::DebugString() 的辅助函数。
  static bool DebugStringCallback(std::stringstream* ss, BufferPool::Page* page);

  /// 该页面所属的客户端。
  Client* const client;

  /// 页面的长度（字节）。
  const int64_t len;

  /// 页面的固定计数。只在传递相关 PageHandle 的上下文中访问，因此无法被多个线程并发访问。
  int pin_count;

  /// 如果固定该页面的读取 I/O 已启动但未完成，则为 true。只在传递相关 PageHandle 的上下文中访问，因此只能通过 PageHandle::GetBuffer() 被多个线程并发访问，因为其他页面句柄操作不是线程安全的。这是原子的，以便 GetBuffer() 可以进行乐观检查以避免获取 'buffer_lock'。
  AtomicBool pin_in_flight;

  /// 如果有写入进行中、页面干净或页面被驱逐，则非空。
  std::unique_ptr<TmpWriteHandle> write_handle;

  /// 当此页面的写入完成时信号的条件变量。由
  /// client->lock_ 保护。
  ConditionVariable write_complete_cv_;

  /// 当页面未固定且未驱逐时访问 'buffer' 必须持有此锁（即如果页面被固定或驱逐，则安全访问 'buffer'）。
  /// 这个锁只配合访问下面的buffer成员变量，不配合上面的write_compelete_cv_使用
  SpinLock buffer_lock;

  /// 包含页面内容的缓冲区。只有在页面被驱逐时关闭。否则打开。
  BufferHandle buffer;
};

/// 围绕 InternalList<Page> 的包装器，用于跟踪列表中的字节数。
class BufferPool::PageList {
 public:
  PageList() : bytes_(0) {}
  ~PageList() {}

  void Enqueue(Page* page) {
    list_.Enqueue(page);
    bytes_ += page->len;
  }

  bool Remove(Page* page) {
    if (list_.Remove(page)) {
      bytes_ -= page->len;
      return true;
    }
    return false;
  }

  Page* Dequeue() {
    Page* page = list_.Dequeue();
    if (page != nullptr) {
      bytes_ -= page->len;
    }
    return page;
  }

  Page* PopBack() {
    Page* page = list_.PopBack();
    if (page != nullptr) {
      bytes_ -= page->len;
    }
    return page;
  }

  void Iterate(const boost::function<bool(Page*)>& fn) { list_.Iterate(fn); }
  void IterateFirstN(const boost::function<bool(Page*)>& fn, int n) {
    list_.IterateFirstN(fn, n);
  }
  bool Contains(Page* page) { return list_.Contains(page); }
  Page* tail() { return list_.tail(); }
  bool empty() const { return list_.empty(); }
  int size() const { return list_.size(); }
  int64_t bytes() const { return bytes_; }

  void DCheckConsistency() {
    DCHECK_GE(bytes_, 0);
    DCHECK_EQ(list_.empty(), bytes_ == 0);
  }

 private:
  InternalList<Page> list_;
  int64_t bytes_;
};

/// 客户端的内部状态。
class BufferPool::Client {
 public:
  Client(BufferPool* pool, TmpFileGroup* file_group, const string& name,
      ReservationTracker* parent_reservation, MemTracker* mem_tracker,
      MemLimit mem_limit_mode, int64_t reservation_limit, RuntimeProfile* profile);

  ~Client() {
    DCHECK_EQ(0, num_pages_);
    DCHECK_EQ(0, buffers_allocated_bytes_);
  }

  /// 释放此客户端的预留。
  void Close() { reservation_.Close(); }

  /// 使用通过 AllocateBuffer() 分配的 'buffer' 创建固定页面。
  /// 调用者不应持有客户端或页面锁。
  BufferPool::Page* BufferPool::Client::CreatePinnedPage(BufferHandle&& buffer) {
        Page* page = new Page(this, buffer.len());
        page->buffer = move(buffer);
        page->pin_count = 1;

        std::lock_guard<std::mutex> lock(lock_);
        // buffer被交给了对应的page去管理，所以通过pinned_pages_.bytes() 而不是buffers_allocated_bytes_记录使用情况
        buffers_allocated_bytes_ -= page->len;
        pinned_pages_.Enqueue(page);
        ++num_pages_;
        DCHECK_CONSISTENCY();
        return page;
}

  /// 重置 'handle'的状态为初始状态，清理对 handle->page 的引用并释放与 handle->page 关联的任何资源。
  /// 从pinned_pages_或dirty_unpinned_pages_或BufferPool::BufferAllocator的某个FreeBufferArena的clean_pages_列表中删除该page。
  /// 同时通过page->write_handle删除对应磁盘上文件上指定的数据区域。
  /// 如果页面是固定状态，可以传入 'out_buffer'，页面的缓冲区将通过out_buffer返回给用户。
  /// 如果没有提供out_buffer，handle中的buffer将被返回给BufferPool::BufferAllocator的某个FreeBufferArena的free_buffers
  /// 调用者不应持有客户端的锁或 handle->page_->buffer_lock。
  void DestroyPageInternal(PageHandle* handle, BufferHandle* out_buffer = NULL);

  /// 将page从pinned_pages移动到dirty_unpinned_pages_队列中以反映 'page' 现在是脏未固定页面。
  /// 调用异步函数WriteDirtyAsync将dirty_unpinned_pages写到磁盘的。
  /// 至于是否有dirty_unpinned_pages被写到磁盘，看当前内存使用状态。
  /// 调用者不应持有客户端的锁或 page->buffer_lock。
  ///  注意学习：page从一个队列移动到另一个队列时，只使用了BufferPool::Client一个锁。
  ///  也就是一个锁保护了两个队列。
  void MoveToDirtyUnpinned(Page* page);

  /// 将未固定页面移动到固定状态，在数据结构之间移动：
         如果page在dirty_unpinned_pages中：
            将该page从diry_unpinned_pages移动到pinned_pages。
        如果page在in_flight_write_pages中：
             等待该page写操作完成（完成后该page在BufferPool::BufferAllocator的某个core的PerSizeLists中的clean_pages里）。
             我们将在上述clean_pages中的page移动到pinned_pages前，需要通过cleanPages保证当前BufferPool::Client有page->len的空间。
             即dirty_unpinned_pages_只能保留 int64_t target_dirty_bytes = reservation_.GetReservation() - buffers_allocated_bytes_- pinned_pages_.bytes() - len;大小。多余的diryty_unpinned_pages都要通过
              WriteDirtyPagesAsync(min_bytes_to_write)写到磁盘。
            dirty_unpinned_pages能保留多少page原则：来自 eviction policy（不变量）：reservation >= buffers + pinned + dirty（unpinned/in-flight）。


  /// 如果必要从磁盘读取。确保页面有缓冲区。如果数据已在
  /// 内存中，确保数据在页面的缓冲区中。如果数据在
  /// 磁盘上，启动异步读取数据并将页面的 'pin_in_flight' 设置
  /// 为 true。调用者不应持有客户端的锁或 page->buffer_lock。
  Status StartMoveToPinned(ClientHandle* client, Page* page) WARN_UNUSED_RESULT;

  /// 将有固定进行中的页面移动回驱逐状态，撤销
  /// StartMoveToPinned()。调用者不应持有客户端的锁或 page->buffer_lock。
  void UndoMoveEvictedToPinned(Page* page);

  /// 如果
  /// page->pin_in_flight 由 StartMoveToPinned() 设置为 true，则完成将驱逐页面的数据带入内存的工作。
  Status FinishMoveEvictedToPinned(Page* page) WARN_UNUSED_RESULT;

  /// 必须在通过 AllocateBuffer() 或
  /// AllocateUnreservedBuffer() API 分配 'len' 的缓冲区之前调用一次。从客户端的预留扣除并更新
  /// 内部计费。如果需要，清理脏页面以满足缓冲池的
  /// 内部不变量。调用者不应持有页面或客户端锁。
  /// 如果 'reserved' 为 true，我们假设内存已预留。如果为
  /// false，则尝试如果需要增加预留。
  ///
  /// 成功时，返回 OK 并如果非 NULL 将 'success' 设置为 true。如果遇到错误，
  /// 例如在清理页面时，返回错误状态。如果无法为未预留分配
  /// 增加预留，返回 OK 并将 'success'
  /// 设置为 false（对于未预留分配，'success' 必须非 NULL）。
  Status PrepareToAllocateBuffer(
      int64_t len, bool reserved, bool* success) WARN_UNUSED_RESULT;

  /// ClientHandle::DecreaseReservationTo() 的实现。
  Status DecreaseReservationTo(int64_t max_decrease, int64_t target_bytes) WARN_UNUSED_RESULT;

  /// ClientHandle::TransferReservationTo() 的实现。
  Status TransferReservationTo(ReservationTracker* dst, int64_t bytes, bool* transferred);

  /// 在通过 FreeBuffer() API 释放 'len' 的缓冲区后调用，以更新
  /// 内部计费并将缓冲区释放到客户端的预留。调用者不应持有页面或
  /// 客户端锁。
  void FreedBuffer(int64_t len) {
    std::lock_guard<std::mutex> cl(lock_);
    reservation_.ReleaseTo(len);
    buffers_allocated_bytes_ -= len;
    DCHECK_CONSISTENCY();
  }

  /// 等待 'page' 的进行中写入完成。
  /// 调用者必须通过 'client_lock' 持有 'lock_'。不应
  /// 持有 page->buffer_lock。
  void WaitForWrite(std::unique_lock<std::mutex>* client_lock, Page* page);

  /// 测试助手：等待所有进行中写入完成。
  /// 调用者不应持有 'lock_'。
  void WaitForAllWrites();

  /// 断言 'client_lock' 持有 'lock_'。
  void DCheckHoldsLock(const std::unique_lock<std::mutex>& client_lock) {
    DCHECK(client_lock.mutex() == &lock_ && client_lock.owns_lock());
  }

  int64_t min_buffer_len() const { return pool_->min_buffer_len(); }
  ReservationTracker* reservation() { return &reservation_; }
  const BufferPoolClientCounters& counters() const { return counters_; }
  bool spilling_enabled() const { return file_group_ != NULL; }
  void set_debug_write_delay_ms(int val) { debug_write_delay_ms_ = val; }
  bool has_unpinned_pages() const {
    // 无锁读取安全，因为其他线程不应调用创建、销毁或 unpin 页面的 BufferPool
    // 函数。
    return pinned_pages_.size() < num_pages_;
  }

  /// 打印关于客户端状态的调试信息。调用者不应持有 'lock_'。
  std::string DebugString();

 private:
  // 检查客户端的一致性，如果不一致则 DCHECK。必须持有 'lock_'。
  void DCheckConsistency() {
    DCHECK_GE(buffers_allocated_bytes_, 0) << DebugStringLocked();
    pinned_pages_.DCheckConsistency();
    dirty_unpinned_pages_.DCheckConsistency();
    in_flight_write_pages_.DCheckConsistency();
    DCHECK_LE(pinned_pages_.size() + dirty_unpinned_pages_.size()
            + in_flight_write_pages_.size(),
        num_pages_) << DebugStringLocked();
    // 检查鉴于我们的驱逐策略，我们是否将足够的页面刷新到磁盘。
    DCHECK_GE(reservation_.GetReservation(), buffers_allocated_bytes_
            + pinned_pages_.bytes() + dirty_unpinned_pages_.bytes()
            + in_flight_write_pages_.bytes()) << DebugStringLocked();
  }

  /// 必须在分配或回收 'len' 的缓冲区之前调用一次。确保
  /// 足够的脏页面刷新到磁盘以在分配后满足缓冲池的内部
  /// 不变量。调用者应通过
  /// 'client_lock' 持有 'lock_'。如果 'lazy_flush' 为 true，仅在需要回收
  /// 'len' 时写入页面，并且如果错误阻止刷新足够的页面，则不返回写入错误。
  Status CleanPages(std::unique_lock<std::mutex>* client_lock, int64_t len,
      bool lazy_flush = false);

  /// 启动脏未固定页面的异步写入到磁盘。确保至少
  /// 'min_bytes_to_write' 字节的写入将异步写入。可能
  /// 更积极地启动写入，以便 I/O 和计算可以重叠。如果
  /// 遇到任何错误，则设置 'write_status_'。因此
  /// 在读取回任何页面之前必须检查 'write_status_'。调用者必须持有 'lock_'。
  /*
      1:至少写出min_bytes_to_write字节，同时启动至少磁盘个数个TmpFileGroup::Write异步写操作。
      2:同时这个函数只启动异步写（TmpFileGroup::write)，而不等待异步写完成。
      3:补充一句，TmpFileGroup写出的page，在TmpFileGroup::write指定的回调函数中将in_flight_write_pages已经完成写操作的page移动到BufferPool::BufferAllocator对应core的FreeArena的对应大小PerSizeLists中的clean_pages中。
      4： WriteDirtyPagesAsync是由函数MoveToDirtyUnpinned 或 CleanPages调用的。
        WriteDiryPageAsync一经调用，dirty_unpinned_pages_ 会持续异步写到磁盘（即使后续不调用 moveToDirtyUnpinned 或 CleanPages），直到dirty_unpinned_pages_为空为止。
        Impala BufferPool 的 spilling（溢出）机制设计为异步、链式（递归）驱动的，一旦 MoveToDirtyUnpinned 或 CleanPages 被调用并触发写操作，脏未固定页面（dirty_unpinned_pages_）就会持续通过 TmpFileGroup::Write 异步写入磁盘，直到没有更多脏页可写（因为每个TmpFileGroup::write的回调函数会在每个page被写完的回调函数继续调用WriteDirtyPagesAsync）。这不是“一次性”操作，而是自我维持的循环，目的是最大化 I/O 重叠和磁盘利用率。
  */
  void WriteDirtyPagesAsync(int64_t min_bytes_to_write = 0);

  /* 1：TmpFileGroup::write的回调函数，当 'page' 的写入完成时调用。
      将page从BufferPool::Client::in_flight_write_pages移动到BufferPool::BufferAllocator对应core的FreeArena的对应大小PerSizeLists中的clean_pages中。
    2：继续调用WriteDirtyPagesAsync(); 启动另一个异步写操作。
    3：通知page和BufferPool::Client各自的write_complete_cv_
  */
  void WriteCompleteCallback(Page* page, const Status& write_status);

  /// 通过分配新缓冲区、启动从磁盘的异步读取并将页面移动到 'pinned_pages_' 将驱逐页面移动到固定状态。调用者必须通过 'client_lock' 锁定 client->impl，并且 handle->page 必须解锁。
  /// 'client_lock' 被释放然后重新获取。
  Status StartMoveEvictedToPinned(
      std::unique_lock<std::mutex>* client_lock, ClientHandle* client, Page* page);

  /// 与 DebugString() 相同，但调用者必须已经持有 'lock_'。
  std::string DebugStringLocked();

  /// 拥有客户端的缓冲池。
  BufferPool* const pool_;

  /// 用于分配 scratch 空间的文件组。如果为 NULL，则禁用 spilling。
  TmpFileGroup* const file_group_;

  /// 标识客户端的名称。
  const std::string name_;

  /// 客户端的预留跟踪器。客户端固定的所有页面计入
  /// 'reservation_' 的使用。
  ReservationTracker reservation_;

  /// 此客户端的 RuntimeProfile 计数器，由客户端的 RuntimeProfile 拥有。
  /// 所有非 NULL。
  BufferPoolClientCounters counters_;

  /// 延迟写入完成的调试选项。
  int debug_write_delay_ms_;

  /// 保护以下成员变量的锁；
  std::mutex lock_;

  /// 当此客户端的写入完成时信号的条件变量。
  /*
        5.3:这个BufferPool::Client::write_complete_cv_配合BufferPool::Client::lock_使用；
  */
  ConditionVariable write_complete_cv_;

  /*1：用于确保 CleanPages() 中一次只有一个线程活跃。
    2：CleanPages函数通常只指定需要腾出多少空间，
    所以CleanPages可能最终会写多个page到磁盘（写出到磁盘的page会从BufferPool::Client的in_flight_write_pages移动到BufferPool::BufferAllocator的某个core的FreeBufferArena的某个size的PerSizeLists中的clean_pages中）
    3：所以CleanPages是批量写出pages
    4：CleanPages是同步接口，所以调用了WriteDirtyPagesAsync后，需要条件变量clean_pages_done_cv_进行同步
    5：因为WriteDiryPagesAsync调用的TmpFileGroup::write每个page写成功后的回调函数都会通知这个write_complete_cv_也会通知page自己的write_complete_cv_）。
       5.1：而CleanPages需要等到足够的page写出到磁盘。所以write_complete_cv_被通知以后，还需要检查条件（写出page个数）是否满足。
       5.2：这和我们通常while （） {condition.wait}避免虚假唤醒的机制不同(这个是真的唤醒)。相同的是都用了while
       5.3只有通过write_complete_cv_被唤醒后再次检测是否由足够的page将被写出（通过while循环）。另一个条件变量clean_pages_done_cv_才通知当前CleanPages完成。
       5.4：也就是连续使用了两个条件变量BufferPool::Client::write_complete_cv_和BufferPool::Client::clean_pages_done_cv.
       5.5 CleanPages是同步接口。由while循环和条件变量BufferPool::Client：：write_complete_cv_保证足够多的page已经被写出。由BufferPool::Client::clean_pages_done_cv_保证只有一个函数调用CleanPages.
       5.5我们发现BufferPool::Client::lock_被三个条件变量使用：BufferPool::Page::write_compelete_cv_和BufferPool::Client::write_complete_cv和BufferPool::Client::clean_pages_done_cv_
 


  */
  bool cleaning_pages_ = false;
  ConditionVariable clean_pages_done_cv_;

  /// 由写入操作返回的所有非 OK 状态合并到此状态中。
  /// 所有依赖页面成功写入磁盘的操作（例如。
  /// 从磁盘读取回页面）必须在继续之前检查 'write_status_'，以
  /// 正确传播异步发生的写入错误。
  /// 写入错误是客户端的全局，因此可以传播到客户端的任何返回 Status 的
  /// 操作（即使是对不同 Pages 或 Buffers 的操作）。
  /// 写入错误不可恢复，因此最好尽快传播它们，
  /// 而不是等待以特定方式传播它们。
  Status write_status_;

  /// 此客户端的总页面数。用于调试和强制执行在客户端之前销毁所有
  /// 页面。
  int64_t num_pages_;

  /// 返回给客户端的 BufferHandles 中的缓冲区总字节数（即从
  /// AllocateBuffer() 或 ExtractBuffer() 获取）。
  int64_t buffers_allocated_bytes_;

  /// 此客户端的所有固定页面。
  PageList pinned_pages_;

  /* 此客户端的脏未固定页面，这些page的还没有开始异步写到磁盘。
     正在进行异步写出到磁盘的page会放在in_flight_write_pages中。
     对于ile_group_==nullptr，也就是禁用spilling的BufferPool::Client的dirty_unpinned_pages_一定是空的
   */
  PageList dirty_unpinned_pages_;

  /* 此客户端的脏未固定页面，正在进行异步写到磁盘的页面，
     从队列尾部开始遍历写出到磁盘，也就是LIFO方式
      因为操作符通常具有顺序访问
      模式，其中最近驱逐的页面将是最后读取的。
  */
  PageList in_flight_write_pages_;
};
}
```