---
title: "MapReduce: 分布式计算系统设计与实现"
date: 2021-03-03 19:42:52
tags:
  - mapreduce
  - distributed system
  - paper
---

本文基于内部分享 <"抄"能力养成系列 -- MapReduce: 分布式计算系统设计与实现> 整理.

2003 年开始 Google 陆续放出三套系统的设计(GFS/MapReduce/Bigtable), 在互联网届掀起云计算狂潮一直影响至今. MapReduce 作为老二出场, 因为它的实现依赖于之前分享的 GFS 作为存储. 该论文一出, 便直接催生了 Hadoop 另一个重量级同名框架 MapReduce 的诞生. 时光荏苒, 虽然后面又出现了 spark/flink, 但是 MapReduce 在批处理领域的地位至今牢固. 下面就让我们一起看看 MapReduce 的设计, 希望为各位后续系统研发提供灵感. (Salute to Jeff).
<!--more-->

# 1 简要介绍

这个模型说起来真是简单至极, 非常符合直觉, 就是说给非互联网行业的人, 也能听明白. 该模型跟函数式语言中的 map-reduce 理念基本一样, 不过这个是分布式的.

map 用于处理原始记录, 输出中间 `<k, v>`; reduce 基于 k 把中间数据合并, 输出 `<k, List<v>>`.

- 并行化
- 容错
- 数据分发
- 负载均衡

最主要的容错机制就是支持重跑任务.

# 2 编程模型

用户要写的就是 Map 函数和 Reduce 函数. Map 负责将输入加工成中间 kv; MapReduce 库负责将同一个 k 的全部 v 收集好发给 Reduce; Reduce 接收中间数据, 然后基于 k, 合并 v, 一般输出一个或零个值.

## 2.1 举例

以单词计数为例, 用户需要干的就是实现自己的 map 函数和 reduce 函数. 剩下的事情由框架负责.

```c++  
// key: 文档名
// value: 文档内容
map(String key, String value):
  // 遍历文档, 每个词输出一个键值对
  for each word w in value:
    EmitIntermediate(w, "1");
// key: 单词
// value: 计数值构成的列表   
reduce(String key, Iterator values):
  // 累加器
  int result = 0;
  // 遍历每个计数值, 将其累加起来
  for each v in values:
    result += ParseInt(v);
  // 得到每个单词出现的次数
  Emit(AsString(result));
```
## 2.2 输入输出类型

```c++
// 输入为两个参数, k1 类型的键, v1 类型的值; 
// 返回值是一个键值对, 每个键值对是一个 <k2 类型的键, v2 类型的值>,
// 整体效果上看相当于输出了一个键值对列表.
map       (k1,v1)         →   list(k2,v2)
// 输入为两个参数, k2 类型的键, 以及其对应的 v2 类型的值的列表;
// 返回值是 v2 类型的值, 效果上看相当于输出了一个列表.
reduce    (k2,list(v2))   →   list(v2)
```

## 2.3 使用场景举例

- 分布式 grep 
  - map 函数处理模式匹配, 一旦匹配输出一行;
  - reduce 函数是一个等价函数(啥也不干), 将中间结果拷贝到输出.
- URL 访问计数
  - map 函数负责处理每个页面的请求日志, 每处理一行便输出 `<URL, 1>`.
  - reduce 函数负责将 URL 一样的值累加, 返回的是 `<URL, 累加值>`
- web 站点链接反转
  - map 函数针对每个在叫 source 的页面中发现的链接 target, 输出 `<target, source>`.
  - reduce 将每个 target 对应的 source 收集为一个列表并输出, 形如 `<target, list<source>>`.
- Term-Vector per Host
  - 背景: 检索词向量是对一个文档或者文档集合中最重要的单词及其词频的统计, 形式为 `<word, frequency>` 列表.
  - map 函数针对每个输入文档, 输出 `<hostname, term vector>`, 其中 hostname 是从文档对应的 URL 中抽取得到.
  - reduce 函数负责处理给定 host 的每个文档的检索词向量, 它将这些词向量加在在一起去除低频检索词, 输出 `<hostname, term vector>` 键值对.
