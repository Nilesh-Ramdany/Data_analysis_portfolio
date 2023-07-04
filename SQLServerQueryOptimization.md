# SQL Server Query Optimization



### Objective

Analyze SQL queries to find performance issues and fix them.

#### Entity Relationship Diagram
![](/execution_plans/'students database.png')


#### 1. Adding Missing Indexes

Let's consider the following query that select all the courses that a particular student has taken

````sql
SELECT	c.departmentcode,
		c.coursenumber,
		c.coursetitle,
		c.credits,
		ce.Grade
FROM CourseEnrollments ce
INNER JOIN courseofferings co on co.courseofferingid = ce.courseofferingid
INNER JOIN courses c on
	co.departmentcode = c.departmentcode AND co.coursenumber = c.coursenumber
where ce.studentid = 29717;
````


Before executing the query we will look at the estimated execution plan below 

We see that the Clustered index Scan on the course enrollment table accounts for 94% cost of the query.
To improve the performance of the query this is the operation that we would want to have a look at. The operation is scanning the whole course enrollments table to find rows where studentid = 29717.

We can use the following to get some statistics when we execute the query.


````sql
set statistics io on
set statistics time on
````

Executing the above query gives us these statistics:

````
Table 'CourseEnrollments'. Scan count 9, logical reads 12197
Table 'Courses'. Scan count 0, logical reads 80
Table 'CourseOfferings'. Scan count 0, logical reads 158

SQL Server Execution Times:
   CPU time = 138 ms,  elapsed time = 18708 ms.
````

We notice a high number of logical reads on the course enrollments table. We want to minimize this to make the query run more efficiently. To do this we can create a missing index:


````sql

USE [Students]
GO
CREATE NONCLUSTERED INDEX IX_CourseEnrollments_StudentID
ON [dbo].[CourseEnrollments] ([StudentId])

GO

````


Now in the estimated execution plan the index scan operation has been replaced by an index seek and a key lookup. The cost of the operation is lower.


we execute the query now and look at the statistics.


````
Table 'Courses'. Scan count 0, logical reads 80
Table 'CourseOfferings'. Scan count 0, logical reads 91
Table 'CourseEnrollments'. Scan count 1, logical reads 134
 SQL Server Execution Times:
 
   CPU time = 0 ms,  elapsed time = 1 ms.
````







#### 2. Rewriting Queries to be more efficient

Let's consider the following query that select all the courses in a particular term that has no enrollment.

````sql
SELECT 
	co.CourseOfferingId,
	co.Departmentcode,
    co.CourseNumber,
    co.TermCOde,
    co.Section
FROM CourseOfferings co
LEFT JOIN CourseEnrollments ce on co.CourseOfferingId = ce.CourseOfferingId
WHERE co.TermCode ='SP2016' AND ce.CourseOfferingId IS NULL
````


Looking at the estimated execution plan, we see that the index seek accounts for 94%
of the cost. This is because SQL server is reading all of the course offerings in that semester and then probes an index on the course enrollments table to see how many enrollments there are for that course. 

We only need to know if there is at least one enrollment therefore we can rewrite the query as follows: 



````sql
SELECT 
	co.CourseOfferingId,
	co.Departmentcode,
    co.CourseNumber,
    co.TermCOde,
    co.Section
FROM CourseOfferings co
WHERE NOT EXISTS
(SELECT 1 FROM CourseEnrollments ce WHERE co.CourseOfferingId = ce.CourseOfferingId)
AND co.TermCode ='SP2016'
````


#### 2. Notes on Indexing

Correctly indexing columns helps us avoid performance issues. These are some practices to consider:

- Add indexes for WHERE clause criteria


- Index foreign key columns


- Understand how users will query the data and what combination of columns are used in the WHERE clauses in theses queries

Consider the following statements

````sql
CREATE iNDEX IX_applicants_FirstNameLastName
	ON Applicants (firstname, lastname, state);

SELECT * FROM applicants WHERE lastname = 'Davis' and state = 'co';
````
We have created an index on the applicants table and we are running a query to find an applicant by lastname and state.


We notice that there is an index scan operation that is going to read the entire index instead of using the tree structure of the index. This is because the WHERE clause does not include the first column of the index which is the firstname column.
The query has over 2600 logical read and is not efficient.

````
Table 'Applicants'. Scan count 1, logical reads 2691

SQL Server Execution Times:
   CPU time = 31 ms,  elapsed time = 100 ms.
````

To improve this we can either include the firstname in the where clause or change the order of the columns in our index. The second option is better since users will lookup students by their lastname more often. Sometimes the user may not know the firstname. Therefore, we will change the index to: 

