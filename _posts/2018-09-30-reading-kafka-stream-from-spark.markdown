---
layout: post
title:  "Reading Kafka stream from Spark job"
date:   2018-09-30 12:23:44 +0300
categories: BigData Spark Kafka
---

Prerequisite
======

Suppose we have already setup Spark and Kafka with Zookeeper on our cluster.
We need the Spark streaming Kafka library which was not automatically setup by Ambari.

This is a little bit wired because Kafka and Spark are both present in the default Ambari stack.

You may want to check if this library is already present on your cluster:
```
$ ls -lah /usr/hdp/current/kafka/libs/spark-streaming-kafka
```
If you don't have the file just download it from Hortonworks repositories:
```
wget http://repo.hortonworks.com/content/repositories/releases/org/apache/spark/spark-streaming-kafka-0-8-assembly_2.11/2.3.0.2.6.5.0-292/spark-streaming-kafka-0-8-assembly_2.11-2.3.0.2.6.5.0-292.jar /usr/hdp/2.6.5.0-292/kafka/libs/spark-streaming-kafka-0-8-assembly_2.11-2.3.0.2.6.5.0-292.jar
```

Setup Kafka topic 
===
First you need to determine where is your Zookeeper host and port.

You can do this through the Ambari UI. I am not going into details as it is trivial task. Something like Zookeeper --> Configs --> and you can search for host and port.

The port is usually 2181. In the command bellow the `testtopic` parameter is the name of the topic.
For the other options see the official Kafka [documentation](http://kafka.apache.org/documentation/).
``` bash
/usr/hdp/current/kafka-broker/bin/kafka-topics.sh --create --zookeeper zookeeper.host:2181 --replication-factor 1 --partitions 1 --topic testtopic
```
Check if the topic is successfully created:
``` bash
/usr/hdp/current/kafka-broker/bin/kafka-topics.sh --zookeeper zookeeper.host:2181 –list
```

Start listen on Kafka channel
===


### The script

```python
from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils

if __name__ == "__main__":
    sc = SparkContext(appName="PythonSparkStreamingKafka")
    sc.setLogLevel("WARN")

    ssc = StreamingContext(sc,60)
    kafkaStream = KafkaUtils.createStream(ssc, 'zookeeper.host:2181', 'spark-streaming', {'testtopic':1})
    lines = kafkaStream.map(lambda x: x[1])
    lines.pprint()

    ssc.start()
    ssc.awaitTermination()
```

1. We will create SparkContext
2. I am setting log level for debugging
3. Create the StreamingContext with batchDuration 60 seconds. For more details about batch setup refer to [Spark docs](https://spark.apache.org/docs/latest/streaming-programming-guide.html#setting-the-right-batch-interval)
4. Create the Kafka stream. Here my host and topic parameters are hardcoded but it is better to pass them as an arguments. `spark-streaming` is just a name for the consumer
5. We get the lines passed from Kafka stream
6. Print the lines
7. Start the streaming context
8. Avoid exiting the application until explicit termination occur


Start the job
===
You can put `--master yarn` to use YARN as a executor or user `local[*]` for testing the job on the local machine with
all CPU cores involved. 
```bash
spark-submit --master local[*] --jars /usr/hdp/2.6.5.0-292/kafka/libs/spark-streaming-kafka-0-8-assembly_2.11-2.3.0.2.6.5.0-292.jar  --deploy-mode client ./kafka.py
```

Start Kafka producer and put some example messages
===
You need to find out the host and the port for the Kafka broker. 
Open the Ambari UI and navigate to
Kafka --> Configs --> you will see the fields related to broker host and port there.
The port is usually 6667.
```
/usr/hdp/current/kafka-broker/bin/kafka-console-producer.sh  --broker-list kafka-broker.host:6667 --topic imagetext
```
This will open a console and we can put whatever messages we want.




