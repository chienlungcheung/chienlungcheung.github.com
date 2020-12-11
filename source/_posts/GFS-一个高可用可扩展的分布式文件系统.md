---
title: 'GFS: 一个高可用可扩展的分布式文件系统'
date: 2020-12-11 21:44:47
tags:
  - gfs
  - distributed system
  - paper
---

<!-- toc -->

本文基于内部分享 <"抄"能力养成系列 -- GFS 设计> 整理.

2003 年开始 Google 陆续放出三套系统的设计(GFS/MapReduce/Bigtable), 在互联网届掀起云计算狂潮一直影响至今. GFS 作为发轫, 目前许多业界知名的分布式系统设计仍然有着它的影子. 下面就让我们一起看看 GFS 的设计, 希望为各位后续系统研发提供灵感。

# 1 GFS 诞生背景

- 1, 组件失效是常态而非例外
- 2, 按传统标准, 现实世界的文件太大了, 一般都几个 GB, 个数多了单机存不下.
- 3, 大多数文件通过 append 而非覆盖进行修改, 所以追加操作是性能优化和需要原子保证的焦点.
- 4, 将上层应用和文件系统 API 联合设计可增加灵活性, 让整个系统都受益. 比如通过放松 GFS 的一致性要求以大规模简化文件系统还不给应用增加负担. 同时也引入了一个原子化的 append 操作以让多个客户端针对同一个文件并发执行 append 操作而无需额外的同步措施.

# 2 GFS 设计概览

## 2.1 几点假设

- 硬件便宜爱故障, 监控完善恢复快.
- 文件个数适中, 以大文件为主, 支持但不优化小文件存储(tradeoff).
- 负载主要是大数据量的流式读取和小数据量的随机读取, 后者可以合并重排减少随机.
- 另一个重要负载是大数据量顺序 append 数据到文件, 也支持随机写但没做优化, 文件一旦写完几乎不再修改.
- 设计良好且高效实现的单文件并发 append 操作. GFS 文件经常用于生产者消费者队列或多路合并.
- 持续稳定的高带宽比时延更加重要. 大多数应用更看重高吞吐而非某个读写操作的响应时间. (long-fat network)

## 2.2 接口

- GFS 支持 create/delete/open/close/read/write 操作但不支持 POSIX 规范(GlusterFS 支持).
- Snapshot: 就是以一个很低的消耗创建一个文件或者目录树的拷贝.
- record append 操作: 允许多个客户端并发地向同一个文件追加数据, 每个追加操作都是原子的, 这为实现生产者-消费者队列或者多路合并提供了便利, 因为多个客户端同时写同一个文件无需加锁.

## 2.3 架构

Figure 1 为 GFS 架构一览.

- 一个 GFS 集群由 1 个 master 和多个 chunkserver 构成, 集群可同时被多个 client 访问.
- Client 代表应用和 master 还有 chunkserver 通信.

![Figure 1](/images/google-gfs-20201211/figure-1-GFS-Architecture.png)

其中文件相关的有:
- 文件被切成固定大小的 chunks(每个 chunk 还会被切成 blocks, 后述). 
- 每个chunk 都有一个全局不可变的 64 bit 的 chunk handle, 这个 handle 由 master 在 chunk 创建时分配.
- 读写 chunk 时需要指定 handle 和字节范围.
- 为了高可用, 每个 chunk 会被在多个 chunkserver 上保存各保存一个副本, 默认三副本.

master 相关的有:
- master 保存整个文件系统的元信息, 包括命名空间/访问控制信息/从 files 到 chunks 的映射/chunks 当前位置.
- master 也控制系统层面的活动如 chunk 租约管理/孤儿 chunks 的垃圾回收/chunkservers 之间的 chunk 迁移.
- master 和各个 chunkservers 通信(心跳)以发送指令和收集状态信息.

client 和 chunkserver 都不会缓存文件数据:
- 就 client 而言, 一方面原因是应用程序一般流式拉取巨大文件或者数据太大无法进行缓存, 另一方面因为无需处理缓存一致性问题可以简化 client 和整个系统的复杂度. 
- 针对 chunkserver 而言, 因为数据本来就存在 chunkserver 本地磁盘, 可以直接使用 linux 的 buffer 来实现经常被访问的数据的缓存, 单机的缓存就交给单机操作系统来处理了.

## 2.4 单 master 集群各组件交互重点

