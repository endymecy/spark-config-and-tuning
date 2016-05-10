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

### 1.3 优雅地处理坏的输入数据

&emsp;&emsp;当处理大量的数据的时候，一个常见的问题是有些数据格式不对或者内容有误。使用`filter`方法可以很容易丢弃坏的输入或者使用`map`方法可以修复可能修复的坏的数据。当你尝试着修复坏的数据，但是丢弃无法被修复的数据时，
`flatMap`函数是最好的选择。让我们考虑下面的输入`json`串。

```scala
input_rdd = sc.parallelize(["{\"value\": 1}",  # Good
                            "bad_json",  # Bad
                            "{\"value\": 2}",  # Good
                            "{\"value\": 3"  # Missing an ending brace.
                            ])
```
&emsp;&emsp;当我们尝试着在`SqlContext`中使用这个输入串时，很明显它会因为格式不对而报错。

```scala
sqlContext.jsonRDD(input_rdd).registerTempTable("valueTable")
# The above command will throw an error.
```
&emsp;&emsp;让我妈用下面的`python`代码修复输入数据。

```python
def try_correct_json(json_string):
  try:
    # First check if the json is okay.
    json.loads(json_string)
    return [json_string]
  except ValueError:
    try:
      # If not, try correcting it by adding a ending brace.
      try_to_correct_json = json_string + "}"
      json.loads(try_to_correct_json)
      return [try_to_correct_json]
    except ValueError:
      # The malformed json input can't be recovered, drop this input.
      return []
```
&emsp;&emsp;经过上面函数的处理之后，我们就可以使用这些数据了。

```scala
corrected_input_rdd = input_rdd.flatMap(try_correct_json)
sqlContext.jsonRDD(corrected_input_rdd).registerTempTable("valueTable")
sqlContext.sql("select * from valueTable").collect()
# Returns [Row(value=1), Row(value=2), Row(value=3)]
```

## 2 常规故障处理

### 2.1 Job aborted due to stage failure: Task not serializable

&emsp;&emsp;如果你看到以下错误：

```
org.apache.spark.SparkException: Job aborted due to stage failure:
Task not serializable: java.io.NotSerializableException: ...
```

&emsp;&emsp;上述的错误在这种情况下会发生：当你在 `master` 上初始化一个变量，但是试图在 `worker` 上使用。
在这个示例中， `Spark Streaming` 试图将对象序列化之后发送到 `worker` 上，如果这个对象不能被序列化就会失败。思考下面的代码片段：

```scala
NotSerializable notSerializable = new NotSerializable();
JavaRDD<String> rdd = sc.textFile("/tmp/myfile");
rdd.map(s -> notSerializable.doSomething(s)).collect();
```
&emsp;&emsp;这段代码会触发上面的错误。这里有一些建议修复这个错误：

- 让 `class` 实现序列化
- 在作为参数传递给 `map` 方法的 `lambda` 表达式内部声明实例
- 在每一台机器上创建一个 `NotSerializable` 的静态实例
- 调用 `rdd.forEachPartition` 并且像下面这样创建 `NotSerializable` 对象：

```java
rdd.forEachPartition(iter -> {
  NotSerializable notSerializable = new NotSerializable();
  // ...Now process iter
});
```

### 2.2 缺失依赖

&emsp;&emsp;在默认状态下，`Maven` 在 `build` 的时候不会包含所依赖的 `jar` 包。当运行一个 `Spark` 任务时，如果 `Spark worker` 机器上没有包含所依赖的 `jar` 包会发生类无法找到的错误(`ClassNotFoundException`)。

&emsp;&emsp;有一个简单的方式，在 `Maven` 打包的时候创建 `shaded` 或 `uber` 任务可以让那些依赖的 `jar` 包很好地打包进去。

&emsp;&emsp;使用 `<scope>provided</scope>` 可以排除那些没有必要打包进去的依赖，对 `Spark` 的依赖必须使用 `provided` 标记，因为这些依赖已经包含在 `Spark cluster`中。在你的 `worker` 机器上已经安装的 `jar` 包你同样需要排除掉它们。

&emsp;&emsp;下面是一个 `Maven pom.xml` 的例子，工程了包含了一些需要的依赖，但是 `Spark` 的 `libraries` 不会被打包进去，因为它使用了 `provided`：

```xml
<project>
    <groupId>com.databricks.apps.logs</groupId>
    <artifactId>log-analyzer</artifactId>
    <modelVersion>4.0.0</modelVersion>
    <name>Databricks Spark Logs Analyzer</name>
    <packaging>jar</packaging>
    <version>1.0</version>
    <repositories>
        <repository>
            <id>Akka repository</id>
            <url>http://repo.akka.io/releases</url>
        </repository>
    </repositories>
    <dependencies>
        <dependency> <!-- Spark -->
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.10</artifactId>
            <version>1.1.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency> <!-- Spark SQL -->
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.10</artifactId>
            <version>1.1.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency> <!-- Spark Streaming -->
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.10</artifactId>
            <version>1.1.0</version>
            <scope>provided</scope>
        </dependency>
        <dependency> <!-- Command Line Parsing -->
            <groupId>commons-cli</groupId>
            <artifactId>commons-cli</artifactId>
            <version>1.2</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <filters>
                        <filter>
                            <artifact>*:*</artifact>
                            <excludes>
                                <exclude>META-INF/*.SF</exclude>
                                <exclude>META-INF/*.DSA</exclude>
                                <exclude>META-INF/*.RSA</exclude>
                            </excludes>
                        </filter>
                    </filters>
                    <finalName>uber-${project.artifactId}-${project.version}</finalName>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```


