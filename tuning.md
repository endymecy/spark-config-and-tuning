# spark性能调优

> 当你开始编写`Apache Spark`代码或者浏览公开的`API`的时候，你会遇到诸如`transformation，action，RDD`等术语。了解到这些是编写`Spark`代码的基础。同样，当你任务开始失败或者你需要透过`web`界面去了解自己的应用为何如此费时的时候，你需要去了解一些新的名词:`job, stage, task`。对于这些新术语的理解有助于编写良好`Spark`代码。这里的良好主要指更快的`Spark`程序。对于`Spark`底层的执行模型的了解对于写出效率更高的`Spark`程序非常有帮助。

# 1 Spark 是如何执行程序的

&emsp;&emsp;一个`Spark`应用包括一个`driver`进程和若干个分布在集群的各个节点上的`executor`进程。<br><br/>

&emsp;&emsp;`driver`主要负责调度一些高层次的任务流（`flow of work`）。`exectuor`负责执行这些任务，这些任务以`task`的形式存在， 同时存储用户设置需要`caching`的数据。 `task`和所有的`executor`的生命周期为程序的整个运行过程（如果使用了`dynamic resource allocation`时可能不是这样的）。一个`executor`可以运行多个任务，任务的运行是并行的。如何调度这些进程是通过集群管理框架完成的（比如`YARN，Mesos，Spark Standalone`），任何一个`Spark`程序都会包含一个`driver`和多个`executor`进程。<br><br/>

<div  align="center"><img src="http://blog.cloudera.com/wp-content/uploads/2015/02/spark-tuning-f1.png" alt="1.1" align="center" /></div><br><br/>

&emsp;&emsp;如上图，在执行层次结构的最上方是一系列`Job`。调用一个`Spark`内部的`action`会产生一个`Spark job`来完成它。为了确定这些`job`的实际内容，`Spark`检查`RDD`的`DAG`再计算出执行`plan`。这个`plan`以最远端的`RDD`为起点（最远端指的是对外没有依赖的`RDD`或者数据已经缓存下来的 `RDD`），以产生结果`RDD`的`action`为结束 。<br><br/>

&emsp;&emsp;执行的`plan`由一系列`stage`组成，`stage`是`job`的`transformation`的组合，`stage` 对应于一系列 `task`， `task` 指的对于不同的数据集执行的相同代码。每个`stage`包含**不需要`shuffle`所有数据的`transformation`的序列**。<br><br/>

&emsp;&emsp;什么决定数据是否需要`shuffle`呢？`RDD`包含固定数目的 `partition`， 每个 `partiton` 包含若干的 `record`。对于那些通过 `narrow tansformation`（窄依赖，比如 `map` 和 `filter`）返回的` RDD`，一个 `partition` 中的 `record` 只需要从父` RDD` 对应的 `partition` 中的 `record` 计算得到。每个对象只依赖于父 `RDD` 的一个对象。有些操作（比如 `coalesce`）可能导致一个 `task` 处理多个输入` partition` ，但是这种` transformation` 仍然被认为是窄的，因为用于计算的多个输入 `record` 始终是来自有限个数的 `partition`。<br><br/>

&emsp;&emsp;然而`Spark`也支持宽依赖的 `transformation`，比如 `groupByKey`，`reduceByKey`。在这种依赖中，计算得到一个 `partition` 中的数据需要从父 `RDD` 中的多个 `partition` 中读取数据。所有拥有相同 `key` 的元组最终会被聚合到同一个 `partition` 中，被同一个 `stage` 处理。为了完成这种操作， `Spark`需要对数据进行 `shuffle`，意味着数据需要在集群内传递，最终生成由新的`partition` 集合组成的新的 `stage`。<br><br/>


&emsp;&emsp;在下面的代码中，只有一个 `action` 以及一系列处理文本的 `transformation`， 这些代码就只有一个 `stage`，因为没有哪个操作需要从不同的 `partition` 里面读取数据。<br><br/>

```scala
sc.textFile("someFile.txt").
  map(mapFunc).
  flatMap(flatMapFunc).
  filter(filterFunc).
  count()
```