注意, GFS 一个集群只有一个 Master(关于它的高可用后面描述).

- master 和 chunkserver 之间周期性的心跳交互.
- client 读取时先问 master 数据存在哪个 chunkserver, 拿到信息后缓存到本地(有超时时间限制), 然后直接和 chunkserver 通信.
- client 把上层应用指定的文件名和偏移量翻译成 chunk index, 发给 master.
- master 返回 handle 和 location, client 用文件名和 index 做 key 缓存该信息.
- 直到元信息缓存失效或者应用重新打开文件, client 都无需和 master 交互, 而是直接和 chunkserver 交互. 这可以大幅降低 master 负载.

## 2.5 chunk 大小设计

- chunk 大小非常关键, GFS 选的是 64MB. 比传统的文件块大多了. 采用 lazy space 分配策略, 可以避免如此大的 chunk size 导致的空间浪费.
- 每个 chunk 都作为一个普通的文件存储在 chunkserver.
- chunk size 选的大有如下好处: 
  - 1, 大幅减少了 client 和 master 的交互, 因为典型的应用就是顺序读取大文件, chunk 大则要访问的 chunks 就少, 就不用频繁与 master交互. client 侧可以为数 TB 数据集数据缓存它们对应的 chunk 元信息. 
  - 2, 因为 chunk 很大,可以覆盖很多操作, 这样 client 跟其所在 chunkserver 长时间维持一个持久 TCP 连接而无需频繁与多个 chunkserver 新建链接, 这就减少了网络开销.
  - 3, 因为 chunk 很大, 个数就会减少, 则对应的元信息也相应减少, 这样 master 可以在内存缓存全部元信息.
- 这么大的 chunk size 也有缺点, 就是对小文件不友好:
  - 因为一个小文件可能只对应一个 chunk, 如果针对小文件并发操作很多, 那么它就会成为热点(redis hotkey 与之类似.). 
  - 因为实际应用中以大文件读写为主所以问题不严重. 
  - 目前解决办法就是针对热点小文件提升其副本因子, 并且把访问该文件的客户端交错启动. 长远的解决办法是允许客户端能从其它客户端而不仅仅是从 chunkserver 读取同样的数据.

## 2.6 元信息

master 保存三种类型的元信息:
- 文件和 chunk 命名空间
- file-to-chunk mapping
- 每个 chunk 副本的位置信息

master 把全部元信息都保存在的内存中:
- 前两种元信息会被 master 通过本地日志持久化变更同时备份到远程机器确保可用性.
- chunk 位置信息不会被持久化, 而是在 master 启动以及 chunkserver 加入集群时询问 chunkserver 有关 chunk 的位置信息.

chunkserver 保存的元信息有两个: 每个 block 的校验和, 以及 chunk 版本号.

在 master 上, 每个文件对应的元信息大约 100 字节, 这也符合 master 内存不会成为系统容量瓶颈的预期. 而且这 100 字节大部分是文件名(已经过前缀压缩). 其它元信息还有文件归属/权限, 从文件到 chunks 的映射, chunk 副本的位置信息, 每个 chunk 的当前版本. 针对 chunk 还会保存一个引用计数用于实现 COW(copy-on-write).

master 和 chunkserver 每个都保存大约 50 到 100 MB 数据, 加载非常快, 但是由于 master 启动时要拉取chunk location 所以启动需要额外 30 到 60 秒.

自从 master 内存结构改成二叉搜索树以后, 命名空间搜索不再是瓶颈.

### 2.6.1 元信息都保存在内存中

master 把元信息存在内存, 操作就会很快.

master 可以在后台高效地周期性扫描整个状态空间以实现:
- chunk GC, 
- 应对 chunkserver 失效重新分配副本, 
- 为实现负载和磁盘空间均衡在 chunkservers 间迁移 chunks.

master 把全部信息保存到内存有个不好的地方就是集群数据量受限于 master 内存大小.  不过因为每个 chunk(64MB 大) 对应元信息不超过 64 字节, 所以实践中不是啥问题.

大多数 chunks 都是满的因为大多数文件各自都包含多个 chunks, 可能仅每个文件对应的最后一个 chunk 不满, 这就让元信息性价比非常高. 

同样, 针对每个文件, 它的文件命名空间数据也不超过 64 字节, 因为其存储的文件名通过使用前缀压缩非常紧凑.

