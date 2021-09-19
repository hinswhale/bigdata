# RDD

> 📌 **关键词：** `RDD`、`stage`、`Transformations& Actions`、`宽依赖&窄依赖`、`分区`


<!-- TOC depthFrom:2 depthTo:3 -->

- [1. rdd是什么](#1-rdd是什么)
- [2. 创建方式](#2-创建方式)
    - [2.1. 通过读取文件生成的](#31-通过读取文件生成的)
    - [2.2. 通过并行化的方式创建RDD](#32-通过并行化的方式创建RDD)
- [3. rdd并行度与分区](#3-rdd并行度与分区)
- [4. 常用RDD算子](#4-环境变量)
    - [4.1. Transformation](#41-Transformation)
    - [4.2. Action](#42-Action)
- [5. RDD依赖关系](#5-RDD依赖关系)
    - [5.1. 宽依赖](#51-宽依赖)
    - [5.2. 窄依赖](#52-窄依赖)
    - [5.3. 总结](#53-总结)
- [6. DAG的生成和划分Stage](#6-DAG的生成和划分Stage)
    - [6.1. stage划分](#61-stage划分)
    - [6.3. DAG/job/Action/分区/关系](#62-DAG/job/Action/分区/关系)
- [7. 持久化/缓存](#7-持久化/缓存)
    - [7.1. cache](#71-cache)
    - [7.2. checkpoint](#71-checkpoint)
- [8. 性能优化](#8-性能优化)

<!-- /TOC -->

# 1. rdd是什么

RDD(Resilient Distributed Dataset) 弹性分布式数据集

### 定义

- 数据集
    - 存储数据的计算逻辑
- 分布式
    - 数据来源：从不同网络结点读取数据
    - 计算：拆分成不同任务，发送给不同的excute
    - 数据存储
- 弹性
    - 依赖关系
        - 维护血缘关系
    - 计算
        - 计算基于内存，性能高，可与磁盘灵活切换
    - 分区
        - 可通过指定算子改变分区数量，并行计算
        - 数据不同，计算逻辑相同
    - 容错
        - 重试处理

### 特性：

* 只读 - 只能通过算子创建新的RDD，原RDD未受影响【算子】
* 分区 - 每个 RDD 被切分成多个分区(partition), 每个分区可能会在集群中不同的节点上进行计算
* 依赖关系可并行 - DAG，划分并行
* 缓存&持久化

# 2. 创建方式

## 2.1. 通过读取文件生成的

- 以行为单位，读取数据都是字符串

```scala
val rdd1 = sc.textFile("hdfs://node1:8020/wordcount/input/words.txt")
```

- 以文件为单位

```scala
val rdd1 = sc.whoTextFiles("hdfs://node1:8020/wordcount/input/words.txt")
```

## 2.2. 通过并行化的方式创建RDD

```scala
num_rdd = sc.parallelize([1,2,3]) //parallelize 并行
```

或

```scala
val rdd = sc.makeRDD(List(1, 2, 3, 4), 2) // 2为分区，不写会有默认值
```

makeRDD方法底层调用了parallelize方法

## 2.3. 其他方式

# 3 rdd并行度与分区

## 分区算法

![img.png](../../pic/分区.png)

- 偏移量 例子： 最小分区：3 文件内容：
  ```text
    1
    2
    3
   ```
  结果： 文件1： 1 2 文件2：3 文件3： 空

# 4. 常用RDD算子

**`惰性求值`**

- Transformations ：从一个已知的 RDD 中创建出来一个新的 RDD
- Actions： 在数据集上计算结束之后, 给驱动程序返回一个值

在 Spark 中几乎所有的transformation操作都是`懒执行`的(lazy).

只有遇到action，才会执行 RDD 的计算(即延迟计）,只有遇到action，才会执行 RDD 的计算(即延迟计）

## 4.1. Transformation

将旧的RDD包装成新的RDD

### 单Value类型

1. map

* 一个分区 数据顺序执行
* 不同分区间 无序

```
scala > val rdd1 = sc.parallelize(1 to 10)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at parallelize at <console>:24
// 得到一个新的 RDD, 但是这个 RDD 中的元素并不是立即计算出来的
scala> val rdd2 = rdd1.map(_ * 2, numslice=2) //numslice 分区
rdd2: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[1] at map at
<console>:26

// 开始计算 rdd2 中的元素, 并把计算后的结果传递给驱动程序
scala> rdd2.collect
res0: Array[Int] = Array(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
  ```  

2. mapPartitions 考虑分区

* 语法

  ```mapPartitions(func) Iterator<T> => Iterator<U>```

* 可应用场景 a. 每个分区最大值或对分区数据做批处理

* 🏠`map vs mapPartitions`
    * mapPartitions 性能更高，每个分区一次拿到所有数据

      e.g. 假设有N个元素，有M个分区，那么map的函数的将被调用N次, 而mapPartitions被调用M次,一个函数一次处理所有分区。
    * mapPartitions 内存有限时不推荐 处理完的数据不会被释放，存在对象引用，数据量较大的时候，容易内存溢出，此时应考虑map
    * map 转换后数量不变，mapPartitions可以改变

```scala
package spark.core.rdd.transform

import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object mapPartitions {
  def main(args: Array[String]): Unit = {
    val sparkConf: SparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 2)
//    val mpRDD: RDD[Int] = rdd.mapPartitions(
//      iter => {
//        println(">>>>")
//        iter.map(_ * 2)
//      }
//    )

    // 最大值
    val mpRDD: RDD[Int] = rdd.mapPartitions(
      iter => {
        List(iter.max).iterator
      }
    )

    mpRDD.collect().foreach(println)

    sc.stop()
  }
}

```

3. mapPartitionsWithIndex 索引号

* 语法

  ``` mapPartitionsWithIndex(func) (Int, Iterator<T>) => Iterator<U>```
* 功能
    * 多提供一个Int值来表示分区的索引
    * 分区数的确定, 和对数组中的元素如何进行分区
* 示例代码 取某个分区数据

```scala
package spark.core.rdd.transform

import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object mapPartitionsWithIndex {
  def main(args: Array[String]): Unit = {
    val sparkConf: SparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 2)
    val mpiRDD: RDD[Int] = rdd.mapPartitionsWithIndex(
      (index, iter) => {
        if (index == 1) {
          iter
        } else {
          Nil.iterator
        }
      }
    )

    mpiRDD.collect().foreach(println)

    sc.stop()

  }
}
```
     
4. flatMap 扁平化
   * 功能 扁平化 
       * 如[[1, 2] [3, 4]] = > [1, 2, 3, 4]
       * 拆分单词 "Hello Scala", "Hello Spark" => Hello  Scala Hello Spark
  
   * 代码

```scala
/**
[1, 2] [3, 4] = > [1, 2, 3, 4]
/**
package spark.core.rdd.transform

import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object flatMap {
  def main(args: Array[String]): Unit = {
    val sparkConf: SparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd: RDD[List[Int]] = sc.makeRDD(List(List(1, 2), List(3, 4)), 2)
    val flatrdd: RDD[Int] = rdd.flatMap(
      List => {List}
    )

    flatrdd.collect().foreach(println)

    sc.stop()

  }
}
  ```

* ～模式匹配～
```scala
val rdd = sc.makeRDD(List(List(1, 2), 3, List(4, 5)))

val flatMapRDD = rdd.flatMap(
  data => {
    data match {
      case list: List[_] => list
      case dat => List(dat)
    }
  }
)
```
5. glom 将每个分区形成一个数组
   * 用法
     ```RDD[Array[T]] ```
   * 功能 
     * 将每个分区形成一个数组
     * 分区个数不变
   * 代码
   
   打印各分区数据

   ```scala
    val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 2)
    val glomRDD: RDD[Array[Int]] = rdd.glom(
    )

    glomRDD.foreach(arr => println(arr.mkString(",")))
   /*
    1,2
    3,4
   */
   ```
   求各分区最大值之和

    ```scala
    val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 2)
    val glomRDD: RDD[Array[Int]] = rdd.glom()
    val maxRDD: RDD[Int] = glomRDD.map(array => {
      array.max
    })
    println(maxRDD.collect().sum)
    // 结果：6
    ```
  
6. groupBy
   * 用法
     ```groupBy(func) RDD[(K, Iterable[T])```
   * 功能 
     * 按照func的返回值做为key进行分组
     * shuffle
     
   * 代码
   
    ```scala
    val rdd = sc.makeRDD(List(1, 2, 3, 4))
    val groupRDD: RDD[(Int, Iterable[Int])] = rdd.groupBy((_%2)
    groupRDD.collect().foreach(println)
    ```

7. filter
   * 用法
     `````
   * 功能 
     * 过滤
     * 产生数据倾斜 分区各区数据差别较大

8. sample 随机采样
   * 用法
    sample算子需要传递三个参数
     * 第一个参数  抽取数据后是否将数据返回 
       * true（返回：（泊松算法）） false（不返回：（伯努利算法））
     * 第二个参数 每个数据出现概率
        * 如果抽取不放回的场合：数据源中每条数据被抽取的概率,基准值的概念
        * 如果抽取放回的场合：表示数据源中的每条数据被抽取的可能次数
     * 第三个参数 随机算法种子 如果不传，则使用当前时间
   * 功能 
     * 数据倾斜时应用

  * 代码

  ```scala
  import org.apache.spark.rdd.RDD
  import org.apache.spark.{SparkConf, SparkContext}
  
  object Spark08_RDD_Operator_Transform {
    def main(args: Array[String]): Unit = {
      val sparkConf: SparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
      val sc = new SparkContext(sparkConf)
  
      // TODO 算子 —— sample
      val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10))
  
      /*val sampleRDD: RDD[Int] = rdd.sample(
        false,
        0.4
        //1
      )*/
  
      val sampleRDD: RDD[Int] = rdd.sample(
        true,
        2
        //1
      )
  
      println(sampleRDD.collect().mkString(","))
      sc.stop()
    }
  }

```

10. distinct
11. coalesce 缩减分区
12. repartition shuffle 随机洗牌
13. sortBy

- K-V类型
  1. map
  2. mapPartitions 分区
  2. mapPartitionsWithIndex
  3. flatMap 扁平化
  4. map  vs mapPartitions  mapPartitions 批处理 性能好，但数据不释放
  5. glom 将每个分区形成一个数组
  6. groupBy
  7. filter
  8. sample 采样，数据倾斜时应用
  9. distinct
  10. coalesce 缩减分区
  11. repartition shuffle 随机洗牌
  12. sortBy
- 
- 双V类型
  1. union
  2. subtract
  2. intersection
  3. zip 一一对应 分区数量相同，分区内数据相同

## 4.2. Action

触发任务调度和作业的执行

  1. reduce
  2. collect
  3. first
  4. take
  5. takeOrdered
  6. aggregate 分区间初始值
  7. foreach
  8. countByKey
  9. save

# 5. RDD依赖关系

是否shuffle
![img.png](../../pic/依赖关系.png)
## 5.1. 宽依赖
包含Shuffle过程，无法实现流水线方式处理
- 父 RDD 的分区被不止一个子 RDD 的分区依赖
- 具有宽依赖的 transformations 包括: sort, reduceByKey, groupByKey, join, 和调用rePartition函数的任何操作.

## 5.2. 窄依赖
可以实现流水线优化
- 父 RDD 中的每个分区最多只有一个子分区, 形象的比喻为独生子女
- 可以在任何的的一个分区上单独执行, 而不需要其他分区的任何信息.

## 5.3 总结
`shuffle` 操作是 spark 中最耗时的操作,应尽量避免不必要的 `shuffle`.

# 6. DAG的生成和划分Stage
![img.png](../../pic/stage.png)

划分stage的依据就是RDD之间的宽窄依赖

Spark任务会根据RDD之间的依赖关系，形成一个DAG有向无环图，DAG会提交给DAGScheduler，
DAGScheduler会把DAG划分成互相依赖的多个stage。

核心算法：回溯算法 从后往前回溯/反向解析，遇到窄依赖加入本Stage，遇见宽依赖进行Stage切分。


## 6.1 stage划分
- 对于窄依赖，partition的转换处理在Stage中完成计算。
- 对于宽依赖，由于有Shuffle的存在，只能在parent RDD处理完成后，才能开始接下来的计算，因此宽依赖是划分Stage的依据。

## 6.2 DAG/job/Action/分区/关系
![img.png](../../pic/stage分析.png)


* 概念
  - Application
  - job  执行一个行动操作，就会执行sc.runJob(...)
  - stages 
  - tasks 最小执行单位  一个分区划一个Task, 每一个 task 表现为一个本地计算

* 联系⚠️
  - - Application->Job->Stage-> Task 1对多
  - 有几个Action，就有几个DAG,一个程序可有多个DAG
  - 一个DAG可以有多个Stage【根据宽依赖/shuffle进行划分】
  - 同一个Stage可以有多个Task并行执行(task数=分区数，如上图，Stage1 中有三个分区P1、P2、P3，对应的也有三个 Task)



# 7. 持久化/缓存
## cache
- 将该 RDD 缓存起来，该 RDD 只有在第一次计算的时候会根据血缘关系得到分区的数据，在后续其他地方用到该 RDD 的时候，会直接从缓存处取而不用再根据血缘关系计算，这样就加速后期的重用
## checkpoint


## 8. 性能优化

## 9. 参考资料

常见的 Java IDE 如下：

- Eclipse - 一个开放源代码的、基于 Java 的可扩展开发平台。
- NetBeans - 开放源码的 Java 集成开发环境，适用于各种客户机和 Web 应用。
- IntelliJ IDEA - 在代码自动提示、代码分析等方面的具有很好的功能。
- MyEclipse - 由 Genuitec 公司开发的一款商业化软件，是应用比较广泛的 Java 应用程序集成开发环境。
- EditPlus - 如果正确配置 Java 的编译器“Javac”以及解释器“Java”后，可直接使用 EditPlus 编译执行 Java 程序。

