# spark参数介绍

## 1 spark on yarn常用属性介绍

| 属性名 | 默认值 | 属性说明 |
|:------:|:------:|:-------|
|spark.yarn.am.memory|512m|在客户端模式（client mode）下，yarn应用master使用的内存数。在集群模式（cluster mode）下，使用spark.driver.memory代替。|