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