## Spark Miscellaneous - Info, commands and tips

- Workload profile with [sparkMeasure](Spark_Performace_Tool_sparkMeasure.md)   
```
bin/spark-shell --packages ch.cern.sparkmeasure:spark-measure_2.11:0.11
val stageMetrics = ch.cern.sparkmeasure.StageMetrics(spark) 
stageMetrics.runAndMeasure(spark.sql("select count(*) from range(1000) cross join range(1000)").show)
```
---
- Spark SQL execution plan and code generation
```
sql("select count(*) from range(10) cross join range(10)").explain(true)
sql("explain select count(*) from range(10) cross join range(10)").collect.foreach(println)

// CBO
sql("explain cost select count(*) from range(10) cross join range(10)").collect.foreach(println)

// Code generation
sql("select count(*) from range(10) cross join range(10)").queryExecution.debug.codegen
sql("explain codegen select count(*) from range(10) cross join range(10)").collect.foreach(println)
```

---
- Example command line for spark-shell/pyspark/spark-submit  
`--master yarn --num-executors 5 --executor-cores 4 --executor-memory 7g --driver-memory 7g`

---
- Allocate Spark Session from API
```
// Scala
import org.apache.spark.sql
val mySparkSession = SparkSession.
    builder().
    appName("my app").
    master("local[*]").   // use master("yarn") for a YARN cluster
    config("spark.driver.memory","2g").  // set all the parameters as needed
    getOrCreate() 

# Python
import pyspark.sql
mySparkSession = SparkSession.builder.appName("my app").master("local[*]").config("spark.driver.memory","2g").getOrCreate()

```

---
- Spark configuration
configuration files are: in SPARK_CONF_DIR (defaults SPARK_HOME/conf)  
get configured parameters from running Spark Session with  
`spark.conf.getAll.foreach(println)`  
get list of driver and executors from Spark Context:  
`sc.getExecutorMemoryStatus.foreach(println)`

---
- Read and set configuration variables from Hadoop  
```
sc.hadoopConfiguration.get("dfs.blocksize")
sc.hadoopConfiguration.getValByRegex(".").toString.split(", ").sorted.foreach(println)
sc.hadoopConfiguration.setInt("parquet.block.size", 256*1024*1024)
```

---
- Print properties
```
println(System.getProperties)
System.getProperties.toString.split(',').map(_.trim).foreach(println)

```

---
- Distribute the Kerberos TGT cache to the executors
```bash
export KRB5CCNAME=/tmp/krb5cc_$UID
kinit 

spark-shell --master yarn --files $KRB5CCNAME#krbcache --conf spark.executorEnv.KRB5CCNAME='FILE:$PWD/krbcache'

pyspark --master yarn --files $KRB5CCNAME#krbcache --conf spark.executorEnv.KRB5CCNAME='FILE:$PWD/krbcache'
```

---
- Run OS commands from Spark
```scala
// Scala, runs locally on the driver
import sys.process._
"uname -a".!  // with one !,  returns exit status
"uname -a".!! // with 2 !, returns output as String
```
- This execute OS commands on Spark executors (relevant for cluster deployments).
It is expected to run on each executor and for each "core"/task allocated.
However, the actual result and order are not guaranteed, a more solid approach is needed
```scala
// Scala, runs on the executors/tasks in a cluster
import sys.process._
sc.parallelize(1 to sc.defaultParallelism).map(_ => "uname -a" !).collect()
sc.parallelize(1 to sc.defaultParallelism).map(_ => "uname -a" !!).collect().foreach(println)
```
```
# Python, run on the executors (see comments in the Scala version) 
# method 1
import os
sc.parallelize(range(0, sc.defaultParallelism)).map(lambda i: os.system("uname -a")).collect()

# method 2
import subprocess
sc.parallelize(range(0, sc.defaultParallelism)).map(lambda i: subprocess.call(["uname", "-a"])).collect()
sc.parallelize(range(0, sc.defaultParallelism)).map(lambda i: subprocess.check_output(["uname", "-a"])).collect()
```

---
- Parquet tables
```
// Read
spark.read.parquet("fileNameAndPath")
// relevant parameters:
spark.conf.set("spark.sql.files.maxPartitionBytes", ..) // default 128MB, small files are grouped into partitions up to this size

// Write
df.write
  .partitionBy("colPartition") // partitioning column if relevant 
  .bucketBy(numBuckets, "colBucket")   // This feature currently gives error, follow SPARK-19256
  .option("compress","snappy") // this is the default, set to none if you don't want compression
  .parquet("fileNameAndPath")

// relevant parameters:
sc.hadoopConfiguration.setInt("parquet.block.size", .. ) // default to 128 MB parquet block size (size of the column groups)
spark.conf.set("spark.sql.files.maxRecordsPerFile", ...) // defaults to 0, use if you need to limit size of files being written  
```

