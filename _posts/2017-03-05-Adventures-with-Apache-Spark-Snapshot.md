---
layout: post
title:  "Adventures with Apache Spark: Creating a Snapshot Table"
summary: This article discusses how to create a snapshot table for OLAP analysis with Apache Spark.
date: 2017-03-05
categories: Streaming
tags: Spark
published: true
--- 


In this article we will discuss a very specific task: Creating a periodic snapshot table optimised for **OLAP/Cube**-Analysis. It's a cube in the classical business intelligence sense, where an MDX query is fired off and and OLAP engine processes the result set. The problem is specific, because, in the multidemensional world the measure has to exist that you are asking for: Imagine we are counting how many JIRA tickets there are in a specific status (e.g. "Open"). The raw data only has one record for last Friday and the following Monday - a simple count. Using a plain **SQL** it would be fairly easy to figure out how many open JIRA cases there were on Saturday. However, in the cube world, if we crossjoin the time member for Saturday with the count measure, we will get an empty tuple returned since that measure does not exist for this particular day. The same applies for Sunday. The way to solve this dilemma is to create artificial records for Saturday and Sunday, basically copying Fridays record and adjusting the date details respectively. While I am not a big fan of this solution (you basically generate Big Data here), I still haven't found any other suitable alternative for this use case.

Previously in [Big Data Snapshot and Accumulating Snapshot](/data-modeling/2015/07/27/Events-and-Snapshot.html) I discussed how to implement this approach with **Hadoop MapReduce**. I've been quite curious for a while on how to implement the same with **Apache Spark**. This article is a follow-up on [Adventures with Apache Spark: How to clone a record](spark/2017/03/04/Adventures-with-Apache-Spark-How-to-clone-a-record.html), which set out the strategy for cloning records with **Apache Spark**, which is an essential element for creating snapshots.

Let's start our exercise: The input data looks like this:

```
event_date, case_id
"2015-01-02", 13
"2015-01-09", 13
"2015-01-01", 12
"2015-01-03", 12
"2015-01-07", 12
"2015-01-11", 12
"2015-01-03", 15
"2015-01-04", 15
"2015-01-07", 15
```


Start **Spark-Shell** and execute the following to read the source data, sort it and provide SQL access to the data via a temporary table:

```scala
val sourceData = (
  spark
    .read
    .option("header", "true")
    .option("ignoreLeadingWhiteSpace","true")
    .option("ignoreTrailingWhiteSpace","true")
    // not really necessary as default date format any ways
    .option("dateFormat","yyyy-MM-dd HH:mm:ss.S") 
    .option("inferSchema", "true")
    .csv("file:///home/dsteiner/git/code-examples/spark/cloning/source-data.txt")
    //.as[Transaction]
  )

// sort data
val sourceDataSorted = sourceData.sort("case_id","event_date")
// next we want to retrieve the next event date for a given case id
// we will do this in Spark SQL as it is easier
sourceDataSorted.createOrReplaceTempView("events")
```

The next step is to figure out how many days are between one record for a particular `case_id` and the next one. This is very easy to accomplish by using a **SQL window function**:

```scala
scala> spark.sql("SELECT case_id, event_date, LEAD(event_date,1) OVER (PARTITION BY case_id ORDER BY event_date) AS next_event_date FROM events").show
+-------+--------------------+--------------------+
|case_id|          event_date|     next_event_date|
+-------+--------------------+--------------------+
|     12|2015-01-01 00:00:...|2015-01-03 00:00:...|
|     12|2015-01-03 00:00:...|2015-01-07 00:00:...|
|     12|2015-01-07 00:00:...|2015-01-11 00:00:...|
|     12|2015-01-11 00:00:...|                null|
|     13|2015-01-02 00:00:...|2015-01-09 00:00:...|
|     13|2015-01-09 00:00:...|                null|
|     15|2015-01-03 00:00:...|2015-01-04 00:00:...|
|     15|2015-01-04 00:00:...|2015-01-07 00:00:...|
|     15|2015-01-07 00:00:...|                null|
+-------+--------------------+--------------------+
```

Let's image it's `2015-01-13` today and this is all the source data we received. We want to build the snapshot up until yesterday. In case the `next_event_date` is `NULL`, we want to set it to `2015-01-13`:

