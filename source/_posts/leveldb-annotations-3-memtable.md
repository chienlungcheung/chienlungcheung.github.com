---
title: "Leveldb 源码详解系列之三: MemTable 设计与实现"
date: 2020-10-01 21:47:36
tags:
  - leveldb
  - LSM-Tree
  - db
---

<!-- toc -->

memtable 可以看作是 log 文件的内存形式, 但是格式不同. 每个 log 文件在内存有一个对应的 memtable, 它和正在压实的 memtable 以及磁盘上的各个 level 包含的文件构成了数据全集. memtable 的本质就是一个 SkipList.

# 1 核心文件和核心类

| 核心类 | 所在文件 | 用途  |
| :----: | :----: | :----: |
| leveldb::MemTable | db/memtable.cc; db/memtable.h | 本文主角 |
| leveldb::Arena | util/arena.cc; util/arena.h | 负责内存管理(仅分配和释放, 无回收) |
| leveldb::SkipList<Key, Comparator> | db/skiplist.h | 作为 MemTable 类底层存储(基于内存) |

# 2 MemTable 核心成员变量

| 字段 | 类型 | 用途  |
| :----: | :----: | :----: |
| comparator_ | struct leveldb::MemTable::KeyComparator | 用于 SkipList 比较 key |
| refs_ | int | 该 memtable 引用计数, memtable 的拷贝构造和赋值构造都是禁用的, 只能通过增加引用计数复用 |
| arena_ | class leveldb::Arena | 内存池, 给 SkipList 分配 Node 时候使用 |
| table_ | typedef leveldb::SkipList<const char *, leveldb::MemTable::KeyComparator> leveldb::MemTable::Table | SkipList, 存储 memtable 里的数据 |

# 3 MemTable 核心方法

| 方法 | 用途  |
| :----: | :----: |
| explicit MemTable(const InternalKeyComparator& comparator) | 传入一个自定义或数据库提供的比较器, 生成一个 MemTable 实例. |
| void Ref() | 递增引用计数, MemTable 是基于引用计数的, 每个 MemTable 实例初始引用计数为 0, 调用者必须至少调用一次 Ref() 方法. |
| void Unref() | 递减引用计数, 计数变为 0 后删除该 MemTable 实例. |
| size_t ApproximateMemoryUsage() | 返回当前 memtable 中数据字节数的估计值, 当 memtable 被修改时调用该方法也是安全的, 该方法底层实现直接用的 arena_ 持有内存的字节数. |
| Iterator* NewIterator() | 返回一个迭代器, 该迭代器可以用来遍历整个 memtable 的内容. |
| void Add(SequenceNumber seq, ValueType type, const Slice& key, const Slice& value) | 根据指定的序列号和操作类型将 user_key 转换为 internal_key 然后和 value 一起向 memtable 新增一个数据项. 该数据项是 key 到 value 的映射, 如果操作类型 type==kTypeDeletion 则 value 为空. 最后数据项写入了底层的 SkipList 中. 每个数据项编码格式为 [varint32 类型的 internal_key_size, internal_key, varint32 类型的 value_size, value] internal_key 由 [user_key, tag] 构成. |
| bool Get(const LookupKey& key, std::string* value, Status* s) | 如果 memtable 包含 key 对应的 value, 则将 value 保存在 *value 并返回 true. 如果 memtable 包含 key 对应的 deletion, 则将 NotFound 错误存在 *status, 并返回 true; 其它情况返回 false.  |

## 3.1 MemTable 写方法 

当需要向 memtable 加入一个 <key, value> 时, 可以调用 `void Add(SequenceNumber seq, ValueType type, const Slice& key, const Slice& value)` 达成目标.

该方法负责将用户 key 转换为一个 InternalKey, 然后连同 value 一起进行编码, 组成一个形如下述的数据项:


    +------------------------------+---------+--------+--------+---------------+-----+
    |     internal key size        | seq num |op type |user key|  value size   |value|
    |      (varint32 type)         |(7 bytes)|(1 byte)|        |(varint32 type)|     |
    |(user key + seq num + op type)|         |        |        |               |     |
    +------------------------------+---------+--------+--------+---------------+-----+

其中该数据项前四个部分构成了 InternalKey, 该结构是 MemTable 做排序的依据. 最后将该数据项插入到底层的 SkipList 中. 具体处理流程见下面代码注释:

```c++
void MemTable::Add(SequenceNumber s, ValueType type,
                   const Slice& key,
                   const Slice& value) {
  // 插入到 memtable 的数据项的数据格式为
  // (注意, memtable key 的构成与 LookupKey 一样, 即 user_key + 序列号 + 操作类型):
  // [varint32 类型的 internal_key_size, 
  //  user_key, 
  //  序列号 + 操作类型, 
  //  varint32 类型的 value_size, 
  //  value]
  // 其中, internal_key_size = user_key size + 8
  size_t key_size = key.size();
  size_t val_size = value.size();
  size_t internal_key_size = key_size + 8;
  // 编码后的数据项总长度
  const size_t encoded_len =
      VarintLength(internal_key_size) + internal_key_size +
      VarintLength(val_size) + val_size; 
  // 分配用来存储数据项的内存    
  char* buf = arena_.Allocate(encoded_len); 
  // 将编码为 varint32 格式的 internal_key_size 写入内存
  char* p = EncodeVarint32(buf, internal_key_size); 
  // 将 user_key 写入内存
  memcpy(p, key.data(), key_size); 
  p += key_size;
  // 将序列号和操作类型写入内存.
  // 注意, 序列号为高 7 个字节;  
  // 将序列号左移 8 位, 空出最低 1 个字节写入操作类型 type. 
  EncodeFixed64(p, (s << 8) | type); 
  p += 8;
  // 将编码为 varint32 格式的 value_size 写入内存
  p = EncodeVarint32(p, val_size); 
  // 将 value 写入内存
  memcpy(p, value.data(), val_size); 
  assert(p + val_size == buf + encoded_len);
  // 将数据项插入跳跃表
  table_.Insert(buf); 
}
```

