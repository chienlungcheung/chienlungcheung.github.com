---
title: "leveldb 原理详解系列之零: 基本介绍与使用举例"
date: 2020-09-11 21:13:08
tags: 
  - leveldb
  - LSM-Tree
  - db
---

<!-- toc -->

**Leveldb** 是一个高速 KV 数据库, 它提供了一个持久性的 KV 存储. 其中 keys 和 values 都是随机字节数组, 并且存储时根据用户指定的比较函数对 keys 进行排序. 

它由 Google 开发的, 其作者为大名鼎鼎的 Sanjay Ghemawat (sanjay@google.com) 和 Jeff Dean (jeff@google.com). 感谢他们对人类的贡献.

# 基本介绍

该部分主要介绍 leveldb 的功能, 局限性以及性能等.

## 特性

  * keys 和 values 都可以是随机的字节数组. 
  * 数据被按照 key 的顺序进行存储. 
  * 调用者可以提供一个定制的比较函数来覆盖默认的比较器. 
  * 基础操作有 `Put(key,value)`, `Get(key)`, `Delete(key)`. 
  * 多个更改可以在一个原子批处理中一起生效. 
  * 用户可以创建一个瞬时快照来获取数据的一致性视图. 
  * 支持针对数据的前向和后向遍历. 
  * 数据通过 [Snappy](http://google.github.io/snappy/) 压缩程序库自动压缩. 
  * 与外部交互的操作都被抽象成了接口(如文件系统操作等), 所以用户可以根据接口定制自己期望的操作系统交互行为. 

## 局限性

  * LevelDB 不是 SQL 数据库. 它没有关系数据模型, 不支持 SQL 查询, 也不支持索引. 
  * 同时只能有一个进程(可能是具有多线程的进程)访问一个特定的数据库. 
  * 该程序库没有内置基于网络的 CS 架构, 有需求的用户可以自己封装. 

## 性能

下面是通过运行 db_bench 程序得出的性能测试报告. 

### 测试配置

我们使用的是一个包含一百万数据项的数据库, 
其中 key 是 16 字节, value 是 100 字节, value 压缩后大约是原来的一半, 测试配置如下:

```bash
    LevelDB:    version 1.1
    Date:       Sun May  1 12:11:26 2011
    CPU:        4 x Intel(R) Core(TM)2 Quad CPU    Q6600  @ 2.40GHz
    CPUCache:   4096 KB
    Keys:       16 bytes each
    Values:     100 bytes each (50 bytes after compression)
    Entries:    1000000
    Raw Size:   110.6 MB (estimated)
    File Size:  62.9 MB (estimated)
```

### 写性能

"fill" 基准测试创建了一个全新的数据库, 以顺序(下面 "seq" 结尾者)或者随机(下面 "random" 结尾者)方式写入. "fillsync" 基准测试每次写操作都将数据从操作系统刷到磁盘; 其它的操作会将数据保存在系统中一段时间. "overwrite" 基准测试做随机写, 这些操作会更新数据库中已有的键. 

```bash
    fillseq      :       1.765 micros/op;   62.7 MB/s
    fillsync     :     268.409 micros/op;    0.4 MB/s (10000 ops)
    fillrandom   :       2.460 micros/op;   45.0 MB/s
    overwrite    :       2.380 micros/op;   46.5 MB/s
```
上述每个 "op" 对应一个 key/value 对的写操作. 也就是说, 一个随机写基准测试每秒大约进行四十万次写操作(1,000,000/2.46). 

每个 "fillsync" 操作时间消耗(大约 0.3 毫秒)少于一次磁盘寻道(大约 10 毫秒). 我们怀疑这是因为磁盘本身将更新操作缓存到了内存, 并且在数据真正落盘前返回响应. 该方式是否安全取决于断电后磁盘是否有备用电力将数据落盘. 

### 读性能

我们分别给出正向顺序读、反向顺序读的性能以及随机查询的性能指标. 注意, 基准测试创建的数据库很小. 因此该性能报告描述的是 leveldb 的全部数据集能放入到内存的场景. 如果数据不在操作系统缓存中, 读取一点数据的性能消耗主要在于一到两次的磁盘寻道. 写性能基本不会受数据集是否能放入内存的影响. 

```bash
    readrandom  : 16.677 micros/op;  (approximately 60,000 reads per second)
    readseq     :  0.476 micros/op;  232.3 MB/s
    readreverse :  0.724 micros/op;  152.9 MB/s
```
LevelDB 会在后台压实底层的数据来改善读性能. 上面列出的结果是在经过一系列随机写操作后得出的. 如果经过压实(通常是自动触发), 那么上述指标会更好. 

```bash
    readrandom  : 11.602 micros/op;  (approximately 85,000 reads per second)
    readseq     :  0.423 micros/op;  261.8 MB/s
    readreverse :  0.663 micros/op;  166.9 MB/s
```

读操作消耗高的地方有一些来自重复解压从磁盘读取的数据块. 如果我们能提供足够的缓存给 leveldb 来将
解压后的数据保存在内存中, 读性能会进一步改善: 

```bash   
    readrandom  : 9.775 micros/op;  (approximately 100,000 reads per second before compaction)
    readrandom  : 5.215 micros/op;  (approximately 190,000 reads per second after compaction) 
```

# 使用举例

下面从构建和头文件介绍开始, 对 leveldb 的基本使用进行介绍.

## 构建

该工程开箱支持 CMake. 

所以构建起来超简单: 

```bash
$ mkdir -p build && cd build
$ cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
$ sudo make install
```

更多高级用法请请参照 CMake 文档和本项目的 CMakeLists.txt. 

## 头文件介绍

LevelDB 对外的接口都包含在 include/*.h 中. 除了该目录下的文件, 用户不应该依赖其它目录下任何文件. 

* **include/db.h**: 主要的接口在这, 使用 leveldb 从这里开始. 

* **include/options.h**: 使用 leveldb 过程各种操作包括读写有关的控制参数. 

* **include/comparator.h**: 比较函数的抽象, 如果你想用逐字节比较 key 那么可以直接使用默认的比较器. 如果你想定制排序逻辑(如处理不同的字符编解码等)可以定制自己的比较函数. 

* **include/iterator.h**: 迭代数据的接口. 你可以从一个 DB 对象获取到一个迭代器. 

* **include/write_batch.h**: 原子地应用多个更新到一个数据库. 

* **include/slice.h**: 类似 string, 维护着指向字节数组的一个指针和相应长度. 

* **include/status.h**: 许多公共接口都会返回 `Status`, 用于指示成功或其它各种错误. 

* **include/env.h**: 操作系统环境的抽象. 在 `util/env_posix.cc` 中有一个该接口的 posix 实现. 

* **include/table.h, include/table_builder.h**: 底层的模块, 大多数客户端可能不会直接用到. 


## 打开(或新建)一个数据库

leveldb 数据库都有一个名字, 该名字对应了文件系统上一个目录, 而且该数据库内容全都存在该目录下. 下面的例子显示了如何打开一个数据库以及在必要情况下创建之. 

```c++
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());
...
```

如果你想在数据库已存在的时候触发一个异常, 将下面这行加到 `leveldb::DB::Open` 调用之前: 

```c++
options.error_if_exists = true;
```

## Status 类型

你可能注意到上面的 `leveldb::Status` 类型了. leveldb 中大部分方法在遇到错误的时候会返回该类型的值. 你可以检查它是否为 ok, 然后打印关联的错误信息即可: 

```c++
leveldb::Status s = ...;
if (!s.ok()) cerr << s.ToString() << endl;
```

## 关闭数据库

当数据库不再使用的时候, 像下面这样直接删除数据库对象就可以了: 

```c++
... open the db as described above ...
... do something with db ...
delete db;
```

非常简单是不是? 因为 `DB` 类的实现是基于 RAII 的, 在 delete 时触发析构方法自动进行清理工作.

## 数据库读写

数据库提供了 `Put`、`Delete` 以及 `Get` 方法来修改、查询数据库. 下面的代码展示了将 key1 对应的 value 移动(先拷贝后删除)到 key2 下. 

```c++
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);
```

## 原子更新

注意, 上一小节中如果进程在 Put 了 key2 之后但是删除 key1 之前挂了, 那么同样的 value 就出现在了多个 keys 之下. 该问题可以通过使用 `WriteBatch` 类原子地应用一组操作来避免. 

```c++
#include "leveldb/write_batch.h"
...
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) {
  leveldb::WriteBatch batch;
  batch.Delete(key1);
  batch.Put(key2, value);
  s = db->Write(leveldb::WriteOptions(), &batch);
}
```

`WriteBatch` 保存着一系列将被应用到数据库的编辑操作, 这些操作会按照添加的顺序依次被执行. 注意, 我们先执行 Delete 后执行 Put, 这样如果 key1 和 key2 一样的情况下我们也不会错误地丢失数据. 

除了原子性, `WriteBatch` 也能加速更新过程, 因为可以把一大批独立的操作添加到同一个 batch 中然后一次性执行. 

## 同步写操作

默认地, leveldb 每个写操作都是异步的: 进程把要写的内容 push 给操作系统后立马返回. 从操作系统内存到底层持久性存储的迁移异步地发生. 当然, 也可以把某个写操作的 sync 标识打开, 以等到数据真正被记录到持久化存储再让写操作返回. (在 Posix 系统上, 这是通过在写操作返回前调用 `fsync(...)` 或 `fdatasync(...)` 或 `msync(..., MS_SYNC)` 来实现的. )

```c++
leveldb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
```

异步写通常比同步写快 1000 倍. 异步写的缺点是, 一旦机器崩溃可能会导致最后几个更新操作丢失. 注意, 仅仅是写进程崩溃(而非机器重启)则不会引起任何更新操作丢失, 因为哪怕 sync 标识为 false, 在进程退出之前写操作也已经从进程内存 push 到了操作系统. 

异步写总是可以安全使用. 比如你要将大量的数据写入数据库, 如果丢失了最后几个更新操作, 你可以重启整个写过程. 如果数据量非常大, 一个优化点是, 每进行 N 个异步写操作则进行一次同步地写操作, 如果期间发生了崩溃, 重启自从上一个成功的同步写操作以来的更新操作即可. (同步的写操作可以同时更新一个标识, 该标识用于描述崩溃重启后从何处开始重启更新操作. )

`WriteBatch` 可以作为异步写操作的替代品. 多个更新操作可以放到同一个 WriteBatch 中然后通过一次同步写(即 `write_options.sync` 置为 true)一起应用. 

## 并发

一个数据库同时只能被一个进程打开. LevelDB 会从操作系统获取一把锁来防止多进程同时打开同一个数据库. 在单个进程中, 同一个 `leveldb::DB` 对象可以被多个并发的线程安全地使用, 也就是说, 不同的线程可以写入或者获取迭代器, 或者针对同一个数据库调用 `Get`, 前述全部操作均不需要借助外部同步设施(leveldb 实现会自动地确保必要的同步). 但是其它对象, 比如 `Iterator` 或者 `WriteBatch` 需要外部自己提供同步保证. 如果两个线程共享此类对象, 需要使用锁进行互斥访问. 具体见对应的头文件. 

## 迭代数据库

下面的用例展示了如何打印数据库中全部的 (key, value) 对. 

```c++
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": "  << it->value().ToString() << endl;
}
assert(it->status().ok());  // Check for any errors found during the scan
delete it;
```

下面的用例展示了如何打印 `[start, limit)` 范围内数据:

```c++
for (it->Seek(start);
   it->Valid() && it->key().ToString() < limit;
   it->Next()) {
  ...
}
```

当然你也可以反向遍历(注意, 反向遍历可能比正向遍历要慢一些, 具体见前面的读性能基准测试). 

```c++
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  ...
}
```

## 快照

快照提供了针对整个 KV 存储的一致性只读视图. `ReadOptions::snapshot` 不为空表示读操作应该作用在 DB 的某个特定版本上; 若为空, 则读操作将会作用在当前版本的一个隐式的快照上.  

快照通过调用 `DB::GetSnapshot()` 方法创建:  

```c++
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
... apply some updates to db ...
// 获取与前面快照对应的数据库迭代器
leveldb::Iterator* iter = db->NewIterator(options);
... read using iter to view the state when the snapshot was created ...
delete iter;
db->ReleaseSnapshot(options.snapshot);
```

注意, 当一个快照不再使用的时候, 应该通过 `DB::ReleaseSnapshot` 接口进行释放. 

## Slice 切片

`it->key()` 和 `it->value()` 调用返回的值是 `leveldb::Slice` 类型的实例. 熟悉 Golang 或者 Rust 的同学对 slice 应该不陌生. slice 是一个简单的数据结构, 包含一个长度和一个指向外部字节数组的指针. 返回一个切片比返回 `std::string` 更加高效, 因为不需要隐式地拷贝大量的 keys 和 values. 另外, leveldb 方法不返回空字符结尾的 C 风格地字符串, 因为 leveldb 的 keys 和 values 允许包含 `\0` 字节. 

C++ 风格的 string 和 C 风格的空字符结尾的字符串很容易转换为一个切片: 

```c++
leveldb::Slice s1 = "hello";

std::string str("world");
leveldb::Slice s2 = str;
```

一个切片也很容易转换回 C++ 风格的字符串: 

```c++
std::string str = s1.ToString();
assert(str == std::string("hello"));
```

注意, 当使用切片时, 调用者要确保它内部指针指向的外部字节数组保持存活. 比如, 下面的代码就有问题: 

```c++
leveldb::Slice slice;
if (...) {
  std::string str = ...;
  slice = str;
}
Use(slice);
```

当 if 语句结束的时候, str 将会被销毁, 切片的底层存储也随之消失了, 后面再用就出问题了. 

## 比较器

前面的例子中用的都是默认的比较函数, 即逐字节按照字典序比较. 你可以定制自己的比较函数, 然后在打开数据库的时候传入. 只需继承 `leveldb::Comparator` 然后定义相关逻辑即可, 下面是一个例子: 

```c++
class TwoPartComparator : public leveldb::Comparator {
 public:
  // Three-way comparison function:
  //   if a < b: negative result
  //   if a > b: positive result
  //   else: zero result
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2);
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
  }

  // Ignore the following methods for now:
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const leveldb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};
```

然后使用上面定义的比较器打开数据库: 

```c++
// 实例化比较器
TwoPartComparator cmp;
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
// 将比较器赋值给 options.comparator
options.comparator = &cmp;
// 打开数据库
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
...
```

### 后向兼容性

比较器 `Name` 方法返回的结果在创建数据库时会被绑定到数据库上, 后续每次打开都会进行检查. 如果名称改了, 对 `leveldb::DB::Open` 的调用就会失败. 因此, 当且仅当在新的 key 格式和比较函数与已有的数据库不兼容而且已有数据不再被需要的时候再修改比较器名称. 总而言之, 一个数据库只能对应一个比较器, 而且比较器由名字唯一确定, 一旦修改名称或者比较器逻辑, 数据库的操作逻辑就统统会出错, 毕竟 leveldb 是一个有序的 KV 存储.

如果非要修改比较逻辑怎么办呢? 你可以根据预先规划一点一点的演进你的 key 格式, 注意, 事先的演进规划非常重要. 比如, 你可以存储一个版本号在每个 key 的结尾(大多数场景, 一个字节足够了). 当你想要切换到新的 key 格式的时候(比如新增 third-part 到上面例子 `TwoPartComparator` 处理的 keys 中), 那么你需要做的是:
- (a) 保持比较器名称不变
- (b) 递增新 keys 的版本号
- (c) 修改比较器函数以让其使用版本号来决定如何进行排序. 

## 性能调优

通过修改 `include/leveldb/options.h` 中定义的类型的默认值来对 leveldb 的性能进行调优. 

### Block 大小

Leveldb 把相邻的 keys 组织在同一个 block 中(具体见后面文章针对 sorted table 文件格式的描述), 而且 block 是 leveldb 把数据从内存到转移到持久化存储和从持久化存储转移到内存的基本单位. 默认的, 压缩前 block 大约为 4KB. 经常处理大块数据的应用可能希望把这个值调大, 而针对数据做"点读" 的应用可能希望这个值小一点, 这样性能可能会更高一些. 但是, 没有证据表明该值小于 1KB 或者大于几个 MB 的时候性能会表现更好. 同时要注意, 针对大的 block size, 压缩效率会更高一些. 

### 压缩

每个 block 在写入持久存储之前都会被单独压缩. 压缩默认是开启的, 因为默认的压缩算法非常快, 而且对于不可压缩的数据会自动关闭压缩功能. 极少有场景会让用户想要完全关闭压缩功能, 除非基准测试显示关闭压缩会显著改善性能. 按照下面方式做就关闭了压缩功能: 

```c++
leveldb::Options options;
options.compression = leveldb::kNoCompression;
... leveldb::DB::Open(options, name, ...) ....
```

### 缓存

数据库的内容存储在文件系统的一组文件里, 每个文件保存着一系列压缩后的 blocks. 如果 `options.block_cache` 不为空, 它就会被用于缓存频繁被使用的 block 内容(已解压缩). 

```c++
#include "leveldb/cache.h"

leveldb::Options options;
// 打开数据库之前分配一个 100MB 的 LRU Cache 用于缓存解压的 blocks
options.block_cache = leveldb::NewLRUCache(100 * 1048576);  // 100MB cache
leveldb::DB* db;
// 打开数据库
leveldb::DB::Open(options, name, &db);
... use the db ...
delete db
delete options.block_cache;
```

注意 cache 保存的是未压缩的数据, 因此应该根据应用程序所需的数据大小来设置它的大小. (已压缩数据的缓存工作交给操作系统的 buffer cache 或者用户提供的定制的 Env 实现去干. )

当执行一个大块数据读操作时, 应用程序可能想要取消缓存功能, 这样读进来的大块数据就不会导致 cache 中当前大部分数据被置换出去, 我们可以为它提供一个单独的 iterator 来达到该目的: 

```c++
leveldb::ReadOptions options;
// 缓存设置为关闭
options.fill_cache = false;
// 用该设置去创建一个新的迭代器
leveldb::Iterator* it = db->NewIterator(options);
// 用该迭代器去处理大块数据
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  ...
}
```

### Key 的布局设计

注意, 磁盘传输的单位以及磁盘缓存的单位都是一个 block. 相邻的 keys(已排序)总是在同一个 block 中. 因此应用程序可以通过把需要一起访问的 keys 放在一起, 同时把不经常使用的 keys 放到一个独立的键空间区域来提升性能. 

举个例子, 假设我们正基于 leveldb 实现一个简单的文件系统. 我们打算存储到这个文件系统的数据项类型如下: 

```bash
    filename -> permission-bits, length, list of file_block_ids
    file_block_id -> data
```

我们可以给上面表示 `filename` 的 key 增加一个字符前缀, 比如 '/', 然后给表示 `file_block_id` 的 key 增加另一个不同的前缀, 比如 '0', 这样这些不同用途的 key 就具有了各自独立的键空间区域, 扫描元数据的时候我们就不用读取和缓存大块文件内容数据了. 

### 过滤器

鉴于 leveldb 数据在磁盘上的组织形式, 一次 `Get()` 调用可能涉及多次磁盘读操作. 可配置的 FilterPolicy 机制可以用来大幅减少磁盘读次数. 

```c++
leveldb::Options options;
// 设置启用基于布隆过滤器的过滤策略
options.filter_policy = NewBloomFilterPolicy(10);
leveldb::DB* db;
// 用该设置打开数据库
leveldb::DB::Open(options, "/tmp/testdb", &db);
... use the database ...
delete db;
delete options.filter_policy;
```

上述代码将一个基于布隆过滤器的过滤策略与数据库进行了关联. 基于布隆过滤器的过滤方式依赖于如下事实, 在内存中保存每个 key 的部分位(在上面例子中是 10 位, 因为我们传给 `NewBloomFilterPolicy` 的参数是 10). 这个过滤器将会使得 `Get()` 调用中非必须的磁盘读操作大约减少 100 倍. 每个 key 用于过滤器的位数增加将会进一步减少读磁盘次数, 当然也会占用更多内存空间. 我们推荐数据集无法全部放入内存同时又存在大量随机读的应用设置一个过滤器策略. 

如果你在使用定制的比较器, 你应该确保你在用的过滤器策略与你的比较器兼容. 举个例子, 如果一个比较器在比较 key 的时候忽略结尾的空格, 那么`NewBloomFilterPolicy` 一定不能与此比较器共存. 相反, 应用应该提供一个定制的过滤器策略, 而且它也应该忽略键的尾部空格. 示例如下: 

```c++
class CustomFilterPolicy : public leveldb::FilterPolicy {
 private:
  FilterPolicy* builtin_policy_;

 public:
  CustomFilterPolicy() : builtin_policy_(NewBloomFilterPolicy(10)) {}
  ~CustomFilterPolicy() { delete builtin_policy_; }

  const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

  void CreateFilter(const Slice* keys, int n, std::string* dst) const {
    // Use builtin bloom filter code after removing trailing spaces
    // 将尾部空格移除后再使用内置的布隆过滤器
    std::vector<Slice> trimmed(n);
    for (int i = 0; i < n; i++) {
      trimmed[i] = RemoveTrailingSpaces(keys[i]);
    }
    return builtin_policy_->CreateFilter(&trimmed[i], n, dst);
  }
};
```

当然也可以自己提供非基于布隆过滤器的过滤器策略, 具体见 `leveldb/filter_policy.h`. 

## 校验和

Leveldb 将一个校验和与它存储在文件系统中的全部数据进行关联. 根据激进程度有两种方式控制校验和的核对: 

`ReadOptions::verify_checksums` 可以设置为 true 来强制核对从文件系统读取的全部数据的进行校验和检查. 默认为 false. 

`Options::paranoid_checks` 在数据库打开之前设置为 true 可以使得数据库一旦检测到数据损毁即报错. 取决于数据库损坏部位, 报错时机可能是打开数据库后的时候, 也可能是在后续执行某个操作的时候. 该配置默认是关闭状态, 这样即使持久性存储部分虽坏数据库也能继续使用. 

如果数据库损坏了(当开启 `Options::paranoid_checks` 的时候可能就打不开了), `leveldb::RepairDB` 函数可以用于对尽可能多的数据进行修复. 

## 近似空间大小

`GetApproximateSizes` 方法用于获取一个或多个键区间占据的文件系统近似大小(单位, 字节). 

```c++
leveldb::Range ranges[2];
ranges[0] = leveldb::Range("a", "c");
ranges[1] = leveldb::Range("x", "z");
uint64_t sizes[2];
leveldb::Status s = db->GetApproximateSizes(ranges, 2, sizes);
```

上述代码结果是, `size[0]` 保存 `[a..c)` 键区间对应的文件系统大致字节数, `size[1]` 保存 `[x..z)` 键区间对应的文件系统大致字节数. 

## 环境变量

由 leveldb 发起的全部文件操作以及其它的操作系统调用最后都会被路由给一个 `leveldb::Env` 对象. 用户也可以提供自己的 `Env` 实现以达到更好的控制. 比如, 如果应用程序想要针对 leveldb 的文件 IO 引入一个人工延迟以限制 leveldb 对同一个系统中其它应用的影响:

```c++
// 定制自己的 Env 
class SlowEnv : public leveldb::Env {
  ... implementation of the Env interface ...
};

SlowEnv env;
leveldb::Options options;
// 用定制的 Env 打开数据库
options.env = &env;
Status s = leveldb::DB::Open(options, ...);
```

## 移植性

如果某个特定平台提供 `leveldb/port/port.h` 导出的类型/方法/函数实现, 那么 leveldb 可以被移植到该平台上, 更多细节见 `leveldb/port/port_example.h`. 

另外, 新平台可能还需要一个新的默认的 `leveldb::Env` 实现. 具体可参考 `leveldb/util/env_posix.h` 实现. 

--End--