# Leveldb 源码详解系列之五: SSTable 设计与实现


leveldb, leveldb, 每个 level 保存的内容就是一组 sorted string table (简称 sstable) 文件.
<!--more-->
# 1 sstable 文件布局

SSTable 即 sorted string table, 是一个有序文件格式.

该文件主要包含五个部分:
- 一系列 data blocks, 这里保存的我们需要的数据.
- 一系列 meta blocks, 这里目前保存的只有布隆过滤器. 通过它, 在不解析 data blocks 的前提下就能知道某个 key 是否存在, 如果可能存在也能快速缩小到可能在哪个 data block.
- 一个 metaindex block, 包含指向 meta blocks 的索引.
- 一个 index block, 包含指向 data blocks 的索引. 
- 一个 footer, sstable 文件入口, 保存着指向 metaindex block 和 index block 的索引, 相当于一个二级指针.

不像 kafka 文件存储结构的数据文件和索引文件是各自独立的(在查询时根据具体 key 先在索引文件确定是哪个数据文件), sstable 把索引和数据保存到了同一个文件中. 每次从文件查询数据时会先查询索引, 索引是指向数据的指针, 具体叫做 BlockHandle, 包含着下述信息: 

    // 目标 block 起始位置在文件中的偏移量
    offset: varint64
    // 目标 block 的大小
    size:   varint64