可以看到由于模块化抽象做得好, 整体来说还是比较简单的. 下面再重点说一下上述处理中有关内存分配(`arena_.Allocate()`)和数据项插入到 SkipList (`table_.Insert()`) 的相关操作.


## 3.2 MemTable 读方法

`bool Get(const LookupKey& key, std::string* value, Status* s)` 用于实现针对 memtable 的读操作.

当用户调用 DB 的 `Get` 方法查询某个 key 的时候, 具体步骤是这样的(具体实现位于 `leveldb::Status leveldb::Version::Get(const leveldb::ReadOptions &options, const leveldb::LookupKey &k, string *value, leveldb::Version::GetStats *stats)`, DB 的 `Get` 方法会调用前述实现.):
- 1 先查询当前在用的 memtable, 查到返回, 未查到下一步
- 2 查询正在转换为 sorted table 的 memtable 中寻找, 查到返回, 未查到下一步 
- 3 在磁盘上采用从底向上 level-by-level 的寻找目标 key. 
  - 由于 level 越低数据越新, 因此, 当我们在一个较低的 level 找到数据的时候, 不用在更高的 levels 找了.
  - 由于 level-0 文件之间可能存在重叠, 而且针对同一个 key, 后产生的文件数据更新所以先将包含 key 的文件找出来按照文件号从大到小(对应文件从新到老)排序查找 key; 针对 level-1 及其以上 level, 由于每个 level 内文件之间不存在重叠, 于是在每个 level 中直接采用二分查找定位 key.

其中上面 1,2 两步骤查询 memtable 使用的即为本节开头提到的 Get 方法.

```c++
// 如果 memtable 包含 key 对应的 value, 则将 value 保存在 *value 并返回 true. 
// 如果 memtable 包含 key 对应的 deletion(memtable 的删除不是真的删除, 而是一个带有删除标记的 Add 操作, 
// 而且 key 对应的 value 为空), 则将 NotFound 错误存在 *status, 并返回 true; 
// 其它情况返回 false. 
// LookupKey 就是 memtable 数据项的前半部分, 布局同 internal key. 
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  // LookupKey 结构同 internal key.
  // 注意由于比较时, 当 userkey 一样时会继续比较序列号, 
  // 而且序列号越大对应 key 越小, 所以外部调用 Get()
  // 方法前用户组装 LookupKey 的时候, 传入的序列号不能小于
  // MemTable 每个 key 的序列号. 外部具体实现是采用 DB 记录的最大
  // 序列号.
  Slice memkey = key.memtable_key();
  // 为底层的 skiplist 创建一个临时的迭代器
  Table::Iterator iter(&table_);
  // 返回第一个大于等于 memkey 的数据项(数据项在 iter 内部存着),
  // 这里的第一个很关键, 当 userkey 相同但是序列号不同时, 序列号
  // 大的那个 key 对应的数据更新, 同时由于 internal key 比较规则
  // 是, userkey 相同序列号越大对应 key 越小, 所以 userkey 相同时
  // 序列号最大的那个 key 肯定是第一个.
  iter.Seek(memkey.data()); 
  // iter 指向有效 node, 即 node 不为 nullptr
  if (iter.Valid()) { 
    // 每个数据项格式如下:
    //    klength  varint32
    //    userkey  char[klength-8] // 源码注释这里有误, 应该是 klength - 8
    //    tag      uint64
    //    vlength  varint32
    //    value    char[vlength]
    // 通过比较 user_key 部分和 ValueType 部分来确认是否是我们要找的数据项. 
    // 不去比较序列号的原因是上面调用 Seek() 的时候已经跳过了非常大的序列号
    // (internal_key 比较逻辑是序列号越大 internal_key 越小, 而我们
    // 通过 Seek() 寻找的是第一个大于等于某个 internal_key 的节点).  
    //
    // 注意, memtable 将底层的 SkipList 的 key(确切应该说是数据项)
    // 声明为了 char* 类型. 这里的 entry 是 SkipList.Node 里包含的整个数据项. 
    const char* entry = iter.key();
    uint32_t key_length;
    // 解析 internal_key 长度
    const char* key_ptr = GetVarint32Ptr(entry, entry+5, &key_length); 
    // 比较 user_key. 
    // 因为 internal_key 包含了 tag 所以任意两个 internal_key 
    // 肯定是不一样的, 而我们真正在意的是 user_key, 
    // 所以这里调用 user_comparator 比较 user_key. 
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8),
            key.user_key()) == 0) {
      // 解析 tag, 包含 7 字节序列号和 1 字节操作类型(新增/删除)
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      // 解析 tag 中的 ValueType. 
      // leveldb 删除某个 user_key 的时候不是通过一个插入墓碑消息实现的吗? 
      // 那怎么确保在 SkipList.Seek() 时候返回删除操作对应的数据项, 
      // 而不是之前同样 user_key 对应的真正的插入操作对应的数据项呢? 
      // 机巧就在于 internal_key 的比较原理 user_key 相等的时候, 
      // tag 越大 internal_key 越小, 
      // 这样后执行的删除操作的 tag(序列号递增了, 即使不递增, 但由于
      // 删除对应的 ValueType 大于插入对应的 ValueType 也可以确保
      // 后执行的删除操作的 tag 大于先执行的插入操作的 tag)
      // 这里有个比较技巧的地方. 
      switch (static_cast<ValueType>(tag & 0xff)) {
        // 找到了
        case kTypeValue: {
          Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
          value->assign(v.data(), v.size());
          return true;
        }
        // 找到了, 但是已经被删除了
        case kTypeDeletion:
          *s = Status::NotFound(Slice());
          return true;
      }
    }
  }
  return false;
}
```

