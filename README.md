
# DE_project_Kafka_SparkStreaming

## 1.Knowledge required for this project

-   Kafka  
-   Spark DataFrame transformation  
-   Spark Structured Streaming  
-   Impala  
-   Hdfs  
-   Kudu  
-   YARN

## 2.create Kafka topic

(_Create a kafka topic named "rsvp_your_id"with 2 partitions and 2 replications._)

    kafka-topics --create --bootstrap-server ip-172-31-94-165.ec2.internal:9092 --replication-factor 2 --partitions 2 --topic rsvp_ivan

## 3.create Kafka data source

(_Use meetup RSVP event as the Kafkadata source.  Ingest meetup_rsvp event to the Kafka topic, use kafka-console-consumer to verify it._)

-   create producer

        kafka-console-producer --broker-list ip-172-31-94-165.ec2.internal:9092 ip-172-31-91-232.ec2.internal:9092 ip-172-31-89-11.ec2.internal:9092 --topic rsvp_ivan

-   create consumer

        kafka-console-consumer --bootstrap-server ip-172-31-94-165.ec2.internal:9092 --topic rsvp_ivan --group group-ivan

-   load RSVP data    

    -   Method 1  

        curl <https://stream.meetup.com/2/rsvps> | kafka-console-producer --broker-list ip-172-31-94-165.ec2.internal:9092, ip-172-31-91-232.ec2.internal:9092, ip-172-31-89-11.ec2.internal:9092 --topic rsvp_ivan  

    -   Method 2 

        ./myscript.sh | kafka-console-producer --broker-list ip-172-31-94-165.ec2.internal:9092, ip-172-31-91-232.ec2.internal:9092, ip-172-31-89-11.ec2.internal:9092 --topic rsvp_ivan 

