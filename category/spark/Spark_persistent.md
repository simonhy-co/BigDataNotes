# Spark 持久化

>将数据集持久化(缓存)到内存中是Spark中的一个重要的功能。当您持久化一个RDD时，每个节点都会将其计算的每个分区数据存储在内存中，并在该数据集(或从该数据集派生的数据集)的其他操作中重用它们。这使得计算效率变得更快(通常是10倍以上)。

## 使用场景

### RDD的重复使用

如果一个对应的 RDD，在多个 Job 中重复使用的时候，此时默认情况下该 RDD 之前的处理步骤，每个 Job 都会重复执行，效率较低下。例如下面的代码：

```scala
val sc = new SparkContext(new SparkConf().setMaster("local").setAppName("test"))
val rdd = sc.parallelize(List("hello word", "word spark", "hello spark"))

val rdd2 =
rdd
  .flatMap(line => {
    println(s"flatMap--> ${line}")
    line.split(" ")
  })
  .map((_, 1))

val rdd3 = rdd2.map(x => (x._1, x._2 + 1))
val rdd4 = rdd2.map(x => (x._1, x._2 + 10))

rdd3.collect()
rdd4.collect()
```

在上面的代码中，`rdd3`和`rdd4`都是使用了 `rdd2` 开启了新的Job。 这两个 Job 重复使用了 rdd2 并且把 rdd2 执行了两遍。所以打印语句打印了6次(两次相同的内容) 

### 血统链过长

血统过长时，当某一个RDD计算错误需要根据依赖链条重新计算得到数据，如果链条过长会导致重复计算时间过长。



## 缓存

可以调用`persist()`或`cache（）`方法将一个RDD的数据持久化。该RDD在第一个计算后，会将数据保存在 Node 的内存或磁盘中。Spark的缓存是有容错性的，如果任何分区的RDD丢失，它会自动使用创建它的转换算子重新计算。

```scala
val sc = new SparkContext(new SparkConf().setMaster("local").setAppName("test"))
// ：
// :
//以上代码相同，省略

//将 rdd2 缓存
val cachedRDD = rdd2.cache()
val rdd3 = cachedRDD.map(x => (x._1, x._2 + 1))
val rdd4 = cachedRDD.map(x => (x._1, x._2 + 10))

rdd3.collect()
rdd4.collect()

```

将 rdd2 缓存后，rdd2只会计算一次，只会从缓存中调用。

### 缓存级别

**cache 与 persist 的区别：** 

- cache 底层调用无参的 persist实现，此时数据将保存在内存中。
- persist可以自由指定缓存级别。可以指定点数据保存在 内存 / 磁盘中

**存储级别：**

| 缓存级别              | 解释                                                         |
| --------------------- | ------------------------------------------------------------ |
| NONE                  | 不存储                                                       |
| DISK_ONLY             | 只保存在磁盘                                                 |
| DISK_ONLY_2           | 只保存在磁盘，数据保存两份                                   |
| MEMORY_ONLY           | 只保存在内存                                                 |
| MEMORY_ONLY_2         | 只保存在内存，数据保存两份                                   |
| MEMORY_ONLY_SER       | 只保存在内存，数据序列化                                     |
| MEMORY_ONLY_SER_2     | 只保存在内存，数据序列化，数据保存两份                       |
| MEMORY_AND_DISK       | 一部分数据在磁盘，一部分数据在内存                           |
| MEMORY_AND_DISK_2     | 一部分数据在磁盘，一部分数据在内存，数据保存两份             |
| MEMORY_AND_DISK_SER   | 一部分数据在磁盘，一部分数据在内存，数据序列化               |
| MEMORY_AND_DISK_SER_2 | 一部分数据在磁盘，一部分数据在内存，数据序列化，数据保存两份 |
| OFF_HEAP              | 数据保存在堆外内存中                                         |

常用：

- MEMORY_ONLY：小数量场景
- MEMORY_AND_DISK：大数据量场景

>### Which Storage Level to Choose?
>
>Spark’s storage levels are meant to provide different trade-offs between memory usage and CPU efficiency. We recommend going through the following process to select one:
>
>- If your RDDs fit comfortably with the default storage level (`MEMORY_ONLY`), leave them that way. This is the most CPU-efficient option, allowing operations on the RDDs to run as fast as possible.
>- If not, try using `MEMORY_ONLY_SER` and [selecting a fast serialization library](http://spark.apache.org/docs/latest/tuning.html) to make the objects much more space-efficient, but still reasonably fast to access. (Java and Scala)
>- Don’t spill to disk unless the functions that computed your datasets are expensive, or they filter a large amount of the data. Otherwise, recomputing a partition may be as fast as reading it from disk.
>- Use the replicated storage levels if you want fast fault recovery (e.g. if using Spark to serve requests from a web application). *All* the storage levels provide full fault tolerance by recomputing lost data, but the replicated ones let you continue running tasks on the RDD without waiting to recompute a lost partition.
>
>-- From [Spark Official Websit](http://spark.apache.org/docs/latest/rdd-programming-guide.html#which-storage-level-to-choose)



## Checkpoint

原因： 缓存保存数据的时候是将数据保存在task所在主机或磁盘中，如果Task 所在主机宕机，数据丢失，此时需要根据RDD依赖关系重新得到数据。所以此时需要将RDD数据持久化到可靠的存储介质中，保证数据不丢失。

使用方式：

1. 指定数据持久化路径: `sc.setCheckpointDir("path")`
2. 持久化RDD：`rdd.checkpoint()`

```scala
val sc = new SparkContext(new SparkConf().setMaster("local").setAppName("test"))
// ：
// :
//以上代码相同，省略
sc.setCheckpointDir("checkpoint")

//将 rdd2 缓存到 HDFS
rdd2.checkpoint()
val rdd3 = rdd2.map(x => (x._1, x._2 + 1))
val rdd4 = rdd2.map(x => (x._1, x._2 + 10))

rdd3.collect()
rdd4.collect()
```

运行完代码后会发现，rdd2 其实被运行了两次，这是因为在`collect()`中调用的方法`runJob（）`方法中，有一行代码：`rdd.doCheckpoint()`。 该方法将 调用checkpoint的RDD重新调用了一次。该方法的注释说：这个方法在job运算完该RDD之后被调用，因此该RDD可能已经被存储到内存中。