以上代码重要地方有如下几点:
- 参数 LookupKey 结构如 internal key, 前文介绍过具体结构不再赘述. 要强调的是, 构造其实例时, user key 部分采用用户传入的, 但是序列号要用数据库目前最大值, 这样可以确保作为比较环节的序列号不会影响我们的查找过程.
- 在底层 SkipList 查找之前, 构造了一个临时的迭代器, 迭代器结构体待后文介绍 SkipList 时详细介绍.
- 迭代器的 Seek 方法返回的是第一个大于等于目标 key 的节点, 这个第一个非常非常重要, 这也是为什么我们前面强调 LookupKey 的序列号必须足够达到不污染查找过程.
- Leveldb 的删除会插入一个墓碑消息, 其标识记录到 key 的 tag 部分, 所以查到数据的时候先不要高兴太早, 需要确认下 tag 的操作类型字段值.

## 3.3 MemTable 迭代器

我们知道, MemTable 和磁盘上的 sstables 文件一起构成了数据全集, 而且 leveldb 支持迭代整个数据库. 这就要求 MemTable 也是可迭代的, 这样才能和磁盘文件迭代器串联在一起构成一个超级迭代器供用户迭代.

从 MemTable 实例返回一个迭代器的方法为:

```c++
Iterator* MemTable::NewIterator() {
  return new MemTableIterator(&table_);
}
```

MemTable 的迭代器实现很简单, 它提供了一个 `class leveldb::MemTableIterator`, 但整个类仅仅是一个 wrapper. 因为 MemTable 依赖的 SkipList 提供了一个完整的迭代器, MemTableIterator 仅仅在内部封装了一个 `leveldb::SkipList<Key, Comparator>::Iterator` 实例即实现了对 MemTable 的迭代, 所以它的实现就很简单了:

```c++
// 该类全部方法都是对 leveldb::SkipList<Key, Comparator>::Iterator 方法的二次封装
class MemTableIterator: public Iterator {
 public:
  explicit MemTableIterator(MemTable::Table* table) : iter_(table) { }

  virtual bool Valid() const { return iter_.Valid(); }
  virtual void Seek(const Slice& k) { iter_.Seek(EncodeKey(&tmp_, k)); }
  virtual void SeekToFirst() { iter_.SeekToFirst(); }
  virtual void SeekToLast() { iter_.SeekToLast(); }
  virtual void Next() { iter_.Next(); }
  virtual void Prev() { iter_.Prev(); }
  virtual Slice key() const { return GetLengthPrefixedSlice(iter_.key()); }
  virtual Slice value() const {
    Slice key_slice = GetLengthPrefixedSlice(iter_.key());
    return GetLengthPrefixedSlice(key_slice.data() + key_slice.size());
  }

  // 该迭代器默认返回 OK
  virtual Status status() const { return Status::OK(); }

 private:
  // 这个是真正干活的家伙
  MemTable::Table::Iterator iter_;
  // 用于 EncodeKey 方法存储编码后的 internal_key
  std::string tmp_; 

  // No copying allowed
  MemTableIterator(const MemTableIterator&);
  void operator=(const MemTableIterator&);
};
```

下文介绍 SkipList 时会着重介绍迭代器.

## 3.4 MemTable 内存估计

MemTable 提供了一个方法 `ApproximateMemoryUsage()` 来返回当前 MemTable 占用的内存空间大小. 它的实现依托底层的 Arena, 实现很简单:

```c++
size_t MemTable::ApproximateMemoryUsage() { return arena_.MemoryUsage(); }
```

Arena 每次分配内存的时候, 都会将分配的内存字节数累加到计数器上.

# 4 MemTable 比较器之 KeyComparator

MemTable 既然是有序的, 那么任何操作就需要一个比较器. Memtable 内部定义了一个结构体 `struct leveldb::MemTable::KeyComparator`, 它封装了 MemTable 的比较逻辑, 具体如下:

```c++
// 一个基于 internal_key 的比较器
// 注意下它的函数调用运算符重载方法的参数类型, 都是 char*, 
// 原因就是 memtable 底层的 SkipList 的 key 类型就是 char*, 
// 而类 KeyComparator 对象会被传给 SkipList 作为 key 比较器. 
struct KeyComparator {
  const InternalKeyComparator comparator;
  explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) { }
  // 注意, 这个操作符重载方法很关键, 该方法的会先从 char* 类型地址
  // 获取 internal_key, 然后对 internal_key 进行比较. 
  // 该方法未在 memtable 直接调用, 而是被底层的 SkipList 使用了.
  int operator()(const char* a, const char* b) const;
};
```

该结构体除了在 `MemTable::Get()` 直接使用比较 user key 以外, 还会被用于构造 SkipList 实例.

其中函数调用运算符的定义如下:

```c++
int MemTable::KeyComparator::operator()(const char* aptr, const char* bptr)
    const {
  // 提取数据项的前半部分, 即 internal_key
  Slice a = GetLengthPrefixedSlice(aptr);
  Slice b = GetLengthPrefixedSlice(bptr);
  // 对 internal_key 进行比较
  return comparator.Compare(a, b);
}
```

