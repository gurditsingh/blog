---
title: "Episode-5 'Spark-submit vs Apache Livy' : Spark Performance Tuning"
layout: post
excerpt: "Once a user application is bundled, it can be launched using the spark-submit script or via REST API apache Livy."
last_modified_at: 2021-05-22T09:10:02-05:00
tags:
  - Spark
  - Performance Tuning
  - Bigdata
  - spark-submit
  - REST API
  - Apache Livy
  - Spark context
  - Spark session
  - ETL
---

Spark framework provides spark-submit command to submit Spark batch jobs.  Apache Livy Server provides the similar functionality via REST API call to run the Spark jobs.

In this blog we will examine how spark-submit works and how Apache Livy as REST API works to run the jobs, monitor the progress of the job and kill the job. First lets look the terms(functionalities) around the spark-submit and Livy.

## What is Spark Context ?
SparkContext is the entry point of Apache Spark functionality. The most important phase of any Spark driver application is to generate SparkContext. It allows your Spark Application to access the Spark Cluster with the help of Resource Managers such as (YARN/Mesos).

SparkContext provides various functions in Spark such as get the current status of Spark Application, set the configuration, Cancel a stage, cancel a job, and much more. Thus, it acts as a backbone.

### Functions of SparkContext in Spark

![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/sep5_sparkcontext.jpg?raw=true)

 - We can access a cluster by using this SparkContext. The SparkContext will be used to run anything on Executors.
 - The SparkContext use to set configuration.
 - The SparkContext can be used to create RDDs, accumulators and broadcast variables.
 - The SparkContext is used to create the tables and run sql on top of that.
 - The SparkContext is used to register the Spark-Listeners.

## What is Spark Session?
In spark 2.0, we have a new entry point build for DataSet and DataFrame API’s called as Spark-Session which will provides a single point of entry to interact with underlying Spark functionality.

Its a combination of SQLContext, HiveContext and StreamingContext. All the API’s available on those contexts are available on SparkSession also SparkSession has a SparkContext for actual computation and also all the above-mentioned contexts can be accessed using the SparkSession object.

![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/sep5_spark_session.jpg?raw=true)

### Need of Spark session when Spark already have Spark context?

 - Spark session unifies all the different contexts in spark and avoids the developer to worry about creating difference contexts.
 - Spark session helps when we have multiple users and they want to share the same spark context.

**lets understand by code:**
Suppose we have multiple users and they want to share the Notebook

 - All have there own set of Configurations.
 - All have there own set of Tables.

```scala
val oldSession=spark
oldSession: org.apache.spark.sql.SparkSession = org.apache.spark.sql.SparkSession@3b0994ad

val newSession=spark.newSession
newSession: org.apache.spark.sql.SparkSession = org.apache.spark.sql.SparkSession@46d15164
```

Spark gives a straight forward API to create a new session which shares the same spark context.`spark.newSession()` creates a new spark session object. if you check oldSession and newSession they have different hashcodes.

