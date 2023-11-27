# Leveldb 源码详解系列之六: 文件缓存设计与实现


上一篇讲了 leveldb 中 `Table` 的设计和实现, 它是磁盘 sstable 文件的内存形式, 但是 `Table` 在实际中不会被用户直接用到, 而是借助 `TableCache`.
<!--more-->

## 1 TableCache: leveldb 的磁盘文件缓存结构  

(代码位于 `db/table_cache.h`)

磁盘上每个 sstable 文件都对应一个 `Table` 实例, `TableCache` 是一个用于缓存这些 Table 实例的缓存. 或者这么说, sstable 文件被加载到内存后, 被缓存到 `TableCache` 中.

每次用户进行查询操作的时候(即调用`DBImpl::Get()`)可能需要去查询磁盘上的文件(即未在 memtable 中查到), 这就要求有个缓存功能来加速. `TableCache` 会缓存 sstable 文件对应的 `Table` 实例, 用于加速用户的查询, 否则每次读文件解析就很慢了. 

目前在用的缓存策略是 LRU 以防内存占用过大. 

每个 db 实例都会持有一个 `TableCache` 实例, 对该缓存的的填充是通过副作用实现的, 即当外部调用 `DBImpl::Get()->Version::Get()->VersionSet::table_cache_::Get()` 进行查询的时候, 如果发现 sstable 对应 `Table` 实例不在缓存就会将其填充进来.
### 1.1 概览

下面是 `TableCache` 类构成: 

```c++
class TableCache {
 public:
  // 为 file_number 标识的 sstable 文件构造对应的迭代器.
  Iterator* NewIterator(const ReadOptions& options,
                        uint64_t file_number,
                        uint64_t file_size,
                        Table** tableptr = nullptr); 
  // 从缓存中查找 internal_key 为 k 的数据项.
  Status Get(const ReadOptions& options,
      uint64_t file_number,
      uint64_t file_size,
      const Slice& k,
      void* arg,
      void (*handle_result)(void*, const Slice&, const Slice&));
  // 驱逐 file_number 对应的 table 对象
  void Evict(uint64_t file_number);

 private:
  // 一个基于特定淘汰算法(如 LRU)的 Cache
  Cache* cache_;
};
```

### 1.2 数据成员

`TableCache` 的核心数据成员就是 `Cache`(实际使用的是 `ShardedLRUCache`, 这里不展开, 后文详述).

### 1.3 方法成员

`TableCache` 的核心方法有三个.

#### 1.3.1 查询

`Get()` 方法负责实现从缓存中查找指定 key 的数据项. 

若 key 对应的 sstable 文件不在缓存则会根据 file_number 读取文件生成 `Table` 实例放到缓存中然后再查询(注意这是个副作用), 查到后调用 handle_result 进行处理. 

针对该方法的调用链为: `DBImpl::Get()->Version::Get()->VersionSet::table_cache_::Get()`.

#### 1.3.2 遍历

遍历就要有迭代器, `NewIterator()` 方法负责生成指定 sstable 文件内容对应的迭代器. 

该方法主要用于在 `Version::AddIterators()` 遍历 level 架构中每一个 sstable 文件时构造对应的迭代器, 这些迭代器加上 memtable 的迭代器, 就能遍历整个数据库的内容了. 

`Version::AddIterators()` 会从其 `table_cache_` 成员根据 file_number 查找其对应的 `Table` 对象, 若查到则返回其对应迭代器; 否则加载文件(这是一个副作用)并生成对应的 `Table` 对象放到 `table_cache_` 然后返回新构造的 `Table` 的 iterator. 

注意, 如果 `NewIterator()` 方法的 `tableptr` 参数非空, 则设置 `*tableptr` 指向返回的 iterator 底下的 `Table` 对象. 返回的 `*tableptr` 对象由 `TableCache` 所拥有, 所以用户不要删除它; 只要 iterator 还活着, 该对象就有效. 

#### 1.3.3 删除

`Evict()` 方法负责驱逐某个 sstable 文件对应的 `Table` 缓存, 它会在 `DBImpl::DeleteObsoleteFiles()` 删除过期文件时候执行.