代码注释中提到的 internal key 在前面章节介绍过了, 不再赘述. 该结构体真正干活的是其封装的 `comparator`, 它是一个 `class leveldb::InternalKeyComparator` 实例, 比较器很重要的有两点:
- 1. 它有一个唯一的名字, 非后向兼容的比较器如果重名则会导致数据库混乱. 毕竟 leveldb 是有序数据库, 顺序至关重要, 一旦被破坏就会混乱.
- 2. 核心是 `Compare` 方法, 它决定了什么叫做顺序. `InternalKeyComparator` 用于比较 internal key, 而 internal key 包含 user key, 而 user key 是用户自己定义的所以比较器也由用户提供. 如果用户不提供, 则采用默认(见 `leveldb::Options::Options()` 定义)的 `class leveldb::<unnamed>::BytewiseComparatorImpl`, 它是一个基于字典序进行逐字节比较的内置 comparator 实现. `InternalKeyComparator` 比较时采用如下处理:
  - 如果 user_key 相等, 则序列号越小 internal_key 越大;
  - 如果序列号也相等, 则操作类型(更新/删除)越小 internal_key 越大(因为 leveldb 可以确保序列号单调递增且唯一, 所以实际上用不到该字段).


# 5 MemTable 内存管理之 Arena

Leveldb 专门为了管理内存定义了一个类 `class leveldb::Arena`, 可以把该类看作是 leveldb 的内存池实现, 但是它只负责对完分配, 并不会做回收重利用. 默认情况下, `Arena` 维护的内存块都是 4KB 大小.

注意, 同 `MemTable` 类一样, `Arena` 类全部操作需要调用者确保线程安全.

该类核心字段如下:

| 字段 | 类型 | 用途  |
| :----: | :----: | :----: |
| alloc_ptr_ | char* | 指向 arena 当前空闲字节起始地址 |
| alloc_bytes_remaining_ | size_t | arena 当前空闲字节数 |
| blocks_ | std::vector<char*> | 存放通过 new[] 分配的全部内存块 |
| memory_usage_ | port::AtomicPointer | arena 持有的全部内存字节数 |

同 `MemTable`, `Arena` 也不支持拷贝构造和赋值构造.

该类核心方法如下:

| 方法 | 用途  |
| :----: | :----: |
| Arena() | 默认构造方法, 无任何限制. |
| char* Allocate(size_t bytes) | 从该 arena 返回一个指向大小为 bytes 的内存的指针. bytes 必须大于 0, 具体见实现. |
| char* AllocateAligned(size_t bytes) | 返回一个对齐后的可用内存的地址. 具体对齐逻辑见实现.  |
| size_t MemoryUsage() const | 返回该 arena 持有的全部内存的字节数的估计值(未采用同步设施). |
| size_t MemoryUsage() const | 返回该 arena 持有的全部内存的字节数的估计值(未采用同步设施). |

## 5.1 如何向 Arena 申请指定大小的内存

可以通过方法 `Arena::Allocate` 申请指定大小的内存, 该方法会被 memtable 的 `Add()` 方法直接调用. 如果当前 `Arena` 内存池剩余内存足够则直接分配, 否则向操作系统进行申请. 

具体处理流程如下:

```c++
inline char* Arena::Allocate(size_t bytes) {
  // 如果我们允许分配 0 字节内存, 那该方法返回什么的语义就有点乱, 
  // 所以我们不允许返回 0 字节(我们内部也没这个需求). 
  assert(bytes > 0);
  // arena 剩余内存够则直接分配
  if (bytes <= alloc_bytes_remaining_) { 
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  // arena 可用内存不能满足用户需求, 向系统申请新内存
  return AllocateFallback(bytes);
}
```

上面的 `AllocateFallback()` 根据用户需求决定向系统申请的内存大小, 具体处理如下:

```c++
// 当 arena 可用内存不够时调用该方法来申请新内存
char* Arena::AllocateFallback(size_t bytes) {
  // 如果用户要申请的内存超过默认内存块大小(4KB)的 1/4, 
  // 则单独按照用户指定大小分配一个内存块, 否则单独分配一个默认
  // 大小(4KB)的内存块给用户会造成太多空间浪费. 
  // 新分配的内存块会被加入 arena 然后将其起始地址返回给用户,
  // 所分配的内存不会纳入 Arena 管理, 但是占用空间会被纳入计数. 
  if (bytes > kBlockSize / 4) {
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  // 直接分配一个默认大小内存块给用户, 此处所分配
  // 内存会被纳入 Arena 管理, 等待用户后续内存申请.
  // alloc_ptr_ 被覆盖之前指向的空间被浪费了.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  // result 是返回给用户的
  char* result = alloc_ptr_;
  // 移动 alloc_ptr_ 指向未被占用内存
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}
```

上述处理中还涉及一个辅助方法 `AllocateNewBlock`, 它负责向系统申请固定大小内存并将其纳入 Arena 管理.

具体处理如下:

```c++
// 分配一个大小为 block_bytes 的块并将其加入到 arena
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);
  // 调用者确保线程安全, 这里无需强制同步.
  memory_usage_.NoBarrier_Store(
      reinterpret_cast<void*>(MemoryUsage() + block_bytes + sizeof(char*)));
  return result;
}
```

## 5.2 如何向 Arena 申请对齐后的内存

除了上面提到的 `Arena::Allocate(size_t bytes)` 方法, Arena 还提供另一个内存分配方法, 它支持按照特定字节大小将所分配的内存进行对齐, 它的签名为 `char* Arena::AllocateAligned(size_t bytes)`, 该方法没有被 memtable 代码直接调用, 而是由其底层依赖的 SkipList 所使用. 后面分析 SkipList 代码时还会再次说明.

