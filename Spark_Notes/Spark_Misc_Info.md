## Spark Miscellaneous - Info, commands and tips

- Workload profile with [sparkMeasure](Spark_Performace_Tool_sparkMeasure.md)   
```
bin/spark-shell --packages ch.cern.sparkmeasure:spark-measure_2.11:0.11
val stageMetrics = ch.cern.sparkmeasure.StageMetrics(spark) 
stageMetrics.runAndMeasure(spark.sql("select count(*) from range(1000) cross join range(1000)").show)
```
---
- Allocate Spark Session from API
```
// Scala
import org.apache.spark.sql._
val mySparkSession = SparkSession.
    builder().
    appName("my app").
    master("local[*]").   // use master("yarn") for a YARN cluster
    config("spark.driver.memory","2g").  // set all the parameters as needed
    getOrCreate() 

# Python
from pyspark.sql import SparkSession
mySparkSession = SparkSession.builder.appName("my app").master("local[*]").config("spark.driver.memory","2g").getOrCreate()

```
---
- Spark commit and PRs, see what's new
  - Spark commits to master: https://github.com/apache/spark/commits/master
  - Spark PRs: https://spark-prs.appspot.com/
  - Documentation: 
     - https://github.com/apache/spark/tree/master/docs 
     - https://spark.apache.org/docs/latest/
     - SQL grammar https://github.com/apache/spark/blob/master/sql/catalyst/src/main/antlr4/org/apache/spark/sql/catalyst/parser/SqlBase.g4
     - https://docs.databricks.com/index.html 

---
- How to build Spark
  - see also https://spark.apache.org/docs/latest/building-spark.html
```
git clone https://github.com/apache/spark.git
cd spark
# export MAVEN_OPTS="-Xmx2g -XX:ReservedCodeCacheSize=512m"
./dev/make-distribution.sh --name custom-spark --tgz --pip -Phadoop-2.7 -Phive -Pyarn -Pkubernetes

# customize Hadoop version
./dev/make-distribution.sh --name custom-spark --tgz --pip -Phadoop-2.7 -Dhadoop.version=3.1.1 -Pyarn -Pkubernetes

# compile a version with cherry-picked changes
# git checkout branch-2.3
# git cherry-pick xxxx

```

---
- Spark configuration
configuration files are: in SPARK_CONF_DIR (defaults SPARK_HOME/conf)  

```Scala  
// get configured parameters from running Spark Session with  
spark.conf.getAll.foreach(println)  
// get list of driver and executors from Spark Context:  
sc.getExecutorMemoryStatus.foreach(println)
```
 
```
# PySpark
from pyspark.conf import SparkConf
conf = SparkConf()
print(conf.toDebugString())
```

