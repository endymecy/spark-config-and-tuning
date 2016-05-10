# Databricks Spark 知识库

## 1 最佳实践

### 1.1 避免使用 GroupByKey

&emsp;&emsp;让我们看一下使用两种不同的方式去计算单词的个数，第一种方式使用 `reduceByKey`， 另外一种方式使用 `groupByKey`：

```scala
val words = Array("one", "two", "two", "three", "three", "three")
val wordPairsRDD = sc.parallelize(words).map(word => (word, 1))
//reduce
val wordCountsWithReduce = wordPairsRDD
  .reduceByKey(_ + _)
  .collect()
//group
val wordCountsWithGroup = wordPairsRDD
  .groupByKey()
  .map(t => (t._1, t._2.sum))
  .collect()
```

&emsp;&emsp;虽然两个函数都能得出正确的结果，`reduceByKey` 更适合使用在大数据集上。 这是因为 `Spark` 知道它可以在每个分区`shuffle`数据之前，聚合`key`值相同的数据。

&emsp;&emsp;借助下图可以理解在 `reduceByKey` 里发生了什么。 注意在数据对被`shuffle`前同一机器上同样 `key`的数据是怎样被组合的(`reduceByKey` 中的 `lamdba` 函数)。然后 `lamdba` 函数在每个区上被再次调用来将所有值 `reduce` 成一个最终结果。

<div  align="center"><img src="imgs/reduce_by.png" alt="reduce_by" align="center" /></div>

&emsp;&emsp;但是，当调用 `groupByKey` 时，所有的键值对(`key-value pair`) 都会被`shuffle`。在网络上传输这些数据非常没有必要。

&emsp;&emsp;为了确定将数据对`shuffle`到哪台主机，`Spark` 会对数据对的 `key` 调用一个分区函数。 当`shuffle`的数据量大于单台执行机器内存总量时，`Spark` 会把数据保存到磁盘上。 不过在保存时每次只会处理一个 `key` 的数据，所以当单个 `key` 的键值对超过内存容量会存在内存溢出的可能。
我们应避免将数据保存到磁盘上，这会严重影响性能。

<div  align="center"><img src="imgs/group_by.png" alt="group_by" align="center" /></div>

&emsp;&emsp;你可以想象一个非常大的数据集，在使用 `reduceByKey` 和 `groupByKey` 时他们的差别会被放大更多倍。

&emsp;&emsp;以下函数应该优先于 `groupByKey` ：

- `combineByKey` 组合数据，但是组合之后的数据类型与输入时值的类型不一样。

- `foldByKey` 合并每一个 `key` 的所有值，在级联函数和“零值”中使用。

### 1.2 不要将大型 RDD 的所有元素拷贝到driver

&emsp;&emsp;如果你的`driver`内存容量不能容纳一个大型 `RDD` 里面的所有数据，不要做以下操作：

```scala
val values = myVeryLargeRDD.collect()
```

&emsp;&emsp;`Collect` 操作会试图将 `RDD` 里面的每一条数据复制到`driver`上，这时候会发生内存溢出和崩溃。相反，你可以调用 `take` 或者 `takeSample` 来确保数据大小的上限。或者在你的 `RDD` 中使用过滤或抽样。
同样，要谨慎使用下面的操作，除非你能确保数据集小到足以存储在内存中：

- `countByKey`

- `countByValue`

- `collectAsMap`

&emsp;&emsp;如果你确实需要将 `RDD` 里面的大量数据保存在内存中，你可以将 `RDD` 写成一个文件或者把 `RDD` 导出到一个容量足够大的数据库中。
