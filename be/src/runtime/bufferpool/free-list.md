```cpp

namespace impala {

using BufferHandle = BufferPool::BufferHandle;

/// 一个非线程安全的空闲缓冲区列表。
///
/// 缓冲区Buffers由调用者分配，并可以使用 AddFreeBuffer() 添加到列表中以供后续检索。
/// 如果列表非空，调用 PopFreeBuffer() 将返回先前使用 AddFreeBuffer() 添加到列表的
/// 其中一个缓冲区。FreeList 对添加到它的缓冲区的大小或其他属性不敏感。
///
/// 列表中的缓冲区Buffers可以在任何时候释放，例如如果列表存储了太多
/// 空闲缓冲区（根据某种策略）。调用者负责实现
/// 该策略，并在适当的时候调用 FreeBuffers() 或 FreeAll()。
///
/// 地址空间碎片化
/// ---------------------------
/// 为了减少内存碎片化，空闲列表首先分hands out发具有较低内存
/// 地址的缓冲区 buffers，并首先释放具有较高内存地址的缓冲区。如果缓冲区
/// 通过不考虑内存地址的策略分发，随着时间的推移，地址空间中空闲缓冲区的
/// 分布将变得本质上随机。然后，如果空闲缓冲区被 unmap，将在虚拟
/// 内存映射中产生许多空洞，在某些情况下会给 OS 带来困难，例如在 Linux 中超过
/// 最大 mmapped() 区域数 (vm.max_map_count)。使用这种方法
/// 将倾向于将空闲缓冲区集中到地址空间的较高部分，从而允许
/// 在大多数情况下合并空洞。

/*关于vm.max_map_count参数
  max_map_count
This file contains the maximum number of memory map areas a process may have. Memory map areas are used as a side-effect of calling malloc, directly by mmap, mprotect, and madvise, and also when loading shared libraries.

While most applications need less than a thousand maps, certain programs, particularly malloc debuggers, may consume lots of them, e.g., up to one or two maps per allocation.

The default value is 65530.

这段话说的是为了减少空洞，这个实现倾向于将空洞集中到地址空间较高的部分，估计将空洞集中到地址空间较低的部分也行
*/
class FreeList {
 public:
  FreeList() {}

  /// 获取一个空闲缓冲区。如果列表非空，返回 true 并将 'buffer' 设置为
  /// 先前使用 AddFreeBuffer() 添加的其中一个缓冲区。否则返回 false。
  bool PopFreeBuffer(BufferHandle* buffer) {
    if (free_list_.empty()) return false;
    std::pop_heap(free_list_.begin(), free_list_.end(), HeapCompare);
    *buffer = std::move(free_list_.back());
    free_list_.pop_back();
    return true;
  }

  /// 将一个空闲缓冲区添加到列表中。
  void AddFreeBuffer(BufferHandle&& buffer) {
    buffer.Poison();
    free_list_.emplace_back(std::move(buffer));
    std::push_heap(free_list_.begin(), free_list_.end(), HeapCompare);
  }

  /// 从列表中获取 'num_buffers' 个具有最高内存地址的缓冲区以释放。
  /// 平均时间复杂度为 n log n，其中 n 是列表的当前大小。
  vector<BufferHandle> GetBuffersToFree(int64_t num_buffers) {
    vector<BufferHandle> buffers;
    DCHECK_LE(num_buffers, free_list_.size());
    // 排序列表，以便我们释放具有较高内存地址的缓冲区。
    // 注意，排序后的列表仍然是一个有效的 min-heap。
    std::sort(free_list_.begin(), free_list_.end(), SortCompare);

    for (int64_t i = 0; i < num_buffers; ++i) {
      buffers.emplace_back(std::move(free_list_.back()));
      free_list_.pop_back();
    }
    return buffers;
  }

  /// 返回当前在列表中的缓冲区数量。
  int64_t Size() const { return free_list_.size(); }

 private:
  friend class FreeListTest;

  DISALLOW_COPY_AND_ASSIGN(FreeList);

  /// 按内存地址排序的比较函数。
  inline static bool SortCompare(const BufferHandle& b1, const BufferHandle& b2) {
    return b1.data() < b2.data();
  }

  /// 按内存地址排序的比较函数。需要是 SortCompare() 的逆，因为 C++ 提供 max-heap。
  inline static bool HeapCompare(const BufferHandle& b1, const BufferHandle& b2) {
    return SortCompare(b2, b1);
  }

  /// 空闲内存缓冲区的列表。作为按缓冲区内存地址排序的 min-heap 维护。
  std::vector<BufferHandle> free_list_;
};
}

#endif
```