这三个方法的具体实现比较简单, 不再详列. 下面重点说一下其最重要的数据成员 -- `Cache`.

#### 1.3.4 修改

嘿, 没有这个接口. 由于 leveldb 是一个 append 类型数据库, 它不会做 inplace 修改, 这同时也避免了解决复杂的数据一致性问题.

## 2 Cache 接口

(代码位于 `include/leveldb/cache.h`)

作为一个缓存定义, 最起码要提供的功能就是增删查, 注意没有改, 缓存只是一个视图, 如果要支持修改可能要引入一系列一致性问题.

该接口的设计还有很重要的一点, 也是 leveldb 设计中很重要的一点, 就是引用计数. `Cache` 中保存的数据项类型为 `Cache::Handle`, 具体实现中存在一个数据成员表示计数, 这样可以根据计数进行内存复用或回收. 这一点贯穿了各个重要方法的实现.


### 2.1 增

```c++
// 插入一对 <key, value> 到 cache 中.
virtual Handle* Insert(const Slice& key, void* value, 
    size_t charge,
    void (*deleter)(const Slice& key, void* value)) = 0;
```

这个方法的参数列表看着有点吓人. 但最重要的就是前两个参数, 后面的参数简单介绍下:
- `charge` 主要用来计算缓存的成本, 也就是内存占用的, 每次插入时候调用方可以将插入的数据字节作为 charge 传进来. 
- `deleter` 是一个用户针对自己插入的数据定制的清理器, 举个简单的例子(具体代码位于 `table_cache.cc`):
```c++
// 一个 deleter, 用于从 Cache 中删除数据项时使用
static void DeleteEntry(const Slice& key, void* value) {
  TableAndFile* tf = reinterpret_cast<TableAndFile*>(value);
  delete tf->table;
  delete tf->file;
  delete tf;
}
```
比较简单轻量, 就是做内存释放. 那 `deleter` 啥时候调用呢? `Cache` 析构的时候.

注意, `Insert` 的时候, 会给新的数据项设置引用计数.

### 2.2 删

```c++
// 如果 cache 包含了 key 对应的映射, 删除之.
virtual void Erase(const Slice& key) = 0;
```
 
注意, 因为引用计数的存在, `Erase` 可能不会发生物理删除, 除非数据项对应引用计数变为 0.

### 2.3 查

```c++
// 如果 cache 中没有针对 key 的映射, 返回 nullptr. 
// 其它情况返回对应该映射的 handle. 
virtual Handle* Lookup(const Slice& key) = 0;
```

查询本身没有什么可讲的, 要重点说的还是引用计数相关, 针对查询到的数据项, 要递增其引用计数防止被其它客户端删除. 而这也引申出另一个需求, 就是当调用 `Lookup` 调用方不再使用数据项的时候, 需要主动调用 `Release(handle)` 来将引用计数减一. 

### 2.4 其它

除了上面三个核心方法. `Cache` 还涉及下面几点:
- `Cache` 对象不支持任何拷贝(用技术语言讲就是它的拷贝构造和赋值构造都被禁掉了), 一个实例只能有一个副本存在, 这样做就省掉了维护各个 `Cache` 副本一致性的问题.
- 除了显式地删除, `Cache` 还有一个 `Prune()` 方法, 用于移除缓存中全部不再活跃的数据项, 具体取决于 `Cache` 具体实现, 以 `LRUCache` 为例就是将不在 `in_use_` 链表中的数据项删除.
  
## 3 ShardedLRUCache 实现

(代码位于 `util/cache.cc`)

前文提到的 `Cache` 是个抽象类, leveldb 通过 `ShardedLRUCache` 继承并实现了相关功能. 

该类位于一个匿名 namespace, 通过下面的方法暴露相关功能:

```c++
Cache* NewLRUCache(size_t capacity) {
  return new ShardedLRUCache(capacity);
}
```

(熟悉 golang 的同学可能看了会会心一笑, 真说不准后来的 golang 相关写法沿用了 Google 在 C++ 代码里的习惯.)

要理解这个类, 先从名字开始, 从右到左关键词分别是 *LRU*, *Sharded*. 这两个关键词说清楚了实现的关键点:
- 缓存基于 LRU 算法做淘汰
- 缓存支持 sharding 即分片

