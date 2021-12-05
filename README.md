# Notebook-of-Exam-70-761-Querying-with-Transact-SQL

This Repository contains my notes of highlights from the book: Exam 70-761 Querying with Transact SQL (T-SQL)

### Keyed-in order:

1. SELECT
2. FROM
3. WHERE
4. GROUP BY
5. HAVING
6. ORDER BY

### Logical query processing order:

1. FROM
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT
6. ORDER BY

Example:
```
SELECT country, YEAR(hiredate) AS yearhired, COUNT(*) AS numemployees
FROM HR.Employees
WHERE hiredate >= '20140101'
GROUP BY country, YEAR(hiredate)
HAVING COUNT(*) > 1
ORDER BY country, yearhired DESC;
```
This query is issued against the HR.Employees table. It filters only employees that were hired in or after the year 2014. It groups the remaining employees by country and the hire 
year. It keeps only groups with more than one employee. For each qualifying group, the query returns the hire year and count of employees, sorted by country and hire year, in descending order.

### 2. FILTER ROWS BASED ON THE WHERE CLAUSE
```
SELECT country, YEAR(hiredate) AS yearhired
FROM HR.Employees
WHERE yearhired >= 2014;
```

If you understand that the WHERE clause is evaluated before the SELECT clause, you realize that this attempt is wrong because at this phase, the attribute yearhired doesn’t yet exist. You can indicate the expression YEAR(hiredate) >= 2014 in the WHERE clause. Better yet, for optimization reasons that are discussed later in Skill 1.3 in the section “Search arguments,” use the form hiredate >= ‘20140101’ as done in the original query.

### 4. FILTER GROUPS BASED ON THE HAVING CLAUSE

It’s important to understand the difference between WHERE and HAVING. The WHERE clause is evaluated before rows are grouped, and therefore is evaluated per row. The HAVING clause is evaluated after rows are grouped, and therefore is evaluated per group.

### 5. PROCESS THE SELECT CLAUSE

```
SELECT empid, country, YEAR(hiredate) AS yearhired, yearhired - 1 AS prevyear
FROM HR.Employees;
```
This query generates an error. The reason that this isn’t allowed is that all expressions that appear in the same logical  query-processing step are treated as a set, and a set has no order. In other words, conceptually, T-SQL evaluates all expressions that appear in the same phase in an all-at-once manner. Note the use of the word conceptually. SQL Server won’t necessarily physically process all expressions at the same point in time, but it has to produce a result as if it did. This behavior is different than many other programming languages where expressions usually get evaluated 

### 6. HANDLE PRESENTATION ORDERING

The sixth phase is applicable if the query has an ORDER BY clause. This phase is responsible for returning the result in a specific presentation order according to the expressions that appear in the ORDER BY list. The query indicates that the result rows should be ordered first by country (in ascending order by default), and then by yearhired, descending, yielding the following output:

| country | yearhired | numemployees |
|-|-|-|
| UK | 2015 | 2 |
| UK | 2014 | 2 |

### The FROM clause

If you aliased the Employees table as E, the reference Employees.empid is invalid; you have to use E.empid, as the following example demonstrates:
```
SELECT E.empid, firstname, lastname, country
FROM HR.Employees AS E;
```

*`!!!`* If you try running this code by using the full table name as the column prefix, the code 
will fail.

Renaming the aliases:
```
SELECT empid AS employeeid, firstname + N' ' + lastname AS fullname
FROM HR.Employees;
```

If duplicates are possible in the result, and you want to eliminate them in order to return a relational result, you can do so by adding a DISTINCT clause
```
SELECT DISTINCT country, region, city
FROM HR.Employees;
```

According to standard SQL, a SELECT query must have at minimum FROM and SELECT clauses. Conversely, T-SQL supports a SELECT query with only a SELECT 
clause and without a FROM clause. Such a query is as if issued against an imaginary table that has only one row. For example, the following query is invalid according to standard SQL, but is valid according to T-SQL:
```
SELECT 10 AS col1, 'ABC' AS col2;
```

The output of this query is a single row with attributes resulting from the expressions with names assigned using the aliases:

| col1 | col2 |
|-|-|
| 10 | ABC |

### Predicates and three-valued-logic

Consider the following query, which filters only employees from the US:
```
SELECT empid, firstname, lastname, country, region, city
FROM HR.Employees
WHERE country = N'USA';
```

In case you’re wondering why the literal ‘USA’ is preceded with the letter N as a prefix, that’s to denote a Unicode character string literal, because the country column is of the data type NVARCHAR. Had the country column been of a regular character string data type, such as VARCHAR, the literal should have been just ‘USA’.

### Check NULL

T-SQL supports the IS NULL and IS NOT NULL operators to check if a NULL is or isn’t present, respectively. Here’s the solution query that correctly handles NULLs:
```
SELECT empid, firstname, lastname, country, region, city
FROM HR.Employees
WHERE region <> N'WA'
 OR region IS NULL;
```
This query generates an output of rows having NULLs.

| empid | firstname | lastname | country | region | city |
|-|-|-|-|-|-|
| 5 | Sven | Mortensen | UK | NULL | London |
| 6 | Paul| Suurs| UK | NULL | London |
| 7 | Russell | King | UK | NULL | London |
| 9 | Patricia | Doyle |UK | NULL | London |

### Combining predicates

