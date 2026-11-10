# 从外部归并排序到 Hadoop MapReduce

## 是外部归并排序直接催生了 MapReduce 吗？

**不能简单地说是。**外部归并排序与 Hadoop MapReduce 有密切的技术联系：Hadoop MapReduce 在处理中间数据时会使用排序和归并。但仅凭这种联系，不能断定外部归并排序直接导致了 MapReduce 的诞生。

Google 在 2004 年发表的 [MapReduce 论文](https://research.google/pubs/mapreduce-simplified-data-processing-on-large-clusters/)提出了一种面向大规模集群数据处理的编程模型及实现。它的目标不仅是排序，还包括将计算并行化，并由运行时处理数据分区、任务调度、机器故障和节点间通信。Hadoop MapReduce 是这一编程模型的一种实现。

因此，更准确的理解是：**外部归并排序是 Hadoop MapReduce 处理大量中间数据时使用的重要技术；MapReduce 则是一套解决分布式数据处理问题的更广泛模型和运行机制。**

## 相似之处：拆分、局部处理、合并

单机外部排序和 MapReduce 都会将大任务拆成可处理的部分，再协调中间结果，但拆分和合并的目的并不完全相同。

| | 单机外部归并排序 | Hadoop MapReduce |
| --- | --- | --- |
| 目标 | 对超过内存容量的数据排序 | 在集群上处理大规模数据，排序只是可能的任务之一 |
| 输入拆分 | 分批读取，生成有序 run | 将输入划分为 split，交给 map 任务处理 |
| 局部处理 | 内存排序后写出有序 run | map 生成中间键值对，框架按分区和键组织中间数据 |
| 合并 | 多路归并有序 run，生成排序结果 | 将中间数据传给对应 reducer，归并排序后按键分组处理 |
| 主要限制 | 内存、磁盘空间和 I/O | 除单机资源外，还涉及网络、调度、数据倾斜和机器故障 |

这里的对应关系只是帮助理解，**map 任务不是简单地生成一个有序 run，reduce 也不只是归并排序**。例如，map 可以提取、转换或过滤记录；reduce 可以对同一个键的值求和，而不仅是输出排序结果。

## Hadoop 中排序和归并出现在哪里？

以包含 reducer 的 Hadoop MapReduce 作业为例：

```text
输入 split
    ↓
map：生成中间键值对
    ↓
按目标 reducer 分区；中间数据排序、必要时溢写到磁盘并归并
    ↓
shuffle：reducer 获取各 map 对应分区的数据
    ↓
归并排序、按键分组
    ↓
reduce：处理每个键及其值
```

一个 map 任务可能产生超过内存容量的中间数据。框架可以将缓冲中的数据排序并溢写到磁盘，再归并中间文件。这与单机外部排序“内存容不下就生成有序段、再归并”的做法相似。

在 reducer 侧，框架从各个 mapper 获取对应分区的排序数据并进行归并，然后把相同键的值分组交给 `reduce`。Hadoop 的 [Reducer 文档](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapreduce/Reducer.html)明确描述了 shuffle 和按键归并排序；获取数据和归并可以同时进行。具体的缓冲、溢写和归并行为取决于框架配置及数据量。

例如，词频统计中 map 输出 `(word, 1)`，框架将同一个单词的数据送到同一 reducer 并分组，reduce 再对这些 `1` 求和。**排序和归并是组织中间数据的手段，词频汇总才是这个作业的目标。**

## MapReduce 默认会产生全局排序结果吗？

**不会。**要区分以下两件事：

1. 框架按键排序的是 reducer 的**输入**；reducer 可以输出不同的键和值，框架不会自动重新排序 reducer 的输出。
2. 多个 reducer 各自处理一个分区。默认的哈希分区并不保证不同分区的键范围有序，所以即使各输出文件内部有序，也不能直接按文件顺序拼成全局有序结果。

若要用多个 reducer 实现全局排序，可以按键的范围分区：先确定有序的分区边界，让前一个分区的键不大于后一个分区的键；每个 reducer 按输入顺序输出保留排序键的记录；最后按分区顺序读取或拼接输出。还需确保分区、排序和输出使用一致的比较规则，并处理重复键和数据倾斜。Hadoop 提供的 [TotalOrderPartitioner](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapreduce/lib/partition/TotalOrderPartitioner.html) 可以利用预先生成的分区边界实现全序分区。

如果只使用一个 reducer，框架会将所有中间数据送到它那里并按键排序；但最终输出仍要由 reducer 按适当顺序写出，而且单个 reducer 会限制并行度。

## MapReduce 比单机排序多解决了什么？

- **分布式调度**：将任务分配到多台机器并并行执行。
- **网络数据交换**：按分区把各 mapper 的中间数据交给相应 reducer。
- **容错**：处理任务或机器故障，必要时重新执行任务。
- **通用计算模型**：通过 map 和 reduce 表达计数、聚合等多种处理任务，而不局限于排序。

这些能力使 MapReduce 能够利用集群处理数据，但也引入了网络传输、分区倾斜、任务调度等单机外部排序没有的成本和复杂性。单机处理 1 TB 的题目不需要为了使用外部归并排序而引入 Hadoop；是否使用集群还取决于时间要求、硬件和运维成本。

## 参考资料

- [Google：MapReduce: Simplified Data Processing on Large Clusters](https://research.google/pubs/mapreduce-simplified-data-processing-on-large-clusters/)
- [Apache Hadoop：MapReduce Tutorial](https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html)
- [Apache Hadoop：Reducer API](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapreduce/Reducer.html)
- [Apache Hadoop：TotalOrderPartitioner API](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapreduce/lib/partition/TotalOrderPartitioner.html)
