# SPARK

## Spark 概述

Spark 是一种基于`内存`的快速、通用、可扩展的大数据`分析计算引擎`。





### MR 与 Spark 框架对比



MR 从数据源获取数据，经过分析计算后，将结果输出，核心是一次计算，不适合迭代计算。每次计算后会落盘，磁盘IO多，迭代计算的话(多job串联)效率低。



spark： 支持迭代式计算，中间不落盘(shuffle 会落盘)



spark快的原因：

1. 中间结果保存在内存中
2. spark task是线程级别的， MR task 是进程级别的



### Spark内置模块





Spark Core：负责任务调度， 切分数据等

Spark SQL： 结构化数据处理

Spark



## Spark 运行模式

### Local模式

#### 安装

解压 Spark 安装包

```bash
tar -zxvf spark-3.0.0-bin-hadoop3.2.tgz -C /opt/module/
mv spark-3.0.0-bin-hadoop3.2 spark-local
```

#### 使用

使用`spark-submit`提交任务

使用官方求π案例进行测试

```bash
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master local[2] \
./examples/jars/spark-examples_2.12-3.0.0.jar \
10

```

参数：

- `--class`: 要执行程序的主类

- `--master local[2]`:
  - local:没有指定线程数，则所有计算都运行在一个线程当中，没有任何并行计算
  - local[**K**]:指定使用 **K** 个 Core 来运行计算，比如 local[2] 就是运行2个 Core 来执行
  - <span style="color:red">local[*]：默认模式。</span>自动帮你按照CPU最多核来设置线程数。比如CPU有8核，Spark帮你自动设置8个线程计算。
- spark-examples_2.12-3.0.0.jar：要运行的程序；
- 10：要运行程序的输入参数（计算圆周率π的次数，计算次数越多，准确率越高）；





### 集群角色

### Standalone 模式

### Yarn模式





### 使用

spark-shell







## WordCount 案例







# Spark Core

