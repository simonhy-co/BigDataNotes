# Spark Core

# RDD 概述

RDD(Resilient Distributed Dataset) 弹性分布式数据集，是Spark中最基本的数据抽象  which is a collection of elements partitioned across the nodes of the cluster that can be operated on in parallel. 

- **弹性(Resilient)**
  
  - 存储的弹性： 内存与磁盘的自动切换
  - 容错的弹性： 数据丢失可以自动恢复
  - 计算的弹性： 计算出错重试机制
  - 分片的弹性： 可根据需要重新分片
  
- **分布式(Distributed)**

  数据储存在大数据集群不同节点。集群中的不同节点计算不同数据

- **数据集**

  不储存数据。RDD封装了计算逻辑，并不保存数据

- **数据抽象**

  RDD是一个抽象类，需要子类具体实现

- **不可变**

  RDD封装了计算逻辑，是不可以改变的，想要改变，只能产生新的RDD，在新的RDD里面封装计算逻辑

- **可分区、并行计算**

  

## RDD 五大特性

>
>
>Internally, each RDD is characterized by five main properties:
>
>- A list of partitions
>
>- A function for computing each split
>
>- A list of dependencis on other RDDS
>
>- Optionally, a Partitioner for key-value RDDs(e.g. to say that the RDD is hash-partitioned)
>
>- Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)
>
>  

每一个RDD具有五个主要的特性，每个特性都对应一个方法。

- 一个分区列表

  ```scala
  protected def getPartitions: Array[Partition]
  ```

- 对每个切片的计算函数

  ```scala
  def compute(split: Partition, context: TaskContext): Iterator[T]
  ```

- 对其他RDD的依赖关系

  ```scala
  protected def getDependencies: Seq[Dependency[_]] = deps
  ```

- 分区器: shuffle时，控制数据去往哪个分区

  ```scala
  val partitioner: Option[Partitioner] = None
  ```

- Partition的优先位置：将计算放在数据所在节点

  ```scala
  protected def gerPreferredLocations(split: Partition): Seq[String] = Nil
  ```

  

## [未完成： RDD 数据流向来解释五大特性]







# RDD 编程

## RDD的创建

pom文件中的依赖:

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.12</artifactId>
        <version>3.0.0</version>
    </dependency>
</dependencies>
<build>
    <finalName>SparkCoreTest</finalName>
    <plugins>
        <plugin>
            <groupId>net.alchim31.maven</groupId>
            <artifactId>scala-maven-plugin</artifactId>
            <version>3.4.6</version>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>testCompile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 本地集合创建

通过 `makeRDD` 或者 `parallelize` 方法创建(一般在测试时使用). `makeRDD` 底层也是调用 `parallelize` 创建 RDD

```scala
@Test
def createRddByLocalCollection(): Unit = {
  //创建SparkContext
  val conf: SparkConf = new SparkConf().setMaster("local[4]").setAppName("test")
  val sc: SparkContext = new SparkContext(conf)

  val data: List[String] = List("hello", "java", "spark")

  //方法一
  val rdd: RDD[String] = sc.makeRDD(data)

  //方法二
  val rdd2: RDD[String] = sc.parallelize(data)
}
```

### 读取文件创建





### 其他的RDD衍生



## 分区规则









## Transformation 转换算子

> Value 类型











### filter

根据指定的条件过滤

>Key-Value 类型

### partitionBy

- 提供两个分区器: 





- 自定义分区器：
  1. 创建一个class继承Partitioner
  2. 重写抽象方法

```
```





### reduceByKey

该操作可以将`RDD[K,V]`中的元素按照相同的`K`对`V`进行聚合。其存在多种重载形式，还可以设置新`RDD`的分区数。

该操作会进行`combiner`操作，性能比`groupby`高

// 先按照KEY进行分组，将value添加到seq中，然后对这个seq做reduce操作。



**函数签名：**