&emsp;&emsp;跟上面的代码不同，下面一段代码需要统计总共出现超过1000次的字母。<br><br/>

```scala
val tokenized = sc.textFile(args(0)).flatMap(_.split(' '))
val wordCounts = tokenized.map((_, 1)).reduceByKey(_ + _)
val filtered = wordCounts.filter(_._2 >= 1000)
val charCounts = filtered.flatMap(_._1.toCharArray).map((_, 1)).
  reduceByKey(_ + _)
charCounts.collect()
```
&emsp;&emsp;这段代码可以分成三个`stage`。`recudeByKey` 操作是各 `stage` 之间的分界，因为计算 `recudeByKey` 的输出需要按照可以重新分配 `partition`。<br><br/>

&emsp;&emsp;这里还有一个更加复杂的 `transfromation` 图，包含一个有多路依赖的 `join transformation`。

<div  align="center"><img src="http://blog.cloudera.com/wp-content/uploads/2015/02/spark-tuning-f2.png" alt="1.2" align="center" /></div><br><br/>

&emsp;&emsp;粉红色的框框展示了运行时使用的 `stage` 图。

<div  align="center"><img src="http://blog.cloudera.com/wp-content/uploads/2015/02/spark-tuning-f3.png" alt="1.3" align="center" /></div><br><br/>

&emsp;&emsp;运行到每个 `stage` 的边界时，数据在父 `stage` 中通过 `task` 写到磁盘上，而在子 `stage` 中经过网络通过 `task`去读取数据。这些操作会导致很重的网络以及磁盘的`I/O`，所以 `stage` 的边界是非常占资源的，在编写 `Spark` 程序的时候需要尽量避免的。父 `stage` 中 `partition` 个数与子 `stage` 的 `partition` 个数可能不同，所以那些产生 `stage` 边界的 `transformation` 常常需要接受一个`numPartition` 的参数来决定子 `stage` 中的数据将被切分为多少个 `partition`。<br><br/>

&emsp;&emsp;正如在调试 `MapReduce` 是选择 `reducor` 的个数是一项非常重要的参数，调整在 `stage` 边届时的 `partition` 个数经常可以很大程度上影响程序的执行效率。我们会在后面的章节中讨论如何调整这些值。

# 2 选择正确的 Operator

&emsp;&emsp;当需要使用 `Spark` 完成某项功能时，程序员需要从不同的 `action`和`transformation`中选择不同的方案以获得相同的结果。但是不同的方案，最后执行的效率可能有云泥之别。避免常见的陷阱,选择正确的方案可以使得最后的表现有巨大的不同。一些规则和深入的理解可以帮助你做出更好的选择。<br></br>

&emsp;&emsp;选择 `Operator` 方案的主要目标是减少 `shuffle` 的次数以及被 `shuffle` 的文件的大小。因为 `shuffle` 是最耗资源的操作，所有 `shuffle` 的数据都需要写到磁盘并且通过网络传递。`repartition`，`join`，`cogroup`，以及任何 `*By` 或者 `*ByKey` 的 `transformation` 都需要 `shuffle` 数据。这些 `Operator` 不是所有都是平等的，但是有些常见的性能陷阱是需要注意的。<br></br>

- **当进行联合规约操作时，避免使用 `groupByKey`。**举个例子，`rdd.groupByKey().mapValues(_ .sum)` 与 `rdd.reduceByKey(_ + _)` 执行的结果是一样的，但是前者需要把全部的数据通过网络传递一遍，而后者只需要根据每个 `key` 局部的 `partition` 累积结果，在 `shuffle` 的之后把局部的累积值相加后得到结果。
- **当输入和输入的类型不一致时，避免使用 `reduceByKey`。**举个例子，我们需要实现为每一个`key`查找所有不相同的 `string`。一个方法是利用`map` 把每个元素的转换成一个 `Set`，再使用 `reduceByKey` 将这些 `Set` 合并起来

