---
title: "Leveldb 源码详解系列之一: 接口与文件"
date: 2020-09-11 23:13:08
tags: 
  - leveldb
  - LSM-Tree
  - db
---

<!-- toc -->

[toc]

## 从哪里着手分析 leveldb 实现

在了解了其基本使用以后, 如果想理解 leveldb 基本原理, 则有两个抓手. 第一个是  `include` 目录下的头文件, 尤其是 `db.h` , 第二个就是它的文件类型及其格式.

下面我们就从接口和文件两个方向来切入 leveldb 的设计与实现.

## leveldb 常用的接口

### Open

```c++
/**
 * 打开一个名为 name 的数据库. 
 *
 * 打开成功, 会把一个指向基于堆内存的数据库指针存储到 *dbptr, 同时返回 OK; 如果打开失败, 
 * 存储 nullptr 到 *dbptr 同时返回一个错误状态. 
 *
 * 调用者不再使用这个数据库时需要负责释放 *dbptr 指向的内存. 
 * @param options 控制数据库行为和性能的参数配置
 * @param name 数据库名称
 * @param dbptr 存储指向堆内存中数据库的指针
 * @return
 */
static Status Open(const Options& options,
                    const std::string& name,
                    DB** dbptr);
```

该方法在数据库启动时调用, 主要工作由 `leveldb::DBImpl::Recover` 方法完成, 后者主要做如下事情:

- 1. 调用其 VersionSet 成员的 `leveldb::VersionSet::Recover` 方法该方法从磁盘读取 CURRENT 文件, 进而读取 MANIFEST 文件内容, 然后在内存建立 level 架构:
  - 读取 CURRENT 文件(不存在则新建)找到最新的 MANIFEST 文件(不存在则新建)的名称
  - 读取该 MANIFEST 文件内容与当前 Version 保存的 level 架构合并保存到一个新建的 Version 中, 然后将这个新的 version 作为当前的 version.
  - 清理过期的文件
  - 这一步我们可以打开全部 sstables, 但最好等会再打开
  - 将 log 文件块转换为一个新的 level-0 sstable
  - 将接下来的要写的数据写入一个新的 log 文件

- 2. 遍历数据库目录下全部文件. 筛选出 sorted table 文件, 验证 VersionSet 包含的 level 架构图有效性; 同时将全部 log 文件筛选换出来后续反序列化成 memtable. 恢复 log 文件时会按照从旧到新逐个 log 文件恢复, 这样新的修改会覆盖旧的, 如果对应 memtable 太大了, 将其转为 sorted table 文件写入磁盘, 同时将其对应的 table 对象放到 table_cache_ 缓存. 若发生 memtable 落盘表示 level 架构新增文件则将 save_manifest 标记为 true, 表示需要写变更日志到 manifest 文件. 恢复 log 文件主要由方法 `leveldb::DBImpl::RecoverLogFile` 负责完成.

### Put

```c++
/**
 * 将 <key, value> 对写入数据库, 成功返回 OK, 失败返回错误状态. 
 * @param options 本次写操作相关的配置参数, 如果有需要可以将该参数中的 sync 置为 true, 不容易丢数据但更慢. 
 * @param key Slice 类型的 key
 * @param value Slice 类型的 value
 * @return 返回类型为 Status
 */
virtual Status Put(const WriteOptions& options,
                    const Slice& key,
                    const Slice& value) = 0;
```

该方法主要依赖 `leveldb::DBImpl::Write` 实现.

### Delete

```c++
/**
 * 从数据删除指定键为 key 的键值对. 如果 key 不存在不算错. 
 *
 * @param options 本次写操作相关的配置参数, 如果有需要可以将该参数中的 sync 置为 true, 不容易丢数据但更慢. 
 * @param key 要删除数据项对应的 key
 * @return
 */
virtual Status Delete(const WriteOptions& options, const Slice& key) = 0;
```

该方法主要依赖 `leveldb::DBImpl::Write` 实现.

### Write

```c++
/**
 * 对数据库进行批量更新写操作.
 *
 * 该方法线程安全, 内部自带同步. 
 *
 * @param options 本次写操作相关的配置参数, 如果有需要可以将该参数中的 sync 置为 true, 不容易丢数据但更慢. 
 * @param updates 要进行的批量更新操作
 * @return
 */
virtual Status Write(const WriteOptions& options, WriteBatch* updates) = 0;
```

该方法是 `leveldb::DBImpl::Write` 原型.

