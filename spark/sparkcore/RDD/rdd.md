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
- [7. 第一个程序：Hello World](#7-第一个程序hello-world)
  - [7.1. cache](#71-cache)
  - [7.2. checkpoint](#71-checkpoint)


<!-- /TOC -->

## 1. rdd是什么
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
  * 只读   - 只能通过算子创建新的RDD，原RDD未受影响【算子】
  * 分区   -  每个 RDD 被切分成多个分区(partition), 每个分区可能会在集群中不同的节点上进行计算
  * 依赖关系可并行   - DAG，划分并行
  * 缓存&持久化

## 2. 创建方式

### 2.1. 通过读取文件生成的

- 以行为单位，读取数据都是字符串
```scala
val rdd1 = sc.textFile("hdfs://node1:8020/wordcount/input/words.txt")
```
- 以文件为单位
```scala
val rdd1 = sc.whoTextFiles("hdfs://node1:8020/wordcount/input/words.txt")
```

### 2.2. 通过并行化的方式创建RDD

```scala
num_rdd = sc.parallelize([1,2,3]) //parallelize 并行
```
或
```scala
val rdd = sc.makeRDD(List(1, 2, 3, 4), 2) // 2为分区，不写会有默认值
```
makeRDD方法底层调用了parallelize方法

### 2.3. 其他方式


## 3 rdd并行度与分区
- 分区算法
![img.png](../../pic/分区.png)
- 偏移量
例子：
  最小分区：3 文件内容：
  ```text
    1
    2
    3
   ```
  结果： 文件1： 1 2   文件2：3  文件3： 空

## 4. 常用RDD算子

**`惰性求值`**

- Transformations ：从一个已知的 RDD 中创建出来一个新的 RDD
- Actions： 在数据集上计算结束之后, 给驱动程序返回一个值

在 Spark 中几乎所有的transformation操作都是`懒执行`的(lazy).

只有遇到action，才会执行 RDD 的计算(即延迟计）,只有遇到action，才会执行 RDD 的计算(即延迟计）

### 4.1. Transformation
- 单Value类型
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
- 双V类型
  1. union
  2. subtract
  2. intersection
  3. zip 一一对应 分区数量相同，分区内数据相同

### 4.2. Action
  1. reduce
  2. collect
  3. first
  4. take
  5. takeOrdered
  6. aggregate 分区间初始值
  7. foreach
  8. countByKey
  9. save

## 5. RDD依赖关系

是否shuffle
![img.png](../../pic/依赖关系.png)
### 5.1. 宽依赖
包含Shuffle过程，无法实现流水线方式处理
- 父 RDD 的分区被不止一个子 RDD 的分区依赖
- 具有宽依赖的 transformations 包括: sort, reduceByKey, groupByKey, join, 和调用rePartition函数的任何操作.

### 5.2. 窄依赖
可以实现流水线优化
- 父 RDD 中的每个分区最多只有一个子分区, 形象的比喻为独生子女
- 可以在任何的的一个分区上单独执行, 而不需要其他分区的任何信息.

### 5.3 总结
`shuffle` 操作是 spark 中最耗时的操作,应尽量避免不必要的 `shuffle`.

## 6. DAG的生成和划分Stage
![img.png](../../pic/stage.png)

划分stage的依据就是RDD之间的宽窄依赖

Spark任务会根据RDD之间的依赖关系，形成一个DAG有向无环图，DAG会提交给DAGScheduler，
DAGScheduler会把DAG划分成互相依赖的多个stage。

核心算法：回溯算法 从后往前回溯/反向解析，遇到窄依赖加入本Stage，遇见宽依赖进行Stage切分。


### 6.1 stage划分
- 对于窄依赖，partition的转换处理在Stage中完成计算。
- 对于宽依赖，由于有Shuffle的存在，只能在parent RDD处理完成后，才能开始接下来的计算，因此宽依赖是划分Stage的依据。

### 6.2 DAG/job/Action/分区/关系
![img.png](../../pic/stage分析.png)


* 概念
  - DAG
  - job 
  - stages 
  - tasks 最小执行单位  每一个 task 表现为一个本地计算

* 联系⚠️
  - - Application->Job->Stage-> Task 1对多
  - 有几个Action，就有几个DAG,一个程序可有多个DAG
  - 一个DAG可以有多个Stage【根据宽依赖/shuffle进行划分】
  - 同一个Stage可以有多个Task并行执行(task数=分区数，如上图，Stage1 中有三个分区P1、P2、P3，对应的也有三个 Task)



## 7. 持久化
### cache
- 将该 RDD 缓存起来，该 RDD 只有在第一次计算的时候会根据血缘关系得到分区的数据，在后续其他地方用到该 RDD 的时候，会直接从缓存处取而不用再根据血缘关系计算，这样就加速后期的重用
### checkpoint


## 8. 第一个程序：Hello World

## 9. 参考资料

常见的 Java IDE 如下：

- Eclipse - 一个开放源代码的、基于 Java 的可扩展开发平台。
- NetBeans - 开放源码的 Java 集成开发环境，适用于各种客户机和 Web 应用。
- IntelliJ IDEA - 在代码自动提示、代码分析等方面的具有很好的功能。
- MyEclipse - 由 Genuitec 公司开发的一款商业化软件，是应用比较广泛的 Java 应用程序集成开发环境。
- EditPlus - 如果正确配置 Java 的编译器“Javac”以及解释器“Java”后，可直接使用 EditPlus 编译执行 Java 程序。

