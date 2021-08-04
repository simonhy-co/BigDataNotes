## 广播变量

假设一个场景： 每个task都需要用到driver中的一份数据。

如果提交参数设置为：

```
bin/spark-submit --class xxx --master yarn --executor-cores 5 --number-executor 20 --executor-memory 2G \
--deploymode cluster
```

此时，本次任务需要的CPU总资源=20 * 5 = 100， 需要内存总资源= 20*2=40.

工作中设置RDD分区数一般设置为CPU总资源的2-3倍 = 300， task个数=分区数=300.

假设此时Driver的需要共享的数据的大小为100M，此时默认情况下，Driver会将该数据给每个task都发送一份。

此时该map在程序运行过程中占用的总内存大小 = task个数 * 数据大小 = 300 * 100M = 30G，此时数据占用的内存过大，导致计算的内存过小。

针对上述情况下数据占用内存空间过大的情况， Spark 提交了广播变量的机制，可以将所有task都需要的数据广播出去，此时数据会广播到executor中，所以此时程序运行过程中该数据占用的总内存空间= executor个数 * 数据大小 = 20 * 100M = 2G

### 使用场景：

1. 算子内用到Driver数据的时候，该数据稍微有点大。减少数据内存的占用
2. 大表 join 小表，可以避免 shuffle 操作

### 使用：

1. 广播数据：`val bc = sc.broadcast(data)`
2. 使用数据：`bc.value`

### 案例一

```scala
val sc: SparkContext = new SparkContext(new SparkConf().setMaster("local[4]").setAppName("test"))
//数据
val rdd1 = sc.parallelize(List("jd","atguigu","pdd","tb"))
val map = Map("jd"->"www.jd.com","atguigu"->"www.atguigu.com","pdd"->"www.pdd.com","tb"->"www.taobao.com")
//广播变量
val bc: Broadcast[Map[String, String]] = sc.broadcast(map)
val rdd2: RDD[String] = rdd1.map(bc.value.getOrElse(_, " "))
println(rdd2.collect().toList)
```

### 案例二-大小表join

需求：

```
有两张表：一张表记录 姓名，年龄，班级编号 另一张表记录 班级编号，班级名称
输出： 姓名，年龄，班级名称
```

代码一： 使用join算子

```scala
val sc: SparkContext = new SparkContext(new SparkConf().setMaster("local[4]").setAppName("test"))
// 两张表的数据
val rdd1 = sc.parallelize(List(("zhangsan",20,"A001"),("lisi",30,"A003"),("wangwu",33,"A001")))
val clasRdd = sc.parallelize(List("A001"->"python班","A003"->"大数据班","A002"->"java班"))

//将两张表的key调整为相同的格式，然后进行join
val rdd2: RDD[(String, ((Int, String), String))] = rdd1.map { 
  case (name, age, cls) => (name, (age, cls)) 
}.join(clasRdd)
val rdd3: RDD[(String, Int, String)] = rdd2.map { case (name, ((age, cls), cls1)) => (name, age, cls1) }
print(rdd3.collect().toList)
```

执行后会发现，这种方法会生成三个stage来运算。

<img src="category/spark/assets/img1.png" alt="截屏2021-08-04 下午1.05.20" style="zoom:50%;" />

代码二： 使用BroadCast

```scala
val sc: SparkContext = new SparkContext(new SparkConf().setMaster("local[4]").setAppName("test"))
//两张表的数据
val rdd1 = sc.parallelize(List(("zhangsan",20,"A001"),("lisi",30,"A003"),("wangwu",33,"A001")))
val cls = Map("A001"->"python班","A003"->"大数据班","A002"->"java班")
//广播 cls 表的数据
val bc = sc.broadcast(cls)
val rdd2: RDD[(String, Int, String)] = rdd1.map(x => (x._1, x._2, bc.value.getOrElse(x._3, " ")))
println(rdd2.collect().toList)
```

使用 BroadCast之后，stage数量只有一个

<img src="category/spark/assets/img2.png" alt="截屏2021-08-04 下午1.29.07" style="zoom:50%;" />