针对调用 db 进行的写操作, 都会生成一个对应的 `struct leveldb::DBImpl::Writer`, 其封装了写入数据和写入进度. 新构造的 writer 会被放入一个队列. 循环检查, 若当前 writer 工作没完成并且不是队首元素, 则当前有其它 writer 在写, 挂起当前 writer 等待条件成熟. 当前 writer 如果被排在前面的 writer 给合并写入了, 那么它的 done 就被标记为完成了. 否则会被其它在写入的 writer 调用其 signal 将其唤醒执行写入工作.

当执行写入工作时(被前一个执行写入并完成工作的 writer 唤醒了), 首先确认是否为本次该 writer 写操作分配新的 log 文件, 如果需要则分配. 因为该 writer 成为队首 writer 了, 则它负责将队列前面若干 writers 的 batch 合并为一个(该工作由`leveldb::DBImpl::BuildBatchGroup` 负责完成), 注意, 被合并的 writers 不出队(待合并写入完成再出队, 具体见后面描述), 所以写 log 期间队首 writer 不变. 具体写入工作由 `leveldb::log::Writer::AddRecord` 负责, 就是将数据序列化为 record 写入 log 文件. 如果追加 log 文件成功,则将被追加的数据插入到内存中的 memtable 中. 待写入完毕, 该 writer 将参与前述 batch group 写入 log 文件的 writer 都取出来并设置为写入完成, 即将其出队, 将其 done 置为 true, 同时向其发送信号将其唤醒, 被唤醒后它会检查其 done 标识并返回. 最后唤醒队首 writer 执行下一个合并写入.

### Get

```c++
/**
 * 查询键为 key 的数据项, 如果存在则将对应的 value 地址存储到第二个参数中. 
 *
 * 如果 key 不存在, 第二个参数不变, 返回值为 IsNotFound Status. 
 *
 * @param options 本次读操作对应的配置参数
 * @param key 要查询的 key, Slice 引用类型
 * @param value 存储与 key 对应的值的指针, string 指针类型
 * @return
 */
virtual Status Get(const ReadOptions& options,
                    const Slice& key, std::string* value) = 0;
```

- 1 先查询当前在用的 memtable(具体工作由 `leveldb::MemTable::Get` 负责, 本质就是 SkipList 查询, 速度很快)
- 2 如果没有则查询正在转换为 sorted table 的 memtable 中寻找
- 3 如果没有则我们在磁盘上采用从底向上 level-by-level 的寻找目标 key. 

针对上述第 3 步, 具体由 db VersionSet 的当前 Version 负责, 因为该结构保存了 db 当前最新的 level 架构信息, 即每个 level 及其对应的文件列表和每个文件的键范围. 对应方法为 `leveldb::Version::Get`, 具体为:
- 从低 level 向高 level 寻找. 由于 level 越低数据越新, 因此, 当我们在一个较低的 level 找到数据的时候, 不用在更高的 levels 找了.
- 由于 level-0 文件之间可能存在重叠, 而且针对同一个 key, 后产生的文件数据更新所以先将包含 key 的文件找出来按照文件号从大到小(对应文件从新到老)排序查找 key;
- 针对 level-1 及其以上 level, 由于每个 level 内文件之间不存在重叠, 于是在每个 level 中直接采用二分查找定位 key.

另外需要注意的的是, 参数 `options` 可以配置一个快照, 快照对应了数据库历史上的一个操作序列号, 查询时仅查询不大于该序列号的操作范围. 针对同样的 key, 如果历史上有多次更新操作, 而用户想查找特定更新, 这就是实现途径. 如果没有配置快照选项, 默认采用当前最大序列号进行查询.

### NewIterator

```go
/**
 * 返回基于堆内存的迭代器, 可以用该迭代器遍历整个数据库的内容. 
 * 该函数返回的迭代器初始是无效的(在使用迭代器之前, 调用者必须在其上调用 Seek 方法). 
 *
 * 当不再使用时, 调用者应该释放该迭代器对应的内存, 而且迭代器必须在数据库释放之前进行释放. 
 * @param options 本次读操作对应的配置参数
 * @return
 */
virtual Iterator* NewIterator(const ReadOptions& options) = 0;
```

该方法负责将内存 memtable(可能有两个, 一个在写, 一个写完待存盘) 和磁盘 sorted table 文件全部数据结构串起来构造一个大一统迭代器, 可以遍历整个数据库.