`def reduceByKey(func: (V, V) => V): RDD[(K, V)]`



**Example:**

```scala
val data = List( ("hello",3),("hello",1),("hello",4),
      "spark"->5,"hello"->10,"spark"->2,"spark"->1,"hello"->7,"spark"->6,"spark"->9 )
val rdd: RDD[(String, Int)] = sc.makeRDD(data,2)
val rdd2: RDD[(String, Int)] = rdd.reduceByKey(_+_)
println(rdd2.collect.toList)
// --> List((hello,25), (spark,23))

//相当于 groupBy+map:
//val rdd2 = rdd.groupBy(_._1).map(x=> (x._1,x._2.map(_._2).sum))
```



**流程：**

[TODO]



### groupByKey

- 按照Key进行分组，将相同key的value值放入同一个seq。

- 与groupBy方法不同，groupby是将每一个元素整体放入seq中。groupBy底层也是调用groupBykey。
- 与reduceByKey相比，groupByKey 没有 combine 操作。工作中推荐使用 reduceByKey 这种高性能 shuffle 算子

```scala
val data = List( ("hello",3),("hello",1),("hello",4), "spark"->5,"hello"->10,"spark"->2,"spark"->1,"hello"->7,"spark"->6,"spark"->9 )
val rdd: RDD[(String, Int)] = sc.makeRDD(data)
val rdd2: RDD[(String, Iterable[Int])] = rdd.groupByKey()
println(rdd2.collect().toList) // List((spark,CompactBuffer(5, 2, 1, 6, 9)), (hello,CompactBuffer(3, 1, 4, 10, 7)))
```





### combineByKey

需要三个函数：

`createCombiner: V => C`: 在combiner阶段，分完组后对每组的第一个value值进行转换
`mergeValue: (C, V) => C`: combiner计算逻辑
`mergeCombiners: (C, C) => C`: reduce计算逻辑

```scala
 //数据
val data = List("语文" -> 60, "英语" -> 80, "语文" -> 70, "英语" -> 100, "数学" -> 40, "英语" -> 90, "语文" -> 70, "数学" -> 100, "数学" -> 100, "语文" -> 30, "英语" -> 80)
val rdd = sc.makeRDD(data, 2)

//获取每门学科的平均分
val rdd2 = rdd.combineByKey(
                (score: Int) => (score, 1),
                (agg: (Int, Int), curr: Int) => (agg._1 + curr, agg._2 + 1),
                (agg: (Int, Int), curr: (Int, Int)) => (agg._1 + curr._1, agg._2 + curr._2)
						)// List((数学,(240,3)), (英语,(350,4)), (语文,(230,4)))

val rdd3 = rdd2.map { case (name, (sumScore, count)) => (name, sumScore.toDouble / count) }
println(rdd3.collect().toList  //List((数学,80.0), (英语,87.5), (语文,57.5))
```

### foldByKey

**函数签名:**

```scala
//zeroValue: 默认值
def foldByKey(zeroValue: V)(func: (V, V) => V): RDD[(K, V)]
```

foldByKey 与 reduceByKey功能相似。 只是 foldByKey 在 combiner 阶段分组之后对每个组所有的 value 第一次聚合的时候，函数==第一个参数==`(agg)`的初始值为传入的默认值。



### aggregateByKey

**函数签名：**

```scala
/**
* zeroValue： 默认值
* seqOp： combiner 聚合逻辑
* combOp： reducer 聚合逻辑
*/
def aggregate[U: ClassTag](zeroValue: U)(seqOp: (U, T) => U, combOp: (U, U) => U): U 
```



**Example：**