```c++
// 按照 8 字节或者指针长度进行内存对齐, 然后返回对齐后分配的内存起始地址
char* Arena::AllocateAligned(size_t bytes) {
  // 如果机器指针的对齐方式超过 8 字节则按照指针的对齐方式进行对齐; 
  // 否则按照 8 字节进行对齐. 
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  // 确保当按照指针大小对齐时, 机器的指针大小是 2 的幂
  assert((align & (align-1)) == 0);
  // 求 alloc_ptr 与 align 的模运算的结果, 
  // 以确认 alloc_ptr 是否恰好为 align 的整数倍. 
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1);
  // 如果 alloc_ptr 的值恰好为 align 整数倍, 
  // 则已经满足对齐要求, 可以从该地址直接进行分配; 
  // 否则, 需要进行手工对齐, 比如 align 为 8, alloc_ptr 等于 15, 
  // 则需要将 alloc_ptr 增加 align - current_mod = 8 - 7 = 1 个字节. 
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  // 虽然用户申请 bytes 个字节, 但是因为对齐要求, 
  // 实际消耗的内存大小为 bytes + slop
  size_t needed = bytes + slop; 
  char* result;
  // 如果 arena 空闲内存满足要求则直接分配
  if (needed <= alloc_bytes_remaining_) { 
    // 将 result 指向对齐后的地址(对齐会导致前 slop 个字节被浪费掉)
    result = alloc_ptr_ + slop; 
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else { 
    // 否则从堆中申请新的内存块, 注意重新分配
    // 内存块时 malloc 会保证对齐, 无序再如上做手工对齐.
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align-1)) == 0);
  return result;
}
```

# 6 MemTable 底层存储之 SkipList

MemTable 是有序的, 为了确保查询和插入迅速, 它采用了 SkipList 作为存储结构.

## 6.1 SkipList 原理简介

开始之前先大体介绍下 SkipList 这个数据结构. 目前该数据结构在开源项目中用的越来越多代替红黑树等存在, 这里包括 Redis 都用到了该数据结构. 

你在学校可能没学过这个数据结构, 但是你肯定在操作系统课上学过多级页表. SkipList 加速原理跟多级页表差不多, 当然后者如此设计也为了节省存储. 整个内存字节空间分成若干页面, 然后把全部页面分成若干组, 然后把上一步若干组进一步分为多个二级组 ...; 查找时从最外层找起, 从外向里索引粒度逐渐减小, 最后一下子定位到内存空间某个字节位置, 这里虽然没说排序, 但是由于每个字节已经被编码所以其实也是有序的. 

SkipList 与之类似, 每个元素所在节点在一个链表上按序排列, 如果我们把每个节点看作内存空间一个字节位置, 那么接下来我们就要为它们建索引了:
- 第 0 级索引就是原始链表, 一个接一个, 共 $n$ 个节点
- 第 $1$ 级索引就是在上一级基础上间隔一个选取一个, 这样构成第 $1$ 级索引, 共 $\frac{n}{2}$ 个节点
- 第 $2$ 级索引也是在上一级基础设间隔一个取一个, 这样构成第 $2$ 级索引, 共 $\frac{n}{2^2}$ 个节点
- ... ...
- 最后从上一级只能提取两个节点了, 这就是最后一级索引了因为没必要再往下分了, 假设共有 h 级, 那么这就是第 h-1 级索引, 该级索引共 $\frac{n}{2^{h-1}}$ 个节点

我们调过头反着看, 每一层索引节点数都是其上一层索引的 $\frac{1}{2}$, 这不就是个平衡二叉树吗? 总的节点数为 $2n$ 个. 每次查询时, 从最外层索引向里找, 每次在每一层只需查询一次, 最多查询 h 次也就是 $\log_2^{2n} = 1 + \log_2^n \approx \log_2^n$ 即可找到或则确认不存在. 我们把最外层看作最高层, 最里层看作最底层, 则 SkipList 的高度为 h.

可以看出时间复杂度同平衡二叉树如红黑树是一样的, 只是空间复杂度要多一倍. 那为什么不用红黑树呢, 嘿嘿, 你手写一次就知道了. 那么 SkipList 是如何确保平衡的呢? 方法强大又朴素, 抛硬币, 五五分. 每次插入元素时, 自上而下定位到其插入位置, 在第 $0$ 级插入元素时, 通过抛硬币的办法决定是否将该元素拔擢到第 $1$ 级索引中, 如果拔擢成功则再次抛硬币决定是否将其继续拔擢到第 $2$ 级索引, 以此类推, 除非上一级拔擢成功则继续抛硬币, 否则停止拔擢过程, 索引插入的时间复杂度也是 $\log_2^n$. 删除类似, 从外层到内层, 查找到该元素后, 则从该层开始递进删除每一层中的该元素, 时间复杂度依然是 $\log_2^n$.

好了, 我们来看看 leveldb 是如何实现 SkipList 的.

## 6.2 SkipList 实现

下面详细说明 SkipList 实现时的细节以及一些辅助的数据结构.

### 6.2.1 SkipList 核心数据成员

该类核心字段如下:

| 字段 | 类型 | 用途  |
| :----: | :----: | :----: |
| compare_ | Comparator | 比较器, 用于比较 key, 初始化以后不可更改 |
| arena_ | Arena* | 用于分配 Node 所用的内存 |
| head_ | Node* | dummy node |
| max_height_ | port::AtomicPointer | 指向存储当前 skiplist 最大高度的变量的地址, max_height_ <= kMaxHeight. |
| rnd_ | Random | 指向存储当前 skiplist 最大高度的变量的地址, max_height_ <= kMaxHeight. |

同 `MemTable`, `SkipList` 也不支持拷贝构造和赋值构造.

### 6.2.2 SkipList 核心方法

该类核心方法如下:

| 方法 | 用途  |
| :----: | :----: |
| explicit SkipList(Comparator cmp, Arena* arena) | 构造方法, cmp 用于比较 keys, arena 用做内存池. |
| void Insert(const Key& key) | 将 key 插入到 SkipList 实例中. |
| bool Contains(const Key& key) const | 当且仅当 sliplist 中存在与 key 相等的数据项时才返回 true. |