上述工作其实是由 `leveldb::Iterator *leveldb::DBImpl::NewInternalIterator` 负责完成的. 该方法实现涉及到 leveldb 特别精巧的迭代器的实现. 这个单独可以写一篇文章来专门介绍. 这里大致说下处理流程:
- 1 初始化一个列表
- 2 把当前 memtable 迭代器加入列表中
- 3 把待写盘 memtable 迭代器追加到列表中
- 4 将当前 version 维护的 level 架构中每个 sorted table 文件对应的迭代器追加到列表中. 针对 level-0 和其它 levels 处理方式不同.
  - 由于 level-0 文件之间可能存在重叠, 所以按照文件生成顺序(这极其重要, 其实就是按照 key 从小到大, 只有这样才能确保最后生成的迭代器能够从小到大按序遍历整个数据库) 为每个文件生成一个两级迭代器(`TwoLevelIterator`, 该结构巧妙地将索引块和数据块结合到了一起)追加到列表中. 
  - 针对 level-1 及其以上 level, 按照从低 level 到高 level(这极其重要, 原因同 level-0), 为每个 level 生成一个两级迭代器, 数据结构依然是 `TwoLevelIterator`, 不过这里把每个 level 的文件列表抽象成了第一级索引, 然后每个文件对应的 table 对象抽象层二级索引.
- 最后将前述全部迭代器构成的迭代器列表再级联成一个大一统的迭代器 `MergingIterator`. 这其实也是一个两级迭代器, 第一级指向迭代器列表, 第二级是某个迭代器指向的内容的迭代器.

最后返回给调用者的就是 `MergingIterator` 实例. 可以调用它的相关方法在整个数据库上寻找目标 key.

### GetSnapshot

```c++
/**
 * 返回当前 DB 状态的一个快照. 
 * 使用该快照创建的全部迭代器将会都指向一个当前 DB 的一个稳定快照. 
 *
 * 当不再使用该快照时, 调用者必须调用 ReleaseSnapshot 将其释放. 
 * @return
 */
virtual const Snapshot* GetSnapshot() = 0;
```

用数据库当前最新的更新操作对应的序列号创建一个快照. 快照最核心的就是那个操作序列号, 因为查询时会把 用户提供的 key(我们叫做 user_key)和操作序列号一起构成一个 internal_key(数据库存储的 key 就是它), 针对 user_key 相等的情况比如针对 hello 这个 user_key Put 多次, 则每次序列号就不一样, 于是根据特定序列号可以查询到特定的那次 Put 写入的 value 值.

这个新生成的快照会被挂载到一个双向链表上, 用完后可以调用 `ReleaseSnapshot` 将其释放掉.

### ReleaseSnapshot

```c++
/**
 * 释放一个之前获取的快照, 释放后, 调用者不能再使用该快照了. 
 * @param snapshot 指向要释放的快照的指针
 */
virtual void ReleaseSnapshot(const Snapshot* snapshot) = 0;
```

从双向链表上删除指定的快照.

### GetProperty

```c++
/**
 * DB 实现可以通过该方法导出自身状态相关的信息. 如果提供的属性可以被 DB 实现理解, 
 * 那么第二个参数将会存储该属性对应的当前值同时该方法返回 true, 其它情况该方法返回 false. 
 *
 * 合法的属性名称包括: 
 *
 * "leveldb.num-files-at-level<N>" - 返回 level <N> 的文件个数, 其中 <N> 是一个数字. 
 *
 * "leveldb.stats" - 返回多行字符串, 描述该 DB 内部操作相关的统计数据. 
 *
 * "leveldb.sstables" - 返回多行字符串, 描述构成该 DB 的全部 sstable 相关信息. 
 *
 * "leveldb.approximate-memory-usage" - 返回被该 DB 使用的内存字节数近似值
 * @param property 要查询的属性名称
 * @param value 保存属性名称对应的属性值
 * @return
 */
virtual bool GetProperty(const Slice& property, std::string* value) = 0;
```

leveldb 实现在内部做了一些统计, 可以通过这个接口进行查询. 不过目前可查询状态不多, 具体如下:
- "leveldb.num-files-at-level<N>" - 返回 level <N> 的文件个数, 其中 <N> 是一个 ASCII 格式的数字.
- "leveldb.stats" - 返回多行字符串, 描述该 DB 内部操作相关的统计数据. 
- "leveldb.sstables" - 返回多行字符串, 描述构成该 DB 的全部 sstable 相关信息. 
- "leveldb.approximate-memory-usage" - 返回被该 DB 使用的内存字节数近似值

### GetAppoximateSizes

```c++
/**
 * 对于 [0, n-1] 中每个 i, 将位于 [range[i].start .. range[i].limit) 
 * 中全部 keys 所占用文件系统空间近似大小存储到 sizes[i] 中. 
 *
 * 注意, 如果数据被压缩过了, 那么返回的 sizes 存储的就是压缩后数据所占用文件系统空间大小. 
 *
 * 返回结果可能不包含最近刚写入的数据所占用空间. 
 * @param range 指定要查询一组 keys 范围
 * @param n range 和 sizes 两个数组的大小
 * @param sizes 存储查询到的每个 range 对应的文件系统空间近似大小
 */
virtual void GetApproximateSizes(const Range* range, int n,
                                  uint64_t* sizes) = 0;
```

