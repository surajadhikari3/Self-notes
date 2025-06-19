https://www.sql-practice.com/learn/function/window_function_basics/ --> Read the window function from here too..

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

| Function                   | Accepts Boolean/Condition? | Behavior / Notes                                            | Example                                        | Result Explanation                               |
| -------------------------- | -------------------------- | ----------------------------------------------------------- | ---------------------------------------------- | ------------------------------------------------ |
| **`SUM()`**                | ✅ Yes                      | Adds up `1` for true, `0` for false; can simulate count     | `SUM(rating < 3)`                              | Adds 1 for each row where rating < 3             |
| **`COUNT(*)`**             | ❌ No (counts all rows)     | Counts all rows, including NULLs                            | `COUNT(*)`                                     | Total row count                                  |
| **`COUNT(expr)`**          | ⚠️ _Only counts non-NULL_  | `COUNT(condition)` counts all rows unless condition is NULL | `COUNT(rating < 3)` → counts all non-NULL rows | **Not recommended** for booleans                 |
| **`COUNT(CASE WHEN ...)`** | ✅ Yes                      | Safest way to count rows matching a condition               | `COUNT(CASE WHEN rating < 3 THEN 1 END)`       | Counts only rows where condition is true         |
| **`AVG()`**                | ✅ Yes                      | Averages boolean values (1 for true, 0 for false)           | `AVG(rating < 3)`                              | Gives proportion as a decimal (e.g., 0.25 → 25%) |
| **`MIN()` / `MAX()`**      | ✅ (sort of)                | Compares boolean or conditional results                     | `MAX(rating < 3)`                              | Returns 1 if any row matches condition           |

---

## ✅ Recommended Use Summary

| Use Case                              | Preferred Expression                                        |
| ------------------------------------- | ----------------------------------------------------------- |
| Count where condition is true         | `COUNT(CASE WHEN condition THEN 1 END)`                     |
| Sum of condition matched rows         | `SUM(condition)`                                            |
| Percentage of rows matching condition | `SUM(condition) / COUNT(*) * 100` or `AVG(condition) * 100` |
| Avoid ambiguous `COUNT(condition)`    | ✅ Use `CASE WHEN` for clarity                               |

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

| Window Function                                                                                                                                   | Purpose   |
| ------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| `ROW_NUMBER()`                                                                                                                                    | Row num   |
| `RANK()`  Ranking with gaps --> if there is two same position lets say 6, 6 then next will be 8 not 7 which is overcome by the dense_rank   6 , 6 |           |
| `DENSE_RANK()`                                                                                                                                    | Ranking   |
| `LAG()`/`LEAD()`                                                                                                                                  | Previous/ |
| `SUM()`/`AVG()`                                                                                                                                   | Runni     |
| `NTILE(n)`                                                                                                                                        |           |
we can write the conditional clause with the case like below:


![[Pasted image 20250607181156.png]]

Note: Self observation..

while using the window function like sum() we give the over()

the order by tells the summing is done on the basis of that field in order.......


## ✅ Syntax: `CASE` Expression (like if / else if / else)

```sql
CASE
  WHEN condition1 THEN result1      -- if
  WHEN condition2 THEN result2      -- else if
  WHEN condition3 THEN result3      -- else if
  ...
  ELSE default_result               -- else
END
```

### 🔸 SQL:

```sql
SELECT 
  amount,
  CASE 
    WHEN amount < 100 THEN 'Low'
    WHEN amount BETWEEN 100 AND 300 THEN 'Medium'
    ELSE 'High'
  END AS category
FROM Sales;
```


Great question!

---

## ✅ What is a CTE?

**CTE** stands for **Common Table Expression**.  
It’s a **temporary result set** that you define at the beginning of a SQL query and use just like a normal table in the main query.

You write it using the `WITH` clause.

---

### 🧠 Think of it as:

> A temporary named result that helps you break a complex query into logical steps — like creating a sub-table that you can reuse once.

---

## 🔹 Syntax

```sql
WITH cte_name AS (
  SELECT ... FROM ...
)
SELECT * FROM cte_name;
```

You can even chain multiple CTEs:

```sql
WITH cte1 AS (...),
     cte2 AS (...)
SELECT ...
FROM cte2
JOIN cte1 ON ...
```

---

## 🎯 Example

### 👇 Let's say we have a `Sales` table:

|id|amount|region|
|---|---|---|
|1|100|East|
|2|200|West|
|3|150|East|

### ✅ Problem:

Find average sales per region, and show only regions above 120 average.

---

### 🧾 With CTE:(Common Table expression )

```sql
WITH avg_sales AS (
  SELECT region, AVG(amount) AS avg_amt
  FROM Sales
  GROUP BY region
)
SELECT * 
FROM avg_sales
WHERE avg_amt > 120;
```

### 🔍 Result:

|region|avg_amt|
|---|---|
|West|200|
|East|125|

---

## ✅ Why Use CTEs?

|Feature|Benefit|
|---|---|
|Readability|Breaks down complex queries|
|Reusability|Use the same logic multiple times|
|Maintainability|Easier to debug and edit|
|Recursion|Can be used for hierarchical data|

---

Note: we can use the sum (with boolean condition) --> it will give the counter value.....

We use it for finding the salary categories 

```sql
# Write your MySQL query statement below 
SELECT 'Low Salary' AS category, 
SUM(income < 20000) AS accounts_count 
FROM Accounts 

UNION 

SELECT 'Average Salary' AS category, 
SUM(income BETWEEN 20000 AND 50000 ) AS accounts_count 
FROM Accounts 

UNION 

SELECT 'High Salary' AS category, 
SUM(income > 50000) AS accounts_count FROM Accounts;
```



