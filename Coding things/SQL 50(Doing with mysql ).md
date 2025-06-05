
charlength() --> Function to count the length of the varchar in mysql

datediff() --> To calculate the date difference for example can be used as..

![[Pasted image 20250603102028.png]]


ROUND(number, decimal_places) --> Can perform the operation and do the rounding...........

![[Pasted image 20250603110614.png]]

AVG() --> gives the average value and make sure to do the group by on the aggregated functions like avg(), sum().

STUDENTS AND EXAMINATIONS

![[Pasted image 20250603120435.png]]
First take the cartesian product which is cross join for the students and subjects table as we have to show all the subjects even though the student never attended the exams. then we will do the left join to the examination table


## ✅ Rule:

### 👉 `WHERE` filters **rows**

- It works **before aggregation**
    
- You **can’t** use `COUNT()`, `SUM()`, etc. inside `WHERE`
    

### 👉 `HAVING` filters **groups**

- It works **after `GROUP BY` and aggregation**
    
- It **can** use aggregate functions like `COUNT()`, `AVG()`, `SUM()`
    

---

## 🔄 SQL Execution Order (Simplified):


```
1. FROM         → get all rows
2. JOIN         → combine tables
3. WHERE        → filter rows (raw data only, no COUNT/SUM)
4. GROUP BY     → group rows
5. HAVING       → filter groups (can use COUNT/SUM/etc.)
6. SELECT       → select final columns
7. ORDER BY     → sort result

```


Great question! Both `COALESCE()` and `IFNULL()` are used in SQL to **handle `NULL` values**, but they work a little differently.

---

## ✅ `IFNULL(expression, value_if_null)`

### ➤ Checks **one value**

- If the first value is `NULL`, it returns the second.
    
- If the first value is **not** `NULL`, it returns it as-is.
    

### 🔹 Syntax:

```sql
IFNULL(expr1, expr2)
```

### 🔸 Example:

```sql
SELECT IFNULL(NULL, 0);       -- → 0
SELECT IFNULL(5, 0);          -- → 5
SELECT IFNULL(NULL, 'N/A');   -- → 'N/A'
```

---

## ✅ `COALESCE(expr1, expr2, expr3, ...)`

### ➤ Checks **multiple values** in order

- Returns the **first non-NULL** value from the list.
    
- If **all are NULL**, it returns `NULL`.
    

### 🔹 Syntax:

```sql
COALESCE(expr1, expr2, expr3, ...)
```

### 🔸 Example:

```sql
SELECT COALESCE(NULL, NULL, 5, 10);   -- → 5
SELECT COALESCE(NULL, 'hello');       -- → 'hello'
SELECT COALESCE(NULL, NULL);          -- → NULL
```

---

## 🧠 Key Differences:

| Feature       | `IFNULL()`               | `COALESCE()`                             |
| ------------- | ------------------------ | ---------------------------------------- |
| Arguments     | 2 only                   | Multiple (2 or more)                     |
| Return value  | 2nd if 1st is NULL       | First non-NULL in the list               |
| Compatibility | MySQL-specific           | Standard SQL (works in PostgreSQL, etc.) |
| Flexibility   | Limited to just 2 values | More powerful with multiple fallbacks    |

---

## ✅ When to use:

- **Use `IFNULL()`** for simple 2-value cases in MySQL.
    
- **Use `COALESCE()`** when:
    
    - You need multiple fallback options
        
    - Writing portable SQL (works in most databases)
        

---

![[Pasted image 20250603142838.png]]


Note sum takes the boolean but the count does not if you want to count on conditional basis then u have to pass the boolean by yourself like below .....................


`count(case when rating < 3 then 1 end)`

---

## ✅ SQL Aggregate Functions: Can You Use Boolean/Conditional Inside?

|Function|Accepts Boolean/Condition?|Behavior / Notes|Example|Result Explanation|
|---|---|---|---|---|
|**`SUM()`**|✅ Yes|Adds up `1` for true, `0` for false; can simulate count|`SUM(rating < 3)`|Adds 1 for each row where rating < 3|
|**`COUNT(*)`**|❌ No (counts all rows)|Counts all rows, including NULLs|`COUNT(*)`|Total row count|
|**`COUNT(expr)`**|⚠️ _Only counts non-NULL_|`COUNT(condition)` counts all rows unless condition is NULL|`COUNT(rating < 3)` → counts all non-NULL rows|**Not recommended** for booleans|
|**`COUNT(CASE WHEN ...)`**|✅ Yes|Safest way to count rows matching a condition|`COUNT(CASE WHEN rating < 3 THEN 1 END)`|Counts only rows where condition is true|
|**`AVG()`**|✅ Yes|Averages boolean values (1 for true, 0 for false)|`AVG(rating < 3)`|Gives proportion as a decimal (e.g., 0.25 → 25%)|
|**`MIN()` / `MAX()`**|✅ (sort of)|Compares boolean or conditional results|`MAX(rating < 3)`|Returns 1 if any row matches condition|

