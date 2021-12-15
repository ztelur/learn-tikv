本文大部分来自于 https://pingcap.com/zh/blog/tidb-transaction-model



### 事务模型概览

TiKV 是 Pincap 团队按照论文做 Spanner 的开源实现。支持分布式事务和水平扩展的 KV 数据库。一个分布式数据库涉及的技术面非常广泛， **今天我们主要探讨的是 TiKV 的 MVCC（多版本并发控制） 和 Transaction 实现。**



#### MVCC



MySQL 中用于避免大量的悲观锁，不过它是一个思维，最直观的实现就是每个 KV 对，都有一个 version 字段。



TiKV 底层依赖 RocksDB，同一个 key 会存储同一个 key 的多个版本。RocksDB 数据存储是有序的 KV Pairs，对于有序 key 的迭代的场景访问效率较高。

```
MetaKey                      --> key 的所有版本信息
DataKey(key+version_1)-->Value_v1
DataKey(key+version_2)-->Value_v2
```



**核心思想是，先读取 meta key 然后通过 meta key 中找到相应的可见版本，然后再读取 data key，由于这些 key 都拥有相同的前缀，所以在实际的访问中，读放大的程度是可以接受的。**



当某个key更新较为频繁时，会导致meta key 对应的value过大，需要进行拆解。对 meta key 建立索引，将一个 meta key 变成多个 meta key





#### 分布式事务模型



本质上来说，在一个分布式系统中实现跨节点事务，只有两阶段提交一种办法（3PC 本质上也是 2PC 的一个优化）。

Spanner 是引入了TrueTime API 来作为事务 ID 生成器从而实现了 Serializable 的隔离级别

TiKV 中最终选择的事务模型和 Spanner 有所区别，采用了 Google 的另一套分布式事务方案 Percolator 的模型。



Percolator 是 Google 的上一代分布式事务解决方案，构建在 BigTable 之上，在 Google 内部用于网页索引更新的业务。原理比较简单，总体来说就是一个经过优化的 2PC 的实现，依赖一个单点的授时服务 TSO 来实现单调递增的事务编号生成，提供 SI 的隔离级别。



传统的分布式事务模型中，一般都会有一个中央节点作为事务管理器，Percolator 的模型通过对于锁的优化，去掉了单点的事务管理器的概念，将整个事务模型中的单点局限于授时服务器上，在生产环境中，单点授时是可以接受的，因为 TSO 的逻辑极其简单，只需要保证对于每一个请求返回单调递增的 id 即可，通过一些简单的优化手段（比如 pipeline）性能可以达到每秒生成百万 id 以上，同时 TSO 本身的高可用方案也非常好做，所以整个 Percolator 模型的分布式程度很高。



Percolator 论文 https://research.google/pubs/pub36726.pdf

相关文章

-  [《Deep Dive TiKV - Percolator》 ](https://tikv.org/deep-dive/distributed-transaction/percolator/)

- [《TiKV 事务模型概览》 ](https://pingcap.com/blog-cn/tidb-transaction-model/)



#### 具体过程



TiKV 的读写事务分为两个阶段：1、Prewrite 阶段；2、Commit 阶段。



客户端会缓存本地的写操作，在客户端调用 client.Commit() 时，开始进入分布式事务 prewrite 和 commit 流程。

Prewrite

- 首先在所有行的写操作中选出一个作为 primary row，其他的为 secondary rows
- PrewritePrimary: 对 primaryRow 写入锁（修改 meta key 加入一个标记），锁中记录本次事务的开始时间戳。上锁前会检查：
  - 该行是否已经有别的客户端已经上锁 (Locking)
  - 是否在本次事务开始时间之后，检查versions ，是否有更新 [startTs, +Inf) 的写操作已经提交 (Conflict)
- 将 primaryRow 的锁上好了以后，进行 secondaries 的 prewrite 流程：
  - 类似 primaryRow 的上锁流程，只不过锁的内容为事务开始时间 startTs 及 primaryRow 的信息
  - 检查的事项同 primaryRow 的一致
  - 当锁成功写入后，写入 row，时间戳设置为 startTs



以上 Prewrite 流程任何一步发生错误，都会进行回滚：删除 meta 中的 Lock 标记 , 删除版本为 startTs 的数据。



Commit 的流程

1. commit primary: 写入 meta 添加一个新版本，时间戳为 commitTs，内容为 startTs, 表明数据的最新版本是 startTs 对应的数据
2. 删除 Lock 标记





值得注意的是，如果 primary row 提交失败的话，全事务回滚，回滚逻辑同 prewrite 失败的回滚逻辑。

如果 commit primary 成功，则可以异步的 commit secondaries，流程和 commit primary 一致， 失败了也无所谓。Primary row 提交的成功与否标志着整个事务是否提交成功。



#### 源码过程



在 Percolator 的设计中，分布式事务的算法都在客户端的代码中，这些客户端代码直接访问 BigTable。TiKV 的设计与 Percolator 在这一方面也有些类似。TiKV 以 Region 为单位来接受读写请求，需要跨 Region 的逻辑都在 TiKV 的客户端中，如 TiDB





客户端的代码会将请求切分并发送到对应的 Region。也就是说，正确地进行事务需要客户端和 TiKV 的紧密配合。本篇文章为了讲解完整的事务流程，也会提及 TiDB 的 tikv client 部分的代码（位于 TiDB 代码的 `store/tikv`目录），大家也可以参考《TiDB 源码阅读系列文章》的 [《TiDB 源码阅读系列文章（十八）tikv-client（上）》 ](https://pingcap.com/blog-cn/tidb-source-code-reading-18/)和 [《TiDB 源码阅读系列文章（十九）tikv-client（下）》 ](https://pingcap.com/blog-cn/tidb-source-code-reading-19/)



由于采用的是乐观事务模型，写入会缓存到一个 buffer 中，直到最终提交时数据才会被写入到 TiKV；而一个事务又应当能够读取到自己进行的写操作，因而一个事务中的读操作需要首先尝试读自己的 buffer，如果没有的话才会读取 TiKV





![图表](https://img1.www.pingcap.com/prod/1_62ed82f6c5.png)

#### prewrite



第一步是 prewrite，即将此事务涉及写入的所有 key 上锁并写入 value。在 client 一端，需要写入的 key 被按 Region 划分，每个 Region 的请求被并行地发送。请求中会带上事务的 `start_ts`和选取的 primary key。





TiKV 的 [`kv_prewrite`](https://github.com/tikv/tikv/blob/5024ad08fc7101ba25f17c46b0264cd27d733bb1/src/server/service/kv.rs#L114)接口会被调用来处理这一请求。接下来，请求被交给 [`Storage::async_prewrite`](https://github.com/tikv/tikv/blob/5024ad08fc7101ba25f17c46b0264cd27d733bb1/src/storage/mod.rs#L1047)来处理，`async_prewrite`则将任务交给 [`Scheduler`](https://github.com/tikv/tikv/blob/5024ad08fc7101ba25f17c46b0264cd27d733bb1/src/storage/txn/scheduler.rs#L239)。





`Scheduler`负责调度 TiKV 收到的读写请求，进行流控，从 engine 取得 snapshot（用于读取数据），最后执行任务。Prewrite 最终在 [`process_write_impl`](https://github.com/tikv/tikv/blob/5024ad08fc7101ba25f17c46b0264cd27d733bb1/src/storage/txn/process.rs#L523)中被实际进行。