当然, 想支持更大的数据集合, 给 master 加内存就行了, 便宜又快捷.

### 2.6.2 chunk 位置信息

虽然 master 不持久化 chunk 位置信息, 但是因为它负责 chunks 布局同时通过心跳监控 chunkserver, 所以它能保持这些位置信息时刻保持更新.

其实我们开始也尝试持久化 chunk 位置信息, 后来发现还是 master 启动后周期性拉取更简单.  这消除了因为 chunkserver 加入或离开集群/改名/失效/重启等等而保持 master 和 chunkservers 数据同步的问题.

### 2.6.3 操作日志

master 上的操作日志包含了关键元信息的变更历史.  这对 GFS来说至关重要. 

这个日志作为逻辑时间线定义了并发操作的顺序. 文件和 chunks 以及它们的版本号, 可以永久通过它们被创建的逻辑时间被唯一识别.

元信息变更被持久化到本地和远程多台机器后, 相关变更才对客户端可见.

master 启动时会读取操作日志并进行重放以恢复系统状态, 所以操作日志不能太大否则启动时间会非常长.

为了避免日志文件过大, master 会在日志文件超过一定大小后 checkpoint 自己当前状态, 这样下次启动时只需从本地磁盘加载和重放最近一次 checkpoint 就可以恢复到最新状态.

因为构造 checkpoint 需要花时间, 为了避免阻塞后续处理, 方法如下: 
- master 切换到新的日志文件并在一个单独的线程中创建 checkpoint. 
- 新的 checkpoint 包含日志切换前的全部变更.
- 大约一分钟左右就可以为一个拥有几百万文件的集群创建一个 checkpoint.
- 创建完成后它将被写入本地和远程磁盘.

master 恢复过程只需最近的 checkpoint 和创建该 checkpoint 后的日志文件. 老的 checkpoints 和日志文件都会被删除.

## 2.7 GFS 的一致性模型

GFS 提供了一个弱一致性模型, 相对简单且容易高效实现.

### 2.7.1 GFS 提供的保证

文件命名空间变更, 如创建文件, 是原子的. 它们均仅由 master 处理.  master 的操作日志为这些操作定义了全局顺序. 

**consistent**: 针对某个文件区域, 如果全部客户端看到的数据是一致的, 不管它们是从哪个副本读取的数据, 我们就说这个文件区域数据是一致的.

**defined**: 针对经历过数据变更的某个文件区域, 如果它是 consistent 的, 并且客户端能看到前述变更的完整内容, 即可预测, 不会因并发而随机那么我们就说这个文件区域是确定的.

注意, defined 和 consistent 是两个层面的东西, 只要写成功, 那么 GFS 的 2PC 能保证 consistent, 也就是 defined 包含了 consistent.  但如果发生了并发变更, 比如多个客户端针对某个文件区域同一个偏移并发写,  此时不同于追加, 这种写操作是互相覆盖的, 最终结果是 undefined 的, 即我们无法预先知晓该偏移处结果是什么, 但 GFS 可以保证各个副本都是同样的 undefined 状态.

![Table 1](/images/google-gfs-20201211/table-1-变更后的文件区域状态.png)

上面 Table 1 展示了 GFS 两种典型的变更操作 write(覆盖写) 和 record append(记录追加)在串行和并发情况下完成后对应的文件区域的状态.

数据变更包括 write(在指定偏移处写入, 如覆盖写)和 record append(即在文件尾部原子地追加记录).

GFS 会保证一系列成功变更后的文件是 defined 的, 措施如下:
- 1, 以同样的顺序将变更应用到 chunk 全部副本.
- 2, 使用 chunk 版本号来检测过期的副本, 相关 chunkserver可能因为下过线错过某些变更.

过期副本不会对外服务而是会被尽快 GC 掉.

master 借助心跳计算每个chunkserver 上数据的校验和以检测数据是否损坏.发现损坏后会尽快从好的副本恢复数据.

如果全部副本丢失, 那么chunk 就不可恢复了. 这种情况下应用得到的响应是数据丢失而非损坏.

### 2.7.2 一致性模型在应用端实现

写
- 依赖于追加而非覆写(相比于覆写, 追加更加高效同时对应用错误更富有弹性), 应用程序一般就是一个文件从头 append 到尾;
- 要么等数据写完后原子地将文件重命名为一个永久性的名字, 要么周期性地 checkpoint 写了多少字节了同时还可以附加一个应用层校验和. 
- checkpoint 让 writers 可以在重启后增量写入而不用重写全部数据, 同时避免 readers 处理已成功写入但不完整的内容(站在应用层角度).