```scala
rdd.map(kv => (kv._1, new Set[String]() + kv._2))
    .reduceByKey(_ ++ _)
```
&emsp;&emsp;这段代码生成了无数的非必须的对象，因为需要为每个 `record` 新建一个` Set`。这里使用 `aggregateByKey` 更加适合，因为这个操作是在 `map` 阶段做聚合。<br></br>

```scala
val zero = new collection.mutable.Set[String]()
rdd.aggregateByKey(zero)(
    (set, v) => set += v,
    (set1, set2) => set1 ++= set2)
```
- **避免 flatMap-join-groupBy 的模式。**当有两个已经按照`key`分组的数据集，你希望将两个数据集合并，并且保持分组，这种情况可以使用 `cogroup`。这样可以避免对`group`进行装箱拆箱的开销。

# 3 什么时候不发生 Shuffle

&emsp;&emsp;当然了解在哪些 `transformation` 上不会发生 `shuffle` 也是非常重要的。当前一个 `transformation` 已经用相同的 `patitioner` 把数据分区了，`Spark`知道如何避免 `shuffle`。参考以下代码：<br></br>

```scala
rdd1 = someRdd.reduceByKey(...)
rdd2 = someOtherRdd.reduceByKey(...)
rdd3 = rdd1.join(rdd2)
```

&emsp;&emsp;因为没有 `partitioner` 传递给 `reduceByKey`，所以系统使用默认的 `partitioner`，所以 `rdd1` 和 `rdd2` 都会使用 `hash` 进行分区。代码中的两个 `reduceByKey` 会发生两次 `shuffle` 。如果 `RDD` 包含相同个数的 `partition`， `join` 的时候将不会发生额外的 `shuffle`。因为这里的 `RDD` 使用相同的 `hash` 方式进行 `partition`，所以全部 `RDD` 中同一个 `partition` 中的 `key`的集合都是相同的。因此，`rdd3`中一个 `partiton` 的输出只依赖`rdd2`和`rdd1`的同一个对应的 `partition`，所以第三次 `shuffle` 是不必要的。<br></br>

&emsp;&emsp;举个例子说，当 `someRdd` 有4个 `partition`， `someOtherRdd` 有两个 `partition`，两个 `reduceByKey` 都使用3个 `partiton`，所有的 `task` 会按照如下的方式执行：<br></br>

<div  align="center"><img src="http://blog.cloudera.com/wp-content/uploads/2015/02/spark-tuning-f4.png" alt="1.4" align="center" /></div><br><br/>

&emsp;&emsp;如果 `rdd1` 和 `rdd2` 在 `reduceByKey` 时使用不同的 `partitioner` 或者使用默认的 `partitioner`, 但是 `partition` 的个数不同，那么在`join`时只用一个 `RDD` (`partiton` 数更少的那个)需要重新 `shuffle`。

&emsp;&emsp;相同的 `tansformation`，相同的输入，不同的 `partition` 个数的情况：

<div  align="center"><img src="http://blog.cloudera.com/wp-content/uploads/2015/02/spark-tuning-f5.png" alt="1.5" align="center" /></div><br><br/>

&emsp;&emsp;当两个数据集需要 `join` 时，避免 `shuffle` 的一个方法是使用 `broadcast variables`。如果一个数据集小到能够塞进一个 `executor` 的内存中，那么它就可以在 `driver` 中写入到一个 `hash table`中，然后 `broadcast` 到所有的 `executor` 中。然后 `map transformation` 可以引用这个 `hash table` 作查询。

# 4 什么情况下 Shuffle 越多越好

&emsp;&emsp;尽可能减少 `shuffle` 的准则也有例外的场合。如果额外的 `shuffle` 能够增加并发那么这也能够提高性能。比如当你的数据保存在几个没有切分过的大文件中时，那么使用 `InputFormat` 产生分 `partition` 可能会导致每个 `partiton` 中聚集了大量的 `record`，如果 `partition` 不够，导致没有启动足够的并发。在这种情况下，我们需要在数据载入之后使用`repartiton` （会导致`shuffle`)提高 `partiton` 的个数，这样能够充分使用集群的`CPU`。<br></br>

