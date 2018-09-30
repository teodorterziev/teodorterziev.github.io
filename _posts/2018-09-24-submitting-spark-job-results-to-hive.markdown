---
layout: post
title:  "Submitting Apache Spark job results to Hive SQL"
date:   2018-09-01 17:23:44 +0300
categories: BigData
---

Prerequisite
======

We presume that we have already setup a cluster with HDFS, Spark2 and Hive running.
The easiest way to do this is through [Apache Ambari](https://docs.hortonworks.com/HDPDocuments/Ambari-2.6.2.2/bk_ambari-installation/content/install-ambari-server.html). 

If this is a raw HDFS installation (no previous setup), we need to create user home directory for the user that we are working from. 

For example if we are using hdfs user to submit Spark jobs we need to create the corresponding home directory on HDFS to be used with that user:
```
hdfs dfs -mkdir /user/hdfs
```
Then we need to find example file with large text. I am using [this file](https://norvig.com/big.txt) which I found in internet. It is only 6.2MB but is big enough for testing and it is still easy to copy it over the network.

```
wget https://norvig.com/big.txt
```
Now we need to put this file in HDFS:

hdfs dfs -put big.txt /user/hdfs/big.txt

The script itself
===

```python
# Some python imports
# Just for using better print functionality
from __future__ import print_function
# Used to count to arguments given to the script
import sys
# Used in reduce phase just to be able to pass + operator as a reference
from operator import add
# Import Spark interface and Spark Hive integration
from pyspark.sql import SparkSession, HiveContext


if __name__ == "__main__":
   # Make sure we have enough arguments to run the job 
   if len(sys.argv) != 2:
        print("Usage: wordcount <file>", file=sys.stderr)
        sys.exit(-1)

    # Create spark session with name PythonWordCount
    spark = SparkSession\
        .builder\
        .appName("PythonWordCount")\
        .getOrCreate()

    # Convert file to RDD
    rdd = spark.sparkContext.textFile(sys.argv[1])
    # Split text on space
    counts = rdd.flatMap(lambda x: x.split(' ')) \
                  # Count every item
                  .map(lambda x: (x, 1)) \
	    # Sum the counts reduced by the key (the word itself)
                  .reduceByKey(add) \
                   # Sort by the second item (count) descending. False as a second argument means descending
                   .sortBy(lambda x: x[1], False)
    # Get the results
    output = counts.collect()
    # Create Data Frame to be used for storing the results
    df = spark.createDataFrame(output)
    # We can also write the results back to HDFS in csv format
    #df.write.csv("hdfs:///user/hdfs/results.csv")
    # Create Hive context to interact with Hive
    hive_ctx = HiveContext(spark)
   # Create the table wordcount if it is not already present
    hive_ctx.sql("CREATE TABLE IF NOT EXISTS wordcount (value STRING, count INT)")
    # Write the results directly to Hive
    df.write.insertInto("wordcount")

spark.stop()
```

Explanation
===

What the script does is basically:
1.	Create Spark session

2.	Convert the text file to resilient distributed dataset (RDD) so after that we are able to do map/reduce jobs in parallel on the cluster

3.	Execute flatMap which will “flatten“ the results: [[word => word], [word => word]] becomes [word, word]

4.	Then map each word with number 1

5.	Reduce all the words by combining them by key and sum the number associated with each occurrence (number 1 in our case see point 4) so the structure now becomes [[word => count], [word => count]] 

6.	We need the words sorted by occurrences and we perform sortBy operation with two arguments. The first one is the function to use for sorting and we return the value of the count. The second argument is the sorting order – False means descending 

7.	Then we need to create Data Frame to prepare the data to be stored in Hive

8.	In the last step we just write the Data Frame directly to Hive

Submitting the job
===
Usually spark-submit command is automatically set in the user $PATH but if not just `cd` to the script 
```
$ cd /usr/hdp/current/spark-client
```
### Through YARN
We can use YARN for resource manager which is the recommended (production) method for submitting the job
```
$ spark-submit --master yarn --deploy-mode client ./wordcount.py
```

### Local submit
Usually suitable for local testing. Will execute the job faster but won't get any benefits for multi node environment.

The `local[*]` setting will tell spark to use all available CPU cores. 
```
$ spark-submit --master local[*] --deploy-mode client ./wordcount.py
```

Possible issues
====
```
Multiple versions of Spark are installed but SPARK_MAJOR_VERSION is not set
Spark1 will be picked by default
Traceback (most recent call last):
  File "/home/hdfs/spark_test/wordcount.py", line 23, in <module>
    from pyspark.sql import SparkSession, HiveContext
ImportError: cannot import name SparkSession
```
If you see this error just set SPARK_MAJOR_VERSION to 2 like this:
export SPARK_MAJOR_VERSION=2