读
- 只验证和处理最后一个 checkpoint 之前的文件区域, 这些区域处于 defined 状态.

record-append 操作的 append-at-least-once 语义确保了不丢数据, 然后由 readers 负责处理重复数据.

由于 record 包含了一个附加的校验和, 所以 readers 可以此校验 records. 

readers 还可以通过每个记录的唯一 ID 过滤掉重复的 records.

## 2.8 系统交互

GFS 在设计的时候就在尽量做到最小化 master 参与各种操作, 给 master 减负.

下面描述 client/master/chunkserver 三者之间如何交互以实现数据变更/原子化记录追加/快照.

## 2.8.1 lease 和 mutation 顺序

chunk 的每个变更都会反映到每个副本上, 具体如下:
- master 通过选择一个副本颁发一个 lease 将其指定为 primary.
- 然后 primary 为 chunk 的并发变更选择一个串行的顺序, 全部副本都遵循该顺序应用变更. 
- 因此全局变更顺序首先由 master 授权 lease 的顺序以及每个 lease 生效期 primary 指定序列号时确定. 

lease 机制可以最小化 master 的管理开销, 因为针对 chunk 的一部分管理工作让 primary 承担了.

每个 lease 都有一个初始的 60 秒超时, 但是只要 chunk 仍在变更, primary 可以发送请求给 master 要求延长 lease 有效期, 这个过程是通过 heartbeat 实现的.

当然 master 可以在 lease 过期之前吊销它, 比如当 mast 想禁止对正在改名的文件进行变更时. 当 master 失去与primary 通信时, master 可以授权新的 lease 给另一个副本.

![Figure 2](/images/google-gfs-20201211/figure-2-write-control-and-data-flow.png)

Figure 2 为执行写操作时的控制流, 具体如下:
- 1, client 询问 master 哪个 chunkserver 持有要访问的 chunk 的 lease 以及其它副本的位置. 如果没人持有 lease, master 会选一个副本授权之.
- 2, master 返回 primary 的 id 以及和其它副本的位置, client 会缓存这些信息, 仅当 primary 不可用或者 primary 明确告知不再持有 lease 时再和 master 通信.
- 3, client 以任意顺序将数据推给全部副本. 每个 chunkserver 将数据首先保存到内部的 LRU 缓存.  图中将控制流和数据流解耦, 基于网络拓扑(该拓扑不关心谁是 primary)调度数据流可以改善性能, 具体后述.
- 4, 当全部副本确认收到数据后, client 再发送一个写请求给 primary,该请求标记了上一步推送给全部副本的数据, primary 会为(可能来自多个 clients 的)全部变更分配连续的序列号,然后 primary 按照此顺序应用这些变更.
- 5, 应用本次变更后, primary 将该写请求转发给全部 secondary 副本,这些副本以同样的顺序应用变更.
- 6, secondary 副本回复 primary 自己已完成操作.
- 7, primary 回复 client 本次写操作成功 or 失败. 如果只有 primary 和部分 secondary 成功, 则再返回第一步重试之前会尝试重复 3-7.

**从 3 到 7 其实就是 2PC (two-phase commit).**

如果一个写操作跨多个 chunk, 那么客户端就会将这个写操作分裂成多个写操作. 它们都按照前面描述的步骤执行, 但是可能和其它客户端的写操作交织在一起并发执行. 因此同一个文件区域可能被多个客户端相互覆盖写入. 尽管这个文件区域的全部副本最终是 consistent, 但是最终结果我们无法预知是什么, 所以最终是 consistent 但 undefined.

### 2.8.2 数据流

将数据流和控制流解耦, 让我们可以更高效地利用网络. 下面看看 GFS 是怎么做的.

就像控制流是从 client->primary->secondaries 管道化传输, 数据流从 client 到各个 replicas 也是管道化传输.

具体地, client 挑选离自己最近的 chunkserver (注意这里根本不管谁是 primary, 只关注网络拓扑), 将数据发给它, 然后

这个  chunk server 一边接收 client 的数据一边转发给离自己最近的 chunkserver, 依此类推  ...  从而链式完成数据从 client 到每个副本所在 chunkserver  的发送. 

