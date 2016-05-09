# spark参数介绍

## 1 spark on yarn常用属性介绍

| 属性名 | 默认值 | 属性说明 |
|:------:|:------:|:-------|
|`spark.yarn.am.memory`|512m|在客户端模式（`client mode`）下，`yarn`应用`master`使用的内存数。在集群模式（`cluster mode`）下，使用`spark.driver.memory`代替。|
|`spark.driver.cores`|1|在集群模式（`cluster mode`）下，`driver`程序使用的核数。在集群模式（`cluster mode`）下，`driver`程序和`master`运行在同一个`jvm`中，所以`master`控制这个核数。在客户端模式（`client mode`）下，使用`spark.yarn.am.cores`控制`master`使用的核。|
|`spark.yarn.am.cores`|1|在客户端模式（`client mode`）下，`yarn`应用的`master`使用的核数。在集群模式下，使用`spark.driver.cores`代替。|
|`spark.yarn.am.waitTime`|100ms|在集群模式（cluster mode）下，`yarn`应用`master`等待`SparkContext`初始化的时间。在客户端模式（`client mode`）下，`master`等待`driver`连接到它的时间。|
|`spark.yarn.submit.file.replication`|3|文件上传到`hdfs`上去的`replication`次数|
|`spark.yarn.preserve.staging.files`|`false`|设置为`true`时，在`job`结束时，保留`staged`文件；否则删掉这些文件。|
|`spark.yarn.scheduler.heartbeat.interval-ms`|3000|`Spark`应用`master`与`yarn resourcemanager`之间的心跳间隔|
|`spark.yarn.scheduler.initial-allocation.interval`|200ms|当存在挂起的容器分配请求时，spark应用master发送心跳给`resourcemanager`的间隔时间。它的大小不能大于`spark.yarn.scheduler.heartbeat.interval-ms`，如果挂起的请求还存在，那么这个时间加倍，直到到达`spark.yarn.scheduler.heartbeat.interval-ms`大小。|
|`spark.yarn.max.executor.failures`|numExecutors * 2，并且不小于3|在失败应用程序之前，executor失败的最大次数。|
|`spark.executor.instances`|2|`Executors`的个数。这个配置和`spark.dynamicAllocation.enabled`不兼容。当同时配置这两个配置时，动态分配关闭，`spark.executor.instances`被使用|
|`spark.yarn.executor.memoryOverhead`|`executorMemory * 0.10`，并且不小于`384m`|每个executor分配的堆外内存。|
|`spark.yarn.driver.memoryOverhead`|`driverMemory * 0.10`，并且不小于`384m`|在集群模式下，每个`driver`分配的堆外内存。|
|`spark.yarn.am.memoryOverhead`|`AM memory * 0.10`，并且不小于`384m`|在客户端模式下，每个`driver`分配的堆外内存|
|`spark.yarn.am.port`|随机|`Yarn` 应用`master`监听的端口。|
|`spark.yarn.queue`|`default`|应用提交的`yarn`队列的名称|
|`spark.yarn.jar`|`none`|`Jar`文件存放的地方。默认情况下，`spark jar`安装在本地，但是`jar`也可以放在`hdfs`上，其他机器也可以共享。|

## 2 客户端模式和集群模式的区别

&emsp;&emsp;这里我们要区分一下什么是客户端模式（`client mode`），什么是集群模式（`cluster mode`）。

&emsp;&emsp;我们知道，当在`YARN`上运行`Spark`作业时，每个`Spark executor`作为一个`YARN`容器(`container`)运行。`Spark`可以使得多个`Tasks`在同一个容器(`container`)里面运行。
`yarn-cluster`和`yarn-client`模式的区别其实就是`Application Master`进程的区别，在`yarn-cluster`模式下，`driver`运行在`AM`(`Application Master`)中，它负责向`YARN`申请资源，并监督作业的运行状况。当用户提交了作业之后，就可以关掉`Client`，作业会继续在`YARN`上运行。然而`yarn-cluster`模式不适合运行交互类型的作业。
在`yarn-client`模式下，`Application Master`仅仅向`YARN`请求`executor`，`client`会和请求的`container`通信来调度他们工作，也就是说`Client`不能离开。下面的图形象表示了两者的区别。

<div  align="center"><img src="imgs/1.1.png" alt="1.1" align="center" /></div><br><br/>

<div  align="center"><img src="imgs/1.2.png"  alt="1.2" align="center" /></div><br><br/>