myscript.sh [click here](https://github.com/1van0705/Kafka_project/blob/master/myscript.sh)

## 4. Write Five spark streaming job

1.  Save events in HDFS in text(json)format.  Use "kafka"source and "file"sink.  Set outputMode to "append".

```scala
val df = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "ip-172-31-94-165.ec2.internal:9092").option("subscribe","rsvp_ivan").option("startingOffsets","latest").option("failOnDataloss","false").load()

df.isStreaming

import org.apache.spark.sql.streaming.Trigger

val DF = df.selectExpr("CAST(value AS STRING)")
 
val write = DF.writeStream.trigger(Trigger.ProcessingTime("60 seconds")).format("text").option("path","/user/ivan0705/kafka_project/Q41").option("checkpointLocation", "/user/ivan0705/spark_streaming/checkpoint_0").outputMode("append").start.awaitTermination
```

2.  Save events in HDFS in parquet format with schema.  Use "kafka"source and "file"sink.  Set outputMode to "append".

```scala

val static = spark.read.json("/user/ivan0705/events")
static.printSchema


val df2 = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "ip-172-31-94-165.ec2.internal:9092").option("subscribe","rsvp_ivan").option("startingOffsets","latest").option("failOnDataloss","false").load()

val DF2 = df2.selectExpr("CAST(value AS STRING)")
Or
val DF2 = df2.select(("value").cast("string"))

val eventDF = DF2.select(from_json(col("value"), static.schema).as("data"))
.select("data.*")

import org.apache.spark.sql.streaming.Trigger

val write_parquet = eventDF.writeStream.trigger(Trigger.ProcessingTime("60 seconds")).format("parquet").option("path", "/user/ivan0705/kafka_project/Q42").option("checkpointLocation", "/user/ivan0705/spark_streaming/checkpoint_2").outputMode("append").start.awaitTermination
```

3.  Show how many events are received, display in a 2-minute tumbling window.  Show result at 1-minute interval.  Use "kafka"source and "console"sink.Set outputMode to "complete". Here is a sampleoutput.

```scala

import org.apache.spark.sql.functions.{window, col}

import org.apache.spark.sql.streaming.Trigger

val static = spark.read.json("/user/ivan0705/events")

val df3 = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "ip-172-31-94-165.ec2.internal:9092").option("subscribe","rsvp_ivan").option("startingOffsets","latest").option("failOnDataloss","false").load()

val DF3 = df3.selectExpr("CAST(value AS STRING)", "timestamp")


val eventDF = DF3.select(from_json(col("value"), static.schema).as("data"), col("timestamp")).select("data.*","timestamp").where("_corrupt_record is null").drop("_corrupt_record").withColumnRenamed("timestamp","event_time")

 eventDF.groupBy(window(col("event_time"), "2 minutes")).count.orderBy("window").writeStream.trigger(Trigger.ProcessingTime("60 seconds")).format("console").outputMode("complete").option("truncate","false").start()

```

![](screen-shot-2020-11-04-at-4-50-46-pm-kh3xr5bo.png)

4.  Show how many events are receivedfor each country, display it in a slidingwindow(set windowDuration to 3 minutes and slideDuration to 1 minutes).  Show result at 1-minute interval.  Use "kafka" source and "console" sink.Set outputMode to "complete". Here is a sampleoutput.

```scala

import org.apache.spark.sql.functions.{window, col}

import org.apache.spark.sql.streaming.Trigger

val static = spark.read.json("/user/ivan0705/events")


val df4 = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "ip-172-31-94-165.ec2.internal:9092").option("subscribe","rsvp_ivan").option("startingOffsets","latest").option("failOnDataloss","false").load()
 

 val DF4 = df4.selectExpr("CAST(value AS STRING)","timestamp")


 val eventDF = DF4.select(from_json(col("value"), static.schema).as("data"), col("timestamp")).select("data.*","timestamp").where("_corrupt_record is null").drop("_corrupt_record").withColumnRenamed("timestamp","event_time")

eventDF.groupBy(col("group.group_country"), window(col("event_time"), "3 minutes","60 seconds")).count.orderBy("window").writeStream.trigger(Trigger.ProcessingTime("60 seconds")).format("console").outputMode("complete").option("truncate","false").option("numRows",100).start

```

![](screen-shot-2020-11-04-at-5-09-19-pm-kh3yesyp.png)

5.  Use impala to create a KUDU table. Do dataframe transformation to extract information and write to the KUDU table. Use "kafka"source and "kudu"sink.    

```scala

create table if not exists rsvp_db.rsvp_kudu_ivan(
	rsvp_id bigint primary key,
	member_id bigint,
	member_name string,
	group_id bigint,
	group_name string,
	group_city string,
	group_country string,
	event_name string,
	event_time bigint)
partition BY HASH (rsvp_id) PARTITIONS 2
STORED AS KUDU;

./spark-shell --packages "org.apache.kudu:kudu-spark2_2.11:1.10.0-cdh6.3.2" --repositories "https://repository.cloudera.com/artifactory/cloudera-repos"

import org.apache.spark.sql.functions.{window, col}

import org.apache.spark.sql.streaming.Trigger

val static = spark.read.json("/user/ivan0705/events")

val df5 = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "ip-172-31-94-165.ec2.internal:9092").option("subscribe","rsvp_ivan").option("startingOffsets","latest").option("failOnDataloss","false").load()

val DF5 = df5.selectExpr("CAST(value AS STRING)","timestamp")

 val eventDF = DF5.select(from_json(col("value"), static.schema).as("data"), col("timestamp")).select("data.*","timestamp").where("_corrupt_record is null").drop("_corrupt_record")


val kudu = eventDF.select("rsvp_id","member.member_id","member.member_name","group.group_id","group.group_name","group.group_city","group.group_country","event.event_name","timestamp").select(expr("*"),unix_timestamp(col("timestamp")).as("event_time")).drop("timestamp")

kudu.writeStream.trigger(Trigger.ProcessingTime("60 seconds")).format("Kudu").option("kudu.master","ip-172-31-89-172.ec2.internal,	ip-172-31-86-198.ec2.internal,	ip-172-31-93-228.ec2.internal").option("kudu.table","impala::rsvp_db.rsvp_kudu_ivan").option("kudu.operation","upsert").option("checkpointLocation", "/user/ivan0705/spark_streaming/checkpoint_q4_5").outputMode("append").start

```