You can deal with this problem in a number of ways. A simple option is to use the *TRY_CAST* function instead of CAST. When the input expression isn’t convertible to the target 
type, TRY_CAST returns a NULL instead of failing. And comparing a NULL to anything yields unknown. Eventually, you will get the correct result, without allowing the query to fail. So your WHERE clause should be revised as follows:
```
WHERE propertytype = 'INT' AND TRY_CAST(propertyval AS INT) > 10
```

### Filtering character data

```
<column> LIKE <pattern>
```
![TABLE 1-1 Wildcards used in LIKE patterns](https://user-images.githubusercontent.com/70437668/144731358-6cdadc15-2617-42fd-b39c-0e009987c571.jpg)

```
SELECT empid, firstname, lastname
FROM HR.Employees
WHERE lastname LIKE N'D%';
```

This query returns the following output:

| empid | firstname | lastname |
|-|-|-|
| 1 | Sara | Davis |
| 9 | Patricia | Doyle |

If you want to look for a character that is considered a wildcard, you can indicate it after a character that you designate as an escape character by using the ESCAPE keyword. For example, the expression col1 ```LIKE ‘!_%’ ESCAPE ‘!’``` looks for strings that start with an underscore (_) by using an exclamation point (!) as the escape character. Alternatively, you can place the wildcard in square brackets, as in col1 ```LIKE ‘[_]%’```

### Filtering date and time data

There are two main approaches. One is to use a form that is considered language-neutral. For example, the form ‘20160212’ is always interpreted as *`ymd`*, regardless of your language. Note that the form ‘2016-02-12’ is considered language-neutral only for the data types DATE, DATETIME2, and DATETIMEOFFSET. Unfortunately, due to historic reasons, this form is considered language-dependent for the types DATETIME and SMALLDATETIME. The advantage of the form without the separators is that it is language-neutral for all date and time types. So the recommendation is to write the query as follows:
```
SELECT orderid, orderdate, empid, custid
FROM Sales.Orders
WHERE orderdate = '20160212';
```
Another approach is to explicitly convert the string to the target type using the CONVERT function, and indicating the style number that represents the style that you used. You can find the documentation of the CONVERT function with the different style numbers that it supports at https://msdn.microsoft.com/en-GB/library/ms187928.aspx. For instance, to use the U.S. style, specify style number 101, as ```CONVERT(DATE, ‘02/12/2016’, 101)```.

When filtering data stored in a DATETIME data type, you need to be very careful with ranges. To demonstrate why, first run the following code to create a table called Sales.Orders2 and populate it with a copy of the data from Sales.Orders, using the DATETIME data type for the orderdate attribute:
```
DROP TABLE IF EXISTS Sales.Orders2;
SELECT orderid, CAST(orderdate AS DATETIME) AS orderdate, empid, custid
INTO Sales.Orders2 
FROM Sales.Orders;
```

Suppose you need to filter only the orders that were placed in April 2016. You use the following query in attempt to achieve this, thinking that the DATETIME type supports a three digit precision as the fraction of the second:
```
SELECT orderid, orderdate, empid, custid
FROM Sales.Orders2
WHERE orderdate BETWEEN '20160401' AND '20160430 23:59:59.999';
```

In practice, though, the precision of DATETIME is three and a third milliseconds. Because 999 is not a multiplication of this precision, the value is rounded up to the next millisecond, which happens to be the midnight of the next day. To make matters worse, when people want to store only a date in a DATETIME type, they store the date with midnight in the time, so besides returning orders placed in April 2016, this query also returns all orders placed in May 1, 2016. Here’s the output of this query, shown here in abbreviated format:
```
orderid orderdate empid custid
----------- ----------------------- ----------- -----------
10990 2016-04-01 00:00:00.000 2 20
...
11063 2016-04-30 00:00:00.000 3 37
11064 2016-05-01 00:00:00.000 1 71
11065 2016-05-01 00:00:00.000 8 46
11066 2016-05-01 00:00:00.000 7 89
(77 row(s) affected)
```
The recommended way to express a date and time range is with a closed-open interval as 
follows:
```
SELECT orderid, orderdate, empid, custid
FROM Sales.Orders2
WHERE orderdate >= '20160401' AND orderdate < '20160501';
```

*This time the output contains only the orders placed in April 2016.*

Another tricky aspect of ordering is treatment of NULLs. Recall that a NULL represents a missing value, so when comparing a NULL to anything, you get the logical result unknown. 
That’s the case even when comparing two NULLs. So it’s not that trivial to ask how NULLs should behave in terms of sorting. Should they all sort together? If so, should they sort before or after non-NULL values? Standard SQL says that NULLs should sort together, but leaves it to the implementation to decide whether to sort them before or after non-NULL values. In SQL Server the decision was to sort them before non-NULLs (when using an ascending direction). As an example, the following query returns for each order the order ID and shipped date, 
ordered by the latter:
```
SELECT orderid, shippeddate
FROM Sales.Orders
WHERE custid = 20
ORDER BY shippeddate;
```

Remember that unshipped orders have a NULL in the shippeddate column; hence, they sort before shipped orders, as the query output shows:
```
orderid shippeddate
----------- -----------
11008 NULL
11072 NULL
10258 2014-07-23
10263 2014-07-31
```

Standard SQL supports the options NULLS FIRST and NULLS LAST to control how NULLs sort, but T-SQL doesn’t support this option. As an interesting challenge, see if you can figure 
out how to sort the orders by shipped date ascending, but have NULLs sort last. (Hint: You can specify expressions in the ORDER BY clause; think of how to use the CASE expression to achieve this task.)

### Filtering data with TOP and OFF-FETCH

#### Filtering data with TOP


