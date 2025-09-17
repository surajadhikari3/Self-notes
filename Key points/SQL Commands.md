
                  SQL
                   |
  -------------------------------------------------
  |       |          |          |          |      |
 DDL     DML        DQL         DCL        TCL
(CREATE) (INSERT) (SELECT)    (GRANT)    (COMMIT)
	(ALTER)  (UPDATE)             (REVOKE)   (ROLLBACK)
(DROP)   (DELETE)                         (SAVEPOINT)
(TRUNCATE)


# 📋 Quick Cheat Sheet (for Interview)

| Category | Commands                              | Purpose              |
| -------- | ------------------------------------- | -------------------- |
| DDL      | CREATE, ALTER, DROP, TRUNCATE, RENAME | Structure management |
| DML      | INSERT, UPDATE, DELETE, MERGE         | Data manipulation    |
| DQL      | SELECT                                | Data retrieval       |
| DCL      | GRANT, REVOKE                         | Access control       |
| TCL      | COMMIT, ROLLBACK, SAVEPOINT           | Transaction control  |


Practise the sql commands like

Finding the salary 
Joins
unions.


Top SQL Asked questions....

### **What is the difference between `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN`, and `FULL OUTER JOIN`?**

**Explanation:**

### Summary Table:

| Join Type  | Description                              | Use Case Example                             |
| ---------- | ---------------------------------------- | -------------------------------------------- |
| INNER JOIN | Matching rows from both tables           | Get customer names for placed orders         |
| LEFT JOIN  | All left + matched right                 | Get all customers, even if no order placed   |
| RIGHT JOIN | All right + matched left                 | Get all orders, even if customer is missing  |
| FULL JOIN  | All records from both tables             | Show full data with or without matches       |
| SELF JOIN  | Join same table to find relations within | Who reports to whom in an employee hierarchy |
    

**Example:**

sql
```
SELECT emp.name, dept.name 
FROM employee emp
LEFT JOIN 
department dept 
ON emp.dept_id = dept.id;
```

### **2. How do you optimize a slow SQL query?**

**Explanation:**

- **Indexing**: Use composite or partial indexes wisely
    
- **Partitioning**: Break large tables into manageable parts
    
- **Query optimization**:
    
    - Avoid `SELECT *`
        
    - Use joins smartly (INNER vs LEFT)
        
    - Use `EXPLAIN PLAN`
        
- **Caching**: Use Hibernate 2nd level cache / Redis
    
- Use **LIMIT**, **pagination**, or **partitioning** for large datasets.


### **What are indexes? What types of indexes exist?**

**Explanation:**  
Indexes are data structures that improve the speed of data retrieval.

- **B-Tree Index**: Default, good for equality and range queries.
    
- **Hash Index**: For equality, faster than B-Tree but no range support.
    
- **Composite Index**: Index on multiple columns.
    
- **Unique Index**: Ensures all values are different.
    
- **Full-text Index**: For searching within large text columns.


### **Explain normalization and denormalization.**

**Explanation:**

- **Normalization** = Reducing redundancy, organizing data in related tables.
    
    - 1NF: Atomic columns
        
    - 2NF: Remove partial dependencies
        
    - 3NF: Remove transitive dependencies
        
- **Denormalization** = Combining tables to improve performance at the cost of redundancy.


| **Normal Form** | **Goal**                       | **Rule**                                                                                                             | **Simple Example**                                                                                            |
| --------------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| **1NF**         | Ensure atomic values           | No repeating groups or multi-valued columns. Each cell = 1 value only.                                               | ❌ `Courses = "Math, English"`✅ Split into separate rows                                                       |
| **2NF**         | Remove partial dependencies    | If a table has a **composite key**, then all non-key columns must depend on the **entire key**, not just part of it. | ❌ `StudentName` depends only on `StudentID`, not `StudentID + Course`✅ Move `StudentName` to a separate table |
| **3NF**         | Remove transitive dependencies | Non-key columns must depend **only** on the primary key, **not on other non-key columns**.                           | ❌ `StudentID → ZipCode → City`✅ Move `City` to a separate `ZipCode` table                                     |


**Example use case:**

- Normalized for **transactional systems**.
    
- Denormalized for **reporting systems**.


### **What is the difference between `WHERE` and `HAVING`?**

**Explanation:**

- `WHERE`: Filters rows **before aggregation**. --> Where does the row level checks. It performs the checks on individual rows..
    
- `HAVING`: Filters rows **after aggregation**. --> Having runs after the aggregation .
										Aggregation are sum(), count(), avg(), min(), max() --> which aggregate or summarize data
    

