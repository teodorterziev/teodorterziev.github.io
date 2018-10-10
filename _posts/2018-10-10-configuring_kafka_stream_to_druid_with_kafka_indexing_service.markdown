---
layout: post
title:  "Configuring Kafka stream to Druid with Kafka Indexing Service"
date:   2018-10-10 17:15:00 +0300
categories: BigData Kafka Druid
---

### Prerequisite

Suppose we have already setup Ambari with at least these working services:

HDFS, Zookeeper, Amabri Metrics, Kafka, Druid and HBase

### Setup Kafka

First I needed to change Kafka listener port to `9092` (it was conflicting with some other service using the same port) and I also changed the domain from localhost to the domain of my virtual machine which is actually [Hortonworks sandbox](https://hortonworks.com/products/sandbox/).

Open Ambari Web UI and navigate to Kafka -> Configs -> Advanced. Open `Kafka broker` section and look for `listeners` parameter then change to `PLAINTEXT://sandbox-hdp.hortonworks.com:9092` or replace `sandbox-hdp.hortonworks.com` with your domain if you are not using the sandbox. It should work with the default configuration for `localhost` but I just wanted to make it look like production cluster.

#### Create Kafka topic

Kafka uses Zookeeper so we need to submit the topic creation through Zookeeper host. My Zookeeper is installed on the same machine so I use localhost. Execute the following:

```bash
/usr/hdp/current/kafka-broker/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

The command above will create Kafka topic named `test`.

### Setup Druid

We need to enable Kafka Indexing Service to make Druid be able to read Kafka stream. This extension is enabled by default in Ambari but just make sure that it is enabled on your site. 

Open Amabri Web UI and navigate to Druid -> Configs -> Advanced. Open `Advanced druid-common` and look for `druid.extensions.loadList`. If `druid-kafka-indexing-service` is not in the list of enabled extensions you should add it and restart Druid.

Then we need to prepare the configuration for the Druid Supervisor which in my case looks like this:

```json
{
  "type": "kafka",
  "dataSchema": {
    "dataSource": "test_kafka",
    "parser": {
      "type": "string",
      "parseSpec": {
        "format": "json",
        "timestampSpec": {
          "column": "time",
          "format": "auto"
        },
        "dimensionsSpec": {
          "dimensions": [
            "channel"
          ]
        }
      }
    },
    "metricsSpec" : [],
    "granularitySpec": {
      "type": "uniform",
      "segmentGranularity": "DAY",
      "queryGranularity": "NONE",
      "rollup": false
    }
  },
  "tuningConfig": {
    "type": "kafka",
    "reportParseExceptions": false
  },
  "ioConfig": {
    "topic": "test",
    "replicas": 1,
    "taskDuration": "PT10M",
    "completionTimeout": "PT20M",
    "consumerProperties": {
      "bootstrap.servers": "sandbox-hdp.hortonworks.com:9092"
    }
  }
}
```

I am not going to through all parameters but the important ones are:

* `type` - Should be `kafka`

* `dataSource` - The name of the Druid data source. Just put whatever name you feel appropriate.

* `parser` - This section basically explains what the scheme of our database. Here we have only two fields `time` and `channel`. The `time` column will be used for Druid partitioning.

* `ioConfig` - The important settings in this section are `topic` the name of the topic that we created in Kafka and `bootstrap.servers` which is the Kafka broker listener that we defined in `listeners` parameter in Kafka configuration.

More about [Druid Ingestion Setup parameters](http://druid.io/docs/latest/ingestion/ingestion-spec.html).

More about [Kafka Indexing Service configuration](http://druid.io/docs/latest/development/extensions-core/kafka-ingestion.html).

Now you can save this file as `kafka_ini.json` and submit to the Druid Supervisor with `curl` request like:

```bash
curl -X 'POST' -H 'Content-Type:application/json' -d @kafka_init.json  http://localhost:8090/druid/indexer/v1/supervisor
```

You can monitor what happened in Druid on `http://yourcluster.name:8090/console.html`. This is the Druid coordinator web interface.

### Testing the setup

First we want to start Kafka producer and put some data which will then be transferred to Druid 

```bash
/usr/hdp/current/kafka-broker/bin/kafka-console-producer.sh --broker-list sandbox-hdp.hortonworks.com:9092 --topic test
```

Now put some data directly in the console in JSON format with the fields we defined above:

```json
{"time":"2018-09-14T00:47:19.591Z","channel":"test123"}
```

We need to create simple Druid query to see if the results are already present in Druid. Put the following in `search.json` file:

```json
{
  "queryType": "search",
  "dataSource": "test_kafka",
  "granularity": "day",
  "searchDimensions": [
  ],
  "intervals": [
    "2013-01-01T00:00:00.000/2018-12-12T00:00:00.000"
  ]
}
```

More about creating [Druid Search Queries](http://druid.io/docs/latest/querying/searchquery.html).

Execute the query against Druid:

``` bash
curl -X 'POST' -H 'Content-Type:application/json' -d @search.json http://sandbox-hdp.hortonworks.com:8082/druid/v2?pretty
```

You should see something similar to this:

```json
[ {
  "timestamp" : "2018-09-14T00:00:00.000Z",
  "result" : [ {
    "dimension" : "channel",
    "value" : "test123",
    "count" : 1
  } ]
} ]
```
