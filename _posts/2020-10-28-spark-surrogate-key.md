---
title: "Surrogate key with Apache Spark - Part-1"
layout: post
excerpt: "You can generate SURROGATE_KEY by apache spark to automatically generate numerical Ids for rows as you enter data into a table."
last_modified_at: 2020-10-28T11:21:01-05:00
tags:
  - Spark
  - Scala
  - Surrogate key
  - database
  - Data Warehouse
  - monotonically_increasing_id
---

## What does Surrogate Key mean?
 
 A surrogate key (or synthetic key, pseudokey, entity identifier, system-generated key, database sequence number, factless key, technical key, or arbitrary unique identifier) in a database is a unique identifier for either an entity in the modeled world or an object in the database. The surrogate key is not derived from application data, unlike a natural (or business) key which is derived from application data.
 
 **Surrogate key in a Data Warehouse**: Surrogate keys are typically meaningless integers used to connect the fact to the dimension tables of a data warehouse. There are various reasons why we cannot simply reuse our existing natural or business keys.

## Let's examine the monotonically_increasing_id provided by Spark

 - **monotonically_increasing_id :** Spark dataframe add unique number is very common requirement especially if you are working on ETL in Spark. You can use monotonically_increasing_id method to generate long number which is monotonically increasing and unique, but not consecutive.
 
 

	>  **Spark Doc :** The generated ID is guaranteed to be monotonically increasing and unique, but not consecutive. The current implementation puts the partition ID in the upper 31 bits, and the record number within each partition in the lower 33 bits. The assumption is that the data frame has less than 1 billion partitions, and each partition has less than 8 billion records.
	
	
	
	**Let’s create a sample job (Job-1) to generate surrogate keys**
	
	```scala
	 def run(args: Array[String]): Unit = {

	    val spark = SparkSession
	      .builder()
	      .master(args(0))
	      .config("spark.sql.warehouse.dir", System.getProperty("user.dir") + "/spark-warehouse")
	      .enableHiveSupport()
	      .getOrCreate()


	    val articles = loadDataFromSource(spark)

	    val attachSurrogateKey = articles.withColumn("sk", functions.monotonically_increasing_id())

	    attachSurrogateKey.write.mode(SaveMode.Append).saveAsTable("articles_tbl")

	  }

	```
	After running this job surrogate keys will generate. But in ETL jobs we going to be inserting the data in batches, maybe a million at a time, maybe 1000 at a time. So we want to see how this surrogate key generation performs over multiple inserts.

	> Run the same job one more time and see how surrogate keys are generated : so when we run the same job again, it generates the duplicate surrogate keys.

	**Lets understand with Example**: 
	

 - In First run we insert 1 million records and spark generates unique 1million surrogate keys.
 - In Second run we again insert 1 million records but it generates duplicates surrogates keys because second run doesn't know about first run keys.
 
	**What is the reason for this massive amount of surrogates keys collisions/duplication ?**
	The thing is with monotonically increasing ID is, it returns a number between zero and some upper bound. And it only guarantees that the numbers are increased monotonically. So there's no guarantee you'll generate the same numbers or won't generate the same for next batches.

	**Possible Solution :** Since monotonically increasing ID starts with zero, we're going to add max value to it. So we will take the max value from the previous run and add monotonically increasing ID from current run to generate SK for the second and subsequent attempts and by this we've achieved uniqueness, which is a very important criteria in surrogate keys.
	
 
	 **Let’s create a sample job (Job-2) to generate surrogate keys with max value**
	```scala
	def run(args: Array[String]): Unit = {

	    val spark = SparkSession
	      .builder()
	      .master(args(0))
	      .config("spark.sql.warehouse.dir", System.getProperty("user.dir") + "/spark-warehouse")
	      .enableHiveSupport()
	      .getOrCreate()

	    val articles = loadDataFromSource(spark)

	    val maxValueOfSK = if (spark.catalog.tableExists("articles_tbl"))
	      spark.read.table("articles_tbl").groupBy().max("sk").collect() match {
	        case Array() => 0L
	        case a => a.head.get(0).asInstanceOf[Long]
	      }
	    else 0L


	    val attachSurrogateKey = articles.withColumn("sk", functions.monotonically_increasing_id().+(maxValueOfSK))

	    attachSurrogateKey.write.mode(SaveMode.Append).saveAsTable("articles_tbl")

	    spark.read.table("articles_tbl").show()

	  }
	```
	 So the good news is that we have distribution and uniqueness. 

	**Evaluation of monotonically_increasing_id()**
	

	 - **Uniqueness :** By using second job (job-2) we have achieved uniqueness.
	 - **Evenly Distributed :** Keys are distributed but have larger values.
	 - **DBA Perspective :** I think that the DBA will complain about the maximum value of surrogate key is way larger than the total number of records in the table. e.g. if your table contains millions records but the max value of surrogate key can be in trillions because of internal logic of generating monotonically_increasing_id() and the subsequent runs again add max value with monotonically_increasing_id() for uniqueness.  