线程安全相关说明：
- 写操作 `Insert` 需要外部同步设施, 比如 mutex. 
- 读操作 `Contains` 需要一个保证, 即读操作执行期间, SkipList 不能被销毁; 只要保证这一点, 读操作不需要额外的同步措施. 

SkipList 运行不变式：
- (1) 已分配的 nodes 直到 SkipList 被销毁才能被删除. 这很容易保证, 因为我们不会删除任何 SkipList nodes. 
- (2) 一个 Node 一旦被链接到 SkipList 上, 那这个 Node 的内容, 除了 next/pre 指针以外, 都是 immutable 的. 


#### 6.2.2.3 辅助数据结构 Node

SkipList 每个节点由类 `template<class Key, class Comparator> struct leveldb::Node<Key, Comparator>` 表示, 它包含两数据成员:
- `key`, 其实叫 entry 更符合实际. 因为该成员其实不只包含 key, 还包含 value 部分. 但是只要涉及查询, 比较时仅比较前半部分即 internal key.
- `next_`, 默认是一个长度为 1 的 `port::AtomicPointer` 数组(但是我估计作者们想分配长度为 0 的数组, 但是标准 C/++ 不允许). 这个数组的最大长度取决于当前节点最大要做第几级索引(从 0 开始计数). 一旦确定第几级, 后续调用 `leveldb::SkipList<Key, Comparator>::NewNode(const Key &key, int height)` 时就能把 Node 的 `key` 成员和当前数组分配到连续内存中, 这样对缓存友好, 而且释放时可以一次性释放. 如果 `next_` 是指向数组的指针则要分多次释放了.

Node 的方法都比较简单, 共四个, 详细注释如下:

```c++
// 自带 acquire 语义, 返回该 node 在第 n 级(计数从 0 开始) 索引层的后继节点的指针
Node* Next(int n) {
  assert(n >= 0);
  // 采用 acquire 语义可以确保如下两点:
  // - 当前线程后续针对 next_[n] 节点的读写不会被重排序到此 load 之前;
  // - 其它线程在此 load 之前针对 next_[n] 节点的全部写操作此时均可见.
  return reinterpret_cast<Node*>(next_[n].Acquire_Load());
}

// 自带 release 语义, 设置该 node 在第 n 级(计数从 0 开始) 索引层的后继节点
void SetNext(int n, Node* x) {
  assert(n >= 0);
  // 采用 release 语义可以确保如下两点:
  // - 在此 store 之前, 当前线程针对 next_[n] 节点的读写不会被重排序到此 store 之后;
  // - 在此 store 之后, 其它线程针对 next_[n] 节点的读写看到的都是此 store 写入的值.
  next_[n].Release_Store(x);
}

// 同 Next, 但无同步防护.
Node* NoBarrier_Next(int n) {
  assert(n >= 0);
  return reinterpret_cast<Node*>(next_[n].NoBarrier_Load());
}

// 同 SetNext, 但无同步防护.
void NoBarrier_SetNext(int n, Node* x) {
  assert(n >= 0);
  next_[n].NoBarrier_Store(x);
}
```

上面提到的 acquire/release 语义, 将会在本文最后一节做详细介绍, 感兴趣可以阅读.

#### 6.2.2.4 用于为新数据分配存储的 NewNode 方法

该方法用于为 SkipList 执行 Insert 方法时为所插入数据分配一个节点. 传入的第一个参数为要存储的数据项(虽然它叫 key, 但只有前半截是所谓的 internal key, 后半截是 value size 和 value); 第二个参数是通过 `RandomHeight()` 方法预先计算的索引层数, 即该节点最多可以做第几级(从 0 开始计数)索引.

```c++
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node*
SkipList<Key,Comparator>::NewNode(const Key& key, int height) {
  // 要分配的空间存储的是用户数据和当前节点在 SkipList 各个索引层的后向指针, 
  // 其中后者是现算出来的.
  char* mem = arena_->AllocateAligned(
      // 为啥减 1? 因为 Node.next_ 已默认分配了一项
      sizeof(Node) + sizeof(port::AtomicPointer) * (height - 1));
  // 此乃定位 new, 即在 mem 指向内存位置创建 Node 对象
  return new (mem) Node(key); 
}
```

这里分配内存用到了我们前面介绍 `Arena` 类时分析的 `AllocateAligned` 方法. 该方法针对指针类型做了内存对齐(`AtomicPointer` 本身仅一个 `void*` 指针类型数据成员).

#### 6.2.2.5 用于确定某个节点索引层数的 RandomHeight 方法

该方法对应前面讲述 SkipList 原理时如何确定一个节点要被拔擢到最高第几层. 那里说是抛硬币, 五五分. Leveldb 实现 SkipList 时采用更为保守的拔擢策略, 每次递进概率仅为 1/4.

