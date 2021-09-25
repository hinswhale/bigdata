[TOC]

## [spark SQL](basics)



- [DataFrame&DataSet](basics/spark-core.md)
- [SQL的基本使用](basics/spark-core.md)
- [DSL语法的基本使用](basics/spark-core.md)
- [RDD之间的转换](basics/spark-core.md)
- [SQL的基本使用](basics/spark-core.md)
- [SQL的基本使用](basics/spark-core.md)



# 基本概念
## DataFrame
```DataFrame = Schema(表结构) + RDD（代表数据）```
## DataSet
   数据的分布式集合.Dataset是在Spark 1.6中添加的一个新接口，是DataFrame之上更高一级的抽象。


# DataFrame

 SparkSession是创建DataFrame和执行SQL的入口

示例文件： user.json
数据格式：
```json
{"username":"zhhangsan", "age":  10}
{"username":"wanwu", "age":  20}
{"username":"zhhangsan", "age":  30}
```
## 基本用法
![img.png](img.png)
![img_1.png](img_1.png)

## DataFrame创建
### 从spark数据源创建
 `spark.read.[json,text, ....]`
从内存中可获取数据类型，但文件中读取获取不到,数字用bigint接受不了

### RDD
- RDD与DataFrame互相转化
    
    ![img_2.png](../pic/sql/RDDandFrame.png)
- 示例
    ![img_2.png](../pic/sql/RDDtoDataFrame.png)

### Hive

### SQL的基本使用

```df.createOrReplaceTempView("user")
```

- table VS View
table 可修改
View 查询
- 普通临时表是session范围内的， df.createOrReplaceGlobalTempView VS df.createOrReplaceTempView

```scala
  spark.newSession.sql("select * from global_user.user")
  spark.newSession.sql("select * from global_temp.emp")
```
### DSL语法的基本使用

- 查看列数据

![img_2.png](../pic/sql/DSL1.png)

- 列数据运算 如"age+1", 每个列前加'或$
  
  ```df.select("age"+1).show ``` 报错

  ![img_3.png](../pic/sql/DSL2.png)

# DataSet

## 创建
- DataFrame => DataSet
```shell
scala> val df = spark.read.json("/Users/liusj/spark-demo/datas/user.json")
df: org.apache.spark.sql.DataFrame = [age: bigint, username: string]

scala> df.show
+---+---------+
|age| username|
+---+---------+
| 10|zhhangsan|
| 20|    wanwu|
| 30|zhhangsan|
+---+---------+

scala> case class emp(username:String, age:Long)
defined class emp

scala> val ds = df.as[emp]
ds: org.apache.spark.sql.Dataset[emp] = [age: bigint, username: string]

scala> ds.show
+---+---------+
|age| username|
+---+---------+
| 10|zhhangsan|
| 20|    wanwu|
| 30|zhhangsan|
+---+---------+
```
- DataSet => DataFrame

  ```ds.toDF()```

- RDD <=> DataSet

```shell
scala> case class emp(username:String, age:Long)
scala> val rdd = sc.makeRDD(List(emp("zhang", 30), emp("HAHAH", 10)))
rdd: org.apache.spark.rdd.RDD[emp] = ParallelCollectionRDD[36] at makeRDD at <console>:26

scala> rdd.toDS
res16: org.apache.spark.sql.Dataset[emp] = [username: string, age: bigint]
scala> rdd.toDS.rdd
res17: org.apache.spark.rdd.RDD[emp] = MapPartitionsRDD[39] at rdd at <console>:26
```

# RDD  DataFrame  DataSet 转化关系

```scala
  type DataFrame = Dataset[Row]

```

![img_2.png](../pic/sql/dataset-rdd-dataframe.png)

```scala
package spark.sql

import org.apache.spark.rdd.RDD
import org.apache.spark.sql.{DataFrame, Dataset, Row, SparkSession}
import org.apache.spark.{SparkConf, SparkContext}

object Basic {
  def main(args: Array[String]): Unit = {
    val sparkConf: SparkConf = new SparkConf().setMaster("local").setAppName("Bc")
    val sess = SparkSession.builder().config(sparkConf).getOrCreate()
    import sess.implicits._

    // DataFrame
    //    val df: DataFrame = sess.read.json("datas/user.json")
    //df.show()

    //DataFrame => SQL
    //    df.createOrReplaceTempView("user")
    //    sess.sql("select age, username from user").show
    //    sess.sql("select avg(age) from user").show

    //DataFrame => DSL
    //    //转换操作,需引入转换规则
    //    df.select("age", "username").show()
    //    df.select($"age"+1).show()


    //    val seq = Seq(1, 2, 3, 4)
    //    val ds: Dataset[Int] = seq.toDS()
    //    ds.show()

    //DataSet DataFrame是特定泛型DataSet
    // DataFrame方法都适合DataSet
    //    val ds = sess.read.json("datas/user.json")
    //    ds.createOrReplaceTempView("user")
    //    sess.sql("select age, username from user").show

    //RDD <=> DataFrame
    val rdd = sess.sparkContext.makeRDD(List((1, "zhangsan", 40), (2, "wangwu", 2)))
    val df = rdd.toDF("id", "name", "age")
    val rdd1: RDD[Row] = df.rdd

    //DataSet <=> DataFrame
    val ds: Dataset[User] = df.as[User] // df => ds
    ds.toDF() // ds => df


    //RDD <=> DataSet
    val ds1: Dataset[User] = rdd.map {
      case (id, name, age) => {
        User(id, name, age)
      }
    }.toDS()
    val rdd2: RDD[User] = ds1.rdd
    sess.close()

  }

  case class User(id: Int, name: String, age: Int)

}

```

# UDF函数
## 弱类型
## 强类型

# 文件读取

![img_4.png](img_4.png)

```scala
scala>  sess.read.load
scala> sess.read.json("filename")
scala> spark.sql("select * from json.`datas/user.json`").show
+---+---------+
|age| username|
+---+---------+
| 10|zhhangsan|
| 20|    wanwu|
| 30|zhhangsan|
+---+---------+
scala>  df.write.save("filename")
scala>  df.write.save("filename") //报错。已存在
scala> df.write.format("json").mode("ignore").save("out") //不报错
```

## csv
```scala
val dataFrame: DataFrame = spark.read.format("csv")
      .option("header", "true")
      .option("encoding", "gbk2312")
      .load(path)
```

## Mysql

![img_5.png](img_5.png)

![img_6.png](img_6.png)

## Hive
### 内置
### 外置
- 连接方法

![img_7.png](../pic/sql/Hive.png)
