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
|`spark.yarn.scheduler.initial-allocation.interval`|200ms|当存在挂起的容器分配请求时，`spark`应用`master`发送心跳给`resourcemanager`的间隔时间。它的大小不能大于`spark.yarn.scheduler.heartbeat.interval-ms`，如果挂起的请求还存在，那么这个时间加倍，直到到达`spark.yarn.scheduler.heartbeat.interval-ms`大小。|
|`spark.yarn.max.executor.failures`|`numExecutors * 2`，并且不小于3|在失败应用程序之前，`executor`失败的最大次数。|
|`spark.executor.instances`|2|`Executors`的个数。这个配置和`spark.dynamicAllocation.enabled`不兼容。当同时配置这两个配置时，动态分配关闭，`spark.executor.instances`被使用|
|`spark.yarn.executor.memoryOverhead`|`executorMemory * 0.10`，并且不小于`384m`|每个`executor`分配的堆外内存。|
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

### 2.1 Spark on YARN集群模式分析

#### 2.1.1 客户端操作

- 1、根据`yarnConf`来初始化`yarnClient`，并启动`yarnClient`；

- 2、创建客户端`Application`，并获取`Application`的`ID`，进一步判断集群中的资源是否满足`executor`和`ApplicationMaster`申请的资源，如果不满足则抛出`IllegalArgumentException`；

- 3、设置资源、环境变量：其中包括了设置`Application`的`Staging`目录、准备本地资源（`jar`文件、`log4j.properties`）、设置`Application`其中的环境变量、创建`Container`启动的`Context`等；

- 4、设置`Application`提交的`Context`，包括设置应用的名字、队列、`AM`的申请的`Container`、标记该作业的类型为`Spark`；

- 5、申请`Memory`，并最终通过`yarnClient.submitApplication`向`ResourceManager`提交该`Application`。

&emsp;&emsp;当作业提交到`YARN`上之后，客户端就没事了，甚至在终端关掉那个进程也没事，因为整个作业运行在`YARN`集群上进行，运行的结果将会保存到`HDFS`或者日志中。

#### 2.1.2 提交到YARN集群，YARN操作

- 1、运行`ApplicationMaster`的`run`方法；

- 2、设置好相关的环境变量。

- 3、创建`amClient`，并启动；

- 4、在`Spark UI`启动之前设置`Spark UI`的`AmIpFilter`；

- 5、在`startUserClass`函数专门启动了一个线程（名称为`Driver`的线程）来启动用户提交的`Application`，也就是启动了`Driver`。在`Driver`中将会初始化`SparkContext`；

- 6、等待`SparkContext`初始化完成，最多等待`spark.yarn.applicationMaster.waitTries`次数（默认为10），如果等待了的次数超过了配置的，程序将会退出；否则用`SparkContext`初始化`yarnAllocator`；

- 7、当`SparkContext、Driver`初始化完成的时候，通过`amClient`向`ResourceManager`注册`ApplicationMaster`;

- 8、分配并启动`Executeors`。在启动`Executeors`之前，先要通过`yarnAllocator`获取到`numExecutors`个`Container`，然后在`Container`中启动`Executeors`。
如果在启动`Executeors`的过程中失败的次数达到了`maxNumExecutorFailures`的次数，`maxNumExecutorFailures`的计算规则如下：

```scala
// Default to numExecutors * 2, with minimum of 3
private val maxNumExecutorFailures = sparkConf.getInt("spark.yarn.max.executor.failures",
    sparkConf.getInt("spark.yarn.max.worker.failures", math.max(args.numExecutors * 2, 3)))
```

&emsp;&emsp;那么这个`Application`将失败，将`Application Status`标明为`FAILED`，并将关闭`SparkContext`。其实，启动`Executeors`是通过`ExecutorRunnable`实现的，而`ExecutorRunnable`内部是启动`CoarseGrainedExecutorBackend`的。

- 9、最后，`Task`将在`CoarseGrainedExecutorBackend`里面运行，然后运行状况会通过`Akka`通知`CoarseGrainedScheduler`，直到作业运行完成。

### 2.2 Spark on YARN客户端模式分析

&emsp;&emsp;和`yarn-cluster`模式一样，整个程序也是通过`spark-submit`脚本提交的。但是`yarn-client`作业程序的运行不需要通过`Client`类来封装启动，而是直接通过反射机制调用作业的`main`函数。下面是流程。

- 1、通过`SparkSubmit`类的`launch`的函数直接调用作业的`main`函数（通过反射机制实现），如果是集群模式就会调用`Client`的`main`函数。

- 2、而应用程序的`main`函数一定都有个`SparkContent`，并对其进行初始化；

- 3、在`SparkContent`初始化中将会依次做如下的事情：设置相关的配置、注册`MapOutputTracker、BlockManagerMaster、BlockManager`，创建`taskScheduler`和`dagScheduler`；