```scala
oldSession.sparkContext
res2: org.apache.spark.SparkContext = org.apache.spark.SparkContext@e073b56

newSession.sparkContext
res3: org.apache.spark.SparkContext = org.apache.spark.SparkContext@e073b56
```
But if we check sparkContext under both the sessions(oldSession, newSession) have the same hashcode. so if we create temp table under the oldSessionand newSession they will be not shared with each other. Check out the complete [Source code](https://github.com/gurditsingh/blog/blob/gh-pages/files/TestSparkSession.html "Source code").

# Ways to run the job on cluster

## 1. Spark Submit
The **spark-submit** script in Spark’s bin directory is used to launch applications on a cluster. It can use all of Spark’s supported cluster managers through a uniform interface to configure and run the applications on cluster.

The **spark-submit** command is a utility to run or submit a Spark application program (or job) to the cluster by specifying options and configurations. The application you are submitting can be written in Scala, Java, or Python (PySpark).

**where is spark-submit :**
Spark binary comes with **spark-submit.sh** script file for Linux, Mac, and `spark-submit.cmd` command file for windows, these scripts are available at **$SPARK_HOME/bin** directory.

Below is a spark-submit command with the commonly used options.
```bash
./bin/spark-submit \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key<=<value> \
  --driver-memory <value>g \
  --executor-memory <value>g \
  --executor-cores <number of cores>  \
  --jars  <comma separated dependencies>
  --class <main-class> \
  <application-jar> \
  [application-arguments]
```

### Basic terminology of cluster
In the below diagram we have edge node and cluster. The Edge node is used by the user (which run the application/job) to run the application on the cluster.

![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/spt_ep4_sparksubmit.jpg?raw=true)

**Edge Node :**

 - The Edge node can be the system used by the user and which is outside of the cluster but have connectivity to the cluster.
 - The Edge node can be system used by the user which is one of the node in the cluster. Generally the driver node can be the edge node.

**Cluster :** 

 - Cluster is consist of number of nodes : number of Master nodes, number of Edge Nodes,
   number of Worker Nodes.
 - In cluster we have configuration of each node: number of cores per node, RAM and
   Disk Volume.

### How to use spark-submit to run the job
![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/spt_ep4_deploysparksubmit.jpg?raw=true)

 - In dev/qa/prod environment we have edge node and one big cluster which can be on-prem or cloud.
 - First user need to copy all the artifacts like **job**(means java, scala jar ), **files** (can be input files or config files), third party **library** to cluster or edge node.
 - Next from edge node run the job by spark submit. By using spark submit parameters user can submit the job on cluster
 - Spark provides the common utility like spark-submit to run the job/application on cluster.
  
 **Example :**
 ```bash
./bin/spark-submit \
--master yarn \
--deploy-mode cluster \
--conf "spark.sql.shuffle.partitions=1000" \
--jars "dependency1.jar,dependency2.jar"
--class com.test.WordCountExample \
spark-example.jar 
```

## 2. REST API Apache Livy

Apache Livy is a service that enables easy interaction with a Spark cluster over a REST interface. It enables easy submission of Spark jobs or snippets of Spark code, synchronous or asynchronous result retrieval, as well as Spark Context management, all via a simple REST interface.

Livy manages Spark Contexts running on the cluster managed by a Resource Manager like YARN. This enables proper fault-tolerance, high availability, session isolation, scalability and security. Livy also provides multiple modes of interaction: REST based jar submission, a thin java client for fine grained job submission and result retrieval, as well as submission of code snippets in string form. Thus Livy enables interactive Applications as well as interactive Notebooks like Jupyter, to leverage a remote Spark cluster.

The following image, taken from the official website, to show the Livy architecture what happens when submitting Spark jobs/code through the Livy REST APIs:

![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/spt-e04-livy-architecture.png?raw=true)

### Features of Livy:
 -   Jobs can be submitted from anywhere, using the REST API.
-   Long running Spark Contexts that can be used for multiple Spark jobs, by multiple clients
-   Share cached RDDs or DataSets across multiple jobs and clients
-   Multiple Spark Contexts can be managed simultaneously, and the Spark Contexts run on the cluster (YARN/Mesos) instead of the Livy Server, for good fault tolerance and concurrency
-   Jobs can be submitted as precompiled jars, snippets of code or via java/scala client API
-   Ensure security via secure authenticated communication


### How to use Livy to submit the Job

In this section we will look at examples with how to use Livy Spark Service to submit batch job, monitor the progress of the job.

![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/spt_e04_livy.jpg?raw=true)

 - Before submit a batch job, first build spark application and create the assembly jar. You must upload the application jar on the cluster storage (HDFS) of the hadoop cluster. The main difference between submitting job through spark-submit and REST API is that jar to be uploaded into the cluster.
 - Next job can be submitted through REST API from remote server. The spark job parameters is in JSON format. below are the sample job parameters.
	```shell
	{
	  "name" : "spark-example",
	  "className" :  "com.test.WordCountExample",
	  "file"  : "/user/example/spark-example.jar",
	  "proxyUser" : "hadoop",
	  "driverMemory" : "3g",
	  "driverCores" : "3",
	  "args" : ["sample", "10" ]
	}
	```
 - Next submit the batch job with REST POST call to http://<livy-server>:8998/batches request.
	```shell
	curl -H "Content-Type: application/json" localhost:8998/livy/batches
	 -X POST --data '{
	  "name" : "spark-example",
	  "className" :  "com.test.WordCountExample",
	  "file"  : "/user/example/spark-example.jar",
	  "proxyUser" : "hadoop",
	  "driverMemory" : "3g",
	  "driverCores" : "3",
	  "args" : ["sample", "10" ]
	}'
	```
 - Next livy server return the batch job response.
	 ```shell
	{"id":0,"state":"starting","appId":null,"appInfo":{"driverLogUrl":null,"sparkUiUrl":null},"log":["stdout: ","\nstderr: ","\nYARN Diagnostics: "]}
	```

please Check out the complete [documentation](https://github.com/gurditsingh/blog/blob/gh-pages/files/TestSparkSession.html "Source code").

## Need of Apache Livy when we already have spark-submit?

Livy is a REST web service for submitting Spark Jobs or accessing – and thus sharing – long-running Spark Sessions from a remote place. Instead of tedious configuration and installation of your Spark client, Livy takes over the work and provides you with a simple and convenient interface.

Another great aspect of Livy, namely, is that you can choose from a range of scripting languages: Java, Scala, Python, R.

Below are the points which creates a difference:

![Spark](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/spt_e04_livy_spark.jpg?raw=true)

### When you should use Apache Livy

- when multiple users want to share a Spark Session.
- when user wants to run low-latency queries.
- when user wants to share the rdd or dataframes. 
- when user need a quick setup to access your Spark cluster.
- when you want to Integrate Spark into an app on your mobile device.
- when need a remote workflow tool for submits spark jobs.

### Lets understand by code to share context and dataframes.

 - User need to create a session first and choose the kind to session.
	```shell
	curl -H "Content-Type: application/json" localhost:8998/livy/sessions
	 -X POST --data '{
	  "kind" : "spark"
	}'
	```
 - After creating the session livy will return a session id which will be used to run the subsequent statements.
	```shell
	curl -H "Content-Type: application/json" localhost:8998/livy/sessions/0/stetements
	 -X POST --data '{
	  "code" : "spark.range(10).count()"
	}'
	```
 - After running the statement livy will return a statement id which will be used to get the result of the statement.

**share Dataframe :** Likewise user can cache or create temp view of dataframe which can be used by other users. 
 - First user can run the statement and create temp view(create temp table in spark) of the data frame on the session id:1 (assume you select session id 1 for the statements).
 - Other users can use the same session id (suppose session id:1) and they will be able to use the already created tables.

## ETL use case for apache Livy
Suppose you have to create a functionality in which user can preview the input data or intermediate data. In the traditional ETL tools like **AB initio**, **Informatica** and etc have the functionality user can preview the input data as well as intermediate data between the components. The traditional tools not only give the preview functionality they will give transformation functionality (apply functions or filter the data) as well.

So if we have any ETL tool and we need to add the functionality of preview data then we can use apache Livy for faster execution with the help of spark context sharing. In the spark context sharing we can save the time to only one time to create the yarn application no need to create for every preview. Create an new yarn application (new yarn application needs some time to launch).

## Next ?

Planning to create multiple blogs episodes on Spark Performance Tuning. Covering the various areas of spark where we can improve the pipeline/job.