```scala
scala> spark.sql("SELECT case_id, event_date, COALESCE(LEAD(event_date,1) OVER (PARTITION BY case_id ORDER BY event_date),CAST('2015-01-13 00:00:00.0' AS TIMESTAMP)) AS next_event_date FROM events").show
+-------+--------------------+--------------------+
|case_id|          event_date|     next_event_date|
+-------+--------------------+--------------------+
|     12|2015-01-01 00:00:...|2015-01-03 00:00:...|
|     12|2015-01-03 00:00:...|2015-01-07 00:00:...|
|     12|2015-01-07 00:00:...|2015-01-11 00:00:...|
|     12|2015-01-11 00:00:...|2015-01-13 00:00:...|
|     13|2015-01-02 00:00:...|2015-01-09 00:00:...|
|     13|2015-01-09 00:00:...|2015-01-13 00:00:...|
|     15|2015-01-03 00:00:...|2015-01-04 00:00:...|
|     15|2015-01-04 00:00:...|2015-01-07 00:00:...|
|     15|2015-01-07 00:00:...|2015-01-13 00:00:...|
+-------+--------------------+--------------------+
```

This looks good now. One thing left to do is to **calculate** the **amount of days** between `event_date` and `next_event_date`:

```scala
import org.apache.spark.sql.types._

val sourceDataEnriched = spark.sql("SELECT case_id, event_date, COALESCE(LEAD(event_date,1) OVER (PARTITION BY case_id ORDER BY event_date),CAST('2015-01-13 00:00:00.0' AS TIMESTAMP)) AS next_event_date FROM events").withColumn("no_of_days_gap", ((unix_timestamp(col("next_event_date")) - unix_timestamp(col("event_date")))/60/60/24).cast(IntegerType))

scala> sourceDataEnriched.show(10)
+-------+--------------------+--------------------+--------------+
|case_id|          event_date|     next_event_date|no_of_days_gap|
+-------+--------------------+--------------------+--------------+
|     12|2015-01-01 00:00:...|2015-01-03 00:00:...|             2|
|     12|2015-01-03 00:00:...|2015-01-07 00:00:...|             4|
|     12|2015-01-07 00:00:...|2015-01-11 00:00:...|             4|
|     12|2015-01-11 00:00:...|2015-01-13 00:00:...|             2|
|     13|2015-01-02 00:00:...|2015-01-09 00:00:...|             7|
|     13|2015-01-09 00:00:...|2015-01-13 00:00:...|             4|
|     15|2015-01-03 00:00:...|2015-01-04 00:00:...|             1|
|     15|2015-01-04 00:00:...|2015-01-07 00:00:...|             3|
|     15|2015-01-07 00:00:...|2015-01-13 00:00:...|             6|
+-------+--------------------+--------------------+--------------+
```

Let's create the **clones** now (since we need to have a record for every day for every `case_id`):

```scala
scala> sourceDataEnriched.flatMap(r => Seq.fill(r.no_of_days_gap)(r)).show
<console>:28: error: value no_of_days_gap is not a member of org.apache.spark.sql.Row
       sourceDataEnriched.flatMap(r => Seq.fill(r.no_of_days_gap)(r)).show
```

The **workaround** to this **error** is to use a **case class**:

```scala
case class SourceDataEnriched(
  case_id:Int
  , event_date:java.sql.Timestamp
  , next_event_date:java.sql.Timestamp
  , no_of_days_gap:Int
)

val sourceDataEnriched = spark.sql("SELECT case_id, event_date, COALESCE(LEAD(event_date,1) OVER (PARTITION BY case_id ORDER BY event_date),CAST('2015-01-13 00:00:00.0' AS TIMESTAMP)) AS next_event_date FROM events").withColumn("no_of_days_gap", ((unix_timestamp(col("next_event_date")) - unix_timestamp(col("event_date")))/60/60/24).cast(IntegerType)).as[SourceDataEnriched]
```

We should be able to create the **clones** now:

```scala
scala> val sourceDataExploded = sourceDataEnriched.flatMap(r => Seq.fill(r.no_of_days_gap)(r))
scala> sourceDataExploded.show(10)
+-------+--------------------+--------------------+--------------+
|case_id|          event_date|     next_event_date|no_of_days_gap|
+-------+--------------------+--------------------+--------------+
|     12|2015-01-01 00:00:...|2015-01-03 00:00:...|             2|
|     12|2015-01-01 00:00:...|2015-01-03 00:00:...|             2|
|     12|2015-01-03 00:00:...|2015-01-07 00:00:...|             4|
|     12|2015-01-03 00:00:...|2015-01-07 00:00:...|             4|
|     12|2015-01-03 00:00:...|2015-01-07 00:00:...|             4|
|     12|2015-01-03 00:00:...|2015-01-07 00:00:...|             4|
|     12|2015-01-07 00:00:...|2015-01-11 00:00:...|             4|
|     12|2015-01-07 00:00:...|2015-01-11 00:00:...|             4|
|     12|2015-01-07 00:00:...|2015-01-11 00:00:...|             4|
|     12|2015-01-07 00:00:...|2015-01-11 00:00:...|             4|
+-------+--------------------+--------------------+--------------+
only showing top 10 rows
```

Let's create a **clone number** (`clone_no`) so we can create the correct **snapshot date** later on:

```scala
scala> sourceDataExploded.createOrReplaceTempView("events_ranked")

scala> val eventsWithCloneNo = spark.sql("SELECT case_id, event_date, no_of_days_gap, ROW_NUMBER() OVER (PARTITION BY case_id, event_date ORDER BY event_date) - 1 as clone_no FROM events_ranked")

scala> eventsWithCloneNo.sort("case_id","event_date","clone_no").show(50)
+-------+--------------------+--------------+--------+
|case_id|          event_date|no_of_days_gap|clone_no|
+-------+--------------------+--------------+--------+
|     12|2015-01-01 00:00:...|             2|       0|
|     12|2015-01-01 00:00:...|             2|       1|
|     12|2015-01-03 00:00:...|             4|       0|
|     12|2015-01-03 00:00:...|             4|       1|
|     12|2015-01-03 00:00:...|             4|       2|
|     12|2015-01-03 00:00:...|             4|       3|
|     12|2015-01-07 00:00:...|             4|       0|
|     12|2015-01-07 00:00:...|             4|       1|
|     12|2015-01-07 00:00:...|             4|       2|
|     12|2015-01-07 00:00:...|             4|       3|
|     12|2015-01-11 00:00:...|             2|       0|
|     12|2015-01-11 00:00:...|             2|       1|
|     13|2015-01-02 00:00:...|             7|       0|
|     13|2015-01-02 00:00:...|             7|       1|
|     13|2015-01-02 00:00:...|             7|       2|
|     13|2015-01-02 00:00:...|             7|       3|
|     13|2015-01-02 00:00:...|             7|       4|
|     13|2015-01-02 00:00:...|             7|       5|
|     13|2015-01-02 00:00:...|             7|       6|
|     13|2015-01-09 00:00:...|             4|       0|
|     13|2015-01-09 00:00:...|             4|       1|
|     13|2015-01-09 00:00:...|             4|       2|
|     13|2015-01-09 00:00:...|             4|       3|
|     15|2015-01-03 00:00:...|             1|       0|
|     15|2015-01-04 00:00:...|             3|       0|
|     15|2015-01-04 00:00:...|             3|       1|
|     15|2015-01-04 00:00:...|             3|       2|
|     15|2015-01-07 00:00:...|             6|       0|
|     15|2015-01-07 00:00:...|             6|       1|
|     15|2015-01-07 00:00:...|             6|       2|
|     15|2015-01-07 00:00:...|             6|       3|
|     15|2015-01-07 00:00:...|             6|       4|
|     15|2015-01-07 00:00:...|             6|       5|
+-------+--------------------+--------------+--------+
```

The only task left to do now is to create the correct **snapshot date**:

```
val snapshot = eventsWithCloneNo.selectExpr("date_add(event_date,clone_no) AS snapshot_date","case_id")

scala> snapshot.sort("case_id","snapshot_date").show(50)

+-------------+-------+
|snapshot_date|case_id|
+-------------+-------+
|   2015-01-01|     12|
|   2015-01-02|     12|
|   2015-01-03|     12|
|   2015-01-04|     12|
|   2015-01-05|     12|
|   2015-01-06|     12|
|   2015-01-07|     12|
|   2015-01-08|     12|
|   2015-01-09|     12|
|   2015-01-10|     12|
|   2015-01-11|     12|
|   2015-01-12|     12|
|   2015-01-02|     13|
|   2015-01-03|     13|
|   2015-01-04|     13|
|   2015-01-05|     13|
|   2015-01-06|     13|
|   2015-01-07|     13|
|   2015-01-08|     13|
|   2015-01-09|     13|
|   2015-01-10|     13|
|   2015-01-11|     13|
|   2015-01-12|     13|
|   2015-01-03|     15|
|   2015-01-04|     15|
|   2015-01-05|     15|
|   2015-01-06|     15|
|   2015-01-07|     15|
|   2015-01-08|     15|
|   2015-01-09|     15|
|   2015-01-10|     15|
|   2015-01-11|     15|
|   2015-01-12|     15|
+-------------+-------+
```

Finally let's see how many cases there are by day (to simulate a call from the **OLAP/Cube tool**):

```scala
scala> snapshot.createOrReplaceTempView("snapshot_jira")
scala> spark.sql("SELECT snapshot_date, COUNT(*) AS cnt FROM snapshot_jira GROUP BY 1 ORDER BY 1").show

+-------------+---+
|snapshot_date|cnt|
+-------------+---+
|   2015-01-01|  1|
|   2015-01-02|  2|
|   2015-01-03|  3|
|   2015-01-04|  3|
|   2015-01-05|  3|
|   2015-01-06|  3|
|   2015-01-07|  3|
|   2015-01-08|  3|
|   2015-01-09|  3|
|   2015-01-10|  3|
|   2015-01-11|  3|
|   2015-01-12|  3|
+-------------+---+
```