- 4、初始化完`taskScheduler`后，将创建`dagScheduler`，然后通过`taskScheduler.start()`启动`taskScheduler`，而在`taskScheduler`启动的过程中也会调用`SchedulerBackend`的`start`方法。
在`SchedulerBackend`启动的过程中将会初始化一些参数，封装在`ClientArguments`中，并将封装好的`ClientArguments`传进`Client`类中，并`client.runApp()`方法获取`Application ID`。

- 5、`client.runApp`里面的做的和上章客户端进行操作那节类似，不同的是在里面启动是`ExecutorLauncher`（`yarn-cluster`模式启动的是`ApplicationMaster`）。

- 6、在`ExecutorLauncher`里面会初始化并启动`amClient`，然后向`ApplicationMaster`注册该`Application`。注册完之后将会等待`driver`的启动，当`driver`启动完之后，会创建一个`MonitorActor`对象用于和`CoarseGrainedSchedulerBackend`进行通信（只有事件`AddWebUIFilter`他们之间才通信，`Task`的运行状况不是通过它和`CoarseGrainedSchedulerBackend`通信的）。
然后就是设置`addAmIpFilter`，当作业完成的时候，`ExecutorLauncher`将通过`amClient`设置`Application`的状态为`FinalApplicationStatus.SUCCEEDED`。

- 7、分配`Executors`，这里面的分配逻辑和`yarn-cluster`里面类似。

- 8、最后，`Task`将在`CoarseGrainedExecutorBackend`里面运行，然后运行状况会通过`Akka`通知`CoarseGrainedScheduler`，直到作业运行完成。

- 9、在作业运行的时候，`YarnClientSchedulerBackend`会每隔1秒通过`client`获取到作业的运行状况，并打印出相应的运行信息，当`Application`的状态是`FINISHED、FAILED`和`KILLED`中的一种，那么程序将退出等待。

- 10、最后有个线程会再次确认`Application`的状态，当`Application`的状态是`FINISHED、FAILED`和`KILLED`中的一种，程序就运行完成，并停止`SparkContext`。整个过程就结束了。

## 3 spark submit 和 spark shell参数介绍

| 参数名 | 格式 | 参数说明 |
|:------:|:------:|:-------|
|--master|MASTER_URL|如spark://host:port|
|--deploy-mode|DEPLOY_MODE|Client或者master，默认是client|
|--class|CLASS_NAME|应用程序的主类|
|--name|NAME|应用程序的名称|
|--jars|JARS|逗号分隔的本地jar包，包含在driver和executor的classpath下|
|--packages||包含在driver和executor的classpath下的jar包逗号分隔的”groupId:artifactId：version”列表|
|--exclude-packages||用逗号分隔的”groupId:artifactId”列表|
|--repositories||逗号分隔的远程仓库|
|--py-files |PY_FILES|逗号分隔的”.zip”,”.egg”或者“.py”文件，这些文件放在python app的PYTHONPATH下面|
|--files|FILES|逗号分隔的文件，这些文件放在每个executor的工作目录下面|
|--conf|PROP=VALUE|固定的spark配置属性|
|--properties-file|FILE|加载额外属性的文件|
|--driver-memory|MEM|Driver内存，默认1G|
|--driver-java-options||传给driver的额外的Java选项|
|--driver-library-path||传给driver的额外的库路径|
|--driver-class-path||传给driver的额外的类路径|
|--executor-memory|MEM|每个executor的内存，默认是1G|
|--proxy-user|NAME|模拟提交应用程序的用户|
|--driver-cores|NUM|Driver的核数，默认是1。这个参数**仅仅在standalone集群deploy模式下使用**|
|--supervise||Driver失败时，重启driver。**在mesos或者standalone下使用**|
|--verbose||打印debug信息|
|--total-executor-cores|NUM|所有executor总共的核数。**仅仅在mesos或者standalone下使用**|
|--executor-core|NUM|每个executor的核数。**在yarn或者standalone下使用**|
|--driver-cores|NUM|Driver的核数，默认是1。**在yarn集群模式下使用**|
|--queue|QUEUE_NAME|队列名称。**在yarn下使用**|
|--num-executors|NUM|启动的executor数量。默认为2。**在yarn下使用**|

&emsp;&emsp;你可以通过`spark-submit --help`或者`spark-shell --help`来查看这些参数。

## 参考文献

【1】[Spark:Yarn-cluster和Yarn-client区别与联系](http://www.iteblog.com/archives/1223)

【2】[Spark on YARN客户端模式作业运行全过程分析](http://www.iteblog.com/archives/1191)

【3】[Spark on YARN集群模式作业运行全过程分析](http://www.iteblog.com/archives/1189)