最近距离计算: 这个管道化传输过程中, 各个节点通过 IP 地址计算前面提到的 “最近”距离的计算, 比如同一个局域网跟定比跨网段的要更近一些. 

管道化传输好处: 这种管道化传输尽可能地利用了每个 chunkserver  的出站带宽, 也最小化了 TCP 连接的时延.

### 2.8.3 原子化的记录追加操作

传统的写操作, 需要指定要写入到的文件偏移量. 并发执行该类操作写同样的区域并不是串行化的, 被写入区域最后状态包含多个客户端的数据片段.

GFS 提供了原子化追加操作, 叫 record append, GFS 会确保至少一次写的语义, 将数据原子化地追加到文件末尾. 这类似于在 unix 编程中, 采用 O_APPEND 模式打开文件而且没有竟态条件.

record append 作为一种 mutation, 执行流程同 Figure 2, 但在 primary 那有些许不同: 
- 1, 首先 client 肯定是将数据 push 给文件的最后一个 chunk(因为是 appende 嘛).
- 2, primary 检查追加这个记录后 chunk 是否超过 64MB,
  - 2.1, 如果超过, 则将这个chunk 填满无效数据, 同时告诉 secondaries 也这么干, 然后告诉客户端说满了, 请将数据写入下个chunk. 为了避免产生过多这类因填充导致的空间碎片, 我们要求 record 大小最多为 chunk 的四分之一.
  - 2.2, 如果不超过, primary 将数据写入本地副本,然后告知secondaries 将数据写入到同样的偏移处, 然后响应客户端. 
- 上述过程, 在任何副本写失败, 客户端就要重试该写入操作. 当然,这个重试会导致 chunk 不同副本数据不一致, 但是 GFS 保证的不是字节级别的一致性, 而是记录级别的, 它保证至少一次 append, 就能保证各副本数据一致, 虽然重复记录但是可以通过 id 去重.

## 2.9 Snapshot 快照

snapshot 操作用于制作当前文件和目录的快照, 速度非常快.

snapshot 用的是 COW (copy-on-write) 技术实现, 具体如下: 
- 当master 收到 snapshot 请求时, 它首先吊销要进行 snapshot 的文件相关 chunks 的 lease, 这样后续针对这些 chunks 的写操作强迫 client 先和 master 交互, 这就为创建快照争取了时间.
- lease 吊销后, master 将 snapshot 操作记录到日志中, 然后将其应用到内存状态上制作快照.
- 新创建的 snapshot 文件指向源文件同样的 chunks, 计数加一. 此时如果客户端要写这些 chunks, master 就会察觉引用计数大于 1, 此时触发 COW 操作.
- COW 过程: 
  - master 指示每个拥有 chunk C 副本的 chunkserver 都新建一个 chunk C‘, 而且 C’ 创建在同 C 一样的 server 上, 这可以使得 COW 在本地而非通过网络进行.
  - 创建完 C‘ 并拷贝完数据后, 集群就有两组一样的 chunk 副本了, 剩下的处理流程就同之前一样了, master 针对 C' 授权 lease 给某个副本并响应客户端, 客户端开始写入数据.

# 3 Master 行为

Master 执行全部和命名空间有关的操作. 

另外, master 管理着整个系统的 chunk 复制: 
- chunk 布局决策
- 创建新 chunks 和其副本
- 协调多种系统级别的活动以保障 chunks 副本健全, 各个 chunkserver 的负载均衡, 回收未使用的存储空间

下面挨个讨论上述话题.

## 3.1 命名空间管理和锁机制

GFS 允许多个操作并发, 通过锁来将其串行化.
- 读锁, 防止目标被删除/重命名/snapshotted;
- 写锁, 防止并发创建同名目标文件.

同 Unix 不同, GFS 没有为每个目录设计一个保存其文件列表的数据结构.

GFS 的命名空间可以看作一个速查表, 保存了从全路径名(文件名或目录名, 可以看作一个前缀树)到元数据的映射. 
- 通过前缀压缩, 可以使得整个表被保存到内存中. 
- 命名空间树上面的每个节点都关联了一个读写锁.

由于没有真正的目录结构, 所以这使得我们可以并发地在同一个目录下创建文件, 只要获取这个所谓的目录的读锁(防止目标被修改)同时获取目标文件的写锁(防止在同一个目录下生成同名文件)即可.

