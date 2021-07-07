## Problem
Data lakes holds large amount of data, and on these data needs to perform updates, merge and delete over the times. With the traditional data lakes it difficult to perform such a simple operations (because no one supports ACID). So that the user need to build there own strategy to perform these operations but in this approach data consistency is big threat, and it can become very difficult for data engineering and data scientists to reason about their data.

## Solution

Delta Lake solves this issue by enabling data analysts to easily query all the data in their data lake using SQL. Then, analysts can perform updates, merges or deletes on the data with a single command, owing to Delta Lake’s ACID transactions.


## What is DML Operation

DML refers to "Data Manipulation Language", a subset of SQL statements which deals with data manipulation. A transaction is a sequence of one or more SQL statements that can SELECT, INSERT, UPDATE, DELETE the data.



















![Delta lake](https://github.com/gurditsingh/blog/blob/gh-pages/_screenshots/dl_ep5_t7.JPG?raw=true)
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTgyMzk5NDcyMiwyODAwNzMzMzEsNTU0Mj
Q5MDUyLC0xMTE0ODQ2ODg1LDU3MzczODQ4OSwtNDA0OTAzMjQx
LDE2NDMzMTY1MSwtMTM4NzE5Nzk5MywxNTg3Mjk5OTAyLC03NT
kyMzE3NzgsOTYxMTU4Njc0LC0xNzM1MjcyNzIzLC0xNDEyMjE2
MTAsMTExODczNDkxLDE5NjY1MTY3NjksODUxMzU3MTAyLC0xNT
U3ODMxNjY5LC0xMjE1Njk0MjEzLC0xNDMxMTAzMjgyLC0xNzIw
NDMwMzkyXX0=
-->