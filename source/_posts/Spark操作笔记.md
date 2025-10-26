---
title: Spark操作笔记
date: 2018-01-10 01:26:49
tags: Spark
---
## Spark简介
### 什么是Spark？
Apache Spark是用于大规模数据处理的快速(fast)和通用(general)引擎，由加州伯克利分校AMP(Algorithms、Machines and People Lab，在算法、机器和人之间通过大规模集成来展现大数据的应用平台)实验室开发的大数据处理框架。
Spark提供了大数据处理的一站式解决方案，以Spark Core为基础推出了Spark SQL、Spark Streaming、MLlib、GraphX、SparkR等组件。整个Spark生态体系称为BDAS，即：伯克利数据分析栈。

<!--more-->
### Spark特点
Spark具有运行速度快、易用性好、通用型强和随处运行的特点。
#### 运行速度快(Speed)
如果Spark基于内存读取，速度是Hadoop的100倍；使用磁盘读取，也是Hadoop的十倍。spark之所以能够比Hadoop快，有两点主要原因：内存计算和引入DAG执行引擎。
#### 易用性好(Ease of Use)
Spark支持Scala、Java、Python、R语言编写应用程序，并且提供了Scala、Python、R的交互式操作shell。
#### 通用型强(generality)
Spark提供了一站式的大数据解决方案，生态圈BADS包含了：提供内存计算框架的Spark Core、用于结构化查询的Spark SQL、用于实时计算的Spark Streaming、用于机器学习的MLlib和用于图计算的GraphX。
#### 随处运行(Runs Everywhere)
Spark提供了本地Local运行模式，用来学习和测试(当然还有许多用途，比如我们正在做的一个项目就是基于Local模式的)。对于集群部署模式，Spark能够以YARN、Mesos和自身提供的Standalone作为资源管理调度框架来执行作业。对于数据源，Spark能够读取HDFS、Cassandra、HBase、S3、Alluxio等数据源数据。
## 资源调度器YARN
### YARN简介
YARＮ是Spark的3种调度器之一,主要是用来管理和分配集群资源
需要根据情况修改YARN默认参数配置(修改yarn-site.xml或者CDH设置):
### 提交任务到集群,根据计算类型分配各自参数:
* IO密集型:每个exector的cores为１就好;
* driver-memory:主要取决于最后聚合输出(如collect操作)的结果大小;
* num-executors:最好与代码里面的partition或coalesce操作的分区数量相匹配;
* executor-memory:取决于每个stage需要的内存大小;
* 为了使代码更加清晰可读，shell代码可以使用\来分行
```
#!/usr/bin/env sh
sbt package &&\
scp target/scala-2.11/metro-data_2.11-1.0.jar \
root@c1:~/chengsluo/ &&\
ssh root@c1 -o StrictHostKeyChecking=no "
cd chengsluo && \
spark2-submit \
--master yarn \
--deploy-mode cluster \
--executor-memory 1g \
--executor-cores 2 \
metro-data_2.11-1.0.jar 
"
```
### 提前中断任务
```
 yarn application -kill application_1516444072266_0002
```
## HIVE和Spark-SQL性能对比
### 逻辑介绍
* 生成OD断面中间表,通过历史运行数据来得到运行线路；
* 由一个历史OD比例表与一个子查询链接,共生成130441786条数据。
### 性能对比
|---|Spark with Python|Spark with Scala|HIVE On MapReduce|
|-|-|-|-|
|耗时| 4.7min | 4.4min | 20min|
其中HIVE on MapReduce可能受大屏项目的其他查询影响，导致速度很可能比正常要慢一些。
### 三种实现
HIVE-SQL语句:
```
insert overwrite table sparktest.tbl_dm_od_cust_mid1 select 
    dm_id     
 ,a.od_begin_cd
    ,a.od_end_cd
    ,a.time_id
    ,line_id
    ,GD_begin_cd
    ,gd_end_cd
    ,lead(gd_end_cd,1,'0000') over(partition by substr(cast(dm_id as string),1,9) order by run_time) as gd_end_cd_nxt
    ,gd_begin_nm
    ,gd_end_nm
    ,a.time_id+run_time/3 as sta_time_id
    ,a.time_id+max_time/3 end_time_id
    ,cast(b.percent*a.cust_rate as float) precent
from
    sptcc_dm.tbl_dm_od_rate a,
    (select
         dm_id
   ,line_id
         ,od_begin_cd
         ,od_end_cd
         ,gd_begin_cd
         ,gd_end_cd
    ,lead(gd_end_cd,1,'0000') over(partition by substr(cast(dm_id as string),1,9) order by run_time) as gd_end_cd_nxt
         ,gd_begin_nm
         ,gd_end_nm
         ,run_time
         ,percent
         ,max(run_time) over(partition by substr(cast(dm_id as string),1,9)) as max_time
    from sptcc_dpa.tbl_dim_od_dm
    ) b
where
    a.od_begin_cd=b.od_begin_cd
    and a.od_end_cd=b.od_end_cd
```
Spark实现代码:
```scala
package org.apache.spark.examples.sql.hive
import org.apache.spark.sql.SparkSession
object SparkHiveExample {
  def main(args: Array[String]) {
    val spark = SparkSession
      .builder()
      .appName("Spark Speed Test")
      .enableHiveSupport()
      .getOrCreate()
    import spark.sql
    sql("insert overwrite table sparktest.tbl_dm_od_cust_mid1 " +
      "select " +
      "dm_id," +
      "a.od_begin_cd," +
      "a.od_end_cd," +
      "a.time_id," +
      "line_id," +
      "GD_begin_cd," +
      "gd_end_cd," +
      "lead(gd_end_cd,1,'0000') over(partition by substr(cast(dm_id as string),1,9) order by run_time) as gd_end_cd_nxt," +
      "gd_begin_nm," +
      "gd_end_nm," +
      "a.time_id+run_time/3 as sta_time_id ," +
      "a.time_id+max_time/3 end_time_id," +
      "cast(b.percent*a.cust_rate as float) precent " +
      "from " +
      "sptcc_dm.tbl_dm_od_rate a, " +
      "(select dm_id ,line_id,od_begin_cd,od_end_cd,gd_begin_cd,gd_end_cd ,lead(gd_end_cd,1,'0000') over(partition by substr(cast(dm_id as string),1,9) order by run_time) as gd_end_cd_nxt,gd_begin_nm,gd_end_nm,run_time,percent,max(run_time) over(partition by substr(cast(dm_id as string),1,9)) as max_time " +
            " from sptcc_dpa.tbl_dim_od_dm) b " +
      "where " +
      "a.od_begin_cd=b.od_begin_cd " +
      "and " +
      "a.od_end_cd=b.od_end_cd"
    )
    spark.stop()
  }
}
```
pyspark实现代码:
```python
from pyspark.sql import HiveContext
from pyspark.sql import SparkSession
from pyspark.context import SparkContext
spark=SparkSession.builder.master('yarn').appName('test').config("spark.hadoop.validateOutputSpecs", "false")\
    .config("spark.sql.parquet.compression.codec","gzip").getOrCreate()
sc = spark.sparkContext
sqlContext = HiveContext(sc)
sqlContext.sql("""insert overwrite table sparktest.tbl_dm_od_cust_mid1 select 
    dm_id     
 ,a.od_begin_cd
    ,a.od_end_cd
    ,a.time_id
    ,line_id
    ,GD_begin_cd
    ,gd_end_cd
    ,lead(gd_end_cd,1,'0000') over(partition by substr(cast(dm_id as string),1,9) order by run_time) as gd_end_cd_nxt
    ,gd_begin_nm
    ,gd_end_nm
    ,a.time_id+run_time/3 as sta_time_id
    ,a.time_id+max_time/3 end_time_id
    ,cast(b.percent*a.cust_rate as float) precent
from
    sptcc_dm.tbl_dm_od_rate a,
    (select
         dm_id
   ,line_id
         ,od_begin_cd
         ,od_end_cd
         ,gd_begin_cd
         ,gd_end_cd
    ,lead(gd_end_cd,1,'0000') over(partition by substr(cast(dm_id as string),1,9) order by run_time) as gd_end_cd_nxt
         ,gd_begin_nm
         ,gd_end_nm
         ,run_time
         ,percent
         ,max(run_time) over(partition by substr(cast(dm_id as string),1,9)) as max_time
    from sptcc_dpa.tbl_dim_od_dm
    ) b
where
    a.od_begin_cd=b.od_begin_cd
    and a.od_end_cd=b.od_end_cd""").show()
```