---

## ✅ Recommended Use Summary

|Use Case|Preferred Expression|
|---|---|
|Count where condition is true|`COUNT(CASE WHEN condition THEN 1 END)`|
|Sum of condition matched rows|`SUM(condition)`|
|Percentage of rows matching condition|`SUM(condition) / COUNT(*) * 100` or `AVG(condition) * 100`|
|Avoid ambiguous `COUNT(condition)`|✅ Use `CASE WHEN` for clarity|

---

## ✅ Example Query:

```sql
SELECT 
  name,
  COUNT(CASE WHEN rating < 3 THEN 1 END) AS low_ratings,
  SUM(rating < 3) AS low_ratings_alt,
  AVG(rating < 3) * 100 AS percent_low,
  ROUND(SUM(rating < 3) / COUNT(*) * 100, 2) AS low_rating_pct
FROM 
  reviews
GROUP BY 
  name;
```


✅ Common MySQL Date Extraction Functions

| Purpose         | Function                       | Example                   | Result     |
| --------------- | ------------------------------ | ------------------------- | ---------- |
| Get year        | `YEAR(date)`                   | `YEAR('2018-12-18')`      | `2018`     |
| Get month       | `MONTH(date)`                  | `MONTH('2018-12-18')`     | `12`       |
| Get day         | `DAY(date)`                    | `DAY('2018-12-18')`       | `18`       |
| Get day name    | `DAYNAME(date)`                | `DAYNAME('2018-12-18')`   | `Tuesday`  |
| Get month name  | `MONTHNAME(date)`              | `MONTHNAME('2018-12-18')` | `December` |
| Get weekday     | `WEEKDAY(date)` _(0 = Monday)_ | `WEEKDAY('2018-12-18')`   | `1`        |
| Get week number | `WEEK(date)`                   | `WEEK('2018-12-18')`      | `51`       |

## IMPORTANT:

You can also format the date using `DATE_FORMAT`:


`SELECT DATE_FORMAT('2018-12-18', '%Y-%m') AS year_month;  -- → '2018-12'`


For finding the monthly transactions I 

![[Pasted image 20250604104455.png]]

LEFT FUNCTION AND RIGTH FUNCTION

Use to extract the substring, date from the expressions 

```
SELECT LEFT('abcdef', 3);     -- → 'abc'
SELECT LEFT('2024-06-01', 4); -- → '2024'   (useful to extract year)
SELECT LEFT('JavaStream', 4); -- → 'Java'
`RIGHT('abcdef', 3)` → `'def'`
```

### Calculating the date differences and range


## ✅ 1. You Know the End Date — Calculate 30-Day Start Range

If the **end date is `'2019-07-27'`**, and you want a 30-day window:


`WHERE activity_date BETWEEN DATE_SUB('2019-07-27', INTERVAL 29 DAY) AND '2019-07-27'`

## ✅ 2. You Know the Start Date — Calculate the End Date (30 days total)

Let’s say you know the **start date is `'2019-06-28'`**, and you want the next 30 days **including the start**:

`WHERE activity_date BETWEEN '2019-06-28' AND DATE_ADD('2019-06-28', INTERVAL 29 DAY)`


Remember between is inclusive so we have to do -1...


Window function 

It does the computation like group by but does not collapse the table like group by 

it takes  over()  --> over takes the `partition by` based on it the partition is done and 
order by() --> based on it sorting is performed.............

![[Pasted image 20250604170502.png]]

| Window Function  | Purpose                                                                                                                             |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `ROW_NUMBER()`   | Row num                                                                                                                             |
| `RANK()`  Ranking with gaps --> if there is two same position lets say 6, 6 then next will be 8 not 7 which is overcome by the dense_rank   6  66  66  |
| `DENSE_RANK()`   | Ranking                                                                                                                             |
| `LAG()`/`LEAD()` | Previous/                                                                                                                           |
| `SUM()`/`AVG()`  | Runni                                                                                                                               |
| `NTILE(n)`       |                                                                                                                                     |