形象地说, sstable 文件具体布局如下: 

    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1]
    ...
    [meta block K]
    [metaindex block]
    [index block]
    [Footer]        (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>

下面具体讲一下每个段的具体布局.

## 1.1 data block 布局

每个 block 包含的数据笼统地讲, 包含 `<一系列数据项 + restart array + restart number>` ". 

为了节省存储空间, block 中的数据项的 key 使用了前缀压缩. 具体来说, 存储某个 key 的时候先计算它和前一个数据项 key 的公共前缀长度, 公共前缀不再重复存储而是仅记录一个长度(shared), 由于 block 保存的数据是按 key 有序的, 排在一起的前缀都是比较相近的, 而且相似前缀可能还比较长所以该策略可以大幅节省存储空间.

block 中有一个至关重要的概念, 叫 **restart point**. 这个概念和前面提到的前缀压缩密切相关, 每个 block 的前缀压缩不是从第一个数据项开始就一直下去, 而是每隔一段(间隔可配置)设置一个新的前缀压缩起点(作为新起点的数据项的 key 保存原值而非做前缀压缩), restart point 指的就是新起点, 从这个地方开始继续做前缀压缩.

block 中每个数据项的格式如下: 
- shared_bytes: varint32(与前一个 key 公共前缀的长度). 注意, 如果该数据项位于 restart 处, 则 shared_bytes 等于 0.
- unshared_bytes: varint32(当前 key 除去公共前缀后的长度)
- value_length: varint32(当前 key 对应的 value 的长度)
- key_delta: char[unshared_bytes](当前 key 除去共享前缀后的字节内容)
- value: char[value_length](当前 key 对应的 value 的数据内容)

block 结尾处有个 trailer, 格式如下: 
- restarts: uint32[num_restarts](保存 restart points 在 block 内偏移量的数组)
- num_restarts: uint32(restart points 偏移量数组大小)
restarts[i] 保存的是第 i 个 restart point 在 block 内的偏移量.

## 1.2 meta block 布局

它由 `<一系列 filters + filter-offset 数组 + filters 部分的结束偏移量(4 字节) + base log 值(1 字节)>` 构成. 注意该 block 最后 5 字节内容是固定的, 这也是该部分的解析入口.

该部分在写入文件时不进行压缩.

## 1.3 meta-index block 布局

只有一个数据项, key 为 `"filter."+过滤器名`, value 为 meta block 的 handle.

## 1.4 index block 布局

同 data block, 每个数据项的 key 是某个 data block 的最后一个 key, 每个数据项的 value 是这个 data block 的 handle. 注意, 由于该 block 数据项数和 data blocks 个数一样, 相对来说非常少, 所以就没做前缀压缩(具体实现就是将 restart point interval 设置为 1).

## 1.5 footer 布局

Footer 虽然位于 sstable 文件尾部, 但它是名副其实的文件入口, 它的**长度固定**, 很容易从文件尾定位到, 它包含:
- 一个指向 metaindex block 的 BlockHandle 
- 一个指向 index block 的 BlockHandle 
- 一个 magic number. 

Footer 具体格式如下:

    // 指向 metaindex block 的 BlockHandle
    metaindex_handle: char[p];     
    // 指向 index block 的 BlockHandle
    index_handle:     char[q];     
    // 用于维持固定长度的 padding 0,
    // (其中 40 == 2*BlockHandle::kMaxEncodedLength)
    padding:          char[40-p-q];
    // 具体内容为 0xdb4775248b80fb57 (小端字节序)
    magic:            fixed64;     

注意 footer 存的都是 *index-of-xx*, 找到 *index* 就可以找到 *xx* 了.

关于 sstable 的其它细节请见 [Leveldb 源码详解系列之一: 接口与文件]({{<relref "leveldb-annotations-1-interfaces-and-files.md">}}). 

了解了布局, 下面让我们来看看针对 sstable 的读写实现.

# 2 sstable 文件的序列化与反序列化

sstable 文件集合保存着 leveldb 实例的数据, 定义在 `db/version_set.h` 中的 `class leveldb::Version` 跟踪每个 level 及其文件, 可以将这个类看做是对 leveldb 全部层级文件架构的抽象. 

下面说明一下针对 sstable 文件的序列化和反序列化.


## 2.1 sstable 文件序列化

完成该工作的是 `class leveldb::TableBuilder`, 该类负责构造 sstable 文件. 

具体构造和写入顺序为:
- 写 data blocks
- 写 meta blocks(目前仅有过滤器)
- 写 meta-index block
- 写 data-index block
- 写 footer

每个分段也都有类似 XXBuilder 的类, 具体构造时会被 TableBuilder 调用. 除此之外, 还有一个类似的地方, 就是每个 XXBuilder 主要干活的基本都叫做 `Add()` 和 `Finish()` , 前者负责将具体数据添加到自己分段中, 后者负责将本段的元数据追加到自己分段尾部从而完成分段构造. 具体执行过程中, 各个 XXBuilder 有交叉的地方. 典型地, BlockBuilder 构造 data block 时会将自己的 BlockHandle 保存到 index block, 同时会将自己的 key 添加到 filter block 的相关状态里. 具体下面详述.

### 2.1.1 总干事 TableBuilder

该类是构造 sstable 的入口, 外部(如 `leveldb::BuildTable()` 方法在被 `leveldb::DBImpl::WriteLevel0Table()` 方法调用将 memtable 转为 sstable 的时候)直接循环调用该类的 `Add()` 方法来向 sstable 追加 k,v 数据, 追加完毕后调用该类 `Finish()` 方法做收尾工作.

下面列出 TableBuilder 比较核心的成员:
```c++
// 该类用于构造 sstable(sorted string table) 文件. 
//
// 如果用户从多个线程调用该类的 const 方法, 线程安全; 
// 如果从多个线程调用非 const 方法, 则需要依赖外部同步设施确保线程安全. 
class LEVELDB_EXPORT TableBuilder {
 public:
  // 将一对 <key,value> 追加到正在构造的 table 中. 
  // 要求 1: key 必须大于任何之前已经添加过的 keys, 
  //        因为该文件是有序的.
  // 要求 2: 还没调用过 Finish() 或者 Abandon(), 
  //        调用了这两个方法表示 table 对应文件被关掉了.
  void Add(const Slice& key, const Slice& value);

  // 该方法由 Add() 和 Finish() 调用, 将缓冲的 data block 写入文件.
  // 要求: 还没调用过 Finish() 或者 Abandon(). 
  void Flush();

  // 完成 table 构建. 该方法返回后停止使用在构造方法中传入的文件. 
  // 要求: 还没调用过 Finish() 或者 Abandon(). 
  // table 构成: data blocks, filter block, metaindex block, index block
  Status Finish();

 private:
  // 将 data block 内容根据设置进行压缩, 然后写入文件;
  // 同时将 data block 在 table 偏移量和 size 设置到
  // handle 中, 写完 block 会将其 handle 写入
  // index block.
  void WriteBlock(BlockBuilder* block, BlockHandle* handle);
  // 将 block 及其 trailer(注意这个 trailer 不是 block 内部的 trailer)
  // 写入 table 对应的文件, 
  // 并将 block 对应的 BlockHandle 内容保存到 handle 中. 
  // 写失败时该方法只将错误状态记录到 r->status, 不做其它任何处理.
  // 该方法由 WriteBlock 调用.
  void WriteRawBlock(const Slice& data, CompressionType, BlockHandle* handle);
  // 这个结构体很重要, 是 Table 实际存储数据的结构体, 下面单独开辟一节讲述.
  struct Rep;
  // 存储构造过程中的 table
  Rep* rep_;
};
```

主要方法有以下两个:
- `void BlockBuilder::Add(const Slice& key, const Slice& value)` 负责向 TableBuilder 对象添加 (key, value), 该工作主要由 `class leveldb::BlockBuilder::Add()` 方法完成.
- `void leveldb::TableBuilder::Finish()` 负责将整个 Table 序列化为一个 sstable 文件并写入磁盘.

`Add()` 方法在构造 data block 和 index block 时用到了 `BlockBuilder` 对应方法, `Add()` 实现如下:
```c++
// 将一对 <key,value> 追加到正在构造的 table 中.
// 该方法追加数据时会同时影响到 data block, data index block, 
// meta block 的构造.
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  // 确保之前没有调用过 Finish() 或者 Abandon()
  assert(!r->closed); 
  if (!ok()) return;
  // 如果该条件成立则说明之前调用过 Add 添加过数据了
  if (r->num_entries > 0) { 
    // 确保待添加的 key 大于之前已添加过的全部 keys
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0); 
  }

  // 需要构造一个新的 data block
  if (r->pending_index_entry) {
    // 与上面紧邻的这个判断条件构成不变式, 为空表示
    // 已经将写满的 data block flush 到文件了.
    assert(r->data_block.empty());
    // 为 pending index entry 选一个合适的 key.
    // 下面这个函数调用结束, last_key 可能不变, 也可能长度更短(省空间)但是值更大, 
    // 但不会 >= 要追加的 key. 因为进入该方法之前关于两个参数
    // 已经有了一个约束: 第一个字符串肯定小于第二个字符串, 这个上面有断言保证了.
    //
    // 为何这么做? 因为在查询数据时, 是先在 data-index block 中定位包含该数据的
    // 目标 data block, 然后再转入目标 data block 中进行查找. 第一个定位靠的
    // 就是这里的 last_key, 它和 data block 对应的 handle 一起构成了 data block 
    // 在 data-index block 中的数据项. 第一个定位主要过程就是在 data-index block 
    // 上查找第一个大于等于要查询数据的数据项, 具体见 TwoLevelIterator::Seek().  
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    // 用于存储序列化后的 BlockHandle
    std::string handle_encoding;
    // 将刚刚 flush 过的 data block 对应的 BlockHandle 序列化
    r->pending_handle.EncodeTo(&handle_encoding);
    // data index block 构造相关:
    // 为刚刚 flush 过的 data block 在 index block 增加一个数据项, 
    // last_key 肯定大于等于其全部所有的 keys 且小于新的 
    // data block 的第一个 key.
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    // 增加过 index entry 后, 可以将其置为 false 了.
    r->pending_index_entry = false;
  }
  
  // meta block 构造相关:
  // 如果该 table 存在 filter block, 则将该 key 加入.
  // (filter block 可以用于快速定位 key 是否存在于 table 中).
  // 加入的 key 在 FilterBlockBuilder 中使用.
  if (r->filter_block != nullptr) {
    r->filter_block->AddKey(key);
  }

  // 用新 key 更新 last_key
  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  // data block 相关:
  // 将 key,value 添加到 data block 中
  r->data_block.Add(key, value); 

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  // 如果当前 data block 大小的估计值大于设定的阈值, 
  // 则将该 data block 写入文件
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

`Add()` 在检测到 data block 大小达到阈值时会调用 `Flush()` 将数据刷入文件. 刷入完成, 会调用 FilterBlockBuilder 为其生成 filter. 具体如下:

```c++
// Add() 依赖 Flush() 将大小满足要求的 block 写入文件中.
void TableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;
  assert(!r->pending_index_entry);
  // 将 data block 压缩并落盘, 在该方法中 data_block 会调用 Reset()
  WriteBlock(&r->data_block, &r->pending_handle); 
  if (ok()) {
    // flush 一个 data block, 就要在 index block 为其增加一个指针.
    r->pending_index_entry = true;
    r->status = r->file->Flush();
  }
  if (r->filter_block != nullptr) {
    // 为已经刷入文件的 data block 计算 filter, 同时
    // 为新的 data block 计算 filter 做准备, 
    // r->offset 为下个 block 在 table 的起始地址, 
    // 该值在上面 WriteBlock 调用 WriteRawBlock 时会进行更新.
    r->filter_block->StartBlock(r->offset); 
  }
}
```

#### 2.1.1.1 TableBuilder 的存储小助手 Rep

从 TableBuilder 定义可以看到, private 部分有一个 Rep, 这是 TableBuilder 用于存放构造中的 sstable 的地方. 因为它太重要, 所以单独拿出来说一下:

```c++
struct TableBuilder::Rep {
  Options options;
  Options index_block_options;
  // table 对应的文件指针
  WritableFile* file; 
  // table 文件当前的最大偏移量
  uint64_t offset; 
  Status status;
  // 用于构造 data block
  BlockBuilder data_block; 
  // 用于构造 data index block
  BlockBuilder index_block; 
  // 最近一次成功调用 Add 添加的 key
  std::string last_key; 
  // 当前 table 中全部 data block entries 个数
  int64_t num_entries; 
  // 指示是否调用过 Finish() 或 Abandon()
  bool closed;          
  // 用于构造 filter block
  FilterBlockBuilder* filter_block; 

  // 直到当追加下一个 data block 第一个 key 的时候, 我们才会将
  // 当前 data block 对应的 index 数据项追加到 index block,  
  // 这样可以让我们在 index block 使用更短的 key. 
  // 举个例子, 假设当前 data block 最后一个
  // key "the quick brown fox", 下一个 data block 
  // 首个 key "the who" 之间, 
  // 则我们可以为当前 data block 的 index 数据项设置
  // key "the r", 因为它 >= 当前 data block 全部 key, 
  // 而且 < 接下来 data block 的全部 key. 
  //
  // 不变式: 仅当 data_block 为空的时候(已经被 flush 过了)
  // r->pending_index_entry 才为 true. 
  // 注意该变量初始值为 false, 也就是写满(大小可配置)并 flush 过一个
  // data block 时候才会写入它对应的 index entry, 
  // 因为此时 index entry 才知道 data block 最大 key 是什么.
  bool pending_index_entry;
  // 与 pending_index_entry 配套, 
  // 指向当前 data block 的 BlockHandle,
  // 会被追加到 index block 中.
  BlockHandle pending_handle;

  std::string compressed_output;