We can also use the condition with the if clause even in order by like below 

```sql
select row_number() over() id, student
from seat
order by if(mod(id,2) = 0, id-1, id+1);
```

For doing the movie rating consider this solution too


```sql
# Write your MySQL query statement below 
(SELECT name AS results
FROM MovieRating JOIN Users USING(user_id)
GROUP BY name 
ORDER BY COUNT(*) DESC,name 
LIMIT 1) 

UNION ALL 

(SELECT title AS results 
FROM MovieRating JOIN Movies USING(movie_id) 
WHERE EXTRACT(YEAR_MONTH FROM created_at) = 202002 
GROUP BY title 
ORDER BY AVG(rating) DESC, title 
LIMIT 1);
```

Note: 
	While using the frame clause in the window function use the range between instead of the row between for the date based cases:
	
```sql
SUM(amount) OVER (
  ORDER BY visited_on 
  RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW
)

```

instead of using the below :

```sql
SUM(amount) OVER (
  ORDER BY visited_on 
  ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
)

```

Note: 
If you want to find the unique element group by that field and check the having count(*) =1;
or
if u want to find the duplicate element group by which ever field u want and do having count(*) > 1;

It is used in one of the sql 50 problem --> Investment in 2016...

```sql
```sql
SELECT ROUND(SUM(tiv_2016), 2) AS tiv_2016
FROM Insurance
WHERE tiv_2015 IN (
    SELECT tiv_2015
    FROM Insurance
    GROUP BY tiv_2015
    HAVING COUNT(*) > 1
)
AND (lat, lon) IN (
    SELECT lat, lon
    FROM Insurance
    GROUP BY lat, lon
    HAVING COUNT(*) = 1
)
```


Substring in the sql 

`SUBSTRING(string, start, length)`

here we give the string the start position and how many character we want and it starts from 1 not zero..


Regexp 

if there is . -> that means there will be zero or more occurence of the token before it 

eg -> ab*c --> means that b will occur 0 or any time (eg.. ac, abc, abbc)

if there is + means there is one or more occurence of it

.* --> mean it is wildcard any number of element can occur between

for case- sensetive regexp check 

```sql
SELECT *

FROM users

WHERE REGEXP_LIKE(mail, '^[a-zA-Z][a-zA-Z0-9_.-]*@leetcode\\.com$', 'c');
```


---

## ✅ 1. `.` → **Dot = any single character**

- Matches **exactly one character** (letter, digit, symbol, etc.)
    
- Does **not** match newline (`\n`) in MySQL by default
    

### 🔸 Example:

```sql
email REGEXP '^a.b@gmail\\.com$'
```

✅ Matches: `a1b@gmail.com`, `axb@gmail.com`  
❌ Doesn't match: `ab@gmail.com` (missing one char in the middle)

---

## ✅ 2. `*` → **Zero or more** of the **preceding token**

- The `*` repeats the character or group **before it**
    

### 🔸 Example:

```sql
email REGEXP 'ab*c'
```

✅ Matches:

- `ac` (`b` occurs 0 times)
    
- `abc`, `abbc`, `abbbc` (1 or more `b`s)
    

---

## ✅ 3. `+` → **One or more** of the **preceding token**

- Like `*`, but requires **at least 1 occurrence**
    

### 🔸 Example:

```sql
email REGEXP 'ab+c'
```

✅ Matches:

- `abc`, `abbc`, `abbbc`
    

❌ Doesn’t match:

- `ac` (because there's no `b` at all)
    

---

## ✅ 4. `.*` → **Zero or more of any character**

- Most common for **wildcard-like matching**
    

### 🔸 Example:

```sql
email REGEXP '^admin.*@gmail\\.com$'
```

✅ Matches:

- `admin@gmail.com`
    
- `administrator@gmail.com`
    
- `admin12345@gmail.com`
    

🔍 Equivalent to saying:  
"Starts with `admin`, followed by anything (including nothing), and ends with `@gmail.com`"

---

## ✅ 5. `\\` → **Escape special characters** in MySQL strings

- MySQL strings treat `\` as an escape character
    
- So you **must double it** to get a literal `\` into the regex
    

### 🔸 Example:

```sql
email REGEXP '\\.'  → matches a literal `.`
```

✅ In MySQL, to match `@gmail.com` you must write:

```sql
'@gmail\\.com$'
```

Why?

- `.` by default means “any character”
    
- So to match a **literal dot**, you need `\\.`
    

---

## 🔍 Summary Table

| Symbol | Meaning                                 | Example             | Matches                      |
| ------ | --------------------------------------- | ------------------- | ---------------------------- |
| `.`    | Any one character                       | `a.b`               | `a1b`, `a_b`                 |
| `*`    | Zero or more of the preceding token     | `ab*c`              | `ac`, `abc`, `abbc`          |
| `+`    | One or more of the preceding token      | `ab+c`              | `abc`, `abbbc`               |
| `.*`   | Any number of any characters (wildcard) | `admin.*@gmail.com` | `admin123@gmail.com`         |
| `\\`   | Escape for special characters in regex  | `@gmail\\.com`      | Matches literal `@gmail.com` |

---

## ✅ Practical Email Match

```sql
SELECT *
FROM users
WHERE REGEXP_LIKE(email, '^[a-z0-9._-]+@gmail\\.com$', 'c');  -- 'c' = case sensitive in MySQL 8+
```