```scala
 //数据
val data = List("语文" -> 60, "英语" -> 80, "语文" -> 70, "英语" -> 100, "数学" -> 40, "英语" -> 90, "语文" -> 70, "数学" -> 100, "数学" -> 100, "语文" -> 30, "英语" -> 80)
val rdd: RDD[(String, Int)] = sc.makeRDD(data, 2)

//获取每门学科的总分和总人数
val rdd2: RDD[(String, (Int, Int))] = rdd.aggregateByKey(
  	(0, 0))
		((agg: (Int, Int), curr: Int) => (agg._1 + curr, agg._2 + 1),
    (agg: (Int, Int), curr: (Int, Int)) => (agg._1 + curr._1, agg._2 + curr._2))
// List((数学,(240,3)), (英语,(350,4)), (语文,(230,4)))

//计算每门学科的平均分
val rdd3: RDD[(String, Double)] = rdd2.map { case (name, (sumScore, count)) => (name, sumScore.toDouble / count) }
//List((数学,80.0), (英语,87.5), (语文,57.5))
    
```



### 四种 ByKey 方法的区别

四种方法的底层都是调用 `combineByKeyWithClassTag`实现。其函数签名为：

```scala
def combineByKeyWithClassTag[C](
    createCombiner: V => C,
    mergeValue: (C, V) => C,
    mergeCombiners: (C, C) => C,
    partitioner: Partitioner,
    mapSideCombine: Boolean = true,
    serializer: Serializer = null)
```

四种方法的调用：

```scala
// reduceByKey
def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] = self.withScope {
    combineByKeyWithClassTag[V]((v: V) => v, func, func, partitioner)
}

// flodByKey
// createZero: 传入的默认值
// cleanedFunc: 传入的方法
combineByKeyWithClassTag[V]((v: V) => cleanedFunc(createZero(), v),
      cleanedFunc, cleanedFunc, partitioner)

//combineByKey
combineByKeyWithClassTag(createCombiner, mergeValue, mergeCombiners,
      partitioner, mapSideCombine, serializer)(null)

//aggregateByKey
combineByKeyWithClassTag[U]((v: V) => cleanedSeqOp(createZero(), v),
      cleanedSeqOp, combOp, partitioner)
```



`reduceByKey`：combiner与reduce计算逻辑一模一样,combiner针对每个组第一次计算时，函数第一个参数的值 = 该组第一个value值[重点]

`foldByKey`： combiner与reduce计算逻辑一模一样，combiner针对每个组第一次计算时，函数第一个参数的值=默认值(传入)

`combinerByKey`: combiner与reduce计算逻辑可以不一样，combiner针对每个组第一次计算时，函数第一个参数的值 = createcombin函数的结果

`aggregateByKey`:combiner与reduce计算逻辑可以不一样，combiner针对每个组第一次计算时，函数第一个参数的值=默认值



### SorBytKey

按照key排序，默认升序

### mapValues

对每个键值对的value进行操作



## Action 行动算子

- reduce
  - reduce 没有shuffle结果
  - reduce是先对RDd每个分区中所有数据聚合，然后将每个分区的聚合结果给Driver汇总

- collect[重要]

  收集每个分区的数据交给Driver，如果RDD结果数据比较大，Driver 内存默认只有1G，所以可能出现Driver内存溢出。一般在工作中都需要调整Driver内存大小，一般通过`bin/spark-submit --driver-memroy`  设置为5-10G

- count: 统计RDD元素个数

- first: 获取RDD第一个元素(从0号分区开始，若为空则获取下一个分区)
- take: 获取前 N 个元素
- takeOrder: 获取排序之后的前N个元素
- aggregate: 在分区内combine 和diver端reduce时都会使用初始值
- fold: 和aggregate类似，只是combine函数和reduce函数相同。
- countByKey: 统计每个key出现的次数。在工作中，一般用于数据倾斜，工作中出现数据倾斜后一般先用sample算子采样数据，然后用countByKey统计样本数据出现的次数
- save：
- foreach
- foreachPartition

## RDD 序列化

Sparkd的初始化工作是在Driver端进行的，而实际运行程序是在Executor端进行的，这就涉及到了跨进程通信，是需要序列化的。