````sql
CREATE iNDEX IX_applicants_LastNameFirstName
	ON Applicants (lastname, firstname, state);
````
Looking at the execution plan, we now have and index seek. The query is traversing the tree structure of the index and is therefore more efficient.


The number of logical reads has decreased significantly.
````
Table 'Applicants'. Scan count 1, logical reads 311

SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 14 ms.

````
- We want our Indexes to be as selective as possible 
	i,e we want to have few rows for each matching key(when an index is not selective enough we get a lot of matches from the index. Looking the values for the matching index ends up being more costly that just reading the table entirely. May lead to SQL optimizer not using the index at all.)
    
Consider the following statements. We have a query to find all students from a particular state and city. An index has been created on the state.

````sql
CREATE INDEX IX_students_state
	ON students (state);

SELECT * from students  
WHERE state = 'WI' AND city = 'Appleton';
````
When we look at the execution plan, we see that SQl server does not even use the index. The index is not selective enough.
    
- If we have a column that is not selective by itself, then we need to use it in conjunction with other columns to make it selective.

We can make the previous index more selective by using the city in the index.


````sql
CREATE INDEX IX_students_StateCity
	ON students (state);
````


#### 3. Use wildcards at the end of a phrase only

````sql
SELECT* FROM applicants
WHERE lastname LIKE '%Harris%'
AND firstname LIKE '%Thomas%';
````

Using a wildcard at the beginning of a search value, SQL server is not able to use the index for that column. This will lead to an entire scan of the index or the table. We need to provide enough information for our query to be selective enough so that sql server can use the index.



#### 4. Functions in the where clause affects indexes.

Suppose we  want to find a student based on the last few digits of their student ID.
We already know that using a leading wildcard in the search value will make our query inefficient. A better way to search would be by reversing the student ID which will allow us to use a wildcard at the end instead.

StudentID being the primary key, we already have a clustered index.

````sql
SELECT * FROM STUDENTS
WHERE REVERSE(studentid) LIKE REVERSE('%07295'); 
````

However looking at the execution plan we find that SQL server is still performing a scan on the entire Index. This is because we have a function on the studentID in the WHERE clause. This query is not an improvement over using a leading wildcard. 


plan9




The computed values REVERSE(studentid) is not sorted in the index. The function has to be run in real time to create the computed values and then SQL server will compare the values.

We are going to add the computed column



````sql
ALTER TABLE students 
		ADD reversedID AS REVERSE(studentID);
````


and the create and index over the computed column.


````sql
CREATE INDEX IX_reversedID
	ON students (reversedID);
````

Now when we execute the query:

````sql
SELECT studentID FROM STUDENTS
WHERE reversedID LIKE REVERSE('%07295');
````

We are able to use the index


plan10

```
Table 'Students'. Scan count 1, logical reads 2

 SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 0 ms.

```



#### 5. Include Columns and covering Index

Creating an INDEX with the keyword INCLUDE

````sql
CREATE INDEX IX_students_email
	ON students (email)
	INCLUDE (Firstname, Lastname);
    
    
SELECT Email, FirstName, LastName
	FROM Students
    WHERE Email = 'PaulDWilliams@gustr.com'
````


When we use the INCLUDE keyword in the above keyword, the values of FirstName, LastName will be store with the index key. This is useful to create a covering index.
SQL server is able to find all the data for the query in the index itself. There is no need to perform a key lookup operation.



plan 11

**Note: we should only consider a covering index when we only need to add one or two columns to give a performance boost to a key query. Including to many columns is basically just making another copy of the table**


#### 6. Over-Indexing

Indexes require maintenance, so we only want to create indexs that are actually used by our SQL statements.
When any sort of DML operation is performed on the table, SQL server has to keep all the indexes on the table in sync with the data in the table. Therefore we should drop indexes that are not being used.
We can use SQL server Dynamic Management Views to view index usage data. However, viewing server state is restricted.



#### 7. Common Performance Practices

##### Parameterized SQL

- When using parameterized SQL statements, once the statement is run the first time, every subsequent execution is going to be able to use the same **cached** execution plane

- If we are dynamically generating our SQL statements, the actual SQL statement is different each time. SQl server has to generate a new execution plan each time.



##### Stored Procedures

- Stored Procedures offer simialr benefits as Parameterized SQl statements
- Outperform dynamically generated SQl



##### Use SELECT fields instead of SELECT *


In the case of large databases, it is not recommended to retrieve all data because this will take more resources on querying a huge volume of data. Also, if the table has been altered (e.g new column added) this can affect the performance of our query.



##### Use TOP to sample query results
The SELECT TOP command is used to set a limit on the number of records to be returned from the database.
















