---
- Read from Oracle via JDBC, example from [Spark_Oracle_JDBC_Howto.md](Spark_Oracle_JDBC_Howto.md)
```
val df = spark.read.format("jdbc")
         .option("url", "jdbc:oracle:thin:@dbserver:port/service_name")
         .option("driver", "oracle.jdbc.driver.OracleDriver")
         .option("dbtable", "MYSCHEMA.MYTABLE")
         .option("user", "MYORAUSER")
         .option("password", "XXX")
         .option("fetchsize",10000).load()
         
// test
df.printSchema
df.show(5)
         
// write data as compressed Parquet files  
df.write.parquet("MYHDFS_TARGET_DIR/MYTABLENAME")
```

---
- Spark-root, read high energy physics data in ROOT format into Spark dataframes
```
bin/spark-shell --packages org.diana-hep:spark-root_2.11:0.1.11

val df = spark.read.format("org.dianahep.sparkroot").load("<path>/myrootfile.root")
```

---
- Turn off dynamic allocation for this session
`--conf spark.dynamicAllocation.enabled=false`

---
- Enable short-circuit reads for Spark on a Hadoop cluster
  - Spark executors need to have libhadoop.so in the library path
  - Short-circuit is a good feature to enable for Spark running on a Hadoop clusters as it improves performance of I/O
  that is local to the Spark executors.
  - Note: the warning message "WARN shortcircuit.DomainSocketFactory: The short-circuit local reads feature cannot be used because libhadoop cannot be loaded"
  is generated after checking on the driver machine. This can be misleading if the driver is not part of the Hadoop cluster, as what is important is that short-circuit is enabled on the executors!  
  - if the library path of the executors as set up on the system defaults does not yet allow to find libhadoop.so, this can be used:
`--conf spark.executor.extraLibraryPath=/usr/lib/hadoop/lib/native --conf spark.driver.extraLibraryPath=/usr/lib/hadoop/lib/native`

---
- Specify JAVA_HOME to use when running Spark on a YARN cluster   
`bin/spark-shell --conf spark.yarn.appMasterEnv.JAVA_HOME=/usr/lib/jvm/java-oracle --conf spark.executorEnv.JAVA_HOME=/usr/lib/jvm/java-oracle`