**Example:**

```
SELECT dept_id, COUNT(*) as emp_count FROM employee GROUP BY dept_id 
HAVING 
COUNT(*) > 5;
```


### **What is the difference between `UNION` and `UNION ALL`?**

**Explanation:**

- `UNION`: Combines results and removes duplicates.
    
- `UNION ALL`: Combines results **with duplicates**.
    

`UNION ALL` is **faster** since it skips duplicate check.

### **Explain ACID properties in SQL databases.**

**Explanation:**

- **Atomicity** – All operations in a transaction complete or none do.
    
- **Consistency** – DB remains in valid state before/after.
    
- **Isolation** – Transactions don't interfere.
    
- **Durability** – Once committed, changes persist.

### **How do `ROWNUM`, `LIMIT`, and `OFFSET` work?**

**Explanation:**  
Used for **pagination**:

**MySQL/Postgres:**

```
`SELECT * FROM employee ORDER BY id LIMIT 10 OFFSET 20;`

```


**Oracle:**

### **`ROWNUM`** (in Oracle SQL)

- `ROWNUM` is a **pseudocolumn** in Oracle databases.
    
- It assigns a **unique number** to each row returned by a query, starting from 1.
    
- It is **evaluated before ORDER BY**, so it can sometimes behave unexpectedly if used with sorting.
    

**Example:**


```
`SELECT * FROM employees WHERE ROWNUM <= 5;`
```
Returns the **first 5 rows** of the result set.

> ⚠️ Since it's assigned before sorting, if you want the _top 5 highest salaries_, you must use a subquery:

```
`SELECT * FROM (   SELECT * FROM employees ORDER BY salary DESC ) WHERE ROWNUM <= 5;`
```


### **`OFFSET`** and **`LIMIT`** (in PostgreSQL, MySQL, SQLite)

These are used for **pagination** (fetching part of the result set).

- `LIMIT` — how many rows to return.
    
- `OFFSET` — how many rows to skip before starting to return rows.
    

**Example:**

`SELECT * FROM employees ORDER BY id LIMIT 5 OFFSET 10;`

This will skip the first 10 rows and return the **next 5 rows**.

- Equivalent to: _"Give me rows 11 through 15."_
    

> ✅ Useful for pagination: `LIMIT` is what to show, `OFFSET` is what to skip.

---

### Summary

|Term|Used In|Purpose|
|---|---|---|
|`ROWNUM`|Oracle|Pseudocolumn for row numbering|
|`OFFSET`|PostgreSQL, MySQL, SQLite|Skip a number of rows|
|`LIMIT`|PostgreSQL, MySQL, SQLite|Restrict number of rows returned|


Q) Finding the second highest salary of the employee?

SELECT DISTINCT SALARY FROM EMPLOYEE ORDER BY SALARY DESC LIMIT 1 OFFSET 1;

note : offset 1 means skip the first row so we will get the second unique highest which is ensured by the distinct keyword....
		ORDER BY --> means sorting by

Similary we can generalize it for the nth highest salary 

SELECT DISTINCT SALARY FROM EMPLOYEE ORDER BY SALARY DESC LIMIT 1 OFFSET N-1; 


### **What is the difference between `TRUNCATE`, `DELETE`, and `DROP`?**

**Explanation:**

- `DELETE`: Deletes rows, can be rolled back.
    
- `TRUNCATE`: Deletes all rows, faster, can't be rolled back (depends on DB).
    
- `DROP`: Removes the table/schema entirely.


### **What is the difference between primary key and unique key?**

**Explanation:**

- **Primary Key**: Uniquely identifies a row; one per table; can't be NULL.
    
- **Unique Key**: Also enforces uniqueness; can have **multiple**; allows **NULLs**.


### **. How does indexing affect `INSERT`, `UPDATE`, and `DELETE` performance?**

**Explanation:**

- Indexes **speed up reads**.
    
- But **slow down writes** (INSERT, UPDATE, DELETE) due to index maintenance overhead.
    

Hence, balance **read vs. write** optimization based on system requirements.

Example of SELF JOIN

Self join is a join of table with itself..

Performing the self join to who reports to whom (Knowing the manager from it)

SELECT E1.name AS Employee, E2.name as Manager FROM employees E1 JOIN ON employees E2 
WHERE E1.id = E2.manager_id;

