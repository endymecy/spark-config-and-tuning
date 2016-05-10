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