---
- Change Garbage Collector algorithm
  - For a discussion on tests with different GC algorithms for spark see the post [Tuning Java Garbage Collection for Apache Spark Applications](https://databricks.com/blog/2015/05/28/tuning-java-garbage-collection-for-spark-applications.html)
  - Example of how to use G1 GC: `--conf spark.driver.extraJavaOptions="-XX:+UseG1GC" --conf spark.executor.extraJavaOptions="-XX:+UseG1GC"` 


---
- Set logging level  
Edit or create the file log4j.properties in $SPARK_CONF_DIR (default SPARK_HOME/conf)
Example for the logging level of the REPL:
`log4j.logger.org.apache.spark.repl.Main=INFO`

---
- Use off-heap memory for caching DF
```
bin/spark-shell --master local[*] --driver-memory 64g --conf spark.memory.offHeap.enabled=true --conf spark.memory.offHeap.size=64g --jars ../spark-measure_2.11-0.11-SNAPSHOT.jar
val df = sql("select * from range(1000) cross join range(10000)")
df.persist(org.apache.spark.storage.StorageLevel.OFF_HEAP)
```

---
- How to deploy Spark shell or a notebook behind a firewall
  - This is relevant when using spark-shell or pyspark or a Jupyter Notebook, 
  running the Spark driver on a client machine with a local firewall and
  accessing Spark executors remotely on a cluster
  - The driver listens on 2 TCP ports that need to be accessed by the executors on the cluster.
  This is how you can specify the port numbers (35000 and 35001 are picked just as an example):
```
--conf spark.driver.port=35000 
--conf spark.driver.blockManager.port=35001
```
  - You can set up the firewall rule on the driver to to allow connections from cluster node. 
  This is a simplified example of rule when using iptables:
```
-A INPUT -m state --state NEW -m tcp -p tcp -s 10.1.0.0/16 --dport 35000 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp -s 10.1.0.0/16 --dport 35001 -j ACCEPT
```
  - In addition clients may want to access the port for the WebUI (4040 by default)
 
 ---
 - Examples of Dataframe creation for testing
 ```
sql("select * from values (1, 'aa'), (2,'bb'), (3,'cc') as (id,desc)").show
+---+----+
| id|desc|
+---+----+
|  1|  aa|
|  2|  bb|
|  3|  cc|
+---+----+

sql("select * from values (1, 'aa'), (2,'bb'), (3,'cc') as (id,desc)").createOrReplaceTempView("t1")
spark.table("t1").printSchema
root
 |-- id: integer (nullable = false)
 |-- desc: string (nullable = false)

spark.sql("create or replace temporary view outer_v1 as select * from values (1, 'aa'), (2,'bb'), (3,'cc') as (id,desc)")

sql("select id, floor(200*rand()) bucket, floor(1000*rand()) val1, floor(10*rand()) val2 from range(10)").show(3)
+---+------+----+----+
| id|bucket|val1|val2|
+---+------+----+----+
|  0|     1| 223|   5|
|  1|    26| 482|   5|
|  2|    42| 384|   7|
+---+------+----+----+
only showing top 3 rows

scala> val df=Seq((1, "aaa", Map(1->"a") ,Array(1,2,3), Vector(1.1,2.1,3.1)), (2, "bbb", Map(2->"b") ,Array(4,5,6), Vector(4.1,5.1,6.1))).toDF("id","name","map","array","vector")
df: org.apache.spark.sql.DataFrame = [id: int, name: string ... 3 more fields]

df.printSchema
root
 |-- id: integer (nullable = false)
 |-- name: string (nullable = true)
 |-- map: map (nullable = true)
 |    |-- key: integer
 |    |-- value: string (valueContainsNull = true)
 |-- array: array (nullable = true)
 |    |-- element: integer (containsNull = false)
 |-- vector: array (nullable = true)
 |    |-- element: double (containsNull = false)


df.show
+---+----+-----------+---------+---------------+
| id|name|        map|    array|         vector|
+---+----+-----------+---------+---------------+
|  1| aaa|Map(1 -> a)|[1, 2, 3]|[1.1, 2.1, 3.1]|
|  2| bbb|Map(2 -> b)|[4, 5, 6]|[4.1, 5.1, 6.1]|
+---+----+-----------+---------+---------------+

scala> case class myclass(id: Integer, name: String, myArray: Array[Double])
scala> val df=Seq(myclass(1, "aaaa", Array(1.1,2.1,3.1)),myclass(2, "bbbb", Array(4.1,5.1,6.1))).toDF
scala> df..show
+---+----+---------------+
| id|name|        myArray|
+---+----+---------------+
|  1|aaaa|[1.1, 2.1, 3.1]|
|  2|bbbb|[4.1, 5.1, 6.1]|
+---+----+---------------+

// Dataset API
scala> df.as[myclass]
res75: org.apache.spark.sql.Dataset[myclass] = [id: int, name: string ... 1 more field]

scala> df.as[myclass].map(v  => v.id + 1).reduce(_ + _)
res76: Int = 5

// Manipulating rows, columns and arrays

// collect_list agregates columns into rows
sql("select collect_list(col1) from values 1,2,3").show
+------------------+
|collect_list(col1)|
+------------------+
|         [1, 2, 3]|
+------------------+

// explode transforms aggregates into columns
sql("select explode(Array(1,2,3))").show
+---+
|col|
+---+
|  1|
|  2|
|  3|
+---+

sql("select col1, explode(Array(1,2,3)) from values Array(1,2,3)").show()
+---------+---+
|     col1|col|
+---------+---+
|[1, 2, 3]|  1|
|[1, 2, 3]|  2|
|[1, 2, 3]|  3|
+---------+---+

// collect_list and explode combined, return to orginial values 
sql("select collect_list(col1) from values 1,2,3").show
sql("select collect_list(col) from (select explode(Array(1,2,3)))").show
+-----------------+
|collect_list(col)|
+-----------------+
|        [1, 2, 3]|
+-----------------+

// How to push a filter on a nested field in a DataFrame
// The general strategy is to unpack the array, apply a filter then repack
// Note, Higher order functions in Spark SQL and other topics relatedon how to improve this
// are discussed at https://databricks.com/blog/2017/05/24/working-with-nested-data-using-higher-order-functions-in-sql-on-databricks.html
// Example:
sql("select col1, collect_list(col) from (select col1, explode(col1) as col from values Array(1,2,3),Array(4,5,6)) where col%2 = 0 group by col1").show()

+---------+-----------------+
|     col1|collect_list(col)|
+---------+-----------------+
|[1, 2, 3]|              [2]|
|[4, 5, 6]|           [4, 6]|
+---------+-----------------+

// Example of usage of laterral view
sql("select * from values 'a','b' lateral view explode(Array(1,2)) tab1").show()
+----+---+
|col1|col|
+----+---+
|   a|  1|
|   a|  2|
|   b|  1|
|   b|  2|
+----+---+

```

---
 - Additional examples of dealing with nested structures in Spark SQL
```
scala> dsMuons.printSchema
root
 |-- muons: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- reco::Candidate: struct (nullable = true)
 |    |    |-- qx3_: integer (nullable = true)
 |    |    |-- pt_: float (nullable = true)
 |    |    |-- eta_: float (nullable = true)
 |    |    |-- phi_: float (nullable = true)
 |    |    |-- mass_: float (nullable = true)
 |    |    |-- vertex_: struct (nullable = true)
 |    |    |    |-- fCoordinates: struct (nullable = true)
 |    |    |    |    |-- fX: float (nullable = true)
 |    |    |    |    |-- fY: float (nullable = true)
 |    |    |    |    |-- fZ: float (nullable = true)
 |    |    |-- pdgId_: integer (nullable = true)
 |    |    |-- status_: integer (nullable = true)
 |    |    |-- cachePolarFixed_: struct (nullable = true)
 |    |    |-- cacheCartesianFixed_: struct (nullable = true)


// the following 2 are equivalent and transform an array of struct into a table-like  format
// explode can be used to deal withArrays
// to deal with structs use "col.*"

dsMuons.createOrReplaceTempView("t1")
sql("select element.* from (select explode(muons) as element from t1)").show(2)

dsMuons.selectExpr("explode(muons) as element").selectExpr("element.*").show(2)

+---------------+----+---------+----------+----------+----------+--------------------+------+-------+----------------+--------------------+
|reco::Candidate|qx3_|      pt_|      eta_|      phi_|     mass_|             vertex_|pdgId_|status_|cachePolarFixed_|cacheCartesianFixed_|
+---------------+----+---------+----------+----------+----------+--------------------+------+-------+----------------+--------------------+
|             []|  -3|1.7349417|-1.6098186| 0.6262487|0.10565837|[[0.08413784,0.03...|    13|      0|              []|                  []|
|             []|  -3| 5.215807|-1.7931011|0.99229723|0.10565837|[[0.090448655,0.0...|    13|      0|              []|                  []|
+---------------+----+---------+----------+----------+----------+--------------------+------+-------+----------------+--------------------+

```
---
- Spark TPCDS benchmark
  - Download and build the Spark package from [https://github.com/databricks/spark-sql-perf]
  - Download and build tpcds-kit for generating data from [https://github.com/databricks/tpcds-kit]
  - Testing
    1. Generate schema
    2. Run benchmark
    3. Extract results

See instructions on the spark-sql-perf package for more info. Here is an example:
```
///// 1. Generate schema
$ bin/spark-shell --num-executors 200 --driver-memory 32g --executor-memory 16g --driver-cores 8 --executor-cores 1 --jars <path>/spark-sql-perf/spark-sql-perf-scala-2.11.jar --packages com.typesafe.scala-logging:scala-logging-slf4j_2.11:2.1.2
// NOTE: actually only 1 core used per executor, so rather have more executors with 1 core. Each executor will spawn dsgen (100 processes max)

import com.databricks.spark.sql.perf.tpcds.Tables
val tables = new Tables(spark.sqlContext, "<path>/tpcds-kit/tools", 10000)
tables.genData("/user/luca/TPCDS/tpcds_10000", "parquet", true, true, false, true, false)

///// 2. Run Benchmark 
bin/spark-shell --num-executors 80 --executor-cores 6 --driver-memory 8g  --executor-memory 55g --jars <path>/spark-sql-perf/spark-sql-perf-scala-2.11.jar --packages com.typesafe.scala-logging:scala-logging-slf4j_2.10:2.1.2 --conf spark.sql.crossJoin.enabled=true

sql("SET spark.sql.perf.results=/user/canali/TPCDS/perftest_results")
import com.databricks.spark.sql.perf.tpcds.Tables
val tables = new Tables(spark.sqlContext, "<path>/tpcds-kit/tools",10000)
tables.createTemporaryTables("/user/luca/TPCDS/tpcds_10000", "parquet")

import com.databricks.spark.sql.perf.tpcds._
val tpcds = new TPCDS (spark.sqlContext)

val experiment = tpcds.runExperiment(tpcds.tpcds1_4Queries)

///// 3. Extract results
experiment.currentResults.toDF.createOrReplaceTempView("currentResults")

spark.sql("select name, min(executiontime) as MIN_Exec, max(executiontime) as MAX_Exec, avg(executiontime) as AVG_Exec_Time_ms from currentResults group by name order by name").show(200)
spark.sql("select name, min(executiontime) as MIN_Exec, max(executiontime) as MAX_Exec, avg(executiontime) as AVG_Exec_Time_ms from currentResults group by name order by name").repartition(1).write.csv("TPCDS/test_results_<optionally_add_date_suffix>.csv")
```