```
employees
+----+--------+------------+
| id | name   | manager_id |
+----+--------+------------+
| 1  | Alice  | NULL       |
| 2  | Bob    | 1          |
| 3  | Carol  | 1          |
| 4  | Dave   | 2          |
+----+--------+------------+

Result:
+----------+---------+
| Employee | Manager |
+----------+---------+
| Bob      | Alice   |
| Carol    | Alice   |
| Dave     | Bob     |
+----------+---------+

```
select E1.name as Employee , E2.name as Manager 
from employee E1
self join employee E2
on E1.id = E2.manager_id;


Performance Tuning (via `Execution plan`)

An **execution plan** shows how the **SQL engine** executes your query — including what **indexes, joins, sort methods**, and **access paths** are used.

I treid it with the active_life_canada where i joined the offered_course table with the course table and get the join table 
applied the `explain analyze` to get the execution plan

```
explain analyze select oc.bar_code, oc.number_of_seats, c.description 
from offered_course oc 
join course c on oc.course_id = c.course_id;
```
 got the following output
 
 ```
   QUERY PLAN
------------------------------------------------------------------------------------------------------
 Hash Join  (cost=11.57..23.10 rows=120 width=1036) (actual time=0.171..0.178 rows=10 loops=1)
   Hash Cond: (oc.course_id = c.course_id)
   ->  Seq Scan on offered_course oc  (cost=0.00..11.20 rows=120 width=528) (actual time=0.078..0.079 rows=10 loops=1)
   ->  Hash  (cost=10.70..10.70 rows=70 width=524) (actual time=0.072..0.072 rows=10 loops=1)
         Buckets: 1024  Batches: 1  Memory Usage: 9kB
         ->  Seq Scan on course c  (cost=0.00..10.70 rows=70 width=524) (actual time=0.060..0.063 rows=10 loops=1)
 Planning Time: 0.327 ms
 Execution Time: 0.302 ms
(8 rows)
```

I see that it is using the hash join and the full table scan so to optimize created the index

```
CREATE INDEX idx_offered_course_course_id ON offered_course(course_id);
CREATE INDEX idx_course_course_id ON course(course_id);

as we are joining on the basis of the course_id;
```

## **Updating Table Statistics**

After adding indexes, **refresh the planner's knowledge** by updating statistics:


`ANALYZE offered_course;
 ANALYZE course;`

## **Force the Query Plan (Optional - Use Carefully)**

Disable sequential scans** just to test how PostgreSQL behaves with indexes:


`SET enable_seqscan = OFF;  

EXPLAIN ANALYZE SELECT oc.bar_code, oc.number_of_seats, c.description  FROM offered_course oc  JOIN course c ON oc.course_id = c.course_id;`


Got the following output after

   QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------
 Merge Join  (cost=0.27..17.42 rows=10 width=50) (actual time=0.094..0.105 rows=10 loops=1)
   Merge Cond: (oc.course_id = c.course_id)
   ->  Index Scan using idx_offered_course_course_id on offered_course oc  (cost=0.14..12.29 rows=10 width=28) (actual time=0.042..0.048 rows=10 loops=1)
   ->  Index Scan using idx_course_course_id on course c  (cost=0.14..12.29 rows=10 width=38) (actual time=0.002..0.004 rows=5 loops=1)
 Planning Time: 2.827 ms
 Execution Time: 0.212 ms
(6 rows)

### Improvements:

| Optimization                 | Result                                          |
| ---------------------------- | ----------------------------------------------- |
| **Index Scan** on both sides | Only relevant rows are accessed — much faster   |
| **Merge Join**               | Efficient when inputs are sorted (indexes help) |
| **Accurate rows estimate**   | Estimated 10, returned 10 — stats now accurate  |
| **Execution Time reduced**   | From **0.302 ms → 0.212 ms**                    |

Steps for analyze the plan

Run the analyze plan using the keyword explain analyze in postgres and read the output
### Read the Output

A typical plan includes:

| Field         | Meaning                                                         |
| ------------- | --------------------------------------------------------------- |
| `id` / `step` | The step number in query execution                              |
| `select_type` | Type of query (SIMPLE, UNION, SUBQUERY)                         |
| `table`       | Table being accessed                                            |
| `type`        | Join/access type (ALL, index, ref, const) – **lower is better** |
| `key`         | Which index is used                                             |
| `rows`        | Estimated number of rows scanned                                |
| `Extra`       | Notes (e.g. "Using where", "Using temporary", "Using filesort") |
Add the indexes on both the table and again checks the execution plan it will reduce the execution time as it does the index based changed instead of flat scan for the table..