### 评价

在本次测试中，只是简单的使用了一下Spark中SQL查询API的使用。实际上并没有体现出Spark的优势。个人觉得Spark的优势在于在一系列的业务处理和查询过程中，可以很方便的把子查询的中间结果大批量的缓存在内存中，这样就给传统的SQL查询带来了很多优化的空间。另外，Spark也可以快速的对处理结果进行RDD编程和流式处理，这样Spark平台就可以支撑起绝大多数业务类型了。

## 附录

### 测试组件版本
* hadoop:2.6.0
* spark:2.1.0
* hive:1.1.0
* java:1.8

### 网页资料

1. [Spark-SQL,DataFrames and Datasets 官方文档](https://spark.apache.org/docs/2.1.0/sql-programming-guide.html)
2. [hadoop HDFS常用文件操作命令](https://segmentfault.com/a/1190000002672666)
3. [YARN的内存和CPU配置](http://blog.javachen.com/2015/06/05/yarn-memory-and-cpu-configuration.html)
4. [JavaChen的博客里面的示例很好](http://blog.javachen.com/categories.html#spark)
5. [Spark的一些调优经验](http://blog.csdn.net/sinat_29581293/article/details/62045523)

### 学习中的一些坑

1. 对于有结果的任务，下一次重新运行时要记得清空结果
2. 对于输入输出路径最好参数化，本地测试和集群运行使用不同的参数即可
3. 对于yarn的参数最好不要写死在代码里面，而是在运行的脚本里面附件参数，这样调试更灵活
4. 编译时要注意各种组件的版本号,本地测试时要配置与集群一致的版本号.
5. 像我这样对API不熟悉的初学者，测试时应该尽量用spark-shell调试代码原型，在本地编辑器中调试不是很方便。
6. 要分清Spark中的RDD/DF和DS区别和相互转化。