```c++
// 返回一个高度值, 返回值落于 [1, kMaxHeight], 
// SkipList 实现默认索引层最多 12 个.
template<typename Key, class Comparator>
int SkipList<Key,Comparator>::RandomHeight() {
  // 以 1/kBranching 概率循环递增 height. 
  // 每次拔擢都是在前一次拔擢成功的前提下再进行, 如果前一次失败则停止拔擢. 
  // 假设 kBranching == 4, 则返回 1 概率为 1/4, 返回 2 概率为 1/16, .... 
  static const unsigned int kBranching = 4;
  // 每个节点最少有一层索引(就是原始链表)
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

#### 6.2.2.6 用于插入数据的 Insert 方法

插入过程说起来也比较简单:
1. 查找待插入数据的前驱节点, 这个是通过从 SkipList 查找第一个不小于待插入数据的节点来做到的.
2. 确定待插入节点的索引层数, 这个就是随机大法.
3. 更新 SkipList 当前索引层数最大值
4. 为待插入数据生成一个新节点
5. 将新节点插入到前驱和后驱之间, 每一个索引层都要插入一遍.

具体代码如下:

```c++
// 该方法非线程安全, 需要外部同步设施. 
template<typename Key, class Comparator>
void SkipList<Key,Comparator>::Insert(const Key& key) {
  // pre 将用于存储 key 对应的各个索引层的前驱节点
  Node* prev[kMaxHeight];
  // 找到第一个大约等于目标 key 的节点, 一会会把 key
  // 插到这个节点前面.
  // 如果为 nullptr 表示当前 SkipList 节点都比 key 小.
  Node* x = FindGreaterOrEqual(key, prev); 

  // 虽然 x 是我们找到的第一个大于等于目标 key 的节点, 
  // 但是 leveldb 不允许重复插入 key 相等的数据项.
  assert(x == nullptr || !Equal(key, x->key));

  // 确定待插入节点的最大索引层数
  int height = RandomHeight();
  // 更新 SkipList 实例维护的最大索引层数
  if (height > GetMaxHeight()) {
    // 如果最大索引层数有变, 则当前节点将是索引层数最多的节点,
    // 需要将前面求得的待插入节点的前驱节点高度补齐.
    for (int i = GetMaxHeight(); i < height; i++) {
      // 新生成了几个 level, key 对应的前驱节点肯定都是 dummy head
      prev[i] = head_; 
    }
    //fprintf(stderr, "Change height from %d to %d\n", max_height_, height);

    // 这里在修改 max_height_ 无需同步, 哪怕同时有多个并发读线程. 
    // 其它并发读线程如果观察到新的 max_height_ 值, 
    // 那它们将会要么看到 dummy head 新的索引层(注意 SkipList 
    // 初始化时会把 dummy head 的索引高度直接初始化为最大, 默认是 12, 
    // 所以不存在越界问题)的值都为 nullptr, 要么看到的是
    // 下面循环将要赋值的新节点 x. 
    max_height_.NoBarrier_Store(reinterpret_cast<void*>(height));
  }

  // 为待插入数据创建一个新节点
  x = NewNode(key, height);
  // 将 x 插入到每一层前后节点之间, 注意是每一层, 
  // 插入的时候都是先采用 no barrier 方式为 x 后继赋值, 此时 x 还不会被其它线程看到; 
  // 然后插入一个 barrier, 则上面 no barrier 的修改针对全部线程都可见了(其中也包括
  // 了 NewNode 时可能发生的通过 NoBarrier_Store 方式修改的 arena_.memory_usage_), 
  // 最后修改 x 前驱的后继为自己. 
  for (int i = 0; i < height; i++) {
    // 注意该循环就下面两步, 而且只有第二步采用了同步设施, 尽管如此,
    // 第一步的写操作对其它线程也是可见的. 
    // 这是 Release-Acquire ordering 语义所保证的. 
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);
  }
}
```

#### 6.2.2.6 用于查找数据是否存在的 Contains 方法

```c++
template<typename Key, class Comparator>
bool SkipList<Key,Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, nullptr);
  if (x != nullptr && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}
```

很简单有没有? 只需检查 `FindGreaterOrEqual` 返回的节点包含的 key 是否与目标 key 一致即可确认.

#### 6.2.2.7 读写都要依赖的辅助方法 FindGreaterOrEqual

上面提到的 `Insert()` 方法和 `Contains()` 方法都用到了 `FindGreaterOrEqual()` 方法, 下面马上要介绍的迭代器而用到了该方法. 有必要单独介绍一下.

```c++
// 返回第一个大于等于目标 key 的 node 的指针; 
// 返回 nullptr 意味着全部 nodes 的 key 都小于参数 key.
//
// 如果想获取目标节点的前驱, 则令参数 pre 非空, 
// 所找到的 node 所在索引层的前驱节点将被保存到 pre[] 对应层.
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node* 
SkipList<Key,Comparator>::FindGreaterOrEqual(const Key& key, Node** prev)
    const {
  // head_ 为 SkipList 原始数据链表的起始节点,
  // 该节点不存储用户数据, 仅用作哨兵.
  Node* x = head_;
  // 每次查找都是从最高索引层开始查找, 只要确认可能存在
  // 才会降到下一级更细致索引层继续查找.
  // 索引层计数从 0 开始, 所以这里减一才是最高层.
  int level = GetMaxHeight() - 1; 
  while (true) {
    // 下面用的 Next 方法是带同步设施的, 其实由于 SkipList 对外开放的操作
    // 需要调用者自己提供同步, 所以这里可以直接用 NoBarrier_Next.
    Node* next = x->Next(level);
    if (KeyIsAfterNode(key, next)) {
      // key 大于 next, 在该索引层继续向后找
      x = next; 
    } else {
      // key 可能存在.
      //
      // 如果 key 比 SkipList 中每个 node 的 key 都小, 
      // 那么最后返回的 node 为 head_->Next(0), 
      // 同时 pre 里面存的都是 dummy head; 
      // 调用者需要使用返回的 node 与自己持有 key进一步进行对比,
      // 以确定是否找到目标节点. 
      if (prev != nullptr) prev[level] = x;
      if (level == 0) {
        // 就是它！如果 key 比 SkipList 里每个 node 的都大, 则 next 最终为 nullptr.
        return next;  
      } else {
        // 确定目标范围, 但是粒度太粗, 下沉一层继续找
        level--;
      }
    }
  }
}
```


#### 6.2.2.8 辅助数据结构 Iterator

上面讲了好些方法, 怎么没有根据 key 查找 value 的方法呢? 别急, 因为实现这个功能依赖一个新的数据结构 `Iterator`.

它包含两个数据成员:
- `list_`, 这是一个 `SkipList*`, 指向自己要迭代的 SkipList 实例.
- `node_`, 指向目前迭代到的位置, 即 SkipList 上的某个节点.

要构造一个迭代器, 非常简单, 只需传入一个要迭代的 SkipList 实例, 剩下的交给迭代器:
```c++
template<typename Key, class Comparator>
inline SkipList<Key,Comparator>::Iterator::Iterator(const SkipList* list) {
  list_ = list;
  node_ = nullptr;
}
```

下面就是我们之前提到的根据某个 key 查找对应 value 的方法了:
```c++
// 定位 key >= target 的第一个 node
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Seek(const Key& target) {
  // 我们只是要查找, 后续不做插入, 所以第二个用于存储 target 前驱节点的数组为 nullptr
  node_ = list_->FindGreaterOrEqual(target, nullptr); 
}
```
实现也非常简单. 查找到的节点保存到迭代器的 `node_` 成员中.

说到迭代器不可避免要谈到向后/向前移动:

```c++
// 将迭代器移动到下个位置. 
// 要求：当前迭代器有效.
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Next() {
  assert(Valid());
  // 因为 level 0 存的是全部 nodes, 所以迭代整
  // 个 skiplist 时只访问 level 0 即可. 
  node_ = node_->Next(0); 
}