- 倒排索引
  - map 函数解析每个文档, 输出一各 `<word, document ID>` 序列.
  - reduce 函数接受给定单词的全部键值对, 将对应的 document IDs 排序, 输出一个 `<word, list(document ID)>` 键值对.
- 分布式排序
  - map 函数从每个 record 抽取 key, 输出一个 `<key, record>` 键值对.
  - reduce 函数原封不动地输出键值对.
  - 该计算场景依赖于后面将要描述的分区设施和排序属性.

# 3 实现

模型很简单, 具体实现取决于硬件环境. 以 Google 为例(快二十年前的数据了):
- 双核 x86 处理器, 运行 Linux, 2-4GB 内存.
- 普通商用网络, 100Mb/s.
- 几百上千台上述机器构成的集群.
- 数据就保存在计算节点上, 普通的 IDE 磁盘. 不过这些数据由 GFS 管理, 确保高可用.
- 用户将 job 提交到调度系统. 每个 job 由多个 tasks 构成, 每个 job 被调度器映射到集群内的一组机器.

## 3.1 执行概览

系统自动将输入数据自动切割成 M 份, 然后在对应机器上部署多个 Mapper, 每个 Mapper 负责处理若干份数据. Mapper 处理输入生成中间数据, 通过分区函数(比如 hash(key) mod R)将中间数据的键空间分成 R 份, 并在其之上部署 Reducer. 具体的分区函数和分成几份, 由用户负责指定.

![Figure 1 ](/images/google-mapreduce-20210303/figure-1-execution-overview.png)

