# Leveldb 源码详解系列之二: log 读写

我们先简单回顾下 log 文件相关的基础知识点, 具体请见 [Leveldb 源码详解系列之一: 接口与文件]({{<relref "leveldb-annotations-1-interfaces-and-files.md">}}).

log 文件(*.log)保存着数据库最近一系列更新操作, 它相当于 leveldb 的 WAL([write-ahead logging](https://en.wikipedia.org/wiki/Write-ahead_logging)). 当前在用的 log 文件内容同时也会被记录到一个内存数据结构中(即 `memtable` ). 每个更新操作都被追加到当前的 log 文件和 `memtable` 中. 当 log 文件大小达到一个预定义的大小时(默认大约 4MB), 这个 log 文件对应的 `memtable` 就会被转换为一个 sorted string table 文件落盘然后一个新的 log 文件就会被创建以保存未来的更新操作. 
<!--more-->

# log 文件结构

log 文件内容是一系列 blocks, 每个 block 大小为 32KB(有时候最后一个 block 可能装不满). 每个 block 由一系列 records 构成, 具体定义如下(熟悉编译原理的应该对下述写法不陌生): 

    // 即 0 或多个 records, 0 或 1 个 trailer.
    // 最大为 32768 字节.
    block := record* trailer?
    record :=
      // 下面提到的 type 和 data[] 的 crc32c 校验和, 小端字节序
      checksum: uint32
      // 下面的 data[] 的长度, 小端字节序
      length: uint16
      // 类型, FULL、FIRST、MIDDLE、LAST 取值之一
      type: uint8
      // 用户数据
      data: uint8[length]

type 取值如下:

    FULL == 1
    FIRST == 2
    MIDDLE == 3
    LAST == 4

FULL 类型的 record 包含了一个完整的用户 record 的内容. 

FIRST、MIDDLE、LAST 这三个类型用于被分割成多个 fragments 的用户 record. FIRST 表示某个用户 record 的第一个 fragment, LAST 表示某个用户 record 的最后一个 fragment, MIDDLE 表示某个用户 record 的中间 fragments. 

如果当前 block 恰好剩余 7 个字节(正好可以容纳 record 中的 checksum + length + type), 并且一个新的非 0 长度的 record 要被写入, 那么 writer 必须在此处写入一个 FIRST 类型的 record(但是 length 字段值为 0, data 字段为空. 用户数据 data 部分需要写入下个 block, 而且下个 block 起始还是要写入一个 header 不过其 type 为 middle)来填满该 block 尾部的 7 个字节, 然后在接下来的 blocks 中写入全部用户数据.

# 读 log

下面分析读 log 相关的类和方法.

## 核心文件与核心类

与读 log 相关的代码定义在下面两个文件中:

    db/log_reader.h
    db/log_reader.cc

核心类为 `class leveldb::log::Reader`. 下面针对这个类核心方法进行分析.

## Reader 构造方法

```c++
// 创建一个 Reader 来从 file 中读取和解析 records, 
// 读取的第一个 record 的起始位置位于文件 initial_offset 或其之后的物理地址. 
// 如果 reporter 不为空, 则在检测到数据损坏时汇报要丢弃的数据估计大小. 
// 如果 checksum 为 true, 则在可行的条件比对校验和. 
// 注意, file 和 reporter 的生命期不能短于 Reader 对象. 
Reader(SequentialFile* file, Reporter* reporter, bool checksum, uint64_t initial_offset)
```

## Reader 读取方法

`bool ReadRecord(Slice* record, std::string* scratch)` 方法负责从 log 文件读取内容并反序列化为 Record. 该方法会在 db 的 `Open` 方法中调用, 负责将磁盘上的 log 文件转换为内存中 memtable. 其它数据库恢复场景也会用到该方法.

所做的事情, 概括地讲就是从文件读取下一个 record 到 `*record` 中. 如果读取成功, 返回 true; 遇到文件尾返回 false. 如果当前读取的 record 没有被分片, 那就用不到 `*scratch` 参数来为 `*record` 做底层存储了; 其它情况需要借助 `*scratch` 来拼装分片的 record data 部分, 最后封装为一个 Slice 赋值给 `*record`. 

具体处理流程见下面详细注释:

```c++
bool Reader::ReadRecord(Slice* record, std::string* scratch) {
  // last_record_offset_ 表示上一次调用 ReadRecord 方法返回的
  // record 的起始偏移量, 注意这个 record 是逻辑的. 
  // initial_offset_ 表示用户创建 Reader 时指定的在文件中寻找第一个 record 的起始地址.
  // 如果条件成立, 表示当前方法是首次被调用.
  if (last_record_offset_ < initial_offset_) {
    // 跳到我们要读取的第一个 block 起始位置
    if (!SkipToInitialBlock()) {
      return false;
    }
  }

  scratch->clear();
  record->clear();
  // 指示正在处理的 record 是否被分片了, 
  // 除非逻辑 record 对应的物理 record 类型是 full, 否则就是被分片了.
  bool in_fragmented_record = false; 

  // 记录我们正在读取的逻辑 record 的起始偏移量. 初值为 0 无实际意义仅为编译器不发警告. 
  // 为啥叫逻辑 record 呢？
  // 因为 block 大小限制, 所以 record 可能被分成多个分片(fragment). 
  // 我们管 fragment 叫物理 record, 一个或多个物理 record 构成一个逻辑 record. 
  uint64_t prospective_record_offset = 0;

  Slice fragment;
  while (true) {
    // 从文件读取一个物理 record 并将其 data 部分保存到 fragment, 
    // 同时返回该 record 的 type.
    const unsigned int record_type = ReadPhysicalRecord(&fragment);

    // 计算返回的当前 record 在 log file 中的起始地址
    //    = 当前文件待读取位置
    //      - buffer 剩余字节数
    //      - 刚读取的 record 头大小
    //      - 刚读取 record 数据部分大小
    // end_of_buffer_offset_ 表示 log file 待读取字节位置
    // buffer_ 表示是对一整个 block 数据的封装, 底层存储为 backing_store_, 
    //    每次执行 ReadPhysicalRecord 时会移动 buffer_ 指针.
    uint64_t physical_record_offset =
        end_of_buffer_offset_ - buffer_.size() - kHeaderSize - fragment.size(); 

    // resyncing_ 用于跳过起始地址不符合 initial_offset_ 的 record,
    // 如果为 true 表示目前还在定位第一个满足条件的逻辑 record 中.
    // 与 initial_offset_ 的比较判断在上面 ReadPhysicalRecord 中进行.
    if (resyncing_) {
      // 只要数据没有损坏或到达文件尾, 而且返回的 record_type 只要
      // 不是 kBadRecord(返回该类型其中一个情况就是起始地址不满足条件)
      // 就说明当前 record 起始地址已经大于 initial_offset_ 了,
      // 但是如果当前 record 的 type 为 middle 或者 last, 
      // 那么逻辑上这个 record 仍然与不符合 initial_offset_ 的
      // 类型为 first 的 record 同属一个逻辑 record, 
      // 所以当前 record 也不是我们要的.
      if (record_type == kMiddleType) {
        continue;
      } else if (record_type == kLastType) {
        resyncing_ = false;
        continue;
        continue;
      } else {
        // 如果是 full 类型的 record, 而且这个 record 起始地址
        // 不小于 inital_offset_(否则 ReadPhysicalRecord 返回的
        //     类型就是 kBadRecord 而非 full), 
        // 满足条件了, 关掉标识.
        // 如果返回 kBadRecord/kEof(没什么可读了)/
        // 未知类型(但是起始位置满足要求), 也会关掉该标识.
        resyncing_ = false;
      }
    }

    // 注意, 下面 switch 有的 case 是 return, 有的是 break.
    switch (record_type) {
      case kFullType:
        if (in_fragmented_record) {
          // 早期版本 writer 实现存在 bug. 
          // 即如果上一个 block 末尾保存的是一个 FIRST 类型的 header, 
          // 那么接下来 block 开头应该是一个 MIDDLE 类型的 record, 
          // 但是早期版本写入了 FIRST 类型或者 FULL 类型的 record. 
          if (!scratch->empty()) {
            ReportCorruption(scratch->size(), "partial record without end(1)");
          }
        }
        prospective_record_offset = physical_record_offset;
        scratch->clear();
        // 赋值构造
        // FULL 类型 record 不用借助 scratch 拼装了
        *record = fragment; 
        last_record_offset_ = prospective_record_offset;
        // 读取到一个完整逻辑 record, 完成任务.
        return true;
      // 注意, 只有 first 类型的 record 起始地址满足大于 inital_offset_ 的时候
      // 才会返回其真实类型 first, 其它情况哪怕是 first 返回也是 kBadRecord.
      case kFirstType:
        if (in_fragmented_record) {
          // 早期版本 writer 实现存在 bug. 
          // 即如果上一个 block 末尾保存的是一个 FIRST 类型的 header, 
          // 那么接下来 block 开头应该是一个 MIDDLE 类型的 record, 
          // 但是早期版本写入了 FIRST 类型或者 FULL 类型的 record. 
          if (!scratch->empty()) {
            ReportCorruption(scratch->size(), "partial record without end(2)");
          }
        }
        // FIRST 类型物理 record 起始地址也是对应逻辑 record 的起始地址
        prospective_record_offset = physical_record_offset;
        // 非 FULL 类型 record 需要借助 scratch 拼装成一个完整的 record data 部分.
        // 注意只有 first 时采用 assign, first 后面的分片要用 append
        scratch->assign(fragment.data(), fragment.size());
        // 除了 FULL 类型 record, 都说明当前读取的 record 被分片了, 
        // 还需要后续继续读取.
        in_fragmented_record = true; 
        // 刚读了 first, 没读完, 继续.
        break;

      case kMiddleType:
        // 都存在 MIDDLE 了, 竟然还说当前 record 没分片, 报错. 
        if (!in_fragmented_record) { 
          ReportCorruption(fragment.size(),
                           "missing start of fragmented record(1)");
        } else {
          // 非 FULL 类型 record 需要借助 scratch 拼装成一个
          // 完整的 record data 部分, 
          // FIRST 类型已经打底了, MIDDLE 直接追加即可. 
          scratch->append(fragment.data(), fragment.size());
        }
        // 还是 middle, 没读完, 继续.
        break;

      case kLastType:
        // 都存在 LAST 了, 竟然还说当前 record 没分片, 矛盾. 
        if (!in_fragmented_record) { 
          ReportCorruption(fragment.size(),
                           "missing start of fragmented record(2)");
        } else {
          // 非 FULL 类型 record 需要借助 scratch 
          // 拼装成一个完整的 record data 部分, 
          // FIRST 类型已经打底了, LAST 直接追加即可. 
          scratch->append(fragment.data(), fragment.size());
          *record = Slice(*scratch);
          last_record_offset_ = prospective_record_offset;
          // 读完了, 完成任务.
          return true;
        }
        break;

      case kEof:
        // 如果都读到文件尾部了, 逻辑 record 还没读全, 那就是文件损坏了. 
        if (in_fragmented_record) {
          // 可能由于 writer 写完一个物理 record 后挂掉了, 
          // 我们不把这种情况作为数据损坏, 直接忽略整个逻辑 record.
          // 数据损坏, 丢掉之前可能已经解析的数据 
          scratch->clear();
        }
        // 文件尾了, 读到读不到都拜拜.
        return false;

      case kBadRecord:
        // 逻辑 record 有分片已经被读取过了, 但是本次读取物理 record 遇到了错误
        if (in_fragmented_record) { 
          ReportCorruption(scratch->size(), "error in middle of record");
          in_fragmented_record = false;
          // 数据损坏, 丢掉之前可能已经解析的数据
          scratch->clear();
        }
        // 遇到错误也不中止, 继续读取数据进行解析直到读取完整逻辑 record 的目标达成
        break;

      default: {
        char buf[40];
        snprintf(buf, sizeof(buf), "unknown record type %u", record_type);
        ReportCorruption(
            (fragment.size() + (in_fragmented_record ? scratch->size() : 0)),
            buf);
        in_fragmented_record = false;
        scratch->clear();
        // 遇到错误也不中止, 继续读取数据进行解析直到读取完整逻辑 record 的目标达成
        break;
      }
    }
  }
  return false;
}
```

上面方法中, 最重要的一个辅助函数为 `ReadPhysicalRecord`, 该方法负责从文件读取 block, 然后再从 block(block 为空但还没读取到一个完整逻辑 record, 会继续从 log 文件读取 block) 读取并解析一个物理 record 并将其 data 部分保存到 result, 同时返回该物理 record 的 type. 返回的 type 为下面几种之一:
- kEof, 到达文件尾
- kBadRecord, 当前 record 损坏, 或者当前物理 record 起始地址小于用户指定的起始地址 inital_offset_(此时其实际 type 可能为 first) 
- first/middle/last/full 之一(注意, 除了 first/full, 其它类型对应物理 record 起始地址虽然不小于用户指定的 inital_offset_, 但是其所归属的逻辑 record 的起始地址可能不满足要求, 所以此时这两类物理 record 也会被 `Reader::ReadRecord` 方法跳过.)

```c++
unsigned int Reader::ReadPhysicalRecord(Slice* result) {
  // 跳出循环需要满足下面条件之一:
  // - 文件损坏
  // - 无有效数据(非 tailer)且到达文件尾
  // - 读到了一个有效 block(然后解析其中的 record)
  while (true) {
    // 先确认要不要读取一个新的 blcok.
    // buffer_ 底层指针会向前移动, 所以其 size 是动态的. 
    // 如果 buffer 剩余内容字节数小于 kHeaderSize 且不为空, 
    // 表示 buffer 里面剩余字节是个 trailer, 可以跳过它去读取解析下个 block 了;
    // 否则, 跳过 if 继续从该 block 解析 record.
    if (buffer_.size() < kHeaderSize) {
      if (!eof_) { // 如果未到文件尾
        // Last read was a full read, so this is a trailer to skip
        buffer_.clear();
        // 从 log file 读取一整个 block 放到 backing_store_, 
        // 然后将 backing_store_ 封装到 buffer_ 中. 
        Status status = file_->Read(kBlockSize, &buffer_, backing_store_);
        // 更新 end_of_buffer_offset_ 至迄今从 log 读取最大位置下一个字节
        end_of_buffer_offset_ += buffer_.size(); 
        // 如果 log 文件损坏
        if (!status.ok()) {
          buffer_.clear();
          // 一个 block 被丢掉
          ReportDrop(kBlockSize, status);
          // 读文件失败我们认为到达文件尾; 
          // 注意, file_->Read 读到文件尾不会报错因为这是正常情况.
          // 只有遇到错误才会报错. 
          eof_ = true; 
          return kEof;
        } else if (buffer_.size() < kBlockSize) {
          // 如果读取的 block 数据小于 block 容量, 
          // 则肯定到达 log 文件尾部了, 处理其中的 records.
          eof_ = true; 
        }
        continue;
      } else {
        // 注意, 如果 buffer_ 非空, 则其内容为一个
        // 位于文件尾的截断的 record header. 
        // 这可能是因为 writer 写 header 时崩溃导致的. 
        // 我们不会把这种情况当做错误, 而是当做读作文件尾来处理. 
        buffer_.clear();
        return kEof;
      }
    }

    /**
     * 注意, 一个 block 可以包含多个 records, 
     * 但是 block 最后一个 record 可能只包含 
     * header(这是由于 block 最后只剩下 7 个字节)
     */
    // 解析 record 的 header.
    // record header, 由 checksum (4 bytes), 
    // length (2 bytes), type (1 byte) 构成. 
    const char* header = buffer_.data(); 
    /**
     * 下面两步骤是解析 length 的两个字节, 小端字节序, 所以需要拼, 
     * 具体见 Writer::EmitPhysicalRecord
     */
    // xxxx|(xx|x|x)xxxxx 将左边括号内容转换为无符号 32 位数,
    // 并取出最后 8 位即括号左起第一个 x
    const uint32_t a = static_cast<uint32_t>(header[4]) & 0xff;
    // xxxx|x(x|x|xx)xxxx 将左边括号内容转换为无符号 32 位数,
    // 并取出最后 8 位即括号左起第一个 x
    const uint32_t b = static_cast<uint32_t>(header[5]) & 0xff;
    // xxxx|xx|(x)|xxxxxx 读取左边括号内容即 record type
    const unsigned int type = header[6]; 
    // b 和 a 拼接构成了 length
    const uint32_t length = a | (b << 8); 
    // 如果解析出的 length 加上 header 长度大于 buffer_ 剩余数据长度, 
    // 则说明数据损坏了, 比如 length 被篡改了. 
    if (kHeaderSize + length > buffer_.size()) {
      size_t drop_size = buffer_.size();
      buffer_.clear();
      // 如果未到文件尾, 报告 length 损坏. 
      if (!eof_) { 
        ReportCorruption(drop_size, "bad record length");
        return kBadRecord;
      }

      // 如果已经到了文件尾, 即当前读取 block 为 log file 最后一个 block.
      // 由于 length  有问题我们也没必要也没办法读取 data 部分, 
      // 我们假设这种情况原因是 writer 写数据时崩溃了. 
      // 这种情况我们不作为错误去报告, 而是当做到达文件尾了. 
      return kEof;
    }

    if (type == kZeroType && length == 0) {
      // 跳过 0 长度的 record, 而且不会报告数据丢弃. 
      // 因为这些 records 产生的原因是 env_posix.cc 
      // 中基于 mmap 的写入代码在执行时会预分配文件区域. 
      buffer_.clear();
      return kBadRecord;
    }

    // Check crc
    if (checksum_) {
      // 读取 crc
      uint32_t expected_crc = crc32c::Unmask(DecodeFixed32(header)); 
      // crc 是基于 type 和 data 来计算的
      uint32_t actual_crc = crc32c::Value(header + 6, 1 + length); 
      if (actual_crc != expected_crc) {
        // crc 校验失败, 可能是 length 字段出错, 数据损坏,
        // 丢弃这个 block 的剩余部分
        size_t drop_size = buffer_.size();
        buffer_.clear();
        ReportCorruption(drop_size, "checksum mismatch");
        return kBadRecord;
      }
    }

    // header 解析完毕, 将当前 record 从 buffer_ 中
    // 移除(通过向前移动 buffer_ 底层存储指针实现)
    buffer_.remove_prefix(kHeaderSize + length); 

    // 当前解析出的 record 起始地址小于用户指定的起始地址, 
    // 则跳过这个 record(返回 kBadRecord 类型). 
    // 注意, 此时真正的 record type 可能为 first 类型.
    if (end_of_buffer_offset_ - buffer_.size() - kHeaderSize - length
        < initial_offset_) {
      result->clear();
      return kBadRecord;
    }
    
    // 将该 record 的 data 部分返回
    *result = Slice(header + kHeaderSize, length); 
    return type;
  }
}
```


# 写 log

下面分析写 log 相关的类和方法.

## 核心文件与核心类

与写 log 相关的代码定义在下面两个文件中:

    db/log_writer.h
    db/log_writer.cc

核心类为 `class leveldb::log::Writer`. 下面针对这个类核心方法进行分析.    

## Writer 构造方法

```c++
// 创建一个 writer 用于追加数据到 dest 指向的文件.
// dest 指向的文件初始必须为空文件; dest 生命期不能短于 writer.
explicit Writer(WritableFile *dest);

// 创建一个 writer 用于追加数据到 dest 指向的文件.
// dest 指向文件初始长度必须为 dest_length; dest 生命期不能短于 writer.
Writer(WritableFile *dest, uint64_t dest_length);
```

## Writer 写方法

如果用户想把数据写入 log, 则需要将这些数据封装为 `Slice`, 然后调用 `Writer::AddRecord` 将其写入 log 文件. 

写入时, 这个 `Slice` 内容即为 record 的 data 部分, 如果数据量太大导致一个 block(默认 32KB) 装不下, 则这些数据会被分片写入. 也就是说, 这些数据属于一个逻辑 record, 但是因为太大, 被分为若干物理 record 写入到 log 文件.

具体写入流程见源码注释:

```c++
Status Writer::AddRecord(const Slice& slice) {
  const char* ptr = slice.data();
  // data 剩余部分长度, 初始值为其原始长度
  size_t left = slice.size(); 

  // 如有必要则将 record 分片后写入文件. 
  // 如果 slice 内容为空, 则我们仍将会写入一个长度为 0 的 record 到文件中. 
  Status s;
  bool begin = true;
  do {
    // 当前 block 剩余空间大小
    const int leftover = kBlockSize - block_offset_; 
    assert(leftover >= 0);
    // 如果当前 block 剩余空间不足容纳 record 的 header(7 字节) 
    // 则剩余空间作为 trailer 填充 0, 然后切换到新的 block.
    if (leftover < kHeaderSize) { 
      if (leftover > 0) {
        assert(kHeaderSize == 7);
        // 最终填充多少 0 由 leftover 决定, 最大 6 字节
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover)); 
      }
      block_offset_ = 0;
    }

    // 到这一步, block (可能因为不足 kHeaderSize 在上面已经切换到了下个 block)
    // 最终剩余字节必定大约等于 kHeaderSize
    assert(kBlockSize - block_offset_ - kHeaderSize >= 0);

    // block 当前剩余空闲字节数.
    // 除了待写入 header, 当前 block 还剩多大空间, 可能为 0; 
    // block 最后剩下空间可能只够写入一个新 record 的 header 了
    const size_t avail = kBlockSize - block_offset_ - kHeaderSize;
    // 可以写入当前 block 的 record data 剩余内容的长度, 可能为 0
    const size_t fragment_length = (left < avail) ? left : avail;

    RecordType type; 
    // 判断是否将 record 剩余内容分片
    const bool end = (left == fragment_length);
    if (begin && end) {
      // 如果该 record 内容第一次写入文件, 而且, 
      // 如果 block 剩余空间可以容纳 record data 全部内容, 
      // 则写入一个 full 类型 record
      type = kFullType;
    } else if (begin) {
      // 如果该 record 内容第一写入文件, 而且, 
      // 如果 block 剩余空间无法容纳 record data 全部内容, 
      // 则写入一个 first 类型 record. 
      // 注意, 此时是 record 第一次写入即它是一个新 record, 
      //    该 block 剩余空间可能只够容纳 header 了, 
      //    则在 block 尾部写入一个 FIRST 类型 header, record data 不写入, 
      //    等下次循环会切换到下个 block, 然后又会重新写入一个
      //    非 FIRST 类型的 header (注意下面会将 begin 置为 false)
      //    而不是紧接着在新 block 只写入 data 部分. 
      type = kFirstType;
    } else if (end) {
      // 如果这不是该 record 内容第一写入文件, 而且, 
      // 如果 block 剩余空间可以容纳 record data 剩余内容, 
      // 则写入一个 last 类型 record
      type = kLastType;
    } else {
      // 如果这不是该 record 内容第一写入文件, 而且, 
      // 如果 block 剩余空间无法容纳 record data 剩余内容, 
      // 则写入一个 middle 类型 record
      type = kMiddleType;
    }

    // 将类型为 type, data 长度为 fragment_length 的 record 写入 log 文件.
    s = EmitPhysicalRecord(type, ptr, fragment_length);
    ptr += fragment_length;
    left -= fragment_length;
    // 即使当前 block 剩余空间只够写入一个新 record 的 FIRST 类型 header, 
    // record 也算写入过了
    begin = false; 
    // 写入不出错且 record 再无剩余内容则写入完毕
  } while (s.ok() && left > 0); 
  return s;
}
```

'AddRecord` 写入 record 时依赖的辅助方法 `EmitPhysicalRecord`. 该方法负责组装 record header, 然后连同 payload 写入文件.

```c++
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr, size_t n) {
  // data 大小必须能够被 16 位无符号整数表示, 因为 record 的 length 字段只有两字节
  assert(n <= 0xffff);
  // 要写入的内容不能超过当前 block 剩余空间大小
  assert(block_offset_ + kHeaderSize + n <= kBlockSize); 

  // buf 用于组装 record header
  char buf[kHeaderSize];
  // 将数据长度编码到 length 字段, 小端字节序
  // length 低 8 位安排在低地址位置
  buf[4] = static_cast<char>(n & 0xff); 
  // 然后写入 length 高 8 位安排在高地址位置
  buf[5] = static_cast<char>(n >> 8); 
  // 将 type 编码到 type 字段, type 紧随 length 之后 1 字节
  buf[6] = static_cast<char>(t); 

  // 计算 type 和 data 的 crc 并编码安排在最前面 4 个字节
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, n);
  crc = crc32c::Mask(crc);
  // 将 crc 写入到 header 前四个字节
  EncodeFixed32(buf, crc);

  // 写入 header
  Status s = dest_->Append(Slice(buf, kHeaderSize)); 
  if (s.ok()) {
    // 写入 payload
    s = dest_->Append(Slice(ptr, n));
    if (s.ok()) {
      // 刷入文件
      s = dest_->Flush();
    }
  }
  // 当前 block 剩余空间起始偏移量. 
  // 注意, 这里不管 header 和 data 是否写成功. 
  block_offset_ += kHeaderSize + n; 
  return s;
}
```

--End--