为避免死锁, 加锁顺序全局一致: 先按照命名空间树不同层次对锁排序, 如果在同一层则按照字典序对锁排序.

## 3.2 chunk 副本布局

GFS 的分布式体现了多个层次:
- 1, 首先 GFS 集群一般涉及上百台 chunkservers
- 2, 这些 chunkservers 一般分布在多个机架.

所以同一个 chunk 的副本布局不但要考虑多 chunkservers 还要考虑这些 chunkservers 不能集中在同一个机架, 这么做就为了实现: 
- 可靠性
- 可用性
- 最大化网络带宽使用率(机架进出带宽可能小于这个机架上的全部机器各自网卡带宽加总.), 比如热点副本集中到同一个 chunkserver 或者同一个机架, 并发读写就会出现瓶颈, 这在后面测量部分会再细述.

## 3.3 副本的新建, 重复制与再平衡

### 3.3.1 副本新建

三种情况下新建副本:
- 1, 新建 chunk 时
- 2, 重复制
- 3, 重平衡

新建 chunk 时新副本安置在哪儿, 主要考虑下面几点: 
- 1, 尽量把新副本放在磁盘使用率低于平均水平的 chunkserver 上, 这样各个机器会逐渐平均.
- 2, 虽然副本创建操作本身消耗不多, 但它预示着大量的写流量即将到达. 所以新建副本时会尽量让每个 chunkserver 近期的副本(不区分 chunk)新建数尽量的差不多, 避免写流量涌入少数 chunkserver.
- 3, 就像前一节讨论的, 让副本尽量分布在多个机架上.

### 3.3.2 副本重复制

当 chunk 副本因子低于设定时, master 就会触发重复制. 

如果有多个 chunk 满足条件, 则执行优先级就是:
- 1, 哪个 chunk 距离目标副本因子越远就谁优先重复制;
- 2, 还有就是活跃的文件 chunks 优先级高于被删除文件的 chunks.
- 3, 另外就是优先那些当前阻塞住客户端的 chunks.

重复制后的副本放到哪个 chunkserver, 原则同副本新建所描述: 
- 1, 让每个磁盘空间使用率尽量相同; 
- 2, 限制同一个 chunkserver 上同时活跃的 clone 操作; 
- 3, 另副本尽量分散到各个机架. 

为了避免副本 clone 流量压倒客户端流量, master 会限制同时活跃的 clone操作个数.  同时, 每个 chunkserver 会限制自己用于 clone 的带宽, 方法就是限制自己到其它源 chunkserver 的读请求数目.

### 3.3.3 副本重平衡

master 会为了更好的磁盘使用率和负载均衡而周期性地做副本 rebalance.

master 做 rebalance 时布局标准同重复制. 

rebalance 选择要移除哪个 chunkserver 的副本时倾向于那些空闲空间低于平均值的 chunkserver, 目的也是最终拉平各个 chunkserver 的磁盘使用.

## 3.4 垃圾回收

### 3.4.1 惰性回收

一个文件被删除后, GFS 不会立即回收它所占用地物理空间, 而是通过周期性地 GC 在文件和 chunk 两个层次进行惰性回收.  我们发现这么做使得整个系统更加简单也更加可靠.

### 3.4.2 回收机制

文件层面:
- 同其他操作, 删除操作会先记录到 master 日志. 
- 然后被删除文件被标记为包含删除时间戳的隐藏名. 
- master 周期性扫描文件系统命名空间时移除那些被标记超过三天(可配置)的文件. 真正移除之前这些文件仍可读, 甚至可以将名字改成正常名字表示不再删除. 
- 当隐藏文件被移除后, 它对应的内存中的元数据也会被擦除, 这相当于断开了这个文件同其 chunks 的连接.

Chunk 层面:
- 同样地, 针对 chunk 命名空间的周期性扫描, master 会将从任何文件都不可达的 chunks 标记为孤儿 chunks. 
- chunkserver 在和 master 周期性的心跳消息中上报自己持有的 chunks, master 会回复那些不再存在的 chunks 标识, chunkserver 收到后自主删除这些 chunks 对应的副本.

### 3.4.3 几点讨论
虽然在编程语言中, 分布式垃圾回收非常难, 但是针对 GFS 却很简单:
- 我们可以很容易地识别到 chunks 的引用: 它们就在 file-to-chunk mappings 中, 这些信息由 master 专门维护. 
- 我们也可以很容易识别全部 chunk 副本: 它们就是 chunkserver 指定目录下的 linux 文件. 这些副本对 master 来说都不是垃圾.