下面从顶向下来描述相关设计与实现.

### 3.1 Sharding

Sharding 算法一般有两种, 基于 hash 的或者基于 range 的, 这里是基于 hash 的. 

不知道大家可发现这里有个问题. 就是 `ShardedLRUCache` 为何要进行 sharding, 明明这个类的一个实例只存在于一个节点的内存里(这么说有点绕但为了尽可能严谨先这么表达了. 换个不严格的问法就是 leveld 非分布式, `ShardedLRUCache` 实例也只在一个机器上, 为啥还要搞成分片的?)?

文档和代码里没有说明, 但我觉得这个问题值得思考一下. 

Sharding 最明显的目的就是分摊压力到各个 shard, 要么是分摊存储压力(多节点, 每个节点不承担一部分存储), 要么是分摊计算压力(多节点, 每个节点承担一部分计算), 总之这个在分布式环境下比较容易理解. 这里如此设计目的我认为是后者, 即计算压力, 更明确地讲是避免针对 `ShardedLRUCache` 锁粒度过大导致访问变为串行. 每个 shard 持有各自的锁, 这样可以尽可能地实现并行处理. 这里的 sharding 实质上是实现了 Java 里的 [Lock Striping](https://www.geeksforgeeks.org/what-is-lock-striping-in-java-concurrency/).


#### 3.1.1 sharding 存储

`ShardedLRUCache` 有一个成员, 它包含一个 shard 数组, 每个 shard 就是一个 `LRUCache`:

```c++
LRUCache shard_[kNumShards];
```

每个 cache 默认 16 个 shards.

#### 3.1.2 增

插入数据的时候, 先计算 hash, 然后基于 hash 寻找对应 shard(即 LRUCache) 做真正插入操作:

```c++
virtual Handle* Insert(const Slice& key, 
    void* value, 
    size_t charge,
    void (*deleter)(const Slice& key, void* value)) {
  // 计算 hash
  const uint32_t hash = HashSlice(key);
  // 基于 hash 做 sharding
  return shard_[Shard(hash)].Insert(key, hash, value, charge, deleter);
}
```

#### 3.1.3 删

和插入数据类似, 先计算 hash, 定位到所在 shard, 然后再做具体删除操作:

```c++
virtual void Erase(const Slice& key) {
  const uint32_t hash = HashSlice(key);
  shard_[Shard(hash)].Erase(key, hash);
}
```

#### 3.1.4 查

与插入数据类似, 先计算 hash, 定位到可能所在的 shard, 然后再做具体查询操作:

```c++
virtual Handle* Lookup(const Slice& key) {
  const uint32_t hash = HashSlice(key);
  return shard_[Shard(hash)].Lookup(key, hash);
}
```

### 3.2 LRUCache

前面介绍 sharding 的时候提到了 `LRUCache`, 它是每个 shard 的实际形态. 虽然它叫 *XXCache*, 但该类并不是 `Cache` 接口的实现(虽然干的活其实差不多)但也无所谓. 由于它是 `ShardedLRUCache` 的实际存储, 所以相关增删查方法都有, 这些根据字面就基本知道干啥了, 不再细说, 我们要特别说说跟 LRU 相关的部分.

#### 3.2.1 存储成员

该类包括两个循环链表, 一个是 `in_use_` 链表, 一个是 `lru_` 链表, 同时还有一个用于快速查询数据项是否存在的哈希表 `table_`. 其中:
- `in_use_` 存的是在用的数据项
- `lru_` 存的是可以淘汰(以节省空间, 调用 `prune()` 方法会将该链表清空)也可以重新提升到 `in_use_` 中的数据项
- `table_` 是 leveldb 自己实现的一个哈希表(当时测试随机读比 g++ 4.4.3 版本的内置 hashtable 快了大约 5%), 存储了出现在 `in_use_` 和 `lru_` 中的全部数据项(用于快速判断某个数据项是否在 `LRUCache` 中). 注意它保存了前面提到的两个链表的数据, 如果响应用户查询时发现数据项在 `lru_` 中则会自动将其提升到 `in_use_` 链表中.

不管是链表还是哈希表, 存储的数据项都是 `LRUHandle`, 它是个变长数据结构, 有必要展示下其核心部分:
```c++
struct LRUHandle {
  void* value;
  void (*deleter)(const Slice&, void* value);
  // 下面这个成员专用于在哈希表中指向与自己同一个桶中的后续元素
  LRUHandle* next_hash; 
  // 下面两个成员用于 in_use_ 或 lru_ 链表
  LRUHandle* next;
  LRUHandle* prev;
  // TODO(可选): 当前 charge 大小只允许 uint32_t
  size_t charge;      
  size_t key_length;
  // 指示该数据项是否还在 cache 中. 
  bool in_cache;      
  // 引用计数, 包含客户端引用数以及 cache 对该数据项引用数(1). 
  uint32_t refs;      
  // 基于 key 的 hash, 用于 sharding 和比较. 
  uint32_t hash;      
  // key 的起始字符, 注意这个地方有个 trick, 
  // 因为 key 本来是变长的, 所以这里需要
  // 将 key_data 作为本数据结构最后一个元素, 
  // 方便构造时根据 key 实际大小延伸. 
  char key_data[1];   

  Slice key() const {
    // 仅当当前 LRUHandle 作为一个空列表的 dummy head 时, 下面的
    // 断言才不成立. dummy head 不保存任何数据.
    assert(next != this); 

    return Slice(key_data, key_length);
  }
};
```

几个关键点:
- 为啥有 `next_hash`? 由于 `LRUCache` 若仍在缓存则必定被哈希表和其中一个链表(`in_use_` 或 `lru_`)引用, 且哈希表是基于桶的, 所以一个 *next* 成员就不够用了, 所以专门定义了一个 `next_hash` 用于哈希表桶里面的链接.
- 为啥单独保存 key? 变长部分就是保存 key 的 `key_data`. 这里有个疑问, 为啥 value 用的指针, 而这里要搞个变长结构单独保存一遍 key? 通过查看 `Cache::Lookup(key)` 的调用可以发现, 外部传入的 key 都是栈上维护的临时变量, 插入 cache 的时候就需要保存, 于是就有了这里的节省内存的变长结构.
- 引用计数 `refs`, 它的重要意义前面讲过了, 引用计数设计贯穿了整个 leveldb 的实现, 不再说了.

#### 3.2.2 LRU 算法如何生效的

`LRUCache` 的 `prune()` 方法会直接清除 `lru_` 链表内容, 无差别地清除, 所以这里未体现 LRU 思想, 那在哪儿体现的呢? 在 `insert()` 方法里.

在 `insert()` 方法最后有这么一段代码(我是真不喜欢这类有副作用的设计):
```c++
  ...
  // 如果本 shard 的内存使用量大于容量并且 lru_ 链表不为空, 
  // 则从 lru_ 链表里面淘汰数据项(lru_ 链表数据当前肯定未被使用), 
  // 直至使用量小于容量或者 lru_ 清空. 
  while (usage_ > capacity_ && lru_.next != &lru_) {
    // 这很重要, lru_.next 是 least recently used 的元素
    LRUHandle* old = lru_.next;
    // lru 链表里面的数据项除了被该 shard 引用不会被任何客户端引用
    assert(old->refs == 1);
    // 从 shard 将 old 彻底删除
    bool erased = FinishErase(table_.Remove(old->key(), old->hash));
    if (!erased) {  
      // to avoid unused variable when compiled NDEBUG
      assert(erased);
    }
  }
  ...

```

这个循环最最重要的是这一行:
```c++
LRUHandle* old = lru_.next;
```

因为新加入的元素都被插在 `lru_.pre` 位置, 所以从 `lru_.next` 开始遍历就是从最老那个元素遍历.

就是这么简单.

## 4 总结

本文介绍了 leveldb 的缓存相关设计和实现, 缓存用于保存 sstable 文件的内存形式, 可加速用户查询过程. Leveldb 缓存基于 hash 做 sharding 同时支持 LRU 淘汰机制, 前者减小了锁粒度提升了并发性, 后者在内存占用和缓存命中之间实现了折中.

--End---
