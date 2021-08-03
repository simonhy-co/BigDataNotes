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