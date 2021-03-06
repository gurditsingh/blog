---
title: "Surrogate key with Apache Spark - Part-2"
layout: post
excerpt: "You can generate SURROGATE_KEY by apache spark to automatically generate numerical Ids for rows as you enter data into a table."
last_modified_at: 2020-11-01T10:21:02-05:00
tags:
  - Spark
  - Scala
  - Surrogate key
  - database
  - Data Warehouse
  - zipwithindex
---

## What does Surrogate Key mean?

 A surrogate key (or synthetic key, pseudokey, entity identifier, system-generated key, database sequence number, factless key, technical key, or arbitrary unique identifier) in a database is a unique identifier for either an entity in the modeled world or an object in the database. The surrogate key is not derived from application data, unlike a natural (or business) key which is derived from application data.
 
 **Surrogate key in a Data Warehouse**: Surrogate keys are typically meaningless integers used to connect the fact to the dimension tables of a data warehouse. There are various reasons why we cannot simply reuse our existing natural or business keys.

## Let's examine the zipwithindex 

 - **zipwithindex :** Spark dataframe add unique number is very common requirement especially if you are working on ELT in Spark. You can use zipwithindex method to generate long number. zipwithindex works on rdd and use  `df.rdd.zipWithIndex()` . 

	>  **Spark Doc :** Zips this RDD with its element indices. The ordering is first based on the partition index and then the ordering of items within each partition. So the first item in the first partition gets index 0, and the last item in the last partition receives the largest index. **This method needs to trigger a spark job when this RDD contains more than one partitions.**
	

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

	    val zippedRDD = articles.rdd.zipWithIndex()

	    val combineRDD = zippedRDD.map { case (row, index) =>
	      Row.fromSeq(Array(index) ++ row.toSeq)
	    }

	    val convertedDF = spark.createDataFrame(combineRDD, getSchema())

	    convertedDF.write.mode(SaveMode.Append).saveAsTable("articles_tbl")

	  }

	```
	
	
	**Noticeable points**
	
	 -  The indexes will be  starting from 0 and the ordering is done by partition.
	 -  Falling back to rdds and then to dataframe  [**can be quite expensive**](https://stackoverflow.com/questions/37088484/whats-the-performance-impact-of-converting-between-dataframe-rdd-and-back)**.**
	 - This method needs to trigger a spark job when this RDD contains  
more than one partitions.
	 

	
	-   After running this job surrogate keys will generate. But in ETL jobs we going to be inserting the data in batches, maybe a million at a time, maybe 1000 at a time. So we want to see how this surrogate key generation performs over multiple inserts.
    
    > Run the same job one more time and see how surrogate keys are generated : so when we run the same job again, it generates the duplicate surrogate keys.
    
    **Lets understand with Example**:
    
-   In First run we insert 1 million records and spark generates unique 1million surrogate keys.
-   In Second run we again insert 1 million records but it generates duplicates surrogates keys because second run doesn’t know about first run keys.
 
	**What is the reason for this massive amount of surrogates keys collisions/duplication ?**
	The thing is with zipwithindex is, it returns a number between zero and some upper bound [ordering is first based on the partition index and then the ordering of items within each partition. So the first item in the first partition gets index 0, and the last item in the last partition receives the largest index]. And it only guarantees that the numbers are unique. So it will generate the same numbers for next batches.

	**Possible Solution :** We will take the max value Plus a range of IDs to generate SK for the second and subsequent attempts and by this we've achieved uniqueness, which is a very important criteria in surrogate keys.
	
 
	 **Let’s create a sample job (Job-2) to generate surrogate keys with max value**
	```scala
	def run(args: Array[String]): Unit = {

	    val spark = SparkSession
	      .builder()
	      .master("local")
	      .config("spark.sql.warehouse.dir", System.getProperty("user.dir") + "/spark-warehouse")
	      .enableHiveSupport()
	      .getOrCreate()

	    val articles = loadDataFromSource(spark)

	    val maxValueOfSK = if (spark.catalog.tableExists("articles_tbl"))
	      spark.read.table("articles_tbl").groupBy().max("sk").collect() match {
	        case Array() => 1L
	        case a => a.head.get(0).asInstanceOf[Long]
	      }
	    else 1L

	    val zippedRDD = articles.rdd.zipWithIndex()

	    val combineRDD = zippedRDD.map { case (row, index) =>
	      Row.fromSeq(Array(index + maxValueOfSK) ++ row.toSeq)
	    }

	    val convertedDF = spark.createDataFrame(combineRDD, getSchema())

	    convertedDF.write.mode(SaveMode.Append).saveAsTable("articles_tbl")

	  }
	```
	
	So the good news is that we have distribution and uniqueness. 

	**Evaluation of zipwithindex**
	

	 - **Uniqueness :** By using second job (job-2) we have achieved uniqueness.
	 - **Evenly Distributed :** Keys are evenly distributed.
	 - **DBA Perspective :** We have equal number of surrogate keys same with the total number of records in the table (e.g. 2 million surrogate keys and in the range from one to 2 million with table having 2 million records). So that's a good point and as DBA prospective there is no leakage or gaps in between keys.
	 - **Performance :** Job performance can hamper because zipwithindex will trigger another job and rdd to dataframe conversion can be quite expensive operation. One more thing which will increase the performance is join right to that table to figure out what the latest value is in order to increase it.
	 