&emsp;&emsp;另外一种例外情况是在使用 `recude` 或者 `aggregate action` 聚集数据到 `driver` 时，如果把 `partititon` 个数很多的数据进行聚合时，单进程执行的 `driver` `merge` 所有 `partition` 的输出时很容易成为计算的瓶颈。为了缓解 `driver` 的计算压力，可以使用 `reduceByKey` 或者 `aggregateByKey` 执行分布式的 `aggregate` 操作把数据分布到更少的 `partition` 上。每个 `partition` 中的数据并行的进行 `merge`，再把 `merge` 的结果发个` driver` 以进行最后一轮 `aggregation`。查看 `treeReduce` 和 `treeAggregate` 查看如何这么使用的例子。<br><br/>

&emsp;&emsp;这个技巧在已经按照 `Key` 聚集的数据集上格外有效，比如当一个应用是需要统计一个语料库中每个单词出现的次数，并且把结果输出到一个`map`中。一个实现的方式是使用 `aggregation`，在每个 `partition` 中本地计算一个 `map`，然后在 `drive`r 中把各个 `partition` 中计算的 `map merge` 起来。另一种方式是通过 `aggregateByKey` 把 `merge` 的操作分布到各个 `partiton` 中计算，然后在简单地通过 `collectAsMap` 把结果输出到 `driver` 中。

# 5 二次排序

&emsp;&emsp;还有一个重要的技能是了解接口 `repartitionAndSortWithinPartitions`。这是一个听起来很晦涩的 `transformation`，但是却能涵盖各种奇怪情况下的排序，这个 `transformation` 把排序推迟到 `shuffle` 操作中，这使大量的数据有效的输出，排序操作可以和其他操作合并。<br></br>

&emsp;&emsp;例如，`Apache Hive on Spark` 在`join`的实现中，使用了这个 `transformation` 。而且这个操作在 `secondary sort` 模式中扮演着至关重要的角色。`secondary sort` 模式是指用户期望数据按照 `key` 分组，并且希望按照特定的顺序遍历 `value`。使用 `repartitionAndSortWithinPartitions` 再加上一部分用户的额外的工作可以实现 `secondary sort`。

# 6 调试资源分配

&emsp;&emsp;在本章中，你将学会压榨出你集群的每一分资源。`spark`推荐的配置将根据不同的集群管理系统（ `YARN、Mesos、Spark Standalone`）而有所不同，我们将主要集中在 `YARN` 上，因为这个是 `Cloudera` 推荐的方式。<br></br>

&emsp;&emsp;`Spark`（以及`YARN`） 需要关心的两项主要的资源是 `CPU` 和 内存， 磁盘和 `IO` 当然也影响着 `Spark` 的性能，但是不管是 `Spark` 还是 `Yarn` 目前都没法对他们做实时有效的管理。<br></br>

&emsp;&emsp;在一个 `Spark` 应用中，每个 `Spark executor` 拥有固定个数的 `core` 以及固定大小的堆大小。core 的个数可以在执行 `spark-submit` 或者`spark-shell` 时，通过参数` --executor-cores `指定，也可以在 `spark-defaults.conf` 配置文件或者 `SparkConf` 对象中设置 `spark.executor.cores` 参数。同样地，堆的大小可以通过 `--executor-memory` 参数或者 `spark.executor.memory` 配置项配置。`core` 配置项控制一个 `executor` 中`task`的并发数。 `--executor-cores 5 `意味着每个` executor` 中最多同时可以有5个 `task` 运行。`memory` 参数影响 `Spark` 可以缓存的数据的大小，也就是在 `group,aggregate` 以及 `join` 操作时 `shuffle` 的数据结构的最大值。<br></br>

&emsp;&emsp;`--num-executors` 命令行参数或者`spark.executor.instances` 配置项控制需要的 `executor` 个数。从 `Spark 1.3` 开始，你可以避免使用这个参数，只要你通过设置 `spark.dynamicAllocation.enabled` 参数打开动态分配 。动态分配可以使的 `Spark` 的应用在有积压的等待 `task` 时请求 `executor`，并且在空闲时释放这些 `executor`。<br></br>

