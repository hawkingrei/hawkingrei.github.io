---
title:     "LevelDB代码阅读：Arena"
date:     "2017-12-09 14:00:00"
tags:     ["LevelDB"]
---



## Arena源代码分析


首先我们来看定义

```c++
class Arena {
 public:
  Arena();
  ~Arena();  

  // Return a pointer to a newly allocated memory block of "bytes" bytes.
  char* Allocate(size_t bytes);

  // Allocate memory with the normal alignment guarantees provided by malloc
  char* AllocateAligned(size_t bytes);

  // Returns an estimate of the total memory usage of data allocated
  // by the arena.
  size_t MemoryUsage() const {
    return reinterpret_cast<uintptr_t>(memory_usage_.NoBarrier_Load());
  }

 private:
  char* AllocateFallback(size_t bytes);
  char* AllocateNewBlock(size_t block_bytes);

  // Allocation state
  char* alloc_ptr_;
  size_t alloc_bytes_remaining_;

  // Array of new[] allocated memory blocks
  std::vector<char*> blocks_;

  // Total memory usage of the arena.
  port::AtomicPointer memory_usage_;

  // No copying allowed
  Arena(const Arena&);
  void operator=(const Arena&);
};
```
从定义来看，除了构造函数和析构函数，用户只需要调用```Allocate```、```AllocateAligned```和```MemoryUsage```就可以完成内存的调用。

其中MemoryUsage的定义如下，

```c++
  size_t MemoryUsage() const {
    return reinterpret_cast<uintptr_t>(memory_usage_.NoBarrier_Load());
  }
```

MemoryUsage直接读取私有的memory_usage，转换成uintptr_t类型。


NoBarrier_Load方法是Leveldb的AtomicPointer实现，其位置在Leveldb的port目录下，是专门处理Leveldb跨平台特性的。之后会来介绍。```Alllocate```的实现如下：


```c++
inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use).
  assert(bytes > 0);
  if (bytes <= alloc_bytes_remaining_) {
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  return AllocateFallback(bytes);
}
```

首先检查输入的```bytes```是否大于0，然后申请内存大小```bytes```是否小于剩余预先分配的内存，如果小于，则移动```alloc_ptr_```指针，修改剩余内存大小，否则使用AllocateFallback向系统申请内存。


下面来一下AllocateFallback的实现

```c++
char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  // We waste the remaining space in the current block.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}
```

AllocateFallback有两种分配内存的方案，通过判断```bytes > kBlockSize / 4```，如果为真，直接按参数的大小来分配内存，否则按照```kBlockSize```的大小```4096```来分配，令```alloc_ptr```指向新分配的区域，但是其被分配的未使用内存则被浪费了。

接着我们来看一下AllocateAligned函数，其作用是分配内存，保证其内存对齐。AllocateAligned函数的定义如下

```c++
char* Arena::AllocateAligned(size_t bytes) {
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  assert((align & (align-1)) == 0);   // Pointer size should be a power of 2
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1);
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  size_t needed = bytes + slop;
  char* result;
  if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // AllocateFallback always returned aligned memory
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align-1)) == 0);
  return result;
}
```

首先通过```(sizeof(void*) > 8) ? sizeof(void*) : 8```获取```align```，在32位系统下是4，在64位系统下是8，
然后我们通过```reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1)```，取当前指针模```align-1```的值，这样我们就可以知道需要补充的内存量， 所有有 slop = align - current_mod, 因此也就有了 needed = bytes + slop 和 result = alloc_ptr + slop。之后的流程就和```Allocate```函数如出一辙了，不再赘述。

最后就是Arena的析构函数和构造函数：

```c++
Arena::Arena() : memory_usage_(0) {
  alloc_ptr_ = NULL;  // First allocation will allocate a block
  alloc_bytes_remaining_ = 0;
}
```

```c++
Arena::~Arena() {
  for (size_t i = 0; i < blocks_.size(); i++) {
    delete[] blocks_[i];
  }
}
```

这里需要注意```Arena::Arena() : memory_usage_(0)```的作用，其作用是来初始化列表相当于在构造函数内进行相应成员变量的赋值，但两者是有差别的。在初始化列表中是对变量进行初始化，而在构造函数内是进行赋值操作。两者的差别在对于const类型数据的操作上表现得尤为明显。const类型的变量必须在定义时进行初始化，而不能对const型的变量进行赋值，因此const类型的成员变量只能（而且必须）在初始化列表中进行初始化。


## Rocksdb的Arena

Rocksdb是Leveldb的魔改升级版，相比leveldb，rocksdb对内存块的分配主要做了两点改进：一是抽象出内存分配Allocator，支持对不同内存管理策略进行定制扩展；二是启用HugePage支持，提高大内存机器下内存分配和访问的性能。这会在之后的文章中来专门介绍。

## 收获

通过看leveldb的Arena实现，了解了memory pool的实现，随便复习了一下c++和CSAPP，看来之后还是要好好做一下CSAPP中Allocator的作业。在写这篇文章的过程中，挖了两个坑***Leveldb的AtomicPointer实现*** 和 ***Rocksdb的Arena介绍***，希望可以尽快填坑=。=

## 参考

- [LevelDB源码剖析之Arena内存管理](http://mingxinglai.com/cn/2013/01/leveldb-arena/)
- [C++标准转换运算符reinterpret_cast](https://csruiliu.github.io/blog/2016/11/01/c++11_basic/)
- [Arena内存管理优化-RocksDB源码剖析(0)](http://www.pandademo.com/2016/09/arena-rocksdb-source-dissect-0/)
