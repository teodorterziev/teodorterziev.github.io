---
layout: post
title:  "HDFS NameNode memory calculation"
date:   2018-11-12 12:15:00 +0300
categories: BigData HDFS NameNode
---

One of the important things configuring Hadoop cluster is to estimate how much RAM you need for the NameNode to be able to serve N amount of data.

The NameNode keeps the whole information about the data blocks in memory for fast access. Each block on the DataNodes is represented with its own metadata which consists of block location, permissions, creation time, etc. If the file consists of 1.5 block it is represented by two metadata blocks. Nevertheless the second block is only the half of the first block size its metainformation is still as big as the first one.

The metadata representing one block is around 300B.

If we assume a perfect scenario where all our files are perfectly dividable by the block size configured for the cluster and all of them have the same replication factor the formula to calculate the needed RAM is pretty simple.

If we have 1PB of information with replication factor 3 (which is the default) and block size of 64MB we can calculate the memory that we need like this:

1. First we calculate the number of the blocks that we need to store in the NameNode which is 1PB/(64MB*3(replicas)). The NameNode doesn't store the information about each replica. So we need only the unique file blocks and that's why we multiply each block by the number of replicas. One unique file block is actually consuming block_size * replication_factor in our storage. Or simply said NameNode meta info block size does not depend on replica number but storage capacity does!
2. Then we need just to multiply this blocks by the memory consumed for the metadata.

1PB/(64MB*3) ~= 5208333.33333 blocks to store in memory

5208333.33333 * 300B = 1562500000B ~= 1.5625 GB

So we need at least 1.6 GB memory to be available on our NameNode. 

**Keep in mind that this is a perfect scenario where our files are perfectly dividable by the block size!** 

In real life the situation is not the same but we can still calculate the NameNode memory is we know what is the average file to block size ratio and how much files do we have in the cluster. 

If we have 100 million files with average ratio 1.5 - our files are between 1 and 2 block size. In this case one file is actually represented by one inode object (around 150B`*`) and two block objects (150B*2).

So we need 450B to store one file metainformation or we need:

memory = 100M * 450B = 45GB

Or vice versa with 45GB of memory we can store 100 million files with size/block_size ratio = 1.5



`*` This was pointed out by [this Cloudera article](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/admin_nn_memory_config.html) but in other whitepaper ["HDFS scalability: the limits to growth"](http://c59951.r51.cf2.rackcdn.com/5424-1908-shvachko.pdf) (2010) by Konstantin V. Shvachko from Yahoo! he suggests 200B per metainformation object.

