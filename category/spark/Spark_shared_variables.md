# 共享变量

>通常情况下，传递给Spark运算的函数都是在集群中的不同节点运行的(RDD计算在Task中，其他代码运行在Driver中)。所以在方法中使用到的变量会复制成副本分发到每个节点，并且这些副本在计算后不会将副本变量的变化更新到Driver中原本的变量上。Spark支持两种方式的共享变量：广播变量(Broadcast variables) 和 累加器 (Accumulators)
>
>-- From [Spark官网](http://spark.apache.org/docs/latest/rdd-programming-guide.html#shared-variables)

例如下面一种情况，变量`sum`定义在RDD计算方法之外，并在RDD中对`sum`进行累加。运行结束后，`sum`的值并没有改变。

```scala
val data = List(1,2,3,4,5,6,7)
var sum:Int = 0
val rdd: RDD[Int] = sc.makeRDD( data )
rdd.foreach( sum + _ )
println(sum) // 0
```

## 累加器

累加器变量可以通过`add`方法对该变量进行相关操作，并且能在并行任务中有效的计算。Spark官方提供的累加器只支持数值类型，开发者可以自定义其他类型。你可以定义命名或匿名的累加器，命名累加器会显示在 Web 页面中。

### 系统累加器

```scala
val sc = new SparkContext(new SparkConf().setMaster("local").setAppName("test"))
val rdd = sc.parallelize(List(1, 2, 3, 4, 5, 6))

val sumAccu = sc.longAccumulator("sum")
rdd.foreach(sumAccu.add(_))

println(sumAccu.value)
```

**累加器的数据流向：**

先将累加器从Driver中发送到 Executor 的 Task 中。在 Task 中完成计算后，所有 Task 中的累加器将会发回给Driver，由Driver 调用 累加器的 `Merge ` 方法进行汇总。

### 自定义累加器

#### 定义与使用

1. 需要继承 AccumulatorV2[IN, OUT]
   - IN: 累加器的元素类型
   - OUT: 累加器最终结果类型
2. 重写抽象方法

3. 创建自定义累加器对象

4. 注册到SparkContext中

```scala
//需求： 使用自定义累加器替换到 WordCount 中的 reduceBy 操作

// 1. 自定义一个累加器 继承AccumulatorV2
class MyAccumulator extends AccumulatorV2[(String, Int), mutable.Map[String, Int]] {
  var res = mutable.Map[String, Int]()

  override def isZero: Boolean = res.isEmpty
  override def copy(): AccumulatorV2[(String, Int), mutable.Map[String, Int]] = new MyAccumulator
  override def reset(): Unit = res.clear()
  override def add(v: (String, Int)): Unit = res(v._1) = res.getOrElse(v._1, 0) + v._2
  override def merge(other: AccumulatorV2[(String, Int), mutable.Map[String, Int]]): Unit = {
    res ++=
      (other.value.toList ++ res.toList)
        .groupBy(_._1)
        .map(x => (x._1, x._2.map(_._2).sum))
  }
  override def value: mutable.Map[String, Int] = res
}


class $01_Accumulator {
  @Test
  def main(): Unit = {
    val sc: SparkContext = new SparkContext(new SparkConf().setMaster("local").setAppName("test"))
    val data = List("hello spark", "hello hadoop", "hadoop spark")
		
    //创建自定义累加器变量
    val acc = new MyAccumulator
    //注册到SparkContext
    sc.register(acc, "wordcount")
    
    sc.makeRDD(data).flatMap(_.split(" ")).map((_, 1)).foreach(acc.add)// 在RDD中使用累加器
    println(acc.value)
  }
}
```

>For accumulator updates performed inside **actions only**, Spark guarantees that each task’s update to the accumulator will only be applied once, i.e. restarted tasks will not update the value. In transformations, users should be aware of that each task’s update may be applied more than once if tasks or job stages are re-executed.
>
>-- From [Spark Official Website](http://spark.apache.org/docs/latest/rdd-programming-guide.html#accumulators);



## 广播变量

假设一个场景： 每个 Task 都需要用到 Driver 中的一份数据。

如果提交参数设置为：

```
bin/spark-submit --class xxx --master yarn --executor-cores 5 --number-executor 20 --executor-memory 2G --deploymode cluster
```

此时，本次任务需要的CPU总资源=20 * 5 = 100， 需要内存总资源= 20 * 2 = 40.

工作中设置 RDD 分区数一般设置为CPU总资源的2-3倍 = 300， Task 个数 = 分区数 = 300.

假设此时 Driver 的需要共享的数据的大小为100M，此时默认情况下，Driver会将该数据给每个task都发送一份。

此时该 map 在程序运行过程中占用的总内存大小 = Task个数 * 数据大小 = 300 * 100M = 30G，此时数据占用的内存过大，导致计算的内存过小。

针对上述情况下数据占用内存空间过大的情况， Spark 提交了广播变量的机制，可以将所有task都需要的数据广播出去，此时数据会广播到 executor 中，所以此时程序运行过程中该数据占用的总内存空间= executor个数 * 数据大小 = 20 * 100M = 2G

### 使用场景：

1. 算子内用到 Driver 数据的时候，该数据稍微有点大。减少数据内存的占用
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

**需求**：

```
有两张表：一张表记录 姓名，年龄，班级编号 另一张表记录 班级编号，班级名称
输出： 姓名，年龄，班级名称
```

**代码一： 使用join算子**

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

执行后会发现，这种方法会生成三个 stage 来运算。

<img src="category/spark/assets/img1.png" alt="截屏2021-08-04 下午1.05.20" style="zoom:50%;" />

**代码二： 使用BroadCast**

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