GC 相比 eager deletion 的好处: 
- 1, 在组件容易故障的分布式系统中更简单可靠. 如果发删除消息则可能丢失, 需要重发送, 维护起来很复杂. GC 提供了统一可靠的清理不再有用的副本的方式. 
- 2, 将 GC 纳入到周期性后台任务(其它还有命名空间扫描和心跳), 批量处理摊销了消耗. 而且这种后台任务仅当 master 相对空闲时候才做可以保证对客户端的响应. 
- 3, GC 的惰性也避免了因误删除(意外且不可逆)导致的数据安全问题.
- 最大的缺点: 不能在存储紧张时候及时回收. 但 GFS 支持连续两次删除立即回收空间.

另外, GFS 支持为不同的命名空间指定副本策略(如副本因子)和删除策略.

## 3.5 过期副本检测

chunkserver 挂掉或者下线就会导致其上的 chunks 过期.

master 维护着一个 chunk 版本号来识别最新和过期副本. 

master 授权一个新的 lease 时就会递增 chunk 版本号并通知其它副本, 并且 master 和这些副本会持久化这个版本号. 当某个副本挂掉重启后, 它会上报自己的 chunks 和版本号给 master, master就能检测到过期.

如果 master 发现有版本号比自己记录的还要大, 则认为自己授权 lease 时候出错并将该版本号作为最新版本号. 

master 在响应客户端哪个 chunkserver 持有它所请求的 chunk 时会在响应中包含版本号以让客户端进行校验; master 在指示某个 chunkserver 去另一个 chunkserver 拷贝数据时也会告诉它当前的版本号, 供其校验.

# 4 容错与检测

设计这个系统最大的挑战时应对频繁的组件失效.

## 4.1 高可用

### 4.1.1 保持高可用的两个手段

- 1, 快速恢复,不管之前如何下线的, master 和 chunkservers  可以几秒内即可重启并恢复数据.
- 2, 多副本, 默认三副本.

### Master 复制

master 的操作日志和 checkpoint 会被保存到多台机器. 一个更新操作仅当其被记录到 master 本地和远程机器上才算被提交.

master 挂了, 监控系统会立即在其它机器上启动一个 master 并快速从日志恢复状态. clients 用的是域名访问 master, 所以可以快速感知这个变化.

另外 GFS 还提供了影子 masters, 它们只提供了读操作, 而且数据可能会稍微落后 primary master 一秒. 针对那些不怎么变动的文件或者应用不太在乎稍微过期的数据的时候, 这些影子 masters 可以为 primary master 分担一些读请求. 影子 masters 会拉取日志副本并应用保持自己更新, 而且它们也会在启动时查询 chunkservers 获取 chunk 副本位置信息(因为这些信息不会被持久化到日志只能自己去主动去查), 也和 chunkservers 保持心跳交换信息以监控它们的状态. 当然副本布局/删除等等变更操作还是由 primary master 负责, 影子 masters 只读.

## 4.2 数据完备性

chunkserver 靠**校验和**来检测数据是否损坏, 而不是靠比对各个副本,那不太可行.

每个 chunk 被切分成多个 64KB 大小的 blocks. 每个 block 都有一个对应的 32 bit 校验和. 校验和也会被保存到内存中同时会和日志一起持久化.

chunkserver 响应请求者数据之前会计算校验和(有点性能消耗)并和存储的校验和比对, 如果不一致则响应错误并上报给 master, 请求者会去其它副本读数据, 同时 master 会指示 chunkserver去从其他副本恢复数据, 然后删掉损坏的副本.

空闲时, chunkserver 会扫描和校验不活跃的 chunks, 检测损坏的 chunks 并上报.  master 就会指示创建新的 chunks 并删除损坏的 chunks.

## 4.3 诊断工具

诊断问题只能**靠日志**, GFS 诊断日志记录了 chunkserver 上下线等重大事件和全部 RPC 请求响应.

可以通过日志来重建完整的交互历史来诊断问题.

日志的开销很小, 尤其和得到的好处比起来.

# 5 测量

接下来, 我们看看 GFS 架构和实现中的瓶颈, 以及真实集群中的若干数字.

注意, 这都是 2003 年的数据, 我们要观察的是 GFS 这么大系统的度量方法以及它如何在当年那种相比现在硬件条件那么差的情况下发挥其价值的.