&emsp;&emsp;同时 `Spark` 需求的资源如何跟 `YARN` 中可用的资源配合也是需要着重考虑的，`YARN` 相关的参数有：<br></br>

- `yarn.nodemanager.resource.memory-mb` 控制在每个节点上 `container` 能够使用的最大内存；
- `yarn.nodemanager.resource.cpu-vcores` 控制在每个节点上 `container` 能够使用的最大`core`个数；

<br><br/>

&emsp;&emsp;请求5个 `core` 会生成向 `YARN` 要5个虚拟`core`的请求。但是从 `YARN`　请求内存相对比较复杂，因为以下的一些原因：<br></br>

- `--executor-memory/spark.executor.memory` 控制 `executor` 的堆的大小，但是 `JVM` 本身也会占用一定的堆空间，比如 `interned String` 或者 `direct byte buffer`， `spark.yarn.executor.memoryOverhead` 属性决定了向 `YARN` 请求的每个 `executor` 的内存大小，默认值为`max(384, 0.7 * spark.executor.memory)`;
- `YARN` 可能会比请求的内存高一点，`YARN` 的 `yarn.scheduler.minimum-allocation-mb` 和 `yarn.scheduler.increment-allocation-mb` 属性控制请求的最小值和增加量。

<br></br>

&emsp;&emsp;下面展示的是 `Spark on YARN` 内存结构：

<div  align="center"><img src="http://blog.cloudera.com/wp-content/uploads/2015/03/spark-tuning2-f1.png" alt="2.1" align="center" /></div><br><br/>

&emsp;&emsp;如果以上这些还不够决定`Spark executor` 个数，那么可以考虑下面一些概念：<br></br>

- 应用程序的`master`，是一个非 `executor` 的容器，它拥有从 `YARN` 请求资源的能力，它自己本身所占的资源也需要被计算在内。在 `yarn-client` 模式下，它默认请求 `1024MB` 和 1个`core`。在 `yarn-cluster` 模式中，应用的 `master` 运行 `driver`，所以使用参数 `--driver-memory` 和 `--driver-cores` 配置它的资源常常很有用。
- 在 `executor` 执行的时候配置过大的 `memory` 经常会导致过长的`GC`延时，`64G`是推荐的一个` executor `内存大小的上限。
- 我们注意到 `HDFS client` 在大量并发线程时会有性能问题。大概的估计是每个`executor` 中最多5个并行的 `task` 就可以占满的写入带宽。
- 运行微型 `executor` （比如只有一个core而且只有够执行一个task的内存）会抛弃在一个`JVM`上同时运行多个`task`的好处。比如 `broadcast` 变量需要为每个 `executor` 复制一遍，这么多小`executor`会导致更多的数据拷贝。

<br></br>

&emsp;&emsp;为了让上面的说明更具体一点。我们举例子说明如何完全用满整个集群的资源。<br></br>

&emsp;&emsp;假设一个集群中`NodeManager`运行在6个节点上，每个节点有16个`core`以及`64GB`的内存。那么`NodeManager`的容量为：`yarn.nodemanager.resource.memory-mb` 和 `yarn.nodemanager.resource.cpu-vcores` 可以设为 `63 * 1024 = 64512 （MB）` 和 15。我们避免使用 100% 的 `YARN` `container` 资源，因为还要为 `OS` 和 `hadoop` 的 `Daemon` 留一部分资源。在上面的场景中，我们预留了1个`core`和1G的内存给这些进程。<br></br>

&emsp;&emsp;所以看起来我们最先想到的配置会是这样的：`--num-executors 6 --executor-cores 15 --executor-memory 63G`。但是这个配置可能无法达到我们的需求，因为：<br></br>

- `63GB+` 的`executor memory` 塞不进只有63GB容量的 `NodeManager`；
- 应用程序的 `master` 也需要占用一个`core`，意味着在某个节点上，没有15个`core`给 `executor` 使用；
- 15个`core`会影响`HDFS IO`的吞吐量。