### 闭包检查



### 序列化

Spark中有两种序列化方式： `Java 序列化` 和`Kryo 序列化`。 



**Java 序列化**：

  能够序列化任何的类。但是比较重，序列化后对象的体积也比较大。在序列化对象的时候会将对象的属性类型、继承关系、包信息、类型的信息全部都序列化进去。



**kryo序列化：**

  在序列化对象的时候，只会序列化属性值等这些最基本信息。Spark 默认使用 Java 序列化，所以工作中一般需要切换成 kryo 序列化。







**使用 Kryo 序列化：**

1. 在创建sparkContext的时候通过参数指定spark的序列化方式：
2. 将需要进行序列化的类进行注册(可以不进行注册)

?> 注册与不注册的区别： 注册后序列化时不会将包的信息序列化进去。不注册会将报信息序列化进去

## RDD 依赖关系

#### 查看血缘关系

通过 toDebugString 查询血统



#### 查看依赖关系

dependencies 方法

`依赖关系`：指两个RDD之间的关系



**依赖关系分为两大类：**

- 宽依赖：有shuffle的称为宽依赖(父RDD一个分区的数据被子RDD多个分区所使用)

- 窄依赖：没有shuffle的称为窄依赖(父RDD一个分区的数据被子RDD一个分区所使用)





**相关概念：**

`Application`： 应用[一个sparkContex为一个应用]

`job`：一个 action 算子产生一个 job　

`stage`：一个job中会根据宽依赖划分stage，job中stage个数=shuffle个数 + 1

`task`： 子任务，一个 `stage` 中 `task` 的个数等于该 `stage` 中最后一个 `RDD` 的分区数



对应关系：

一个Application中可以产生多个job，多个job之间串行执行

一个job中多个stage，多个stage之间是串行

一个stage中有多个task，多个task是并行



stage切分：

当rdd调用Action算子的时候，此时会根据调用Action算子的rdd的依赖关系从后往前一次查询，如果两个RDD之间是宽依赖则划分stage，如果不是，继续往前直到找到第一个RDD为止

一个job执行的时候，stage的执行时从前往后执行





#### stage 任务划分(面试重点)



#### stage 任务划分 源码



## Spark 持久化

**出现原因：** 

​	如果一个对应的RDD，在多个job中重复使用的时候，此时默认情况下该RDD之前的处理步骤，每个 job 都会重复执行，效率较低



**使用场景：**

- RDD 在多个 job 中重复使用

- 血统过长时，当RDD计算链尾端出错时会从头执行

  

**RDD 持久化方式：**

- 缓存： 将数据保存在 task 所在主机的内存/磁盘中

- 使用方法： rdd.cache() / rdd.persist
  - cache底层调用无参的persist。 此时将数据保存在内存中
  - persist可以自由指定数据存储级别(指定将数据保存在内存 / 磁盘中)

- 存储级别：

  `NONE` 不存储
  `DISK_ONLY`只保存在磁盘
  `DISK_ONLY_2` 只保存在磁盘，数据保存两份
  `MEMORY_ONLY` 只保存在内存
  `MEMORY_ONLY_2`
  `MEMORY_ONLY_SER` 只保存在内存，数据序列化
  `MEMORY_ONLY_SER_2`
  `MEMORY_AND_DISK` 一部分数据在磁盘，一部分数据在内存
  `MEMORY_AND_DISK_2`
  `MEMORY_AND_DISK_SER`
  `MEMORY_AND_DISK_SER_2`
  `OFF_HEAP`

  工作常用： MEMORY_ONLY:小数量场景、MEMORY_AND_DISK：大数据量场景



**CheckPoint:**

​	将数据保存在HDFS中

- 原因： 
- 使用方式：
  1. 指定数据持久化的路径： 
  2. 持久化： rdd.checkpoitn()
  在 job 中，某一个 rdd 如果