计算 range 包含的键区间在磁盘上占用的空间大小, 每个子区间占用会保存到 sizes 对应位置.

计算过程也很简单, 就是遍历 range 列表, 针对每个子区间起止 key, 去数据库中确认其大致字节偏移, 然后"止"-"始" 即为子区间占用空间的大致大小.

### CompactRange

```c++
/**
 * 将 key 范围 [*begin,*end] 对应的底层存储压紧, 注意范围是左闭右闭. 
 *
 * 尤其是, 压实过程会将已经删除或者复写过的数据会被丢弃, 同时会将数据
 * 重新安放以减少后续数据访问操作的成本. 
 * 这个操作是为那些理解底层实现的用户准备的. 
 *
 * 如果 begin==nullptr, 则从第一个键开始; 如果 end==nullptr 则到最后一个键为止. 
 * 所以, 如果像下面这样做则意味着压紧整个数据库: 
 *
 * db->CompactRange(nullptr, nullptr);
 * @param begin 起始键
 * @param end 截止键
 */
virtual void CompactRange(const Slice* begin, const Slice* end) = 0;
```

手动触发与目标键区间重叠的文件压实. 具体为:
- 检查每个 level, 确认其包含的键区间释放与目标键区间有交集.
- 因为当前在写 memtable 可能与目标键区间有交集, 所以强制触发一次 memtable 压实(即将当前 memtable 文件转为 sorted table 文件并写入磁盘)并生成新 log 文件和对应的 memtable.
- 针对与目标键区间有交集的各个 level 触发一次手动压实

具体压实过程后续会写一篇文章进行介绍.

## leveldb 的文件类型

下面分别介绍 leveldb 最重要的几个文件类型.

### log 文件

一个 log 文件(*.log)保存着最近一系列更新操作, 它相当于 leveldb 的 WAL(write-ahead log). 每个更新操作都被追加到当前的 log 文件中. 当 log 文件大小达到一个预定义的大小时(默认大约 4MB), 这个 log 文件就会被转换为一个 sorted table (见下文)然后一个新的 log 文件就会被创建以保存未来的更新操作. 

当前 log 文件内容同时也会被记录到一个内存数据结构中(即 `memtable` ). 这个结构加上全部 sorted tables (*.ldb) 才是完整数据, 一起确保每个读操作都能查到当前最新. 

#### log 文件格式

log 文件内容是一系列 blocks, 每个 block 大小为 32KB. 唯一的例外就是, log 文件末尾可能包含一个不完整的 block. 

每个 block 由一系列 records 构成, 具体定义如下(熟悉编译原理的应该对下述写法不陌生): 

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

如果一个 block 剩余字节不超过 6 个(checksum 字段长度 + length 字段长度 + type 字段长度 = 7), 则不会再构造任何 record, 如前括号解释因为大小不合适. 这些剩余空间会被用于构造一个 trailer, reader 读取该文件时候会忽略之. 

此外, 如果当前 block 恰好剩余 7 个字节(正好可以容纳 record 中的 checksum + length + type), 并且一个新的非 0 长度的 record 要被写入, 那么 writer 必须在此处写入一个 FIRST 类型的 record(但是 length 字段值为 0, data 字段为空. 用户数据 data 部分需要写入下个 block, 而且下个 block 起始还是要写入一个 header 不过其 type 为 middle)来填满该 block 尾部的 7 个字节, 然后在接下来的 blocks 中写入全部用户数据.

未来可能加入更多的 record 类型. Readers 可以跳过它们不理解的 record 类型, 也可以在跳过时进行报告. 

    FULL == 1
    FIRST == 2
    MIDDLE == 3
    LAST == 4

FULL 类型的 record 包含了一个完整的用户 record 的内容. 

FIRST、MIDDLE、LAST 这三个类型用于被分割成多个 fragments(典型的理由是某个 record 跨越了多个 block 边界) 的用户 record. FIRST 表示某个用户 record 的第一个 fragment, LAST 表示某个用户 record 的最后一个 fragment, MIDDLE 表示某个用户 record 的中间 fragments. 

举例: 考虑下面一系列用户 records: 
    A: 长度 1000
    B: 长度 97270
    C: 长度 8000 
**A** 会被作为 FULL 类型的 record 存储到第一个 block, 第一个 block 剩余空间为 32768 - 7 - 1000 = 31761; 