// 将迭代器倒退一个位置. 
// 要求：当前迭代器有效.
template<typename Key, class Comparator>
inline void SkipList<Key,Comparator>::Iterator::Prev() {
  // 注意, Node 结构是没有 pre 指针的, 但因为 SkipList nodes 本来
  // 就是按序从左到右排列, 所以直接采用二分查找来定位最后一个 key 小于
  // 迭代器当前指向的 node 的 key 节点即可. 
  assert(Valid());
  node_ = list_->FindLessThan(node_->key);
  if (node_ == list_->head_) {
    node_ = nullptr;
  }
}
```

# 7 番外: C++ 的内存排序

本节内容主要参考 [cppreference](https://en.cppreference.com/w/cpp/atomic/memory_order). Leveldb 内部多处用到同步设施, 这一块内容比较多, 后续会单独开辟一片文章介绍 leveldb 的原子类型实现.

`std::memory_order` 指定了如何围绕原子操作对内存访问（包括常规的、非原子的内存访问）进行排序。当无任何顺序限制的时候, 在多核系统中, 当多个线程同时读写多个变量时, 一个线程观测到的某个值的变化顺序可能与负责写的线程的写顺序不同. 实际上, 甚至多个读线程之间各自观测到的变化顺序也互不相同.  此类情况哪怕在单核系统上也会发生, 因为只要内存模型允许, 编译器会在编译程序时进行指令重排序. 

库中的全部原子操作的默认行为提供了顺序一致的排序(具体见后文). 这可能会造成性能损失, 但是可以给库中的原子操作提供额外的 `std::memory_order` 参数来指定具体的顺序限制, 除了保障原子性, 编译器和处理器必须执行这类限制确保顺序性.

- memory_order_relaxed
  - Relaxed 操作: 仅保证该操作的原子性, 不会针对其它读写操作施加同步或者顺序限制(此即 Relaxed ordering)
- memory_order_consume 
  - 带有此限制的 load 操作针对受影响内存执行一个 consume 行为: 执行 load 操作的当前线程中后续针对该变量的读写操作不会被重排序到 load 操作之**前**, 从而确保后续读写用的都是最新加载的数据. 其它线程中, 如果先发生过针对同一个原子变量的写操作, 那么在它们 release 该原子变量之前针对该原子变量依赖的其它变量的写操作, 在当前线程 load 该原子变量时也都是可见的. 在大多数平台上, 这仅仅会影响编译器优化(此即 Release-Consume ordering).
- memory_order_acquire 
  - 带有此限制的 load 操作针对受影响内存执行一个 acquire 行为: 执行 load 操作的当前线程中后续依赖该变量的读写操作不会被重排序到该 load 操作之**前**, 从而确保后续读写用的都是最新加载的数据. 其它线程中, 如果先发生过针对同一个原子变量的写操作, 那么在它们 release 该原子变量之前针对其它变量的写操作(**无论该原子变量是否依赖这些变量, 这比 consume 语义更强烈**), 在当前线程 load 该原子变量时都是可见的(此即 Release-Acquire ordering).
- memory_order_release 
  - 带有此限制的 store 操作执行一个 release 行为: 执行 store 操作的当前线程中其它针对该变量的读写操作都不会被重排序到当前 store 操作之**后**, 从而确保之前的读写用的都是本次修改之前的值. 在此 store 之前的针对其它变量的全部写操作对 acquire 该原子变量的其它线程也都是可见的(此即 Release-Acquire ordering), 同时针对该原子变量所依赖的变量的写操作针对 consume 该原子变量的其它线程也都是可见的(此即 Release-Consume ordering).
- memory_order_acq_rel 
  - 带有此限制的 read-modify-write 操作既是一个 acquire 操作也是一个 release 操作. 执行 read-modify-write 操作的当前线程针对同一个变量的其它读写操作不能被重排序到该 store 操作之前或之后. 其它线程中针对该原子变量的写操作在 release 之后针对当前线程的 modify 是可见的, 同理当前线程的 modify 在 release 之后对其它 acquire 该原子变量的其它线程也是可见的.
- memory_order_seq_cst
  - 带有此限制的 load 操作执行一个 acquire 行为, 带有此限制的 store 操作执行一个 release 行为, 带有此限制的 read-modify-write 操作既执行一个 acquire 行为也执行一个 release 行为, 此外, 该限制确保了一个单一全序, 也就是说, 全部线程观测道德全部修改行为都是一个顺序(此即 Sequentially-consistent ordering). 该限制语义最强.

--End--  