  Rep(const Options& opt, WritableFile* f)
      : options(opt),
        index_block_options(opt),
        file(f),
        offset(0),
        data_block(&options),
        index_block(&index_block_options),
        num_entries(0),
        closed(false),
        filter_block(opt.filter_policy == nullptr ? nullptr
                     : new FilterBlockBuilder(opt.filter_policy)),
        pending_index_entry(false) {
    // index block 的 key 不需要做前缀压缩, 
    // 所以把该值设置为 1, 表示每个 restart 段长度为 1.
    index_block_options.block_restart_interval = 1; 
  }
};
```
### 2.1.2 写 data blocks

BlockBuilder 被 TableBuilder 使用来构造 sstable 文件里的 block, 注意, 该类用于构造 block 的序列化形式, 也就是构造 sstable 时候使用; 反序列化用的是 Block 类.

要理解这里必须要清楚 block 布局, 这个在文章开头有详细描述. 建议回过头看一眼, 尤其 restart point 的设计. 

同 TableBuilder 类似, BlockBuilder 最重要的两个方法也是 `Add()` 和 `Finish()`. TableBuilder 会重复使用 BlockBuilder 实例, 每写入一个 block 就会调用其 `Reset()` 将其恢复原状.

BlockBuilder 核心成员如下:

```c++
class BlockBuilder {
 public:
  explicit BlockBuilder(const Options* options);

  // 重置 BlockBuilder 对象状态信息, 就像该对象刚刚被创建时一样. 
  void Reset();

  // 与 Finish() 分工, 负责追加一个数据项到 buffer. 
  // 前提: 
  // 1. 自从上次调用 Reset() 还未调用过 Finish(); 
  // 2. 参数 key 要大于任何之前已经添加过的数据项的 key,
  // 因为这是一个 append 类型操作. 
  void Add(const Slice& key, const Slice& value);

  // 该方法负责 block 构建的收尾工作, 具体是将 restart points
  // 数组及其长度追加到 block 的数据内容之后完成构建工作, 最后返回
  // 一个指向 block 全部内容的 slice. 
  // 返回的 slice 生命期同当前 builder, 若 builder 调用了
  // Reset() 方法则返回的 slice 失效. 
  Slice Finish();

 private:
  // 存储目标 block 内容的缓冲区
  std::string           buffer_;
  // 存储目标 block 的全部 restart points
  // (即每个 restart point 在 block 中的偏移量, 
  // 第一个 restart point 偏移量为 0)
  std::vector<uint32_t> restarts_;
  // 该 BlockBuilder 上次调用 Add 时追加的那个 key
  std::string           last_key_;
};
```

最重要的 `Add()` 和 `Finish()` 实现如下:

```c++
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  // 上次调用 Add 追加的 key, 用于计算公共前缀
  Slice last_key_piece(last_key_);
  // 不能往一个完成构建的 block 里追加数据.
  assert(!finished_);
  // 自上个 restart 之后追加的 key 的个数没有超过要求的两个
  // restart points 之间 keys 的个数.
  assert(counter_ <= options_->block_restart_interval);
  // 当前要追加的 key 要大于任何之前追加到 buffer 中的 key
  assert(buffer_.empty() // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);
  size_t shared = 0;
  // 如果自上个 restart 之后追加的 key 的个数小于所配置的
  // 两个相邻 restart 之间 keys 的个数. 
  if (counter_ < options_->block_restart_interval) {
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    // 计算当前要追加的 key 与上次追加的 key 的公共前缀长度. 
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
    // 否则, 新增一个 restart point, 而且作为 restart 的 key 不进行压缩. 
    // - restart 就是一个 offset, 具体值为当前 buffer 所占空间大小. 
    // - restart 后第一个数据项的 key 不进行压缩, 即不计算与前一个 key 
    //   的公共前缀了, 而是把这个 key 整个保存起来, 但是本 "restart"  
    //   段, 从这个 key 开始后面的 keys 都要进行压缩.
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }
  const size_t non_shared = key.size() - shared;

  // Add "<shared><non_shared><value_size>" to buffer_
  //
  // buffer 里面的每个记录的格式为: 
  // <varint32 类型的当前 key 与上个 key 公共前缀长度>
  // <varint32 类型的当前 key 长度减去公共前缀后的长度>
  // <varint32 类型的当前 value 的长度>
  // <与前一个 key 公共前缀之后的部分>
  // <value>
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

  // 将 key 非公共部分和 value 追加到 buffer_
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());

  // 更新状态
  last_key_.resize(shared);
  // 将 last_key 更新为当前 key
  last_key_.append(key.data() + shared, non_shared); 
  assert(Slice(last_key_) == key);
  // 将自上个 restart 之后的记录数加一
  counter_++; 
}

Slice BlockBuilder::Finish() {
  /**
   * 先将 restarts 数组编码后追加到 buffer, 
   * 然后将 restarts 数组长度编码后追加到 buffer 并将 finished 置位, 
   * 最后根据 buffer 构造一个新的 slice 返回(注意该 slice 引用的内存是 
   * buffer, 所以生命期同 builder, 除非 builder 调用了 Reset)
   */
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }
  PutFixed32(&buffer_, restarts_.size());
  finished_ = true;
  return Slice(buffer_);
}
```

### 2.1.3 写 meta(filter) block

不同于 data block 和 data index block, filter block 有专用的 builder, 叫做 `FilterBlockBuilder`. 它的核心方法是 `AddKey()` 和 `Finish()`.

```c++
// 该类在其它地方定义. 用于定义过滤器逻辑.
class FilterPolicy;

// FilterBlockBuilder 用于构造 table 的全部 filters. 
// 最后生成一个字符串保存在 Table 的一个 meta block 中. 
//
// 该类的方法调用序列必须满足下面的正则表达式: 
//      (StartBlock AddKey*)* Finish
// 最少调用一次 Finish, 而且 AddKey 和 Finish 之间不能插入 StartBlock 调用. 
class FilterBlockBuilder {
 public:
  explicit FilterBlockBuilder(const FilterPolicy*);

  void StartBlock(uint64_t block_offset);
  void AddKey(const Slice& key);
  Slice Finish();

 private:
  void GenerateFilter();

  const FilterPolicy* policy_;
  // 调用 AddKey() 时每个 key 都会被
  // 追加到这个字符串中(用于后续构造 filter 使用)
  std::string keys_;
  // 与 keys_ 配套, 每个被 AddKey() 方法追加的 key 在 
  // keys_ 中的起始索引.
  std::vector<size_t> start_;
  // 每个新计算出来的 filter 都是一个字符串, 
  // 都会被追加到 result_ 中.
  // filter block 保存的内容就是 result_.
  std::string result_;
  // 是 keys_ 的列表形式, 临时变量, 每个成员是 Slice 类型,
  // 用于 policy_->CreateFilter() 生成构造器.
  std::vector<Slice> tmp_keys_;   
  // 与 result_ 配套, 保存每个 filter 在 result_ 
  // 中的起始偏移量.
  std::vector<uint32_t> filter_offsets_; 