## 5.1 测量用的微基准

测试集群由 1 master, 2 master replicas, 16 chunkservers, 16 clients 构成. 这么搭就为了方便测试, 真实的集群有几百个 chunkservers 和几百个 clients 构成.

所有机器都是双核 1.4GHz, 2GB 内存, 两个每分钟 5400 转的 80GB 磁盘, 一个 100Mbps 的全双工以太网. master 和 chunkserver 共 19 台机器都连接到同一个 HP 2524 交换机上, 16 个 clients 连接到另一个交换机上, 两个交换机通过 1Gbps 链路连接.

### 5.1.1 reads 测试

N 个 clients 同时从系统读写. 每个 client  从一个 320GB 的文件集合中读取一个随机选择的 4MB 区域. 全部 chunkservers 共 32GB 内存, 所以我们期待最多可以有 10% 的几率命中 linux buffer cache.

![Figure 3-a](/images/google-gfs-20201211/figure-3-a-reads-测试.png)

Figure 3.a 显示 N 个 clients 的读取速率聚合以及它的理论上限. 
- 当两个交换机之间的 1Gbps 跑满的时候达到理论上限 125MB/s; 当每个 client 的 100Mbps 跑满时, 单个 client 机器达到理论上限 12.5MB/s.
- 当仅有一个 client 读取时, 观察到的读取速率为 10MB/s, 大约是每个 client 上限的 80%(即 10/12.5).
- 当 16 个 clients 一起读取时, 观察到的聚合读取速率为 94MB/s, 大约为理论上限的 75%(即 94/125); 或者 每个 client 大约 6MB/s.
- 从 80% 降到 75% 的原因是: 当读取客户端变多, 多个客户端同时从同一个 chunkserver 读取的概率也变大, 从而导致 chunkserver 的网卡带宽被多个客户端争用的更厉害了.

### 5.1.2 writes 测试

N 个 clients 同时写多个 N 个不同的文件. 

![Figure 3-b](/images/google-gfs-20201211/figure-3-b-writes-测试.png)

每个 client 写 1GB 数据到一个新文件, 每次写入 1MB. 聚合写入速率和理论上限如 3.b 所示.
- 理论上限为 67MB/s, 因为我们需要将每个字节写到 16 个 chunkservers 中的 3 个里, 每个 chunkserver 入口带宽上限为 12.5MB/s.
- 如果单个 client 写入, 则观察到的写入速率为 6.3MB/s, 大约是上限的一半. 罪魁祸首是网络协议栈, 因为它与我们把数据 push 给 chunk 副本的管道化方案不太搭配. 数据从一个副本传播给另一个副本延迟降低了整体的写入速率.
- 如果是 16 个 clients 一起写入, 观察到的聚合写入速率为 35MB/s(每个 client 平均 2.2MB/s), 大约是理论上限的一半(即 35/67). 就像读取测试中, 随着执行写操作的 clients 增多, 则同一个 chunkserver 被多个 clients 并发写的几率增大, 此 chunkserver 的入口带宽被更多 clients 争用, 而且 16 个 clients 的写要比读竞争更激烈, 因为一个字节要写到三个 chunkserver. 




### 5.1.3 record appends 测试

N 个 clients 同时从系统读写. 每个 client  从一个 320GB 的文件集合中读取一个随机选择的 4MB 区域. 全部 chunkservers 共 32GB 内存, 所以我们期待最多可以有 10% 的几率命中 linux buffer cache.

![Figure 3-c](/images/google-gfs-20201211/figure-3-c-record-appends-测试.png)

Figure 3.c 显示了记录追加操作的性能. N 个 clients 同时向同一个文件追加.
性能被存储最后一个 chunk 的 chunkservers 的网络带宽限制住了, 与 clients 个数无关.
当只有一个 client 写入时, 速率为 6.0MB/s, 当 16 个 clients 一起写入时, 速率降到了 4.8MB/s, 主要原因是网络拥塞和抖动.
实际使用中, 我们的应用程序倾向于并发生成若干前述文件而不是一个. 换句话说, N 个 clients 同时向 M 个共享文件追加, N 和 M 都是几千级别的. 因此前面提到的网络拥塞在实际中不是啥大问题, 因为 client 写入一个 chunkserver 时, 当一个文件忙的时候, 可以写另一个文件.

--End--