# RDD 序列化

Sparkd的初始化工作是在Driver端进行的，而实际运行程序是在Executor端进行的，这就涉及到了跨进程通信，是需要序列化的。

## 闭包检查



## 序列化

Spark中有两种序列化方式： `Java 序列化` 和`Kryo 序列化`。 

### Java 序列化：

能够序列化任何的类。但是比较重，序列化后对象的体积也比较大。在序列化对象的时候会将对象的属性类型、继承关系、包信息、类型的信息全部都序列化进去。

### kryo序列化:

在序列化对象的时候，只会序列化属性值等这些最基本信息。Spark 默认使用 Java 序列化，所以工作中一般需要切换成 kryo 序列化。

### 使用 Kryo 序列化：

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