  // No copying allowed
  FilterBlockBuilder(const FilterBlockBuilder&);
  void operator=(const FilterBlockBuilder&);
};
```

`TableBuilder::Add()` 会在追加 k,v 的时候调用 FilterBlockBuilder 的
`AddKey()` 将 k 追加到 FilterBlockBuilder 中, 其实现逻辑比较简单:
```c++
// 向 keys_ 中增加一个 key, 同时将 key 在 keys_ 中
// 起始偏移量保存到 start_ 向量中.
void FilterBlockBuilder::AddKey(const Slice& key) {
  Slice k = key;
  start_.push_back(keys_.size());
  keys_.append(k.data(), k.size());
}
```

当 TableBuilder flush 一个 data block 到文件后, 就要为其生成 filter, 该过程通过调用 FilterBlockBuilder 的 `StartBlock()` 达成:

```c++
// 为前一个已写入 table 文件的 data block 生成
// filter, 生成完毕后重置当前 FilterBlockBuilder 的状态为生成下一个
// filter 做准备.
void FilterBlockBuilder::StartBlock(uint64_t block_offset) {
  // 计算以 block_offset 为起始地址的 block 对应的 filter 在
  // filter-offset 数组中的索引.
  // 默认每两 KB 数据就要生成一个 filter, 
  // 如果 block size 超过 2KB, 则会生成多个 filters.
  uint64_t filter_index = (block_offset / kFilterBase);
  assert(filter_index >= filter_offsets_.size());
  // filter 是一个接一个构造的, 对应的索引数组也是对应着逐渐增长的, 
  // 而非一次性构造好往里面填, 毕竟不知道要生成多少个 filters
  while (filter_index > filter_offsets_.size()) {
    // 这里虽然是个循环, 但是因为每次生成 filter 
    // 都会清空相关状态(keys_, start_ 等等), 
    // 所以下个循环并不会再生成 filter 了, 具体见
    // GenerateFilter() 的 if 部分.
    GenerateFilter();
  }
}
```

`StartBlock()` 方法有个循环调用 `GenerateFilter()` 方法的地方, 比较绕, 这部分逻辑要结合 `GenerateFilter()` if 部分和 `FilterBlockReader::KeyMayMatch()` 的 limit 计算部分一起看:
```c++
// 由于即将生成的 block 不能与当前已写入 table 文件的 block 
// 的 keys 共用 filter 了, 所以为当前已写入 table 文件的 block 的 
// keys_ 生成一个 filter. 生成完毕后清空当前 FilterBlockBuilder 
// 相关相关状态以为下个 filter 计算所用.
void FilterBlockBuilder::GenerateFilter() {
  // keys_ 为空, 无须生成新的 filter.
  const size_t num_keys = start_.size();
  if (num_keys == 0) {
    // 没有 key 需要计算 filter, 则直接把上一个 filter 
    // 的结束地址(每个 filter 都是一个字符串, 所以保存到 result_ 
    // 时候既有起始地址又有结束地址)填充到 filter-offset 数组中,
    // 这么做一方面为了对齐(方便 FilterBlockReader::KeyMayMatch() 
    // 直接通过移位计算 filter 索引), 另一方面方便计算 filter 结束偏移量(
    // 就是 FilterBlockReader::KeyMayMatch() 计算 limit 的步骤).
    filter_offsets_.push_back(result_.size());
    return;
  }

  // 将扁平化的 keys_ 转换为一个 key 列表.
  // 将 keys_ 大小放到 start_ 中作为最后一个 key 的结束地址, 
  // 这样下面可以直接用 start_[i+1] - start_[i] 计算
  // 每个 key 长度.
  start_.push_back(keys_.size());  
  tmp_keys_.resize(num_keys);
  // 将字符串 keys_ 保存的每个 key 提取出来封装
  // 成 Slice 并放到 tmp_keys 列表中
  for (size_t i = 0; i < num_keys; i++) {
    // 第 i 个 key 在 keys 中的起始地址
    const char* base = keys_.data() + start_[i]; 
    // 第 i 个 key 的长度
    size_t length = start_[i+1] - start_[i];
    // 将第 i 个 key 封装成 Slice 并保存到 tmp_keys 
    // 用于后续计算 filter 
    tmp_keys_[i] = Slice(base, length); 
  }

  // 为当前的 key 集合生成 filter.
  // 先将新生成的 filter 在 result_ 中的
  // 起始偏移量保存到 filter_offsets_.
  filter_offsets_.push_back(result_.size());
  // 根据 tmp_keys_ 计算 filter 并追加到 result
  policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);

  // 重置当前 FilterBlockBuilder 相关的状态以方便为
  // 下个 data block 计算 filter 使用.
  tmp_keys_.clear();
  keys_.clear();
  start_.clear();
}
```

与 FilterBlockBuilder 相反, 将一个 filter block 解析出来, 然后用来查询某个 key 是否在某个 block 中. 其成员 `data_` 指向 filter block 起始地址, `offset_` 指向 filter block 尾部 offset array 的起始地址, 这也是 filter block 的末尾:
```c++
// 通过过滤器查询 key 是否在以 block_offset 为起始地址的 block 中
bool FilterBlockReader::KeyMayMatch(uint64_t block_offset, const Slice& key) {
  // 计算 filter 的时候是每隔 base 大小为一个区间, 
  // 每个区间对应一个 filter. 
  // 起始偏移量落在对应区间内的 blocks, 
  // 会将其全部 keys 作为整体输入计算一个 filter. 
  
  // 注意, 这里的算法正好解答了 
  // leveldb::FilterBlockBuilder::GenerateFilter() 的疑问, 
  // 针对多出来那个重复的 filter offset, 会被自动跳过.
  // 下面这一步相当于除以 base 取商, 从而得到 block_offset 
  // 对应的 filter-offset array 数组索引. 
  uint64_t index = block_offset >> base_lg_;
  if (index < num_) {
    // 计算计算索引为 index 的 filter 的 offset, 
    // 具体为先定位到保存目标 filter offset 的地址(每个地址长度为 4 字节), 
    // 然后将其解码为一个无符号 32 位数.
    uint32_t start = DecodeFixed32(offset_ + index*4);
    // 下面这个计算分两种情况理解:
    // - 当 index < num_ - 1 时, 又要分为两个子情况
    //   - 如果 GenerateFilter() 方法在生成索引为 index 的 filter 时,
    //     至少多填充了一次 filter 结束偏移量到 filter-offset 数组, 那么
    //     该操作等价于取出索引为 index 的 filter 的结束偏移量
    //    (注意 GenerateFilter() 方法填充 filter 结束地址到
    //   filter-offset 的情况)
    //   - 如果 GenerateFilter() 方法在生成索引为 index 的 filter 时, 
    //     仅为该 filter 在 filter-offset 数组填入了起始偏移量, 那么该操作
    //     相当于计算索引为 index+1 的 filter 的起始 offset.
    // - 当 index == num_ - 1 时, 也要再分为两个子情况
    //   - 同 index < num_ - 1 时的第一个子情况.
    //   - 如果 GenerateFilter() 方法在生成索引为 index 的 filter 时, 
    //     仅为该 filter 在 filter-offset 数组填入了起始偏移量, 那么该操作
    //     等价于直接取出索引为 index 的 filter 的结束偏移量. 
    //     为什么看起来和第一个子情况一样? 其实不然. 因为 filter block 构成分四块, 
    //     分别是"一系列filters+filters-offset数组+filters和filters-offset
    //     数组的分隔符+base的log值", 这里提到的分隔符用的就是
    //     最后一个 filter 的结束地址). 
    // 不管是哪种情况, limit - start 都是索引为 index 的 filter 的长度.
    uint32_t limit = DecodeFixed32(offset_ + index*4 + 4);
    // 注意 start 和 limit 都是相对于 data_ 的 offset 而非绝对地址.
    if (start <= limit && limit <= static_cast<size_t>(offset_ - data_)) {
      // 取出 filter
      Slice filter = Slice(data_ + start, limit - start);
      // 在 filter 中查询 key 是否存在
      return policy_->KeyMayMatch(key, filter);
    } else if (start == limit) {
      // 如果 start == limit 说明索引为 index 的 filter 为空
      return false;
    }
  }
```

TableBuilder 的 Finish() 会调用 FilterBlockBuilder 的 Finish() 完成 filter block 写入:
```c++
// 计算最后一个 filter 对应的偏移量, 然后将 
// filter-offset array 并追加到 result_.
// 执行结束, result_ 保存的就是一个完整的 filter block
Slice FilterBlockBuilder::Finish() {
  // 若 keys_ 不为空需要为其生成一个 filter
  if (!start_.empty()) { 
    GenerateFilter();
  }

  // 下面马上会把每个 filter 在 result_ 中对应的
  // 起始地址也编码追加到 result_ 中, 这样前面是 filters, 
  // 后面是 filters 的起始偏移量, 那这两部分反序列化时候怎么区分呢?
  // 我们提前记录追加 filters 总字节数就可以了.
  // array_offset 保存全部 filters 的总字节数, 
  // 这个值在追加完全部 filters 
  // 的起始地址后也会被追加到 result_ 作为最后一个元素, 
  // 也是反序列化解析入口.
  const uint32_t array_offset = result_.size();
  // 将每个 filter 在 result_ 中的起始偏移量编码追加到 result_
  for (size_t i = 0; i < filter_offsets_.size(); i++) {
    PutFixed32(&result_, filter_offsets_[i]);
  }

  // 将 filter-offset array 的起始偏移量编码追加到 result_, 
  // 用于反序列化时区分 filters 与 filters 偏移量数组.
  PutFixed32(&result_, array_offset);
  // 将 base 的 log 值追加到 result, 用于反序列化时计算 base
  result_.push_back(kFilterBaseLg);
  // 到此为止, result_ 已经是一个完整的 filter block 了, 
  // 将其封装为 Slice 后返回.
  return Slice(result_);
}
```

最终在 TableBuilder 的 Finish() 将 filter block 刷入文件:
```c++
// 2 如果存在 filter block, 则将其写入文件; 
// 写完后, filter_block_handle 保存着该 block 
// 对应的 BlockHandle 信息, 包括起始偏移量和大小
if (ok() && r->filter_block != nullptr) {
  // 写完 data blocks 该写入 filter block 了.
  // 注意 filter block 写入时不进行压缩.
  WriteRawBlock(r->filter_block->Finish(), kNoCompression, 
  // 将计算出 filter block 对应的 BlockHandle 
  // 信息保存在 filter_block_handle 中.
                &filter_block_handle); 
}
```
### 2.1.4 写 meta-index block

这部分比较简单, 其在 TableBuilder 的 Finish() 方法里完成:
```c++
// 3 filter block 就是 table_format.md 中提到的 
// meta block, 写完 meta block 该写它对应的索引
// metaindex block 到文件中了.
if (ok()) {
  // 虽然 meta block 有独立的 FilterBlockBuilder, 
  // 但是其对应的 index block 用的还是通用的 
  // BlockBuilder.
  BlockBuilder meta_index_block(&r->options);
  if (r->filter_block != nullptr) { 
    // 如果 meta block 存在, 则将其对应的 key 
    // 和 BlockHandle 写入 metaindex block,
    // 具体为 <filter.Name, filter 数据地址>.
    std::string key = "filter.";
    key.append(r->options.filter_policy->Name());
    std::string handle_encoding;
    filter_block_handle.EncodeTo(&handle_encoding);
    meta_index_block.Add(key, handle_encoding);
  }
  // 将 metaindex block 写入文件
  WriteBlock(&meta_index_block, &metaindex_block_handle); 
}
```
### 2.1.5 写 data-index block

这部分比较简单, 其在 TableBuilder 的 Finish() 方法里完成:

```c++
// 4 将 index block 写入 table 文件, 它里面保存的
// 都是 data block 对应的 BlockHandle.
if (ok()) {
  // 最后构建的 data block 对应的 index block entry 还没有写入
  if (r->pending_index_entry) { 
    r->options.comparator->FindShortSuccessor(&r->last_key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    // 写入最后构建的 data block 对应的 index block entry
    r->index_block.Add(r->last_key, Slice(handle_encoding)); 
    r->pending_index_entry = false;
  }
  WriteBlock(&r->index_block, &index_block_handle);
}
```
### 2.1.6 写 footer

footer 是 sstable 文件的入口, 结构比较简单:

```c++
// Footer 封装一个固定长度的信息, 它位于每个 table 文件的末尾. 
//
// 在每个 sstable 文件的末尾是一个固定长度的 footer, 
// 它包含了一个指向 metaindex block 的 BlockHandle
// 和一个指向 index block 的 BlockHandle 以及一个 magic number. 
class Footer {
 public:
  Footer() { }
  void EncodeTo(std::string* dst) const;
  Status DecodeFrom(Slice* input);

  // Footer 长度编码后的长度. 注意, 它就固定这么长. 
  // Footer 包含了一个 metaindex_handle、一个 index_handle、以及一个魔数. 
  enum {
    // Footer 长度, 两个 BlockHandle 最大长度 + 固定的 8 字节魔数
    kEncodedLength = 2*BlockHandle::kMaxEncodedLength + 8
  };

 private:
  BlockHandle metaindex_handle_;
  BlockHandle index_handle_;
};
```

其在 TableBuilder 的 Finish() 方法里完成写入:

```c++
// 5 最后将末尾的 Footer 写入 table 文件
if (ok()) {
  Footer footer;
  footer.set_metaindex_handle(metaindex_block_handle);
  footer.set_index_handle(index_block_handle);
  std::string footer_encoding;
  footer.EncodeTo(&footer_encoding);
  r->status = r->file->Append(footer_encoding);
  if (r->status.ok()) {
    r->offset += footer_encoding.size();
  }
}
```

## 2.2 sstable 文件反序列化

`class leveldb::Table` 可以看做是 sstable 文件的反序列化表示. 它负责对 sstable 进行反序列化并解析其内容, 该类是对 sstable 文件的抽象, 具体底层存储由 Table 的 helper 类 `struct leveldb::Table::Rep` 负责.

该类并不直接被客户代码调用, 用户调用 `DBImpl::Get()` 查询某个 key 的时候, 如果不在 memtable, 则会查询 sstable 文件, 此时会调用 `VersionSet::current_::Get()`, 并进而调用 `leveldb::TableCache::Get()` 查询被缓存的 Table 对象, 如果还没缓存文件对应的 Table 对象, 则会先读取然后将其加入缓存, 这里的读取操作就是 `Table::Open()` 方法提供的反序列化功能. 拿到 Table 对象后, 会调用其 `InternalGet()` 查询数据.

### 2.2.1 总干事 Table 类

`Table` 是 sstable 文件反序列化后的内存形式, 包括 data blocks, data-index block, filter block 等, 核心成员如下:

```c++
// Table 是不可变且持久化的. 
// Table 可以被多个线程在不依赖外部同步设施的情况下安全地访问. 
class LEVELDB_EXPORT Table {
 public:
  // 打开一个保存在 file 中 [0..file_size) 里的
  // 有序 table, 并读取必要的 metadata 数据项
  // 以从该 table 检索数据. 
  //
  // 如果成功, 返回 OK 并将 *table 设置为新打开
  // 的 table. 当不再使用该 table 时候, 客户端负责删除之. 
  // 如果在初始化 table 出错, 将 *table 设置
  // 为 nullptr 并返回 non-OK. 
  // 而且, 在 table 打开期间, 客户端要确保数据源持续有效,
  // 即当 table 在使用过程中, *file 必须保持有效. 
  static Status Open(const Options& options,
                     RandomAccessFile* file,
                     uint64_t file_size,
                     Table** table);

  // 返回一个基于该 table 内容的迭代器. 
  // 该方法返回的结果默认是无效的(在使用该迭代器之前, 
  // 调用者在使用前必须调用其中一个 Seek 方法来
  // 使迭代器生效.)
  Iterator* NewIterator(const ReadOptions&) const;
 private:
  struct Rep;
  Rep* rep_;

  // Seek(key) 找到某个数据项则会自动
  // 调用 (*handle_result)(arg, ...);
  // 如果过滤器明确表示不能做则不会调用.
  friend class TableCache;
  Status InternalGet(
      const ReadOptions&, const Slice& key,
      void* arg,
      void (*handle_result)(void* arg, const Slice& k, const Slice& v));


  void ReadMeta(const Footer& footer);
  void ReadFilter(const Slice& filter_handle_value);
};
```

读取 sstable 的入口为 `Table::Open()` 方法, 读取过程和 sstable 布局密切相关: 读 footer(这是文件入口), 读 data-index block, 再读 meta-index block 和 meta block. 没错, 该方法没有读取 data block. 

该方法最后返回一个 `class leveldb::Table` 对象, 该对象会被调用方用作查询数据使用. 

具体代码如下:

```c++
// 将 file 表示的 sstable 文件反序列化为 Table 对象, 具体保存
// 实际内容的是 Table::rep_.
//
// 如果成功, 返回 OK 并将 *table 设置为新打开的 table. 
// 当不再使用该 table 时候, 需要调用方负责删除之. 
// 如果初始化 table 出错, 将 *table 设置为 nullptr 并返回 non-OK. 
// 注意, 在 table 打开期间, 调用方要确保数据源即 file 持续有效. 
Status Table::Open(const Options& options,
                   RandomAccessFile* file,
                   uint64_t size,
                   Table** table) {
  /**
   * 1 解析 footer: 它是 sstable 的入口.
   */
  *table = nullptr;
  // 每个 table 文件末尾是一个固定长度的 footer
  if (size < Footer::kEncodedLength) { 
    return Status::Corruption("file is too short to be an sstable");
  }

  char footer_space[Footer::kEncodedLength];
  Slice footer_input;
  // 读取 footer, 放到 footer_input
  Status s = file->Read(size - Footer::kEncodedLength, Footer::kEncodedLength,
                        &footer_input, footer_space);
  if (!s.ok()) return s;

  Footer footer;
  // 解析 footer
  s = footer.DecodeFrom(&footer_input);
  if (!s.ok()) return s;

  /**
   * 2 解析 data-index block:
   * 根据已解析的 Footer, 解析出 index block(它保存了指向全部 data blocks 的索引) 
   * 存储到 index_block_contents.
   */
  BlockContents index_block_contents;
  if (s.ok()) {
    ReadOptions opt;
    if (options.paranoid_checks) {
      opt.verify_checksums = true;
    }
    // 读取 index block, 它对应的 BlockHandle 存储在 footer 里面
    s = ReadBlock(file, opt, footer.index_handle(), &index_block_contents);
  }

  if (s.ok()) {
    // 已经成功读取了 Footer 和 index block, 是时候读取 data 了. 
    Block* index_block = new Block(index_block_contents);
    Rep* rep = new Table::Rep;
    rep->options = options;
    rep->file = file;
    // filter-index block 对应的指针 (二级索引), 解析 footer 时候就拿到了.
    rep->metaindex_handle = footer.metaindex_handle();
    // data-index block 
    // (注意它只是一个索引, 即 data blocks 的索引, 
    //  真正使用的时候是基于 data-index block 做二级迭代器来进行查询,
    //  一级索引跨度大, 二级索引粒度小, 可以快速定位数据,
    //  具体见 Table::NewIterator() 方法)
    rep->index_block = index_block;
    // 如果调用方要求缓存这个 table, 则为其分配缓存 id
    rep->cache_id = (options.block_cache ? options.block_cache->NewId() : 0);
    // 接下来跟 filter 相关的两个成员将在下面 ReadMeta 进行填充.
    rep->filter_data = nullptr;
    rep->filter = nullptr;
    *table = new Table(rep);
    /**
     * 3 解析 meta-index block 和 meta block:
     * 根据已解析的 Footer 所包含的 metaindex block 指针, 
     * 解析出 metaindex block, 再基于此解析出 mate block 
     * 存储到 Table::rep_.
     */
    // 读取并解析 filter block 到 table::rep_, 
    // 它一般为布隆过滤器, 可以加速数据查询过程.
    (*table)->ReadMeta(footer);
  }

  // 是的, 该方法没有解析 data blocks.

  return s;
}
```

总结下, 该方法主要干了下面三件事:
1. 先解析 sstable 文件结尾的 Footer, 它是 sstable 的入口.
2. 根据已解析的 Footer, 解析出 (data) index block 存储到 Table::rep_.
3. 根据已解析的 Footer, 解析出 meta block 存储到 Table::rep_.

注意, `Open` 方法并未去解析 data block 部分, 仅仅是解析了它对应的 index block 部分 和 meta block(它包含的是过滤器, 用来快速确认 key 是否存在). 

那什么时候才解析 data block 呢? 答案是调用 `InternalGet()` 时候:
```c++
// 在 table 中查找 k 对应的数据项. 
// 如果 table 具有 filter, 则用 filter 找; 
// 如果没有 filter 则去 data block 里面查找, 
// 并且在找到后通过 saver 保存 key/value. 
Status Table::InternalGet(const ReadOptions& options, const Slice& k,
                          void* arg,
                          void (*saver)(void*, const Slice&, const Slice&)) {
  Status s;
  // 针对 data index block 构造 iterator
  Iterator* iiter = rep_->index_block->NewIterator(rep_->options.comparator);
  // 在 data index block 中寻找第一个大于等于 k 的数据项, 这个数据项
  // 就是目标 data block 的 handle.
  iiter->Seek(k);
  if (iiter->Valid()) {
    // 取出对应的 data block 的 BlockHandle
    Slice handle_value = iiter->value(); 
    FilterBlockReader* filter = rep_->filter;
    BlockHandle handle;
    // 如果有 filter 找起来就快了, 如果确定
    // 不存在就可以直接反悔了.
    if (filter != nullptr &&
        handle.DecodeFrom(&handle_value).ok() &&
        !filter->KeyMayMatch(handle.offset(), k)) {
      // 没在该 data block 对应的过滤器找到这个 key, 肯定不存在
    } else { 
      // 如果没有 filter, 或者在 filter 中查询时无法笃定
      // key 不存在, 就需要在 block 中进行查找.
      // 看到了没? Open() 方法没有解析任何 data block, 解析
      // 是在这里进行的, 因为这里要查询数据了.
      Iterator* block_iter = BlockReader(this, options, iiter->value());
      block_iter->Seek(k);
      if (block_iter->Valid()) {
        // 将找到的 key/value 保存到输出型参数 arg 中, 
        // 因为后面会将迭代器释放掉.
        (*saver)(arg, block_iter->key(), block_iter->value()); 
      }
      s = block_iter->status();
      delete block_iter;
    }
  }
  if (s.ok()) {
    s = iiter->status();
  }
  delete iiter;
  return s;
}
```

`Open()` 方法会读取 block 构造 Block 然后放到 `Table::Rep` 中, 解析过程也用到了 `Block` 类. 在分别介绍各个组成部分读取解析之前先说一下这两个类.

#### 2.2.1.1 Table 类的小助手之一 Rep

该类负责存储 sstable 反序列化后的各个部分内容:

```c++
// 存储 table 中各个元素的结构
struct Table::Rep {
  ~Rep() {
    delete filter;
    delete [] filter_data;
    delete index_block;
  }

  // 控制 Table 的一些选项, 比如是否进行缓存等.
  Options options;
  Status status;
  // table 对应的 sstable 文件
  RandomAccessFile* file; 
  // 如果该 table 具备对应的 block_cache, 
  // 该值与 block 在 table 中的起始偏移量一起构成 key, value 为 block
  uint64_t cache_id; 
  // 解析出来的 filter block
  FilterBlockReader* filter; 
  // filter block 原始数据
  const char* filter_data; 

  // 从 table Footer 取出来的, 指向 table 的 metaindex block
  BlockHandle metaindex_handle;
  // index block 原始数据, 保存的是每个 data block 的 BlockHandle
  Block* index_block;
};
```

#### 2.2.1.2 Table 类的小助手之二 Block

`class leveldb::Block` 定义在 `table/block.h` 和 `table/block.cc` 文件, sstable 中的每个 block 都会被反序列化为一个 `Block` 对象. 具体类定义如下:

```c++
class Block {
 public:
  // 使用特定的 contents 来构造一个 Block
  explicit Block(const BlockContents& contents);

  ~Block();

  size_t size() const { return size_; }
  // 根据用户定制的 comparator 构造该 block 的一个迭代器
  Iterator* NewIterator(const Comparator* comparator);

 private:
  uint32_t NumRestarts() const;

  // block 全部数据(数据项 + restart array + restart number)
  const char* data_;
  // block 总大小 
  size_t size_; 
  // block 的 restart array 在 block 中的起始偏移量
  uint32_t restart_offset_;
  // 如果 data_ 指向的空间是在堆上分配的, 
  // 那么该 block 对象销毁时需要释放该处空间, 该成员使用见析构方法.
  bool owned_;                  

  // 不允许拷贝 block 
  Block(const Block&);
  void operator=(const Block&);

  // 为迭代 block 内容服务的迭代器, block 相当于迭代器的数据源.
  class Iter;
};
```

构造 `Block` 对象只有一种方式, 就是先读取文件内容构造 `BlockContents`, 然后基于 `BlockContents` 构造 `Block`, 这个在 `Table::Open()` 中解析 data-index block 和 meta-index block 时多次用到. 

```c++
Block::Block(const BlockContents& contents)
    : data_(contents.data.data()),
      size_(contents.data.size()),
      owned_(contents.heap_allocated) { 
  //　当数据存储在堆上的时候 owned_ 才为 true
  if (size_ < sizeof(uint32_t)) { 
    // block 最后 4 字节用于存储 restart 个数, 
    // 所以 block 长度最小为 4 字节长度.
    size_ = 0;  // Error marker
  } else {
    // 该 Block 最多可以分配的 restart 的个数, 其中每个 restart 
    // 为 4 字节偏移量.
    // 在构建 block 的时候会每隔一段设置一个 restart point(用于前缀压缩), 
    // 位于 restart point 的数据项的 key 保留原样, 此项之后
    // 的数据项会相对于前一个数据项进行前缀压缩直至下一个 restart  point. 
    // block 最后 4 字节用于存储 restart 个数, 所以计算时不能算在内. 
    size_t max_restarts_allowed = (size_-sizeof(uint32_t)) / sizeof(uint32_t);
    if (NumRestarts() > max_restarts_allowed) {
      // 如果实际的 restart 总数超过了上面计算的最大值, 
      // 那该 Block 空间太小了, 肯定有问题
      size_ = 0; 
    } else {
      // 最后一个 uint_32 存的是 restart 个数, 不能用于存放 restart; 
      // 全部 restart 占用字节数为 NumRestarts() * sizeof(uint32_t); 
      // 所以 restart array 的起始 offset 为下面的值. 
      restart_offset_ = size_ - (1 + NumRestarts()) * sizeof(uint32_t);
    }
  }
}
```

### 2.2.2 读 footer

footer 解析比较简单, 主要就是获取 meta-index block 和 data-index block 分别在文件中的地址和大小:

```c++
// 从 input 指向内存解码出一个 Footer, 
// 先解码最后 8 字节的魔数(按照小端模式), 
// 然后一次解码两个 BlockHandle. 
Status Footer::DecodeFrom(Slice* input) {
  // 1 按照小端模式解析末尾 8 字节的魔数
  const char* magic_ptr = input->data() + kEncodedLength - 8;
  const uint32_t magic_lo = DecodeFixed32(magic_ptr);
  const uint32_t magic_hi = DecodeFixed32(magic_ptr + 4);
  const uint64_t magic = ((static_cast<uint64_t>(magic_hi) << 32) |
                          (static_cast<uint64_t>(magic_lo)));
  if (magic != kTableMagicNumber) {
    return Status::Corruption("not an sstable (bad magic number)");
  }

  // 2 解析 meta-index block 的 handle
  // (包含 meta index block 起始偏移量及其长度)
  Status result = metaindex_handle_.DecodeFrom(input);
  if (result.ok()) {
    // 3 解析 index block 的 handle
    // (包含 index block 起始偏移量及其长度)
    result = index_handle_.DecodeFrom(input);
  }
  if (result.ok()) { 
    // 4 跳过 padding
    // meta-index handle + data-index handle + padding + 魔数.
    // 此时 input 包含的数据只剩下可能的 padding 0 了, 跳过.
    // end 为 footer 尾部
    const char* end = magic_ptr + 8;
    // 第二个参数为值为 0. 生成下面这个 slice 后面没有使用. 
    *input = Slice(end, input->data() + input->size() - end); 
  }
  return result;
}
```

### 2.2.3 读 data-index block

拿到 data-index block 地址和大小后就可以解析它了:

```c++
/**
  * 2 解析 data-index block:
  * 根据已解析的 Footer, 解析出 index block(它保存了指向全部 data blocks 的索引) 
  * 存储到 index_block_contents.
  */
BlockContents index_block_contents;
if (s.ok()) {
  ReadOptions opt;
  if (options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  // 读取 index block, 它对应的 BlockHandle 存储在 footer 里面
  s = ReadBlock(file, opt, footer.index_handle(), &index_block_contents);
}
```

上面 `ReadBlock()` 最后一个输出型参数即为 data-index block 内容, 可以将其反序列化为 `Block` 对象:

```c++
Block* index_block = new Block(index_block_contents);
```

### 2.2.4 读 meta-index block

解析出来的 meta-index block handle 放到了 footer 对象中, 根据它就可以解析 meta-index block 和 meta block 了:

```c++
/**
  * 3 解析 meta-index block 和 meta block:
  * 根据已解析的 Footer 所包含的 metaindex block 指针, 
  * 解析出 metaindex block, 再基于此解析出 mate block 
  * 存储到 Table::rep_.
  */
// 读取并解析 filter block 到 table::rep_, 
// 它一般为布隆过滤器, 可以加速数据查询过程.
(*table)->ReadMeta(footer);
```

这个过程统一在 `ReadMeta()` 完成, 具体读取 meta block 会有专用的 `ReadFilter()` 完成:

```c++
// 解析 table 的 metaindex block(需要先解析 table 的 footer);
// 然后根据解析出来的 metaindex block 解析 meta block(目前 meta block
// 仅有 filter block 一种). 
// 这就是我们要的元数据, 解析出来的元数据会被放到 Table::rep_ 中. 
void Table::ReadMeta(const Footer& footer) {
  // 如果压根就没配置过滤策略, 那么无序解析元数据
  if (rep_->options.filter_policy == nullptr) {
    return;
  }

  /**
   * 根据 Footer 保存的 metaindex BlockHandle 
   * 解析对应的 metaindex block 到 meta 中
   */
  ReadOptions opt;
  if (rep_->options.paranoid_checks) {
    // 如果开启了偏执检查, 则校验每个 block 的 crc
    opt.verify_checksums = true; 
  }
  BlockContents contents;
  // 1 根据 handle 读取 metaindex block (从 rep_ 到 contents)
  if (!ReadBlock(rep_->file, opt, footer.metaindex_handle(), &contents).ok()) {
    // 由于 filter block 不是必须的, 没有 filter 最多就是读得慢一些;
    // 所以出错也不再继续传播, 而是直接返回.
    return;
  }

  // 这个变量叫 metaindex 更合适
  Block* meta = new Block(contents);

  // 为 metaindex block 创建一个迭代器
  Iterator* iter = meta->NewIterator(BytewiseComparator()); 
  // 具体见 table_format.md
  // metaindex block 有一个 entry 包含了 FilterPolicy name 
  // 到其对应的 filter block 的映射
  std::string key = "filter.";
  // filter-policy name 在调用方传进来的配置项中
  key.append(rep_->options.filter_policy->Name());
  // 在 metaindex block 搜寻 key 对应的 meta block 的 handle
  iter->Seek(key); 
  if (iter->Valid() && iter->key() == Slice(key)) {
    // 2 找到了, 迭代器对应的 value 即为 meta block handle,
    // 根据其解析对应的 filter block(就是 meta block), 解析出来的
    // 内容会放到 rep_ 中.
    ReadFilter(iter->value()); 
  }
  delete iter;
  delete meta;
}
```

### 2.2.5 读 meta block

前一节提到了读取 meta(filter) block 读取:
```c++
// 2 找到了, 则解析对应的 filter block, 解析出来的
// 内容会放到 rep_ 中.
ReadFilter(iter->value()); 
```    

具体读取由 `ReadFilter()` 方法完成:

```c++
// 解析 table 的 filter block(需要先解析 table 的 metaindex block)
// 解析出的数据放到了 table.rep.filter
void Table::ReadFilter(const Slice& filter_handle_value) {
  Slice v = filter_handle_value;
  BlockHandle filter_handle;
  // 解析出 filter block 对应的 blockhandle, 它包含 filter block 
  // 在 sstable 中的偏移量和大小
  if (!filter_handle.DecodeFrom(&v).ok()) {
    return;
  }

  ReadOptions opt;
  if (rep_->options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  BlockContents block;
  // 读取 filter block(从 rep_ 到 block)
  if (!ReadBlock(rep_->file, opt, filter_handle, &block).ok()) {
    return;
  }

  // 如果 blockcontents 中的内存是从堆分配的, 
  // 需要将其地址赋值给 rep_->filter_data 以方便后续释放(见 ~Rep())
  if (block.heap_allocated) {
    rep_->filter_data = block.data.data();
  }
  // 反序列化 filter block (从 block.data 到 FilterBlockReader)
  rep_->filter = new FilterBlockReader(rep_->options.filter_policy, block.data);
}
```

### 2.2.6 读 block 内容的通用方法

每个 block(data block, meta block, meta-index block, data-index block 四类) 在写入 sstable 后都会紧接着追加一字节长的压缩类型和四字节长的 crc(它是 block + 压缩类型一起算出来的), 负责读取解析 block + 压缩类型 + crc 的方法为位于 `table/format.cc` 中的 `ReadBlock()` 方法:

```c++
// 从 file 去读 handle 指向的 block:
// - 读取整个块, 包含数据+压缩类型(1 字节)+crc(4 字节)
// - 校验 crc: 重新计算 crc 并与保存 crc 比较
// - 解析压缩类型, 根据压缩类型对数据进行解压缩
// - 将 block 数据部分保存到 BlockContents 中
// 失败返回 non-OK; 成功则将数据填充到 *result 并返回 OK. 
Status ReadBlock(RandomAccessFile* file,
                 const ReadOptions& options,
                 const BlockHandle& handle,
                 BlockContents* result) {
  result->data = Slice();
  result->cachable = false;
  result->heap_allocated = false;

  /**
   * 解析 block.
   * 读取 block 内容以及 type 和 crc. 
   * 具体见 table_builder.cc 中构造这个结构的代码.
   */
  // 要读取的 block 的大小
  size_t n = static_cast<size_t>(handle.size()); 
  // 每个 block 后面紧跟着它的压缩类型 type (1 字节)和 crc (4 字节)
  char* buf = new char[n + kBlockTrailerSize]; 
  Slice contents;
  // handle.offset() 指向对应 block 在文件里的起始偏移量
  Status s = file->Read(handle.offset(), n + kBlockTrailerSize, &contents, buf);
  if (!s.ok()) {
    delete[] buf;
    return s;
  }
  if (contents.size() != n + kBlockTrailerSize) {
    delete[] buf;
    return Status::Corruption("truncated block read");
  }

  /**
   * 校验 type 和 block 内容加在一起对应的 crc
   */
  const char* data = contents.data();
  if (options.verify_checksums) {
    // 读取 block 末尾的 crc(始于第 n+1 字节)
    const uint32_t crc = crc32c::Unmask(DecodeFixed32(data + n + 1));
    // 计算 block 前 n+1 字节的 crc
    const uint32_t actual = crc32c::Value(data, n + 1);
    // 比较保存的 crc 和实际计算的 crc 是否一致
    if (actual != crc) {
      delete[] buf;
      s = Status::Corruption("block checksum mismatch");
      return s;
    }
  }

  /**
   * 解析 type, 并根据 type 解析 block data
   */
  // type 表示 block 的压缩状态
  switch (data[n]) {
    case kNoCompression:
      if (data != buf) {
        delete[] buf;
        result->data = Slice(data, n);
        result->heap_allocated = false;
        result->cachable = false;  // Do not double-cache
      } else {
        result->data = Slice(buf, n);
        result->heap_allocated = true;
        result->cachable = true;
      }

      // Ok
      break;
    case kSnappyCompression: {
      size_t ulength = 0;
      // 获取 snappy 压缩前的数据的大小以分配内存
      if (!port::Snappy_GetUncompressedLength(data, n, &ulength)) {
        delete[] buf;
        return Status::Corruption("corrupted compressed block contents");
      }
      char* ubuf = new char[ulength];
      // 将 snappy 压缩过的数据解压缩到上面分配的内存中
      if (!port::Snappy_Uncompress(data, n, ubuf)) {
        delete[] buf;
        delete[] ubuf;
        return Status::Corruption("corrupted compressed block contents");
      }
      delete[] buf;
      result->data = Slice(ubuf, ulength);
      result->heap_allocated = true;
      result->cachable = true;
      break;
    }
    default:
      delete[] buf;
      return Status::Corruption("bad block type");
  }

  return Status::OK();
}
```


## 2.3 Table 和两级迭代器的结合

前面讲过了, 打开 sstable 文件后会生成对应的 Table 对象, 该对象会被放到 TableCache 缓存中. 如果要访问其内容, 需要一个迭代器, 该工作通过 `leveldb::Iterator *leveldb::Table::NewIterator` 完成:

```c++
// 先为 data-index block 数据项构造一个迭代器 index_iter, 
// 然后基于 index_iter 查询时, 为其指向的具体 data block 
// 构造一个迭代器 data_iter, 进而可以迭代该 data block 里
// 的全部数据项.
// 这样就构成了一个两级迭代器, 从而实现遍历全部 data blocks 的数据项. 
Iterator* Table::NewIterator(const ReadOptions& options) const {
  return NewTwoLevelIterator(
      rep_->index_block->NewIterator(rep_->options.comparator),
      &Table::BlockReader, const_cast<Table*>(this), options);
}
```

返回的迭代器为一个 `leveldb::<unnamed>::TwoLevelIterator`, 该迭代器处于匿名的命名空间所以未直接对外暴露, 仅能通过返回的指针访问其从 `class leveldb::Iterator` 继承的方法. 

每个 sstable 文件对应一个两级迭代器, 然后将全部 sstable 对应的两级迭代器级联起来, 就相当于为整个 leveldb 构造了一个迭代器(见 `leveldb::Version::AddIterators()`, 后续会详解该类), 从而实现在整个 leveldb 上轻松实现迭代或者查询.

--End--