**B** 会被分割为 3 个 fragments: 第一个 fragment 占据第一个 block 剩余空间, 共存入 31761 - 7 = 31754, 剩余 65516; 第二个 fragment 占据第二个 block 的全部空间, 存入 32768 - 7 = 32761, 剩余 65516 - 32761 = 32755; 第三个 fragment 占据第三个 block 的起始空间共 7 + 32755 = 32762. 所以最后在第三个 block 剩下 32768 - 32762 = 6 个字节, 这几个字节会被填充 0 作为 trailer. 

**C** 将会被作为 FULL 类型的 record 存储到第四个 block 中. 

MANIFEST 文件的格式同 log 文件, 只是记录的具体内容不同, 前者记录的针对 level 架构的文件级别变更(新增/删除), 后者记录的是用户数据 key-value 变更.

#### log 文件格式的好处

log 文件格式的好处是(总结一句话就是容易划分边界): 

1. 不必进行任何启发式地 resyncing(可以理解为寻找一个 block 的边界) —— 直接跳到下个 block 边界进行扫描即可, 因为每个 block 大小是固定的(32768 个字节, 除非文件尾部的 block 未写满). 如果数据有损坏, 直接跳到下个 block. 这个文件格式的附带好处是, 当一个 log 文件的部分内容作为一个 record 嵌入到另一个 log 文件时(即当一个逻辑 record 分为多个物理 records, 一部分 records 位于前一个 log 文件, 剩下 records 位于下个 log 文件), 我们不会分不清楚. 
2. 在估计出来的边界处做分割(比如为 mapreduce 应用)变得简单了: 找到下个 block 的边界, 如果起始是 MIDDLE 或者 LAST 类型的 record, 则跳过直到我们找到一个 FULL 或者 FIRST record 为止, 就可以在此处做分割, 一部分投递到一个计算任务, 另一部分(直到分界处)投递到另一个计算任务.

#### log 文件的缺点(并不是)

log 文件格式的缺点: 

- 1. 没有打包小的 records. 通过增加一个新的 record 类型可以解决这个问题, 所以这个问题是当前实现的不足而不是 log 格式的缺陷. 
- 2. 没有压缩. 同样地, 这个也可以通过增加一个新的 record 类型来解决. 

#### log 文件主要接口

下面介绍下 log 文件的读写实现.

##### 写 log

```c++
leveldb::Status leveldb::log::Writer::AddRecord(const leveldb::Slice &slice)
```

该接口做的事情就是把外部传入的 Slice 封装成若干 records 追加到 log 文件中.

该方法会被 `leveldb::Status leveldb::DBImpl::Write(const leveldb::WriteOptions &options, leveldb::WriteBatch *my_batch)` 调用以响应用户的写操作. `DBImpl` 是 `DB` 的派生类, 其 `Put` 和 `Delete` 方法真正工作是由派生类的 `Write` 负责的.

##### 读 log

```c++
bool leveldb::log::Reader::ReadRecord(leveldb::Slice *record, string *scratch)
```

该方法负责从 log 文件读取内容并反序列化为 Record. 该方法会在 db 的 `Open` 方法中调用, 负责将磁盘上的 log 文件转换为内存中 memtable. 其它数据库恢复场景也会用到该方法.

#### 与 log 文件配套的 memtable

memtable 可以看作是 log 文件的内存形式, 但是格式不同.

##### 结构

它的本质就是一个 SkipList.

##### 用途

我们已经知道, 每个 log 文件在内存有一个对应的 memtable, 它和正在压实的 memtable 以及磁盘上的各个 level 包含的文件构成了数据全集. 所以当调用 DB 的 `Get` 方法查询某个 key 的时候, 具体步骤是这样的(具体实现位于 `leveldb::Status leveldb::Version::Get(const leveldb::ReadOptions &options, const leveldb::LookupKey &k, string *value, leveldb::Version::GetStats *stats)`, DB 的 `Get` 方法会调用前述实现.):
- 1 先查询当前在用的 memtable, 查到返回, 未查到下一步
- 2 查询正在转换为 sorted table 的 memtable 中寻找, 查到返回, 未查到下一步 
- 3 在磁盘上采用从底向上 level-by-level 的寻找目标 key. 
  - 由于 level 越低数据越新, 因此, 当我们在一个较低的 level 找到数据的时候, 不用在更高的 levels 找了.
  - 由于 level-0 文件之间可能存在重叠, 而且针对同一个 key, 后产生的文件数据更新所以先将包含 key 的文件找出来按照文件号从大到小(对应文件从新到老)排序查找 key; 针对 level-1 及其以上 level, 由于每个 level 内文件之间不存在重叠, 于是在每个 level 中直接采用二分查找定位 key.