Figure 1 显示了一个 MapReduce 操作的执行概览. 用户程序调用 MapReduce 函数, 然后接下来框架内部陆续发生如下动作:
1. MapReduce 将输入切分成 M 份, 并在一组机器上启动多个用户程序拷贝(fork).
2. 上一步的 fork, 其中有 M 个 map workers, R 个 reduce workers, 还有一个特殊的作为 Master 负责分配任务.
3. map worker 负责读取和解析输入的 key/value 并传给用户定义的 Map 函数, 后者输出中间状态的 key'/value', 这些中间数据起初被缓存到内存中.
4. map worker 缓存的中间数据会被周期性的写到本地磁盘, 同时会被划分成 R 个分区(如 hash(key') mod R), 注意由于分区函数无法保证原空间和像空间一一映射, 所以每个分区的 key' 可能不唯一(比如 R 为 7, 则 key' 为 70 和 700 的落在同一个分区内). **这些中间数据 key'/value' 的位置会被上报给 Master, 它会负责把这些位置信息转发给 reduce workers, 每个 reduce worker 负责一个分区**. 尤其注意一点, 这一步的中间数据会被写到 map tasks 的本地磁盘, 而不是 GFS.
5. 当 reduce worker 收到上面提到的位置信息的时候, 它发起一个 RPC 读取那个 map workers 磁盘缓存的数据. 当数据都被读取过来之后, reduce worker 根据中间 key' 对数据进行排序, 于是相同的 keys 就会被排列到一起. **之所以需要排序, 是因为会有不同的 keys 落到同一个 reduce worker(毕竟像 hash(key') mod R 这种算法无法保证原空间和像空间是一一映射)**. 如果数据大到无法装进内存, reduce worker 就会采用外部排序算法.
6. **reduce worker 迭代排序后的数据, 针对每个唯一 key', 它会把其连同对应的一组 value' 传给用户编写的 Reduce 函数, 该函数输出会被追加到当前 reduce 分区的文件中**. 注意, 不同于 map tasks, reduce worker 的输出是写到 GFS.
7. 当全部 map tasks 和 reduce tasks 执行完成后, master 就会唤醒用户程序. MapReduce 调用返回至用户代码. 执行成功后, mapreduce 结果保存到了 R 个输出文件中(每个 reduce 任务一个输出文件). **一般用户无需合并这 R 个文件, 因为这些文件会被作为下个阶段的 MapReduce 调用的输入, 或者作为其它可以处理多个输入文件的分布式应用的输入**.

## 3.2 Master 节点的作用

master 保存着 map tasks 和 reduce tasks 的状态信息以及它们对应的机器 id. 

master 是一个将中间文件位置信息从 map 传递到 reduce 的中介.

master 保存 map tasks 生成的 R 个中间文件区域的位置信息和大小, 因为 map task 成功完成而产生的这些信息的更新都会被 master 接收并增量推送给正在执行的 reduce tasks.

## 3.3 容错

### 3.3.1 worker 故障

(这里的 worker 指的是机器.)

master 周期性 ping 各个 worker 来检活, 如果故障了, 则 master 会在其它机器上重新调度其上跑的 tasks.

故障机器上运行中的 map task 或者 reduce task 会被被重置为 idle 状态, 因此可以在其它 workers 上再次被调度.

失败的 map tasks 会在其它机器上调度重新执行一遍, 因为它们的输出都在故障机器本地磁盘上, 所以这些数据就丢了; 但是 reduce tasks 失败后在其它机器上被调度后无需从头重新执行, 因为它们的输出在类似 GFS 的分布式文件系统中, 继续从失败处继续运行即可.

如果 map task 换机器重新执行, 那么这个情况会被告知给全部 reduce workers, 毕竟这个 map task 输出的中间数据可能会覆盖全部 reduce workers 对应的分区.

MapReduce 可以容忍大批机器集体故障几分钟, 只需将故障机器上跑的任务重新在其它机器上重新调度执行就可以保证进度进行下去.

### 3.3.2 master 故障

除了心跳以外, 周期性 checkpoint 是提升容错的另一个利器. master 可以周期性的 checkpointing 自己的状态, 如果失效, 则从最后一个 checkpoint 重启新的 master 即可.

即使是单一 master 架构, 但也容易失效, 如果失效, 客户端可以选择重试 MapReduce 计算.

### 3.3.3 故障存在场景下的语义保证

当用户提供的 map 和 reduce 算子是其输入值的确定性函数时(绝大多数计算场景都这样)，即输入确定则输出也是确定的, 我们的分布式实现产生的输出与单体程序的无故障顺序执行所产生的输出相同。**但达成这一点依赖于 map 和 reduce 的输出能原子化地提交**, 下面详述.

每个进行中的 task 会生成自己的私有临时文件:
- 每个 reduce task 会生成一个文件; 
- 每个 map task 会生成 R 个文件, 每个文件对应一个 reduce task. 

其中, map task 被重新调度会丢弃之前的输出会重新从头计算, 成功完成后会将自己生成的 R 个文件的名字上报给 master, master 会将其记录到本地. 

reduce task 完成后会将自己的临时输出文件重命名为最终输出文件, 这个重命名过程是原子化的, 看过之前 GFS 分享的应该很清楚. 如果同样的 reduce task 因为故障被在多个机器上先后执行, 那么同一个最终输出文件会被重命名多次. 但由于输出都一样, 所以文件内容也都一样. 我们依赖底层文件系统如 GFS 提供的原子化重命名操作来保证最终的文件状态与同样的 reduce task 只运行一次结果相同.

前面说的都是算子是确定性函数的情形, 如果算子具有不确定性呢?

针对不确定性情形, 我们提供了比较弱但是仍合理的语义. 当存在不确定算子时, 某个 reduce task $R_{1}$ 的输出等价于一个不确定程序顺序执行时的输出, 而另一个 reduce task $R_{2}$ 的输出可能对应前述不确定程序的另一个顺序执行的输出. 考虑 map task $M$ 和 reduce tasks $R_{1}$ 和 $R_{2}$, 令 $e(R_{i})$ 代表 $R_{i}$ 的执行过程(一次恰好仅有一个该执行过程, 因为只有 task 故障了才会执行另一个). 因为 $e(R_{1})$ 可能读取了 $M$ 某次执行的输出, 而 $e(R_{2})$ 可能读取了 $M$ 的另一次执行的输出, 所以弱语义就保证了.

## 3.4 数据局部性

由 GFS 管理的输入数据就保存在 MapReduce 集群的磁盘上. MapReduce master 在调度 map 任务时会把输入文件的位置信息也考虑进来, 尽量把 map 任务调度到对应数据副本所在机器上, 如果该项尝试失败, 则将 map 任务调度到离着输入数据比较近的(同一局域网或同一个交换机连接的网络)机器上. 

在大型 MapReduce 计算过程中, 数据局部性可以极大地减少网络消耗.

## 3.5 任务颗粒度

**map 任务数 M 和 reduce 任务数 R 加起来要远大于 worker 机器数**. 一般一个 worker 同时执行多个任务, 这可以提升动态负载均衡.

master 要做出 $O(M + R)$ 调度策略, 在内存中持有 $O(M * R)$ 个状态(每个 map/reduce 对对应状态大约一个字节).

用户一般想要控制 R 的大小, 因为每个 reduce 任务单独输出一个文件, **控制 R 可以控制最终文件个数**. **我们倾向于选择大的 M 以使得每个 map 任务处理的数据量在 16MB 到 64MB 之间, 令 R 为 workers 的一个很小的倍数**. 比如, 如果有 2,000 workers, 那么 R 选为 5,000, 而 M 选为 200,000.

## 3.6 后备任务

拖长 mapreduce 计算时间的就是最后完成 map 或者 reduce 任务的机器. 原因一般是机器某些硬件比较差, 比如磁盘 IO 很慢; 或者集群调度系统(除了调度 MR 任务也调度其它的)把很多任务调度到了最后几个任务所在机器上导致资源争用严重.

我们有一个通用的缓解拖后腿问题的机制: 当一个 mapreduce 操作接近完成的时候 master 就会针对仍处于执行阶段的任务调度对应的后备任务, 不管主任务还是后备任务结束, 相关任务就会被标记为完成. 该机制大幅减少了大型 mapreduce 任务的时间消耗, 而资源消耗仅增加几个点.

# 4 调优

尽管对大部分需求, 写写 Map 和 Reduce 函数就够了, 但还是发现了一些可以优化的地方.

## 4.1 分区函数

用户在指定 reduce tasks 个数(也即输出文件个数) R 的值的时候, 也可以指定分区函数, 即如何根据中间数据 key 将数据分散到这 R 个文件. 默认的分区函数就是 `hash(key) mod R`. 

用户可以基于具体需求指定分区函数, 比如当中间 key 是 URL 时候, 如果想把同一个网站的数据刚到同一个文件, 则可以这样 `hash(Hostname(urlkey)) mod R`.

## 4.2 顺序保证

前面讲了, reduce task 在调用 Reduce 函数之前会将本分区数据就行排序, 所以可以保证一个分区内的中间数据会按照升序处理. 这使得为每个分区生成有序输出文件变得简单, 而且也使得针对在每个分区文件进行随机 key 查询变得高效(有序就可以二分查找了).

## 4.3 合并函数(combiner function)

在某些情况下, Reduce 函数满足交换性和结合性, 比如 word counting, 此时可以在每个中间 kv 通过网络传输给 Reduce 函数之前做一件事情, 即允许用户指定一个可选的 Combiner 函数, 它负责在**网络传输前**先对部分数据进行合并再发送, 这可以大幅减少网络数据交互量.

Combiner 函数在执行 map task 的机器上运行, 因为它针对的是 map 生成的数据. 

一般情况下, Combiner 函数代码与 Reduce 函数代码就是同一份. 这两者唯一不同就是 MapReduce  库如何处理它们的输出上. Reduce 函数的输出被写到最终输出文件上, 但是 Combiner 函数输出被写到一个中间文件, 该文件内容将会被发给 reduce task.

## 4.4 输入输出类型

MapReduce 库支持读取多种不同的文件类型. 比如 "text" 类型, 把每一行当作一个 key/value 对: key 是文件偏移量, value 是偏移处一行文件内容. 不管哪种文件类型, 库里对应的代码都知道如何把数据切分成有意义的范围给对应的 map task 去处理. 当然用户也可以实现相应接口来定制支持特定的文件类型.

MapReduce 对输出类型的支持也像输入一样灵活.


## 4.5 跳过坏记录

有时候用户编写的代码或者依赖的第三方库有 bugs, 导致 mapreduce 任务处理不了某些数据总是挂掉, 第一种情况还好说, 第三方库的就不好搞了. 这时候你可能希望能跳过这些记录. 怎么做到呢?

每个工作进程可以安装一个信号处理器用来捕获段错误或者总线错误. 在调用用户编写的 Map 或者 Reduce 之前, MapReduce 库存储一个序列号到全局变量中. 如果用户代码生成一个信号, 那么前面提到的信号处理器就会发送一个包含前述序列号的 UDP 包给 master. 如果 master 发现, 针对某个记录已经收到不止一次上报了, 它就会在重新执行挂掉的 Map/Reduce task 时跳过这条记录.

## 4.6 状态信息

master 除了做任务调度, 还提供了一个 HTTP server, 用户可以访问它来获取整个集群的状态统计信息. 比如计算进展, 每个 task 的输出等等.

## 4.7 全局计数器

MapReduce 库提供了计数器设施, 用户可以利用它在 Map/Reduce 函数中针对一些事件进行统计, 比如在单词计数应用中针对大写的单词进行统计:

```c++
Counter* uppercase;
uppercase = GetCounter("uppercase");
map(String name, String contents):
  for each word w in contents:
    // 如果当前单词大写, 则将全局计数器加 1
    if (IsCapitalized(w)):
      uppercase->Increment();
    EmitIntermediate(w, "1");
```

这些计数值会周期性地随心跳响应传递到 master, 然后 master 进行聚合, 聚合时 master 会去重(比如重复调度执行的任务或为了加速完成而启动的后备任务都会造成重复计数). 用户可以从前面提到的 http server 页面查看值的变化.

# 5 性能

下面以排序程序为例, 说明下各阶段数据传输速率比较以及后备任务对性能的影响.

![Figure 2](/images/google-mapreduce-20210303/figure-2-data-transfer-rate-for-different-executions-of-sort.png)

如上图所示, 横向分为三部分, 分别是正常执行情况, 无后备任务执行情况, 手动干掉 200 个进程的执行情况. 其中, 每种执行情况纵向列出三个指标, 分别是输入数据速率, 排序数据速率, 输出数据速率. 具体每个图, 横轴是时间, 纵轴是速率.

1. a 列上图是数据读取速率, 显著快于下面的排序和输出, 这全都拜前面提到的数据局部性所赐.
2. a 列中图是排序, 可以看到第一个 map task 完成后即启动了排序. 第一个高峰是 1700 个 reduce tasks 执行盛况(这个 sort 程序用了 1700 台机器, 每个机器执行不超过 1 个 reduce task.), 大约 300 秒后第一批数据排序完成, 然后对剩下数据进行排序, 大约 600 秒时完成全部排序任务. 从该图可以看出数据传输速率高于下面的输出, 原因是下面输出要写到 GFS 多副本, 比较耗时.
3. a 列下图是输出, 输出就是 reduce tasks 将数据写入到 GFS. 可以看到第一批数据排序完成到开始输出有一个延迟, 原因是这段时间内机器忙着排序中间数据. 
4. b 列下图显示最终完成时间要显著多余 a 列, 原因是最后 5 个 reduce tasks 严重拖后腿了, 整体耗时增长 44%. 这可以看出后备任务的对性能的显著提升.
5. c 列上图显示显示手动干掉 200 个进程(机器还正常运行)后速率变成负的了, 原因是部分 map tasks 丢了, 需要重跑. 集群调度器快速在这些机器上重新运行相关任务, 最后 c 列下图显示仅仅比正常情况多了 5% 耗时.

--End--




