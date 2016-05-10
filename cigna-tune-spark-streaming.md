# Cigna优化Spark Streaming实时处理应用

## 1 框架一览

&emsp;&emsp;事件处理的架构图如下所示。

<div  align="center"><img src="http://blog.cloudera.com/wp-content/uploads/2016/01/Spark_Kafka_diagram.png" alt="locality" align="center" /></div>

## 2 优化总结

&emsp;&emsp;当我们第一次部署整个方案时，`kafka`和`flume`组件都执行得非常好，但是`spark streaming`应用需要花费4-8分钟来处理单个`batch`。这个延迟的原因有两点，一是我们使用`DataFrame`来强化数据，而强化数据需要从`hive`中读取大量的数据；
二是我们的参数配置不理想。

&emsp;&emsp;为了优化我们的处理时间，我们从两方面着手改进：第一，缓存合适的数据和分区；第二，改变配置参数优化spark应用。运行spark应用的`spark-submit`命令如下所示。通过参数优化和代码改进，我们显著减少了处理时间，处理时间从4-8分钟降到了低于25秒。

```shell
/opt/app/dev/spark-1.5.2/bin/spark-submit \
 --jars  \
/opt/cloudera/parcels/CDH/jars/zkclient-0.3.jar,/opt/cloudera/parcels/CDH/jars/kafka_2.10-0.8.1.1.jar,\
/opt/app/dev/jars/datanucleus-core-3.2.2.jar,/opt/app/dev/jars/datanucleus-api-jdo-3.2.1.jar,/opt/app/dev/jars/datanucleus-rdbms-3.2.1.jar \
--files /opt/app/dev/spark-1.5.2/conf/hive-site.xml,/opt/app/dev/jars/log4j-eir.properties \
--queue spark_service_pool \
--master yarn \
--deploy-mode cluster \
--conf "spark.ui.showConsoleProgress=false" \
--conf "spark.driver.extraJavaOptions=-XX:MaxPermSize=6G -XX:+UseConcMarkSweepGC -Dlog4j.configuration=log4j-eir.properties" \
--conf "spark.sql.tungsten.enabled=false" \
--conf "spark.eventLog.dir=hdfs://nameservice1/user/spark/applicationHistory" \
--conf "spark.eventLog.enabled=true" \
--conf "spark.sql.codegen=false" \
--conf "spark.sql.unsafe.enabled=false" \
--conf "spark.executor.extraJavaOptions=-XX:+UseConcMarkSweepGC -Dlog4j.configuration=log4j-eir.properties" \
--conf "spark.streaming.backpressure.enabled=true" \
--conf "spark.locality.wait=1s" \
--conf "spark.streaming.blockInterval=1500ms" \
--conf "spark.shuffle.consolidateFiles=true" \
--driver-memory 10G \
--executor-memory 8G \
--executor-cores 20 \
--num-executors 20 \
--class com.bigdata.streaming.OurApp \ /opt/app/dev/jars/OurStreamingApplication.jar external_props.conf
```
&emsp;&emsp;下面我们将详细介绍这些改变的参数。

### 2.1 driver选项

&emsp;&emsp;这里需要注意的是，`driver`运行在`spark on yarn`的集群模式下。因为`spark streaming`应用是一个长期运行的任务，生成的日志文件会很大。为了解决这个问题，我们限制了写入日志的消息的条数，
并且用`RollingFileAppender`限制了它们的大小。我们也关闭了`spark.ui.showConsoleProgress`选项来禁用控制台日志消息。

&emsp;&emsp;通过测试，我们的`driver`因为永久代空间填满而频繁发生内存耗尽（永久代空间是类、方法等存储的地方，不会被重新分配）。将永久代空间的大小升高到6G可以解决这个问题。

```shell
spark.driver.extraJavaOptions=-XX:MaxPermSize=6G
```

### 2.2 垃圾回收

&emsp;&emsp;因为我们的`spark streaming`应用程序是一个长期运行的进程，在处理一段时间之后，我们注意到`GC`暂停时间过长，我们想在后台减少或者保持这个时间。调整`UseConcMarkSweepGC`参数是一个技巧。

```shell
--conf "spark.executor.extraJavaOptions=-XX:+UseConcMarkSweepGC -Dlog4j.configuration=log4j-eir.properties" \
```

### 2.3 禁用Tungsten

&emsp;&emsp;`Tungsten`是`spark`执行引擎主要的改进。但是它的第一个版本是有问题的，所以我们暂时禁用它。

```shell
spark.sql.tungsten.enabled=false
spark.sql.codegen=false
spark.sql.unsafe.enabled=false
```

### 2.4 启用反压

&emsp;&emsp;`Spark Streaming`在批处理时间大于批间隔时间时会出现问题。换一句话说，就是`spark`读取数据的速度慢于`kafka`数据到达的速度。如果按照这个吞吐量执行过长的时间，它会造成不稳定的情况。
即接收`executor`的内存溢出。设置下面的参数解决这个问题。

```shell
spark.streaming.backpressure.enabled=true
```

### 2.5 调整本地化和块配置

&emsp;&emsp;下面的两个参数是互补的。一个决定了数据本地化到`task`或者`executor`等待的时间，另外一个被`spark streaming receiver`使用对数据进行组块。块越大越好，但是如果数据没有本地化到`executor`，它将会通过网络移动到
任务执行的地方。我们必须在这两个参数间找到一个好的平衡，因为我们不想数据块太大，并且也不想等待本地化太长时间。我们希望所有的任务都在几秒内完成。

&emsp;&emsp;因此，我们改变本地化选项从3s到1s，我们也改变块间隔为1.5s。

```shell
--conf "spark.locality.wait=1s" \
--conf "spark.streaming.blockInterval=1500ms" \
```

### 2.6 合并临时文件

&emsp;&emsp;在`ext4`文件系统中，推荐开启这个功能。因为这会产生更少的临时文件。

```scala
--conf "spark.shuffle.consolidateFiles=true" \
```

### 2.7 开启executor配置

&emsp;&emsp;在你配置`kafka Dstream`时，你能够指定并发消费线程的数量。然而，`kafka Dstream`的消费者会运行在相同的`spark driver`节点上面。因此，为了从多台机器上面并行消费`kafka topic`，
我们必须实例化多个`Dstream`。虽然可以在处理之前合并相应的`RDD`，但是运行多个应用程序实例，把它们都作为相同`kafka consumer group`的一部分。

&emsp;&emsp;为了达到这个目的，我们设置20个`executor`，并且每个`executor`有20个核。

```shell
--executor-memory 8G
--executor-cores 20
--num-executors 20
```

### 2.8 缓存方法

&emsp;&emsp;使用`RDD`之前缓存`RDD`，但是记住在下次迭代之前从缓存中删除它。缓存那些需要使用多次的数据非常有用。然而，不要使分区数目过大。保持分区数目较低可以减少，最小化调度延迟。下面的公式是我们使用的分区数的计算公式。

```shell
# of executors * # of cores = # of partitions
```

# 参考文献

【1】[How Cigna Tuned Its Spark Streaming App for Real-time Processing with Apache Kafka](http://blog.cloudera.com/blog/2016/01/how-cigna-tuned-its-spark-streaming-app-for-real-time-processing-with-apache-kafka/)
