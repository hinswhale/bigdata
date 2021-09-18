# RDD

> 📌 **关键词：** JAVA_HOME、CLASSPATH、Path、环境变量、IDE

<!-- TOC depthFrom:2 depthTo:3 -->

- [1. rdd是什么](#1-rdd是什么)
- [2. 常用RDD算子](#2-常用RDD算子)
- [3. 环境变量](#3-环境变量)
  - [3.1. Windows](#31-windows)
  - [3.2. Linux](#32-linux)
- [4. 测试安装成功](#4-测试安装成功)
- [5. 开发工具](#5-开发工具)
- [6. 第一个程序：Hello World](#6-第一个程序hello-world)
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
        - spark通过特别的处理方案简化依赖关系
    - 计算
        - 计算基于内存，性能高，可与磁盘灵活切换
    - 分区
        - 可通过指定算子改变分区数量
    - 容错
        - 重试处理
- 特性：
  * 不可变，只能转化【算子】
  * 可分区
  * 缓存【防止血缘关系断掉，根据DAG保存中间过程减少运算量】
  * 依赖关系【DAG，划分并行】

## 2. 创建方式

### 2.1. 通过读取文件生成的
### 2.2. 通过并行化的方式创建RDD
### 2.3. 其他方式


## 2. 常用RDD算子

### 2.1. Transformation
### 2.2. Action


## 3. RDD依赖关系

### 3.1. 宽依赖


### 3.2. 窄依赖


## 4. DAG的生成和划分Stage

执行命令 `java -version` ，如果安装成功，会打印当前 java 的版本信息。

## 5. 参考资料

常见的 Java IDE 如下：

- Eclipse - 一个开放源代码的、基于 Java 的可扩展开发平台。
- NetBeans - 开放源码的 Java 集成开发环境，适用于各种客户机和 Web 应用。
- IntelliJ IDEA - 在代码自动提示、代码分析等方面的具有很好的功能。
- MyEclipse - 由 Genuitec 公司开发的一款商业化软件，是应用比较广泛的 Java 应用程序集成开发环境。
- EditPlus - 如果正确配置 Java 的编译器“Javac”以及解释器“Java”后，可直接使用 EditPlus 编译执行 Java 程序。


## 6. 第一个程序：Hello World