### sorted table 文件

sorted table(*.ldb) 文件就是 leveldb 的数据库文件了. 每一个 level 都对应一组有序的 sorted table 文件. 每个 sorted table 文件保存着按 key 排序的一系列数据项. 每个数据项要么是一个与某个 key 对应的 value, 要么是某个 key 的删除标记. (删除标记其它地方又叫墓碑消息, 用于声明时间线上在此之前的同名 key 对应的记录都失效了, 后台线程负责对这类记录进行压实, 即拷贝到另一个文件时物理删除这类记录.). 注意, leveldb 是一个 append 类型而非 MySQL 那种 in-place 修改的数据库.

sorted tables 文件被组织成一系列 levels. 一个 log 文件生成的对应 sorted table 文件会被放到一个特殊的 **young** level(也被叫做 level-0). 当 young 文件数目超过某个阈值(当前是 4), 全部 young 文件就会和 level-1 与之重叠的全部文件进行合并, 进而生成一系列新的 level-1 文件(每 2MB 数据就会生成一个新的 level-1 文件). 

level-0 的文件之间可能存在键区间重叠, 但是其它每层 level 内部文件之间是不存在重叠情况的. 我们下面来说下 level-1 及其以上的 level 的文件如何合并. 当 level-L (L >= 1)的文件总大小超过了 $10^L$ MB(即 level-1 超过了 10MB, level-2 超过了 100MB, ...), 此时一个 level-L 文件就会和 level-(L+1) 中与自己键区间重叠的全部文件进行合并, 然后为 level-(L+1) 生成一组新的文件. 这些合并操作可以实现将 young level 中新的 updates 一点一点搬到最高的那层 level, 这个迁移过程使用的都是块读写(最小化了昂贵的 seek 操作的时间消耗). 

#### sorted table 文件格式

leveldb sorted table (又叫 sstable) 文件主要包含五个部分, 即多个 data blocks, 多个 meta blocks, 一个 metaindex block, 一个 index block 以及一个 footer, 具体格式如下: 

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

不像 kafka 存储结构数据文件和索引文件是各自独立的(在查询时索引文件用了根据具体 key 定位是哪个数据文件), 该文件把索引和数据保存到了一个文件中. 每次从文件查询数据时会先查询索引, 索引是指向数据的指针, 具体叫做 BlockHandle, 包含着下述信息: 

    // 对应 block 起始位置在文件中的偏移量
    offset: varint64
    // 对应 block 的大小
    size:   varint64