&emsp;&emsp;配置成 `--num-executors 17 --executor-cores 5 --executor-memory 19G` 可能会效果更好，因为： <br></br>
- 这个配置会在每个节点上生成3个 `executor`，除了应用的`master`运行的机器，这台机器上只会运行2个 `executor`;
- `--executor-memory` 被分成3份（63G/每个节点3个executor）=21。 `21 * （1 - 0.07） ~ 19`。

# 7 调试并发

&emsp;&emsp;我们知道 `Spark` 是一套数据并行处理的引擎。但是 `Spark` 并不是神奇得能够将所有计算并行化，它没办法从所有的并行化方案中找出最优的那个。每个 `Spark stage` 中包含若干个 `task`，每个 `task` 串行地处理数据。在调试 `Spark` 的`job`时，`task` 的个数可能是决定程序性能的最重要的参数。<br></br>

&emsp;&emsp;那么这个数字是由什么决定的呢？前文介绍了 `Spark` 如何将 `RDD` 转换成一组 `stage`。`task` 的个数与 `stage` 中上一个 `RDD` 的 `partition` 个数相同。而一个 `RDD` 的 `partition` 个数与被它依赖的 `RDD` 的 `partition` 个数相同。除了以下的情况： `coalesce transformation` 可以创建一个具有更少 `partition` 个数的` RDD`，`union transformation` 产出的 `RDD` 的 `partition` 个数是它父 `RDD` 的 `partition` 个数之和， `cartesian` 返回的 `RDD` 的 `partition` 个数是它们的积。<br></br>

&emsp;&emsp;如果一个 `RDD` 没有父 `RDD` 呢？ 由 `textFile` 或者 `hadoopFile` 生成的 `RDD` 的 `partition` 个数由它们底层使用的 `MapReduce InputFormat` 决定的。一般情况下，每读到的一个 `HDFS block` 会生成一个` partition`。通过 `parallelize` 接口生成的 `RDD` 的 `partition` 个数由用户指定，如果用户没有指定则由参数 `spark.default.parallelism` 决定。<br></br>

&emsp;&emsp;要想知道 `partition` 的个数，可以通过接口 `rdd.partitions().size()` 获得。<br></br>

&emsp;&emsp;这里最需要关心的问题在于 `task` 的个数太小。如果运行时 `task` 的个数比实际可用的 `slot` 还少，那么程序解没法使用到所有的 `CPU` 资源。<br></br>

&emsp;&emsp;过少的 `task` 个数可能会导致在一些聚集操作时， 每个 `task` 的内存压力会很大。任何 `join，cogroup，*ByKey` 操作都会在内存生成一个 `hash-map`或者 `buffer` 用于分组或者排序。`join， cogroup ，groupByKey` 会在 `shuffle` 时在 `fetching` 端使用这些数据结构，` reduceByKey ，aggregateByKey` 会在 `shuffle` 时在两端都会使用这些数据结构。<br></br>

&emsp;&emsp;当需要进行这个聚集操作的 `record` 不能完全轻易塞进内存中时，一些问题就会暴露出来。首先，在内存中保有大量这些数据构的 `record` 会增加`GC`的压力，可能会导致流程停顿下来。其次，如果数据不能完全载入内存，`Spark` 会将这些数据写到磁盘，这会引起磁盘`IO`和排序。这可能是导致 `Spark Job` 慢的首要原因。<br></br>

&emsp;&emsp;那么如何增加你的 `partition` 的个数呢？如果你的 `stage` 是从 `Hadoop` 读取数据，你可以做以下的选项： <br></br>

- 使用 `repartition` 选项，会引发 `shuffle`；
- 配置 `InputFormat`，将文件分得更小；
- 写入 `HDFS` 文件时使用更小的`block`。

&emsp;&emsp;如果`stage` 从其他` stage `中获得输入，引发` stage` 边界的操作会接受一个 `numPartitions `的参数，比如

```scala
val rdd2 = rdd1.reduceByKey(_ + _, numPartitions = X)
```

