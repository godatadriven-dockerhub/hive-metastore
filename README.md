# hive-metastore

Simple Hive metastore to run some CI tests.

## Run it local using Beeline

First start the Hive metastore, and the Postgres database using Docker compose: `docker-compose up`:

After that you can connect to the metastore. On OSX you can install the Beeline client using `brew install hive`.

```
$ beeline
Beeline version 3.1.2 by Apache Hive
beeline> !connect jdbc:hive2://localhost:10000/default
Connecting to jdbc:hive2://localhost:10000/default
Enter username for jdbc:hive2://localhost:10000/default:
Enter password for jdbc:hive2://localhost:10000/default:
Connected to: Apache Hive (version 3.1.2)
Driver: Hive JDBC (version 3.1.2)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://localhost:10000/default> show tables;
INFO  : Compiling command(queryId=root_20191103211512_d28f08e5-e3a5-4903-9188-8f6efb00ba87): show tables
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:tab_name, type:string, comment:from deserializer)], properties:null)
INFO  : Completed compiling command(queryId=root_20191103211512_d28f08e5-e3a5-4903-9188-8f6efb00ba87); Time taken: 1.783 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=root_20191103211512_d28f08e5-e3a5-4903-9188-8f6efb00ba87): show tables
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=root_20191103211512_d28f08e5-e3a5-4903-9188-8f6efb00ba87); Time taken: 0.099 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+-----------+
| tab_name  |
+-----------+
+-----------+
No rows selected (2,499 seconds)
0: jdbc:hive2://localhost:10000/default>
```

## Run it local using Iceberg (Incubating)

One of the main reasons of building this Docker container was to play around with Iceberg, and it works really well:

```
$ spark-shell --jars runtime/build/libs/iceberg-spark-runtime-5fbe134.dirty.jar
Spark context available as 'sc' (master = local[*], app id = local-1572815898320).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.4
      /_/

Using Scala version 2.11.12 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_172)
Type in expressions to have them evaluated.
Type :help for more information.

scala> import org.apache.iceberg.spark.SparkSchemaUtil
import org.apache.iceberg.spark.SparkSchemaUtil

scala> import org.apache.iceberg.catalog.TableIdentifier
import org.apache.iceberg.catalog.TableIdentifier

scala> val df = spark.read.json("/Users/fokkodriesprong/Desktop/people.json")
df: org.apache.spark.sql.DataFrame = [age: bigint, name: string]

scala> df.show()
+----+-------+
| age|   name|
+----+-------+
|null|Michael|
|  30|   Andy|
|  19| Justin|
+----+-------+

scala> val schema = SparkSchemaUtil.convert(df.schema)
schema: org.apache.iceberg.Schema =
table {
  0: age: optional long
  1: name: optional string
}

scala> import org.apache.iceberg.hive.HiveCatalog
import org.apache.iceberg.hive.HiveCatalog

scala> val conf = spark.sparkContext.hadoopConfiguration
conf: org.apache.hadoop.conf.Configuration = Configuration: core-default.xml, core-site.xml, mapred-default.xml, mapred-site.xml, yarn-default.xml, yarn-site.xml, hdfs-default.xml, hdfs-site.xml, __spark_hadoop_conf__.xml

scala> conf.set("hive.metastore.uris", "thrift://localhost:9083");

scala> val catalog = new HiveCatalog(conf)
catalog: org.apache.iceberg.hive.HiveCatalog = org.apache.iceberg.hive.HiveCatalog@de4bee9

scala> val name = TableIdentifier.of("people", "names")
name: org.apache.iceberg.catalog.TableIdentifier = people.names

scala> val name = TableIdentifier.of("default", "people")
name: org.apache.iceberg.catalog.TableIdentifier = default.people

scala> val table = catalog.createTable(name, schema)
table: org.apache.iceberg.Table = default.people

scala> df.write.format("iceberg").mode("overwrite").save("default.people")

scala> spark.read.format("iceberg").load("default.people.history").show(truncate = false)
+----------------------+-------------------+---------+-------------------+
|made_current_at       |snapshot_id        |parent_id|is_current_ancestor|
+----------------------+-------------------+---------+-------------------+
|2019-11-03 22:25:12.74|2656556620040667351|null     |true               |
+----------------------+-------------------+---------+-------------------+


scala> spark.read.format("iceberg").load("default.people.snapshots").show(truncate = false)
+----------------------+-------------------+---------+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
|committed_at          |snapshot_id        |parent_id|operation|manifest_list                                                                                                                                                        |summary                                                                                                                                                  |
+----------------------+-------------------+---------+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
|2019-11-03 22:25:12.74|2656556620040667351|null     |overwrite|file:/Users/fokkodriesprong/Desktop/incubator-iceberg/spark-warehouse/default.db/people/metadata/snap-2656556620040667351-1-35a66bbe-7823-473c-8252-79480084c37d.avro|[spark.app.id -> local-1572815898320, added-data-files -> 1, added-records -> 3, changed-partition-count -> 1, total-records -> 3, total-data-files -> 1]|
+----------------------+-------------------+---------+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------+
```
