---
title: Spark Thrift Server running out of memory
date: 2017-04-18 17:25:01
categories:
- Spark
- SparkSql
- Spark Thrift Server
---

# Introduction
Instead of submitting SQL queries from our BI tool via MapReduce we want to utilize Spark for faster processing.
To provide the possibilities of Spark we need to configure a Spark Thrift Server that can be accessed via ODBC/JDBC from
most BI tools like Tableau. With help of that you can access hive table and run SQL on that.

Unfortunately, the default configuration for retrieving and forwarding a result set to the bi tool is not suited for large (big data) results.
In the case you also hit following error message, read on and consider to apply the provided config.


## Error message
Retrieving a result set via Spark Thrift Server can lead to following error message if driver has not enough memory:

```bash
#
# java.lang.OutOfMemoryError: Java heap space
# -XX:OnOutOfMemoryError="kill -9 %p
kill -9 %p"
#   Executing /bin/sh -c "kill -9 22978
kill -9 22978"...

```


## Cause
 
The reason is, that the driver tries to fetch the complete result set before sending it to the client (BI tool in our case).

How to determine the result fetch behavior? Well, unfortunately at time of writing this is not documented,
but analyzing source code of spark thrift server pointed my to the [implementation](https://github.com/apache/spark/blob/master/sql/hive-thriftserver/src/main/scala/org/apache/spark/sql/hive/thriftserver/SparkExecuteStatementOperation.scala)
of `SparkExecuteStatementOperation`.
 
```
 HiveThriftServer2.listener.onStatementParsed(statementId, result.queryExecution.toString())
      iter = {
        if (sqlContext.getConf(SQLConf.THRIFTSERVER_INCREMENTAL_COLLECT.key).toBoolean) {
          resultList = None
          result.toLocalIterator.asScala
        } else {
          resultList = Some(result.collect())
          resultList.get.iterator
        }
      }
```      

Obviously, there is an option implemented that distinguishes 
between a `collect()` that would cause the driver to fetch the complete result set and an Iterator based approach.
`toLocalIterator()` only consumes as much as memory as the largest partition in the retrieved Dataset.

### enable THRIFTSERVER_INCREMENTAL_COLLECT 

How to enable executors to send their result per partition?

Investigating the [SQLConf](https://github.com/apache/spark/blob/61b5df567eb8ae0df4059cb0e334316fff462de9/sql/catalyst/src/main/scala/org/apache/spark/sql/internal/SQLConf.scala#L406)
shows the name of config parameter that needs to be overwritten:

```
val THRIFTSERVER_INCREMENTAL_COLLECT =
    buildConf("spark.sql.thriftServer.incrementalCollect")
      .internal()
      .doc("When true, enable incremental collection for execution in Thrift Server.")
      .booleanConf
      .createWithDefault(false)
```

With this additional config parameter the spark-thriftserver can be started in incremental-fetch-mode `sudo /usr/lib/spark/sbin/start-thriftserver.sh --conf spark.sql.thriftServer.incrementalCollect=true`


 