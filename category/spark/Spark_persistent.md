## Spark 持久化

**出现原因：** 

	如果一个对应的RDD，在多个job中重复使用的时候，此时默认情况下该RDD之前的处理步骤，每个 job 都会重复执行，效率较低



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

```
将数据保存在HDFS中
```

- 原因： 
- 使用方式：
  1. 指定数据持久化的路径： 
  2. 持久化： rdd.checkpoitn()
  在 job 中，某一个 rdd 如果