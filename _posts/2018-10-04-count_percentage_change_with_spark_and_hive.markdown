---
layout: post
title:  "Bachelor students count percentage change from 2012 by subject. Analytics using Apache Hive and Spark"
date:   2018-10-04 18:23:00 +0300
categories: BigData Spark Hive
---

# Bachelor students count percentage change from 2012 by subject. Analytics using Apache Hive and Spark

Data source [National statistical institute](http://nsi.bg/en) - Bulgaria. Source [file](http://nsi.bg/en/content/4897/students-educational-qualification-degree-and-narrow-field-education) 

This is just an exercise showing how we can use Hive and Spark for creating some interested statistics from randomly picked public data reports.

The goal is to calculate bachelor students rise from 2012 to 2017 grouped by subject of study.

### Prerequisite

It is supposed that we have already setup a cluster with running HDFS, Spak, Yarn and Hive services and configured for interoperability. The easiest way is to use Apache Ambari. We are using Python with `pyspark` 

### Preparing the data

First we need to download the data file from the Bulgarian [National statistical institute](http://nsi.bg/en) and put the file in HDFS. The [file](http://nsi.bg/en/content/4897/students-educational-qualification-degree-and-narrow-field-education) is in XLS and we need to convert it into CSV format. I am not going into details how to do that. It is a matter of copy/pasting the cells that we need in a new file and export the new file into CSV with semicolon field delimiter. So the final file will look like:

|                                                |      |        |       |      |        |       |      |        |       |      |        |       |      |        |       |
| ---------------------------------------------- | ---- | ------ | ----- | ---- | ------ | ----- | ---- | ------ | ----- | ---- | ------ | ----- | ---- | ------ | ----- |
| Teacher training and education science         | 346  | 14 886 | 3 611 | 360  | 14 153 | 4 397 | 375  | 14 144 | 5 342 | 398  | 13 634 | 5 534 | 420  | 13 139 | 5 197 |
| Arts                                           | 53   | 5 857  | 1 049 | 58   | 5 690  | 1 198 | 66   | 5 662  | 1 134 | 56   | 5 572  | 1 114 | 43   | 5 256  | 1 058 |
| Humanities                                     | 63   | 11 378 | 2 294 | 87   | 11 014 | 2 193 | 80   | 10 316 | 2 050 | 88   | 9 594  | 1 804 | 94   | 8 583  | 1 541 |
| Social and behavioural science                 | 430  | 24 255 | 6 637 | 356  | 22 436 | 6 745 | 253  | 21 099 | 7 026 | 84   | 19 995 | 6 575 | 92   | 18 082 | 5 596 |
| Journalism, mass communication and information | -    | 3 193  | 442   | -    | 2 972  | 357   | -    | 2 938  | 319   | -    | 2 826  | 353   | -    | 2 417  | 283   |

Create directory into HDFS to store the file:

```bash
hdfs dfs -mkdir /user/username/students
```



Put the file into HDFS:

```bash
hdfs dfs -put students.csv /user/username/students/students.csv
```



Create Hive table to import the data file.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS students(
    Education STRING, 
	ProfBach1213 STRING,
  	Bach1213 STRING, 
    Master1213 STRING,
  
  	ProfBach1314 STRING,
  	Bach1314 STRING, 
    Master1314 STRING,
  
  	ProfBach1415 STRING,
  	Bach1415 STRING, 
    Master1415 STRING,
  
  	ProfBach1516 STRING,
  	Bach1516 STRING, 
    Master1516 STRING,
  
  	ProfBach1617 STRING,
  	Bach1617 STRING, 
    Master1617 STRING
	)
     ROW FORMAT DELIMITED
     FIELDS TERMINATED BY '\u003B'
     STORED AS TEXTFILE
     LOCATION 'hdfs:///user/admin/students';
```

The `TERMINATED BY` contains some special character which is actually `;` (semicolon). We can't use `;` because it is a special symbol in Hive and the query will fail. 

Now we need a little transformation on the Hive table because the numbers in the source file are containing spaces as a decimal delimiter and we need to put them as integers. I am using only string type in the table definition which for sure is not the best practice but some of the fields are containing only `-` (dash) and I want to avoid making more complex transformations on the source. I will try to use what I have. Also it will be better not to create new table but transform the source file using Spark some other service and then import it into file. This way we will avoid doubling the data but for the experiment this is the fastest way.

The query bellow will create a new table removing whitespaces from the numbers so in future we will be able to represent them as normal integers.

```sql
CREATE TABLE student_stats AS SELECT students.education,         
                translate(trim(profbach1213), ' ', '') as profbach1213,
                translate(trim(bach1213), ' ', '') as bach1213,
                translate(trim(master1213), ' ', '') as master1213,
                translate(trim(profbach1314), ' ', '') as profbach1314,
                translate(trim(bach1314), ' ', '') as bach1314,
                translate(trim(master1314), ' ', '') as master1314,
                translate(trim(profbach1415), ' ', '') as profbach1415,
                translate(trim(bach1415), ' ', '') as bach1415,
                translate(trim(master1415), ' ', '') as master1415,
                translate(trim(profbach1516), ' ', '') as profbach1516,
                translate(trim(bach1516), ' ', '') as bach1516,
                translate(trim(master1516), ' ', '') as master1516,
                translate(trim(profbach1617), ' ', '') as profbach1617,
                translate(trim(bach1617), ' ', '') as bach1617,
                translate(trim(master1617), ' ', '') as master1617 FROM students
```



### Doing the statistic itself

Now the goal is to calculate the percentage change from each student class to another and we will see what are the tendencies.

The script for the spark job looks like:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType


if __name__ == "__main__":

        def calc_change(from_clm, to_clm):
                if from_clm == '-':
                        from_clm = 0
                if to_clm == '-':
                        to_clm = from_clm
                if from_clm == 0 and to_clm == 0:
                        return 0
                if to_clm == 0:
                        to_clm = from_clm
                return 100*((int(to_clm)-int(from_clm))/int(to_clm))

        calc_change_udf = udf(calc_change, StringType())

        sparkSession = (SparkSession
                .builder
                .appName('StudentStatistics')
                .enableHiveSupport()
                .getOrCreate())
        
        df = sparkSession.sql("SELECT education, bach1213, bach1314, bach1415, bach1516, bach1617 FROM student_stats")
        df = df.withColumn('change1213-1314', calc_change_udf(df.bach1213, df.bach1314))
        df = df.withColumn('change1314-1415', calc_change_udf(df.bach1314, df.bach1415))
        df = df.withColumn('change1415-1516', calc_change_udf(df.bach1415, df.bach1516))
        df = df.withColumn('change1516-1617', calc_change_udf(df.bach1516, df.bach1617))
        df = df.withColumn('change1213-1617', calc_change_udf(df.bach1213, df.bach1617))
        df.show()
sparkSession.stop()
```

1. First we define the function that will calculate the percent change 

2. Then we need to use `udf` to convert Spark DataFrame Column to string so will be able to do some simple string comparisons after that 

3. We initialize the Spark session with `enableHiveSupport` to be able to do some query directly through Spark session

4. Execute SQL to select all the records from the table that we prepare before and receive DataFrame object to interact with the data.

5. We add five new columns with data aggregated from the other columns. Each column is the change from one year to another and the last column is total change from the first year to the last.

6. Display the whole DataFrame



The Spark job submitted with a simple command:

```bash
spark-submit --master yarn students.py
```

As a result we should see something like:

| education                                        | bach1213 | bach1314 | bach1415 | bach1516 | bach1617 | change1213-1314     | change1314-1415      | change1415-1516     | change1516-1617     | change1213-1617     |
| ------------------------------------------------ | -------- | -------- | -------- | -------- | -------- | ------------------- | -------------------- | ------------------- | ------------------- | ------------------- |
| Teacher training and education science           | 14886    | 14153    | 14144    | 13634    | 13139    | -5.179113968769872  | -0.06363122171945701 | -3.7406483790523692 | -3.767410000761093  | -13.296293477433593 |
| Arts                                             | 5857     | 5690     | 5662     | 5572     | 5256     | -2.9349736379613356 | -0.49452490286117984 | -1.615218951902369  | -6.012176560121765  | -11.43455098934551  |
| Humanities                                       | 11378    | 11014    | 10316    | 9594     | 8583     | -3.3048846922099147 | -6.766188445133772   | -7.525536793829477  | -11.779098217406501 | -32.56437143190027  |
| Social and behavioural science                   | 24255    | 22436    | 21099    | 19995    | 18082    | -8.107505794259225  | -6.336793212948481   | -5.521380345086271  | -10.579581904656564 | -34.13892268554363  |
| "Journalism, mass communication and information" | 3193     | 2972     | 2938     | 2826     | 2417     | -7.43606998654105   | -1.1572498298162015  | -3.9631988676574665 | -16.921803889118742 | -32.10591642532064  |
| Business and administration                      | 40521    | 39384    | 38691    | 37419    | 34540    | -2.886959171237051  | -1.7911142126075832  | -3.399342579972741  | -8.335263462651998  | -17.316155182397218 |

Also we may insert the results back to another Hive table and use Jupiter or something similar to create some fancy graphics.