&emsp;&emsp;`X` 应该取什么值？最直接的方法就是做实验。不停的将 `partition` 的个数从上次实验的 `partition` 个数乘以1.5，直到性能不再提升为止。<br></br>

&emsp;&emsp;同时也有一些原则用于计算 `X`，但是也不是非常的有效是因为有些参数是很难计算的。这里写到不是因为它们很实用，而是可以帮助理解。这里主要的目标是启动足够的 `task` 可以使得每个 `task` 接受的数据能够都塞进它所分配到的内存中。<br></br>

&emsp;&emsp;每个 `task` 可用的内存通过这个公式计算：`spark.executor.memory * spark.shuffle.memoryFraction * spark.shuffle.safetyFraction)/spark.executor.cores` 。 `memoryFraction` 和 `safetyFractio` 默认值分别 0.2 和 0.8。

&emsp;&emsp;在内存中所有 `shuffle` 数据的大小很难确定。最可行的是找出一个 `stage` 运行的 `Shuffle Spill（memory）` 和 `Shuffle Spill(Disk)` 之间的比例。再用所有`shuffle`写乘以这个比例。但是如果这个 `stage` 是 `reduce` 时，可能会有点复杂：<br></br>

<div  align="center"><img src="http://blog.cloudera.com/wp-content/uploads/2015/03/spark-tuning2-f2.png" height=40 alt="2.2" align="center" /></div><br><br/>

&emsp;&emsp;在有所疑虑的时候，使用更多的 `task` 数（也就是 `partition` 数）通常效果会更好，这与 `MapRecuce` 中建议 `task` 数目选择尽量保守的建议相反。这个因为 `MapReduce` 在启动 `task` 时相比需要更大的代价。

# 8 压缩数据结构

&emsp;&emsp;`Spark` 的数据流由一组 `record` 构成。一个 `record` 有两种表达形式:一种是反序列化的 `Java `对象，另外一种是序列化的二进制形式。通常情况下，`Spark` 对内存中的 `record` 使用反序列化之后的形式，对要存到磁盘上或者需要通过网络传输的 `record` 使用序列化之后的形式。也有计划在内存中存储序列化之后的 `record`。<br></br>

&emsp;&emsp;`spark.serializer` 控制这两种形式之间的转换的方式。`Kryo serializer，org.apache.spark.serializer.KryoSerializer` 是推荐的选择。但不幸的是它不是默认的配置，因为 `KryoSerializer` 在早期的 `Spark` 版本中不稳定，而 `Spark` 不想打破版本的兼容性，所以没有把 `KryoSerializer` 作为默认配置，但是 `KryoSerializer` 应该在任何情况下都是第一的选择。

&emsp;&emsp;`record `在这两种形式切换的频率对 `Spark` 应用的运行效率具有很大的影响。检查传递的数据的类型，看看能否改进是非常值得一试的。

&emsp;&emsp;过多的反序列化`record` 可能会导致数据`spill`到磁盘更加频繁，减少缓存的对象的个数。

&emsp;&emsp;过多的序列化` record` 导致更多的磁盘和网络`IO`，同样也会使得能够 `Cache` 在内存中的` record` 个数减少，这里主要的解决方案是把所有的用户自定义的 `class` 都通过 `SparkConf#registerKryoClasses` 的`API`定义和传递。

# 9 数据格式

&emsp;&emsp;任何时候你都可以决定你的数据以怎样的格式保存在磁盘上，可以使用可扩展的二进制格式比如：`Avro，Parquet，Thrift`或者`Protobuf`。当人们在谈论在`Hadoop`上使用`Avro，Thrift`或者`Protobuf`时，意味着每个 `record` 是一个 `Avro/Thrift/Protobuf `结构，并保存成 `sequence file`。而不是`JSON`格式。

# 参考文献

【1】[How-to: Tune Your Apache Spark Jobs (Part 1)](http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-1/)

【2】[How-to: Tune Your Apache Spark Jobs (Part 2)](http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-2/)

【3】[Tuning Spark](http://spark.apache.org/docs/latest/tuning.html)