如果你没用过 protobuf 之类的二进制编解码协议, 可能对 varint64 不太熟悉, 可以参考这里 [varints](https://developers.google.com/protocol-buffers/docs/encoding#varints) 了解一下. 本质就是对数据类型进行(二次)无损编码, 使其更加紧凑, 可以节省带宽或者存储空间.

下面详细解释下上面提到的文件格式: 

- 1. 文件里存的是一系列 key/value 对, 而且按照 key 排过序了, 同时被划分到了多个 blocks 中. 这些 blocks 从文件起始位置开始一个接一个. 每个 data block 组织形式在 `block_builder.cc` 定义, 用户可以选择对 data block 进行压缩. 
- 2. 全部 data blocks 之后是一组 meta blocks. 已经支持的 meta block 类型见下面描述, 将来可能会加入更多的类型. 每个 meta block 组织形式在 `block_builder.cc` 定义, 同样地, 用户可以选择对其进行压缩. 
- 3. 全部 meta blocks 后是一个 metaindex block. 每个 meta block 都有一个对应的 entry 保存在该部分, 其中 key 就是某个 meta block 的名字, value 是一个指向该 meta block 的 BlockHandle. 
- 4. 紧随 metaindex block 之后是一个 index block. 针对每个 data block 都有一个对应的 entry 包含在该部分, 其中 key 为大于等于对应 data block 最后(也是最大的, 因为排序过了)一个 key 同时小于接下来的 data block 第一个 key 的字符串; value 是指向一个对应 data block 的 BlockHandle. 
- 5. 在每个文件的末尾是一个固定长度的 footer, 固定长度的好处就是读取文件时, 用 file size 减去这个固定长度就能定位到 footer 起始偏移, 然后就可以解析了. 它包含了一个指向 metaindex block 的 BlockHandle 和一个指向 index block 的 BlockHandle 以及一个 magic number. 具体格式如下:

          // 指向 metaindex 的 BlockHandle
          metaindex_handle: char[p];     
          // 指向 index 的 BlockHandle
          index_handle:     char[q];     
          // 用于维持固定长度的 padding 0,
          // (其中 40 == 2*BlockHandle::kMaxEncodedLength)
          padding:          char[40-p-q];
          // 具体内容为 0xdb4775248b80fb57 (小端字节序)
          magic:            fixed64;     

#### "filter" Meta Block

如果打开(创建)数据库的时候指定了一个 `FilterPolicy`, 那么一个 filter block 就会被存储到每个 sstable 中. metaindex block 包含了一个 entry, 它是从 `filter.<Name>` 到 filter block 的 BlockHandle 的映射. 其中, `<Name>` 是一个由 filter policy 的 `Name()`方法返回的字符串. 

filter block 保存着一系列 filters, 其中 filter i 包含了方法

```c++
void leveldb::FilterPolicy::CreateFilter(const Slice *keys, int n, string *dst) const
``` 

针对入参 keys 的计算结果(存储在输出型参数 `dst`). 参数 keys 属于一个 data block, 该 data block 对应的文件偏移量落在下面的范围里: 

    [ i*base ... (i+1)*base-1 ]

当前, 上面的 base 是 2KB. 举个例子, 如果 block X 和 block Y 起始地址落在 `[ 0KB .. 2KB ]` 范围内, 则 X 和 Y 中的全部 keys 将会在调用 `FilterPolicy::CreateFilter()` 时被转换为一个 filter, 然后这个 filter 会作为一个(为啥是第一个, 因为 X、Y 起始地址落在第一个地址空间 `[ 0KB .. 2KB ]` 里) filter 被保存在 filter block 中. (用大白话再说一遍, 每个 FilterPolicy 都有一个唯一的名字, 在 metaindex block 通过这个名字就能找到对应的 filter block 了. 而 filter block 存的就是用这个 FilterPolicy 构造的一系列 filters, 为啥是一系列呢？因为 data blocks 太多了, 所以分了区间, 每几个 data blocks 对应一个 filter, 具体几个根据上面那个带 base 的公式来算. 再说说 filter 是怎么回事. data block 保存的不是键值对构成的 records 嘛, 根据前面说的键区间限制, 把每几个 blocks 的全部键根据某个 FilterPolicy 算一下就得到了一个 filter, 然后把这个 filter 保存到了 filter block 的第 i 个位置. )

具有 N 个 filter 的 filter block 格式如下: 

    [filter 0]
    [filter 1]
    [filter 2]
    ...
    [filter N-1]
    // 下面的 [offset of filter X] 布局其实不太准确, 准确地
    // 说, 除了 [offset of filter 0], 其它的都可能多出现一次.
    // 具体原因见 leveldb::FilterBlockBuilder::GenerateFilter() 的疑问.
    [offset of filter 0]                  : 4 bytes
    [offset of filter 1]                  : 4 bytes
    [offset of filter 2]                  : 4 bytes
    ...
    [offset of filter N-1]                : 4 bytes
    // 上面 [offset of filter 0] 相对于 filter block 首地址的相对偏移量, 
    // 基于该值可以将 fiter 部分和 filter offset 部分分辨出来.
    [offset of beginning of offset array] : 4 bytes
    // base 以 2 为底的对数, 目前 base 是 2048, 则这里就是 11
    lg(base)                              : 1 byte

其中, 位于 filter block 尾部的 offset 数组可以使得我们快速定位到某个 filter. 

由 `leveldb::Slice leveldb::FilterBlockBuilder::Finish()` 负责构造上述格式, 然后由 `leveldb::FilterBlockReader::FilterBlockReader()` 构造方法负责解析上述格式.

#### "stats" Meta Block

下面的 meta block 保存着一组统计信息. key 是统计量的名称, value 是具体的统计值. 

    data size
    index size
    key size (uncompressed)
    value size (uncompressed)
    number of entries
    number of data blocks

遗憾的是, 目前这个还没实现, 🤭.


#### sorted table 文件主要接口

下面说明一下 sorted table 文件主要的操作接口, 主要是读与写.

##### sorted table 文件读接口

完成该工作的是 `class leveldb::Table`, 该类是对 sorted table 文件的抽象, 负责对 sorted table 文件进行读操作.

具体底层存储由 helper 类 `struct leveldb::Table::Rep` 负责.

打开一个 sstable 文件的入口为 `leveldb::Status leveldb::Table::Open`, 该方法依次动作为:
- 1 读取文件末尾固定长度的 footer (具体长度为两个 BlockHandle 最大长度 + 固定的 8 字节魔数)
- 2 解析 footer 从而得到 index block 偏移量和大小以及 meta block 偏移量和大小.
- 3 将解析出来的 index block 放到 Table 对象中, 通过输出型参数返回.

从外部(这里强调外部, 是因为还有一个私有方法 `leveldb::Status leveldb::Table::InternalGet` 可以访问 Table 对象内容)打开文件后如果要访问其内容, 需要一个迭代器, 该工作通过 `leveldb::Iterator *leveldb::Table::NewIterator` 完成. 返回的迭代器为一个 `leveldb::<unnamed>::TwoLevelIterator`, 该迭代器处于匿名的命名空间所以未直接对外暴露, 仅能通过返回的指针访问其从 `class leveldb::Iterator` 继承的方法. 该迭代器设计精巧, 会单独写文章介绍.

##### sorted table 文件写接口

完成该工作的是 `class leveldb::TableBuilder`, 该类负责构造 sstable 文件构造.

主要方法有以下两个:
- `void BlockBuilder::Add(const Slice& key, const Slice& value)` 负责向 TableBuilder 对象添加 (key, value), 该工作主要由 `class leveldb::BlockBuilder::Add()` 方法完成.
- `void leveldb::TableBuilder::Finish()` 负责将整个 Table 序列化为一个 sstable 文件并写入磁盘, 具体写入顺序为:
  - 写 data blocks
  - 写 meta blocks(目前仅有过滤器)
  - 写 meta-index block
  - 写 data-index block
  - 写 footer

### MANIFEST 文件

MANIFEST 文件可以看作 leveldb 存储元数据的地方. 它列出了每一个 level 及其包含的全部 sorted table 文件, 每个 sorted table 文件对应的键区间, 以及其它重要的元数据. 每当重新打开数据库的时候, 就会创建一个新的 MANIFEST 文件(文件名中嵌有一个新生成的数字). MANIFEST 文件被格式化成形同 log 文件的格式, 针对它所服务的数据的变更都会被追加到该文件后面. 比如每当某个 level 发生文件新增或者删除操作时, 就会有一条日志被追加到 MANIFEST 中. 

MANIFEST 文件在实现时又叫 descriptor 文件, 文件格式同 log 文件, 所以写入/读取方法就复用了. 其每条日志就是一个序列化后的 `leveldb::VersionEdit`. 每次针对 level 架构有文件增删时都要写日志到 manifest 文件.

#### 与 MANIFEST 相关的数据结构之 VersionSet

每个 db 都有一个 `class leveldb::VersionSet` 实例, 它保存了 db 当前的 level 架构视图(具体存储结构为其 Version 成员). MANIFEST 文件可以看作是它所维护的信息的反映. 它的重要方法有:
- `VersionSet::Recover` 负责在打开数据库时将 MANIFEST 文件反序列化构造 level 架构视图, 这个过程会依赖 VersionEdit 类.
- `VersionSet::LogAndApply` 负责将当前 VersionEdit 和当前 Version 进行合并, 然后序列化为一条日志记录到 MANIFEST 文件. 最后把新的 version 替换当前 version.

#### 与 MANIFEST 相关的 Version 结构

`class leveldb::Version` 是 leveldb 数据库 level 架构的内存表示, 它存储了每一个 level 及其全部的文件信息(文件名, 键范围等等). 每次调用 db 的 Get 方法在 memtable 找不到目标 key 时就会到各个 level 的文件去搜寻, 这个搜寻过程所依赖的就是数据库 VersionSet(下面介绍) 保存的当前 Version 存储的 level 架构信息进行的, 具体实现见 `leveldb::Version::Get` 方法.

当条件满足时, VersionSet 会将当前 Version 和当前 VesionEdit 合并生成一个新的 Version 替换当前 Version.


#### 与 MANIFEST 相关的 VersionEdit

MANIFEST 文件的每一条日志就是一个序列化的 `class leveldb::VersionEdit`. 它可以看作一个 on-fly 的 Version. 它会记录 db 运行过程中删除的文件列表和新增的文件列表.

### CURRENT 文件

CURRENT 文件是一个简单的文本文件. 由于每次重新打开数据库都会生成一个 MANIFEST 文件, 所以需要一个地方记录最新的 MANIFEST 文件是哪个, CURRENT 就干这个事情, 它相当于一个指针, 其内容即是当前最新的 MANIFEST 文件的名称. 

### 文件位置与命名

各类型文件位置与命名如下:

    dbname/CURRENT
    dbname/LOCK
    dbname/LOG
    dbname/LOG.old
    dbname/MANIFEST-[0-9]+
    dbname/[0-9]+.(log|sst|ldb)

其中 dbname 为用户指定.

--End--