---
- Read and set configuration variables of Hadoop environment from Spark.
  Note this code works with the local JVM, i.e. the driver (will not read/write on executors's JVM)  
```
// Scala:
sc.hadoopConfiguration.get("dfs.blocksize")
sc.hadoopConfiguration.getValByRegex(".").toString.split(", ").sorted.foreach(println)
sc.hadoopConfiguration.setInt("parquet.block.size", 256*1024*1024)
```

```
# PySpark
sc._jsc.hadoopConfiguration().get("dfs.blocksize")
sc.hadoopConfiguration.set(key,value)
```

---
- Read filesystem statistics from all registered filesystem in Hadoop (notably HDFS and local).
  Note this code reports statistics for the local JVM, i.e. the driver (will not read stats from executors)
```
scala> org.apache.hadoop.fs.FileSystem.printStatistics()
  FileSystem org.apache.hadoop.hdfs.DistributedFileSystem: 0 bytes read, 213477639 bytes written, 8 read ops, 0 large read ops, 10 write ops
  FileSystem org.apache.hadoop.fs.RawLocalFileSystem: 213444546 bytes read, 0 bytes written, 0 read ops, 0 large read ops, 0 write ops
```

---
How to use the Spark Scala REPL to access Hadoop API, examples:

```
// get Hadoop filesystem object
val fs = org.apache.hadoop.fs.FileSystem.get(sc.hadoopConfiguration)
// alternative:
val fs = org.apache.hadoop.fs.FileSystem.get(spark.sessionState.newHadoopConf)

//get local filesystem
val fslocal = org.apache.hadoop.fs.FileSystem.getLocal(spark.sessionState.newHadoopConf)

// get file status
fs.getFileStatus(new org.apache.hadoop.fs.Path("<file_path>"))

scala> fs.getFileStatus(new org.apache.hadoop.fs.Path("<file_path>")).toString.split("; ").foreach(println)
FileStatus{path=hdfs://analytix/user/canali/cms-dataset-20/20005/DE909CD0-F878-E211-AB7A-485B398971EA.root
isDirectory=false
length=2158964874
replication=3
blocksize=268435456
modification_time=1542653647906
access_time=1543245001357
owner=canali
group=supergroup
permission=rw-r--r--
isSymlink=false}


fs.getBlockSize(new org.apache.hadoop.fs.Path("<file_path>"))

fs.getLength(new org.apache.hadoop.fs.Path("<file_path>"))

// get block map
scala> fs.getFileBlockLocations(new org.apache.hadoop.fs.Path("<file_path>"), 0L, 2000000000000000L).foreach(println)
0,268435456,host1.cern.ch,host2.cern.ch,host3.cern.ch
268435456,268435456,host4.cern.ch,host5.cern.ch,host6.cern.ch
...
```
---
Example od analysis of Hadoop file data block locations using Spark SQL

```
bin/spark-shell
// get filesystem obejct
val fs = org.apache.hadoop.fs.FileSystem.get(sc.hadoopConfiguration)
// get blocks list (with replicas)
val l1=fs.getFileBlockLocations(new org.apache.hadoop.fs.Path("mydataset-20/20005/myfile1.parquet.snappy"), 0L, 2000000000000000L)
// transform in a Spark Dataframe
l1.flatMap(x => x.getHosts).toList.toDF("hostname").createOrReplaceTempView("filemap")
// query
spark.sql("select hostname, count(*) from filemap group by hostname").show

+-----------------+--------+
|         hostname|count(1)|
+-----------------+--------+
|mynode01.cern.ch|       5|
|mynode12.cern.ch|       4|
|mynode02.cern.ch|       4|
|mynode08.cern.ch|       3|
|mynode06.cern.ch|       6|
|...             |        |
+-----------------+--------+
```
---
- Print properties
```
println(System.getProperties)
System.getProperties.toString.split(',').map(_.trim).foreach(println)

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
- Example command line for spark-shell/pyspark/spark-submit on YARN  
`spark-shell --master yarn --num-executors 5 --executor-cores 4 --executor-memory 7g --driver-memory 7g`

---
- How to turn off dynamic allocation
`--conf spark.dynamicAllocation.enabled=false`

---
- Specify JAVA_HOME to use when running Spark on a YARN cluster   
```
export JAVA_HOME=/usr/lib/jvm/myJAvaHome # this is the JAVA_HOME of the driver
bin/spark-shell --conf spark.yarn.appMasterEnv.JAVA_HOME=/usr/lib/jvm/myJAvaHome --conf spark.executorEnv.JAVA_HOME=/usr/lib/jvm/myJAvaHome
```

---
- Run Pyspark on a jupyter notebook
```
export PYSPARK_DRIVER_PYTHON=jupyter-notebook
# export PYSPARK_DRIVER_PYTHON=jupyter-lab
export PYSPARK_DRIVER_PYTHON_OPTS="--ip=`hostname` --no-browser --port=8888"
pyspark ...<add options here>
```
---
- Change Garbage Collector algorithm
  - For a discussion on tests with different GC algorithms for spark see the post [Tuning Java Garbage Collection for Apache Spark Applications](https://databricks.com/blog/2015/05/28/tuning-java-garbage-collection-for-spark-applications.html)
  - Example of how to use G1 GC: `--conf spark.driver.extraJavaOptions="-XX:+UseG1GC" --conf spark.executor.extraJavaOptions="-XX:+UseG1GC"` 

---
---
- Set log level in spark-shell and PySpark
If you have a SparkContext, use `sc.setLogLevel(newLevel)`

Otherwise edit or create the file log4j.properties in $SPARK_CONF_DIR (default SPARK_HOME/conf)
/bin/vi conf/log4j.properties
  
Example for the logging level of PySpark REPL  
```
log4j.logger.org.apache.spark.api.python.PythonGatewayServer=INFO
#log4j.logger.org.apache.spark.api.python.PythonGatewayServer=DEBUG
```

Example for the logging level of the Scala REPL:  
`log4j.logger.org.apache.spark.repl.Main=INFO`

---
- Caching dataframes using off-heap memory
```
bin/spark-shell --master local[*] --driver-memory 64g --conf spark.memory.offHeap.enabled=true --conf spark.memory.offHeap.size=64g --jars ../spark-measure_2.11-0.11-SNAPSHOT.jar
val df = sql("select * from range(1000) cross join range(10000)")
df.persist(org.apache.spark.storage.StorageLevel.OFF_HEAP)
```
---
- Other options for caching dataframes
```
df.persist(org.apache.spark.storage.StorageLevel.
DISK_ONLY     MEMORY_AND_DISK     MEMORY_AND_DISK_SER     MEMORY_ONLY     MEMORY_ONLY_SER     
DISK_ONLY_2   MEMORY_AND_DISK_2   MEMORY_AND_DISK_SER_2   MEMORY_ONLY_2   MEMORY_ONLY_SER_2   OFF_HEAP)
```

---
- Spark-root, read high energy physics data in ROOT format into Spark dataframes
```
bin/spark-shell --packages org.diana-hep:spark-root_2.11:0.1.16

val df = spark.read.format("org.dianahep.sparkroot").load("<path>/myrootfile.root")
val df = spark.read.format("org.dianahep.sparkroot.experimental").load("<path>/myrootfile.root")
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
- Get username and security details via Apache Hadoop security API
```
scala> org.apache.hadoop.security.UserGroupInformation.getCurrentUser()
res1: org.apache.hadoop.security.UserGroupInformation = luca@MYDOMAIN.COM (auth:KERBEROS)
```
---
- Distribute the Kerberos TGT cache to the executors
```bash
kinit    # get a Kerberos TGT if you don't already have one
klist -l # list details of Kerberos credentials file

spark-shell --master yarn --files <path to kerberos credentials file>#krbcache --conf spark.executorEnv.KRB5CCNAME='FILE:./krbcache'

pyspark --master yarn --files path to kerberos credentials file>#krbcache --conf spark.executorEnv.KRB5CCNAME='FILE:./krbcache'
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
Alternative method to run OS commands on Spark executors in Scala
```
val a = sc.parallelize(1 to sc.defaultParallelism).map(x => org.apache.hadoop.util.Shell.execCommand("uname","-a")).collect()
val a = sc.parallelize(1 to sc.defaultParallelism).map(x => org.apache.hadoop.util.Shell.execCommand("/usr/bin/bash","-c","echo $PWD")).collect()
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
spark.conf.set("spark.sql.parquet.compression.codec","xxx") // xxx= none, gzip, lzo, snappy, {zstd, brotli, lz4} 
df.write
  .partitionBy("colPartition1", "colOptionalSubPart") // partitioning column(s) 
  .bucketBy(numBuckets, "colBucket")   // This feature currently gives error with save, follow SPARK-19256
  .format("parquet")
  .save("filePathandName")             // you can use saveAsTable as an alternative

// relevant parameters:
sc.hadoopConfiguration.setInt("parquet.block.size", .. ) // default to 128 MB parquet block size (size of the column groups)
spark.conf.set("spark.sql.files.maxRecordsPerFile", ...) // defaults to 0, use if you need to limit size of files being written  
```

---
- Repartition / Compact Parquet tables

Parquet table repartition is an operation that you may want to use in the case you ended up with
multiple small files into each partition folder and want to compact them in a smaller number of larger files.
Example:  
```
val df = spark.read.parquet("myPartitionedTableToComapct")
df.repartition('colPartition1,'colOptionalSubPartition)
  .write.partitionBy("colPartition1","colOptionalSubPartition")
  .format("parquet")
  .save("filePathandName")
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
- Enable short-circuit reads for Spark on a Hadoop cluster
  - Spark executors need to have libhadoop.so in the library path
  - Short-circuit is a good feature to enable for Spark running on a Hadoop clusters as it improves performance of I/O
  that is local to the Spark executors.
  - Note: the warning message "WARN shortcircuit.DomainSocketFactory: The short-circuit local reads feature cannot be used because libhadoop cannot be loaded"
  is generated after checking on the driver machine. This can be misleading if the driver is not part of the Hadoop cluster, as what is important is that short-circuit is enabled on the executors!  
  - if the library path of the executors as set up on the system defaults does not yet allow to find libhadoop.so, this can be used:
`--conf spark.executor.extraLibraryPath=/usr/lib/hadoop/lib/native --conf spark.driver.extraLibraryPath=/usr/lib/hadoop/lib/native`

---
- Spark-shell power mode and change config to avoid truncating print for long strings
  - Enter power mode set max print string to 1000:
  - BTW, see more spark shell commands: `:help`

```
spark-shell
scala> :power
Power mode enabled. :phase is at typer.
import scala.tools.nsc._, intp.global._, definitions._
Try :help or completions for vals._ and power._

vals.isettings.maxPrintString=1000
```

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
bin/spark-shell --master yarn --num-executors 80 --driver-memory 32g --executor-memory 90g --driver-cores 4 --executor-cores 3 --jars /home/luca/spark-sql-perf-new/target/scala-2.11/spark-sql-perf_2.11-0.5.0-SNAPSHOT.jar --packages com.typesafe.scala-logging:scala-logging-slf4j_2.10:2.1.2

NOTES:
  - Each executor will spawn dsdgen to create data, using the parameters for size (e.g. 10000) and number of partitions (e.g. 1000)
  - Example: bash -c cd /home/luca/tpcds-kit/tools && ./dsdgen -table catalog_sales -filter Y -scale 10000 -RNGSEED 100 -parallel 1000 -child 107
  - Each "core" in the executor spawns one dsdgen
  - This workloads is memory hungry, to avoid excessive GC activity, allocate abundant executor memory

val tables = new com.databricks.spark.sql.perf.tpcds.TPCDSTables(spark.sqlContext, "/home/luca/tpcds-kit/tools", "10000")
tables.genData("/user/canali/TPCDS/tpcds_10000", "parquet", true, true, false, false)

///// 2. Run Benchmark 
export SPARK_CONF_DIR=/usr/hdp/spark/conf
export HADOOP_CONF_DIR=/etc/hadoop/conf
export LD_LIBRARY_PATH=/usr/hdp/hadoop/lib/native/
cd spark-2.3.0-bin-hadoop2.7

bin/spark-shell --master yarn --num-executors 60 --executor-cores 7 --driver-cores 4 --driver-memory 32g  --executor-memory 100g --jars /home/luca/spark-sql-perf-new/target/scala-2.11/spark-sql-perf_2.11-0.5.0-SNAPSHOT.jar --packages com.typesafe.scala-logging:scala-logging-slf4j_2.10:2.1.2 --conf spark.sql.crossJoin.enabled=true --conf spark.sql.hive.filesourcePartitionFileCacheSize=4000000000 --conf spark.executor.extraLibraryPath=/usr/hdp/hadoop/lib/native --conf spark.sql.shuffle.partitions=800

sql("SET spark.sql.perf.results=/user/luca/TPCDS/perftest_results")
import com.databricks.spark.sql.perf.tpcds.TPCDSTables
val tables = new TPCDSTables(spark.sqlContext, "/home/luca/tpcds-kit/tools",10000)

///// 3. Setup tables and run benchmask

tables.createTemporaryTables("/user/luca/TPCDS/tpcds_10000", "parquet")
val tpcds = new com.databricks.spark.sql.perf.tpcds.TPCDS(spark.sqlContext)

// for spark 2.3, avoid regression at scale on q14a q14b and q72
val benchmarkQueries = for (q <- tpcds.tpcds1_4Queries if !q.name.matches("q14a-v1.4|q14b-v1.4|q72-v1.4")) yield(q)

val experiment = tpcds.runExperiment(benchmarkQueries)

///// 4. Extract results
experiment.currentResults.toDF.createOrReplaceTempView("currentResults")

spark.sql("select name, min(executiontime) as MIN_Exec, max(executiontime) as MAX_Exec, avg(executiontime) as AVG_Exec_Time_ms from currentResults group by name order by name").show(200)
spark.sql("select name, min(executiontime) as MIN_Exec, max(executiontime) as MAX_Exec, avg(executiontime) as AVG_Exec_Time_ms from currentResults group by name order by name").repartition(1).write.csv("TPCDS/test_results_<optionally_add_date_suffix>.csv")

///// Use CBO, modify step 3 as follows

// one-off: setup tables using catalog (do not use temporary tables as in example above
tables.createExternalTables("/user/canali/TPCDS/tpcds_1500", "parquet", "tpcds1500", overwrite = true, discoverPartitions = true)
// compute statistics
tables.analyzeTables("tpcds1500", analyzeColumns = true) 

tables.createExternalTables("/user/canali/TPCDS/tpcds_1500", "parquet", "tpcds10000", overwrite = true, discoverPartitions = true)
tables.analyzeTables("tpcds10000", analyzeColumns = true) 

spark.conf.set("spark.sql.cbo.enabled",true)
// --conf spark.sql.cbo.enabled=true
sql("use tpcds10000")
sql("show tables").show

spark.conf.set("spark.sql.cbo.enabled",true)
// --conf spark.sql.cbo.enabled=true
```

---
- Generate simple benchmark load, CPU-bound with Spark
  - Note: scale up the tests by using larger test tables, that is extending the (xx) value in "range(xx)"
```  
bin/spark-shell --master local[*]

// 1. Test Query 1
spark.time(sql("select count(*) from range(10000) cross join range(1000) cross join range(100)").show)
  
// 2. Test Query 2
// this other example exercices more code path in Spark execution
sql("select id, floor(200*rand()) bucket, floor(1000*rand()) val1, floor(10*rand()) val2 from range(1000000)").cache().createOrReplaceTempView("t1")
sql("select count(*) from t1").show()
 
spark.time(sql("select a.bucket, sum(a.val2) tot from t1 a, t1 b where a.bucket=b.bucket and a.val1+b.val1<1000 group by a.bucket order by a.bucket").show())
```

---
- Monitor Spark workloads with Dropwizard metrics for Spark, Influxdb Grafana   
  - Three main steps: (A) configure [Dropwizard (codahale) Metrics library](https://metrics.dropwizard.io) for Spark
   (B) sink the metrics to influxdb
   (C) Setup Grafana dashboards to read the metrics from influxdb 

  - (A) Configure Dropwizard Metrics as described in the [Spark monitoring guide](https://spark.apache.org/docs/latest/monitoring.html) 
    - Option 1: use con use the metrics.properties
    - Option 2: use Spark configuration parameters of the form `--conf spark.metrics.conf.<property_name>=value`

  - Example of using metrics.proerties file:
  - Note: when using metrics.properties you need to set`--conf spark.metrics.conf`. The file 
  metrics.properties need to be visible to all the executors as well as the driver. Example for yarn:
    - copy the file to the current directory and use the --files option to distribute it to the 
    YARN containers:  
     `--files metrics.properties --conf spark.metrics.conf=metrics.properties`

  ```
  metrics.properties file content for using a graphite sink:
  
  *.sink.graphite.class=org.apache.spark.metrics.sink.GraphiteSink
  *.sink.graphite.host=<graphiteendpoint_influxdb_listening_host>
  *.sink.graphite.port=<listening_port>
  *.sink.graphite.period=10   # Configurable
  *.sink.graphite.unit=seconds
  *.sink.graphite.prefix=luca # Optional value/label
  *.source.jvm.class=org.apache.spark.metrics.source.JvmSource # Optional JVM metrics
  ```

  - Example of using Spark configuration parameters:
  ```
  spark-submit/spark-shell/pyspark:
  --conf "spark.metrics.conf.*.sink.graphite.class"="org.apache.spark.metrics.sink.GraphiteSink" \
  --conf "spark.metrics.conf.*.sink.graphite.host"="graphiteendpoint_influxdb_listening_host>" \
  --conf "spark.metrics.conf.*.sink.graphite.port"=<listening_port> \
  --conf "spark.metrics.conf.*.sink.graphite.period"=10 \
  --conf "spark.metrics.conf.*.sink.graphite.unit"=seconds \
  --conf "spark.metrics.conf.*.sink.graphite.prefix"="luca" \
  --conf "spark.metrics.conf.*.source.jvm.class"="org.apache.spark.metrics.source.JvmSource" \
  ```

  - (B) Influxdb can provide a graphite sink. Configure it in `/etc/influxdb/influxdb.conf`   
  Note in particular the configuration of the template, Example:
  ```
  [[graphite]]
    enabled = true
    bind-address = ":2003"
    database = "graphite"
    ...
    templates = [
    "*.*.jvm.pools.* username.applicationid.process.namespace.namespace.measurement.measurement",
    "username.applicationid.process.namespace.measurement.measurement.measurement",
    ]
    
  ```

  - (C) Setup Grafana dashboards 
    - Create a Grafana data source that connets to influxdb "graphite" database
    - create dashboards for the metrics of interest, including active tasks metrics
    - executor task metrics emit aggregate values, you will need to use derivatives 
    to get values of interest for monitoring, as in:  
   `SELECT non_negative_derivative(last("value"), 1s)  / 1000000000 FROM "cpuTime.count" WHERE ("applicationid" =~ /^$ApplicationId$/) AND $timeFilter GROUP BY time($__interval), "process"`  
---

- How to access AWS s3 Filesystem with Spark  
  -  Deploy the jars for hadoop-aws with the implementation of the s3a filesystem in Hadoop.  
  Note, I have tested this with Spark compiled for Hadoop 3.1.1 ("-Dhadoop.version=3.1.1") it did not work for me yet for Spark on Hadoop 2.7.x
  ```
  bin/spark-shell --packages org.apache.hadoop:hadoop-aws:3.1.1  # customize for the relevant Hadoop version
  ```
  -  Set the Hadoop configuration for s3a in the Hadoop client of the driver
  (note, Spark executors will take care of setting in the executors's JVM Hadoop client,
   see org.apache.spark.deploy.SparkHadoopUtil.scala)
  ```
  export AWS_SECRET_ACCESS_KEY="XXXX"
  export AWS_ACCESS_KEY_ID="YYYY"
  bin/spark-shell \
    --conf spark.hadoop.fs.s3a.endpoint="https://s3.cern.ch" \
    --conf spark.hadoop.fs.s3a.impl="org.apache.hadoop.fs.s3a.S3AFileSystem" \
    --packages org.apache.hadoop:hadoop-aws:3.1.1 # edit Hadoop version
  ```
  - Other options (alternatives to the recipe above): 
    - Set config in driver's Hadoop client
     ```
     sc.hadoopConfiguration.set("fs.s3a.endpoint", "https://s3.cern.ch") 
     sc.hadoopConfiguration.set("fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
     sc.hadoopConfiguration.set("fs.s3a.secret.key", "XXXXXXXX..")
     sc.hadoopConfiguration.set("fs.s3a.access.key", "YYYYYYY...")
     
     // note for Python/PySpark use sc._jsc.hadoopConfiguration().set(...)
     ```
    - Set config in Hadoop client core-site.xml 
     ```
     <property>
       <name>fs.s3a.secret.key</name>
       <value>XXXX</value>
     </property>
   
     <property>
       <name>fs.s3a.access.key</name>
       <value>YYYY</value>
     </property>
   
     <property>
       <name>fs.s3a.endpoint</name>
       <value>https://s3.cern.ch</value>
     </property>
   
     <property>
       <name>fs.s3a.impl</name>
       <value>org.apache.hadoop.fs.s3a.S3AFileSystem</value>
     </property>
    ```
