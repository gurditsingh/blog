---
title: "Episode-1 'Overview' : Spark Performance Tuning"
layout: post
excerpt: "A Spark job can be optimized by many techniques so let’s dig deeper into those techniques. Apache Spark optimization helps with in-memory data computations. The bottleneck for these spark optimization computations can be CPU, memory or any resource in the cluster."
last_modified_at: 2021-04-06T09:10:02-05:00
tags:
  - Spark
  - Performance Tuning
  - Bigdata
  - Memory
  - Spark Configurations
  - Cluster Parameters
---

## What is Spark Performance Tuning?

Spark Performance Tuning refers to the process of adjusting settings to record for memory, cores, and instances used by the system. This process guarantees that the Spark has a flawless performance and also prevents bottlenecking of resources in Spark.
![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/spark-tuning.jpg?raw=true)

 - **Data serialization** also determines a good network performance. You will be able to obtain good results in Spark performance by serialization. Spark supports two serialization libraries Java Serialization, Kryo Serialization.
 - **Memory Tuning** is necessary or important step in tuning. we need to give as much memory so that the entire dataset has to fit in memory.
 -  **Data Structure Tuning** One option to reduce memory consumption is by staying away from java features. Avoid the nested structure with lots of small objects and pointers. Instead of using strings for keys use numeric.
 -  **Garbage Collection Tuning** In order to avoid the large “churn” related to the RDDs that have been previously stored by the program, java will dismiss old objects in order to create space for new ones. However, by using data structures that feature fewer objects the cost is greatly reduced. One such example would be the employment an array of Ints instead of a linked list.
 -  **Memory Management** An efficient memory use is essential to good performance. Spark uses memory mainly for storage and execution. Storage memory is used to cache data that will be reused later. On the other hand, execution memory is used for computation in shuffles, sorts, joins, and aggregations.

## What We can Tune ?

![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/spark-tuning2.png?raw=true)

### Cluster Parameters

 - To apply any parameters first we need the information regarding the Cluster Size accordingly we can supply the parameters.
 - We need the information regarding the Instance type used in the Cluster like General Purpose, Compute-optimized, Memory-optimized, Storage-optimized and etc.
 - We can set the number of processors/cores so that we can choose the number of partitions/task of spark job.
 - We can set the memory size accordingly so that we can run the job in-memory and cache the data whenever required.
 - We need the information regarding the type of disk like SSD, RAM_DISK, HDD and etc. which can reduce the I/O operation and we can decide cache property like memory_only, memory_and_disk and etc.  

### Spark Configurations
Because of the in-memory nature of most Spark computations, Spark programs can be bottlenecked by any resource in the cluster: CPU, network bandwidth, or memory. 

Spark configurations like parallelism, shuffle, storage, JVM tuning flags, feature flags and you know there are probably hundreds of configs you don't need to know or tune all of them but they exist they're hard-coded and it's up to you to configure them by performance tuning.

## Life Cycle of Parameters tuning

First thing is How do we start with parameter tuning or job optimization. Basically parameter tuning is a manually and iterative process.

![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/spark-tuning-lifecycle.jpg?raw=true)

 - **Run the Job :** We can start the job with the default parameters and job can take hours to complete. Basically we can start from any point either if we have any knowledge than we can setup some of the tuning parameters or we can start with default.
 - **Analyze Logs :** Once the job completes and if it take longer time then we can further tune the job. To tune the job we need to analyze the job logs. you can find the job logs on spark UI, yarn logs, job history or per node matrix.
 - **Pick new Params :** After analysing the job logs we can then use some expertise or some intuition to figure out is this application running smoothly or we rune  fast enough and then maybe pick better parameters to improve the stability or improve the performance.

**Important :**  It is not you get it right on the first shot it's an iterative process it's not an exact science it's more of a trial and error loop. while this works really well when you have a handful of pipelines/jobs this manual process is hard to scale to thousands hundreds or thousands of pipelines/jobs.


## Performance Tuning is a iterative process

### Step 1: Run with Rules of thumb parameters
First start the job with optimal tuning parameters. In spark we have some Rules of thumb for parameters.

 - Number of partitions are 3x the number of cores in the cluster
 - Number of cores per executor are 4-8. best setting for cores is 5 per executor.
 - Memory per executor calculated by total node memory divide by number of executor and then 85% of the memory we can use for memory per executor.
 - Set the shuffle partition " spark.sql.shuffle.partitions" property because Shuffle is an expensive operation as it involves moving data across the nodes in your cluster.
 - Use caching when the same operation is computed multiple times in the pipeline flow.
 - Broadcast  HashJoin  is most performant, but may not be applicable if both relations in join are large.
 - Use Kryo Serialization because kryo give you better performance over java serialization.
 
 For the other parameters use some expertise or some intuition to figure out suitable parameters for your pipeline/job.

### Step 2: Ensure stability of the job
After running the job either the job can crash/fail or it doesn't meet the SLA. There are several reasons for crash/fail. The pipeline/job can fail with out of memory exception. Pipeline/Job doesn't have sufficient memory to store the data in-memory. Failure reasons can be any so the first thing is to make the job stable and make it in running state.

### Step 3: Solve Performance Issues 
After make the job stable then we can improve the job performance. if the job is running very slow then we can tune the job to complete within the time. solving performance issue is a critical aspect of the Tuning because pipeline/job can be slow due to spark operators, cluster resources, bad code or data quality. To fix the performance issue needs an overall picture of the pipeline/job which includes everything. 

### Step 4: Speed and Cost trade-off
Once we solved all the inefficiencies. Then we can think of speed and cost perspective. Do we need more speed to complete the job in lesser time. Do we need to add more machine for better performance. basically it's a trade-off between speed and cost means if you try of make the job much faster at the expense of spending more cost on getting more machines.


## Next ?

Planning to create multiple blogs episodes on Spark Performance Tuning. Understand and covering the various areas of spark where we can improve the pipeline/job.
