# Notebook-of-Exam-70-761-Querying-with-Transact-SQL

This Repository contains my notes of highlights (not all content) from the book: Exam 70-761 Querying with Transact SQL (T-SQL)

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

```
SELECT TOP (3) orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC;
```

Specify the number of rows you want to filter:
```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11077 2016-05-06 65 1
11076 2016-05-06 9 4
11075 2016-05-06 68 8
```
*`EXAM TIP`*

T-SQL supports specifying the number of rows to filter using the TOP option in SELECT  queries without parentheses, but that’s only for backward-compatibility reasons. The correct syntax is with parentheses.

Specify a percent of rows to filter instead of a number:

```
SELECT TOP (1) PERCENT orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC;
```

```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11074 2016-05-06 73 7
11075 2016-05-06 68 8
11076 2016-05-06 9 4
11077 2016-05-06 65 1
11070 2016-05-05 44 2
11071 2016-05-05 46 1
11072 2016-05-05 20 4
11073 2016-05-05 58 2
11067 2016-05-04 17 1
```

Specifiy a self-contained expression:
```
SELECT TOP (1) PERCENT orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC;
```

```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11074 2016-05-06 73 7
11075 2016-05-06 68 8
11076 2016-05-06 9 4
11077 2016-05-06 65 1
11070 2016-05-05 44 2
```

But there’s no guarantee that the same rows will be returned if you run the query again. If you are really after three arbitrary rows, it might be a good idea to add an ORDER BY clause with the expression (SELECT NULL) to let people know that your choice is intentional and not 
an oversight. Here’s how your query would look:
```
SELECT TOP (3) orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY (SELECT NULL);
```

Note that even when you do have an ORDER BY clause, in order for the query to be completely deterministic, the ordering must be unique. For example, consider again the first 
query from this section:
```
SELECT TOP (3) orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC;
```
```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11077 2016-05-06 65 1
11076 2016-05-06 9 4
11075 2016-05-06 68 8
```

But what if there are other rows in the result without TOP that have the same order date as in the last row here? You don’t always care about guaranteeing deterministic or repeatable results; but if you do, two options are available to you. One option is to ask to include all ties with the last row by adding the WITH TIES option, as follows:
```
SELECT TOP (3) WITH TIES orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC;
```

Of course, this could result in returning more rows than you asked for, as the output of this query shows:
```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11074 2016-05-06 73 7
11075 2016-05-06 68 8
11076 2016-05-06 9 4
11077 2016-05-06 65 1
```

Now the selection of rows is deterministic, but still the presentation order between rows with the same order date isn’t. The other option to guarantee determinism is to break the ties by adding a tiebreaker that makes the ordering unique. For example, in case of ties in the order date, suppose you wanted to use the order ID, descending, as the tiebreaker. To do so, add orderid DESC to your ORDER BY clause, as follows:
```
SELECT TOP (3) orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC, orderid DESC;
```
Now both the selection of rows and presentation order are deterministic. This query generates the following output:
```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11077 2016-05-06 65 1
11076 2016-05-06 9 4
11075 2016-05-06 68 8
```

### Filtering data with OFFSET-FETCH

Unlike TOP, it is standard, and also has a skipping capability, making it useful for ad-hoc paging purposes.

- Right after the ORDER BY clause (require ORDER BY)
- OFFSET: indicating how many rows you want to skip (0 if you don’t want to skip any); 
- FETHC: then optionally indicating how many rows you want to filter.

Define ordering based on order date descending, followed by order ID descending; it then skips the first 50 rows and fetches the next 25 rows:
```
SELECT orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC, orderid DESC
OFFSET 50 ROWS FETCH NEXT 25 ROWS ONLY;
```
Here’s an abbreviated form of the output:
```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11027 2016-04-16 10 1
11026 2016-04-15 27 4
...
11004 2016-04-07 50 3
11003 2016-04-06 78 3
```

In T-SQL, the OFFSET-FETCH option requires an ORDER BY clause to be present. Also, in T-SQL—contrary to standard SQL—a FETCH clause requires an OFFSET clause to be present. So if you do want to filter some rows but skip none, you still need to specify the OFFSET clause with 0 ROWS.

In order to make the syntax intuitive, you can use the keywords NEXT or FIRST interchangeably. When skipping some rows, it might be more intuitive to you to use the keywords 
FETCH NEXT to indicate how many rows to filter; but when not skipping any rows, it might be more intuitive to you to use the keywords FETCH FIRST, as follows:
```
SELECT orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC, orderid DESC
OFFSET 0 ROWS FETCH FIRST 25 ROWS ONLY
```

In T-SQL, a FETCH clause requires an OFFSET clause, but the OFFSET clause doesn’t require 
a FETCH clause.  

Skip 50 rows, returning all the rest
```
SELECT orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC, orderid DESC
OFFSET 50 ROWS;
```
This query generates the following output, shown here in abbreviated form:
```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11027 2016-04-16 10 1
11026 2016-04-15 27 4
...
10249 2014-07-05 79 6
10248 2014-07-04 85 5
(780 row(s) affected)
```

But what if you need to filter a certain number of rows based on arbitrary order? To do so, you can specify the expression (SELECT NULL) in the ORDER BY clause, as follows:
```
SELECT orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY (SELECT NULL)
OFFSET 0 ROWS FETCH FIRST 3 ROWS ONLY;
```

This code simply filters three arbitrary rows. Here’s the output that I got when running this query on our system:
```
orderid orderdate custid empid
----------- ---------- ----------- -----------
10248 2014-07-04 85 5
10249 2014-07-05 79 6
10250 2014-07-08 34 4
```

With both the OFFSET and the FETCH clauses, you can use expressions as inputs. This is very handy when you need to compute the input values dynamically. For example, suppose 
that you’re implementing a paging solution where you return to the user one page of rows at a time. The user passes as input parameters to your procedure or function the page number they are after (@pagenum parameter) and page size (@pagesize parameter). This means that you need to skip as many rows as @pagenum minus one times @pagesize, and fetch the next @pagesize rows. This can be implemented using the following code (using local variables for simplicity):
```
DECLARE @pagesize AS BIGINT = 25, @pagenum AS BIGINT = 3;
SELECT orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC, orderid DESC
OFFSET (@pagenum - 1) * @pagesize ROWS FETCH NEXT @pagesize ROWS ONLY;
```
With these inputs, the code returns the following output
```
orderid orderdate custid empid
----------- ---------- ----------- -----------
11027 2016-04-16 10 1
11026 2016-04-15 27 4
...
11004 2016-04-07 50 3
11003 2016-04-06 78 3
```

*`EXAM TIP`*
In terms of logical query processing, the TOP and OFFSET-FETCH filters are processed after the FROM, WHERE, GROUP, HAVING and SELECT phases. You can consider these filters as being an extension to the ORDER BY clause. So, for example, if the query is a grouped query, and also involves a TOP or OFFSET-FETCH filter, the filter is applied after grouping. The same applies if the query has a DISTINCT clause and/or ROW_NUMBER calculation as part of the SELECT clause, as well as a TOP or OFFSET-FETCH filter. The filter is applied after the DISTINCT clause and/or ROW_NUMBER calculation.

### ON TOP AND OFFSET-FETCH

For more information on the TOP and OFFSET-FETCH filters, see the free sample chapter of the book, “T-SQL Querying: Chapter 5 - TOP and OFFSET-FETCH.” This book is more advanced, and includes detailed coverage of optimization aspects. 

## Combining sets with set operators

#### UNION and UNION ALL 

The ```UNION``` operator unifies the results of the two input queries. As a set operator, UNION has an implied DISTINCT property, meaning that it does not return duplicate rows.

The ```UNION ALL``` operator unifies the results of the two input queries, but doesn’t try to eliminate duplicates.

The ```INTERSECT``` operator returns only distinct rows that are common to both sets. 

The ```EXCEPT``` operator performs set difference. 

With UNION and INTERSECT, the order of the input queries doesn’t matter. However, with EXCEPT, there’s different meaning to:
```
<query 1> EXCEPT <query 2>
```
Versus:
```
<query 2> EXCEPT <query 1
```

Set operators have precedence: INTERSECT precedes UNION and EXCEPT, and UNION and EXCEPT are evaluated from left to right based on their position in the expression. 

Consider the following set operators:
```
<query 1> UNION <query 2> INTERSECT <query 3>;
```

First, the intersection between query 2 and query 3 takes place, and then a union between the result of the intersection and query 1. You can always force precedence by using parentheses. So, if you want the union to take place first, you use the following form:
```
(<query 1> UNION <query 2>) INTERSECT <query 3>;
```

When you’re done, run the following code for cleanup:
```
DROP TABLE IF EXISTS Sales.Orders2
```

### JOINS

Add a row to the Suppliers table:
```
USE TSQLV4;
INSERT INTO Production.Suppliers
 (companyname, contactname, contacttitle, address, city, postalcode, country, phone)
 VALUES(N'Supplier XYZ', N'Jiru', N'Head of Security', N'42 Sekimai Musashino-shi',
 N'Tokyo', N'01759', N'Japan', N'(02) 4311-2609');
```

```CROSS JOIN ``` performs a multiplication between the tables, yielding a row for each combination of rows from both sides. If you have m rows in table T1 and n rows in table T2, the result of a cross join between T1 and T2 is a virtual table with m × n rows. Figure 1-6 provides an illustration of a cross join.

Consider an example from the TSQLV4 sample database. This database contains a table called dbo. Nums that has a column called n with a sequence of integers from 1 and on. Your 
task is to use the Nums table to generate a result with a row for each weekday (1 through 7) and shift number (1 through 3), assuming there are three shifts a day. The result can later be used as the basis for building information about activities in the different shifts in the different days. With seven days in the week, and three shifts every day, the result should have 21 rows.

Here’s a query that achieves the task by performing a cross join between two instances of the Nums table—one representing the days (aliased as D), and the other representing the 
shifts (aliased as S):
```
USE TSQLV4;
SELECT D.n AS theday, S.n AS shiftno 
FROM dbo.Nums AS D
 CROSS JOIN dbo.Nums AS S
WHERE D.n <= 7
 AND S.N <= 3
ORDER BY theday, shiftno;
```

Here’s the output of this query:
```
theday shiftno
----------- -----------
1 1
1 2
1 3
2 1
2 2
2 3
3 1
3 2
3 3
4 1
4 2
4 3
5 1
5 2
5 3
6 1
6 2
6 3
7 1
7 2
7 3
```

With an ```INNER JOIN``` you can match rows from two tables based on a predicate—usually one that compares a primary key value in one side to a foreign key value in another side. 

*`NOTE`*  Because SQL Server doesn’t create such indexes automatically, it’s your responsibility to identify the cases where they can be useful and create them. So when working on index tuning, one interesting area to examine is foreign key columns, and evaluating the benefits of creating indexes on those.

In terms of logical query processing, the WHERE is evaluated right after the FROM, so conceptually it is equivalent to concatenating the predicates with an AND operator, forming a conjunction of predicates. SQL Server knows this, and therefore can internally rearrange the order in which it evaluates the predicates in practice, and it does so 
based on cost estimates. For these reasons, if you wanted, you could rearrange the placement of the predicates from the previous query, *`specifying both ON and WHERE in the ON clause` (in OUTER JOIN, it is not like that)*, and still retain the original meaning, as follows:
```
SELECT
 S.companyname AS supplier, S.country,
 P.productid, P.productname, P.unitprice
FROM Production.Suppliers AS S
 INNER JOIN Production.Products AS P
 ON S.supplierid = P.supplierid
 AND S.country = N'Japan'
```
With ```OUTER JOIN```, you can request to preserve all rows from one or both sides of the join, never mind if there are matching rows in the other side based on the ON predicate.

### LEFT JOIN

Unlike in the inner join, the left row with the key A is returned even though it has no match in the right side. It’s returned with NULLs as placeholders in the right side.

```
SELECT
 S.companyname AS supplier, S.country,
 P.productid, P.productname, P.unitprice
FROM Production.Suppliers AS S
 LEFT OUTER JOIN Production.Products AS P
 ON S.supplierid = P.supplierid
WHERE S.country = N'Japan';
```
This query generates the following output:
```
supplier country productid productname unitprice
--------------- -------- ----------- -------------- ----------
Supplier QOVFD Japan 9 Product AOZBW 97.00
Supplier QOVFD Japan 10 Product YHXGE 31.00
Supplier QOVFD Japan 74 Product BKAZJ 10.00
Supplier QWUSF Japan 13 Product POXFU 6.00
Supplier QWUSF Japan 14 Product PWCJB 23.25
Supplier QWUSF Japan 15 Product KSZOI 15.50
Supplier XYZ Japan NULL NULL NULL
```

It is very important to understand that, with outer joins, the ON and WHERE clauses play very different roles, and therefore, they aren’t interchangeable. The WHERE clause still plays a simple filtering role—namely, it keeps true cases and discards false and unknown cases. In our query, the WHERE clause filters only suppliers from Japan, so suppliers that aren’t from Japan simply don’t show up in the output.

However, the ON clause doesn’t play a simple filtering role; rather, it’s a more sophisticated matching role. In other words, a row in the preserved side will be returned whether the ON predicate finds a match for it or not. So the ON predicate only determines which rows from the nonpreserved side get matched to rows from the preserved side—not whether to return the rows from the preserved side. In our query, the ON clause matches rows from both sides by comparing their supplier ID values. Because it’s a matching predicate (as opposed to a filter), the join won’t discard suppliers; instead, it only determines which products get matched to each supplier. But even if a supplier has no matches based on the ON predicate, the supplier is still returned. In other words, ON is not final with respect to the preserved side of the join. WHERE is final. So when in doubt, whether to specify the predicate in the ON or WHERE clauses, ask yourself: *`Is the predicate used to filter or match? Is it supposed to be final or nonfinal?`*

Compare the above and below queries:
```
SELECT
 S.companyname AS supplier, S.country,
 P.productid, P.productname, P.unitprice
FROM Production.Suppliers AS S
 LEFT OUTER JOIN Production.Products AS P
 ON S.supplierid = P.supplierid
 AND S.country = N'Japan';
```

Observe what’s different in the result (shown here in abbreviated form) and see if you can explain in your own words what the query returns now:
```
supplier country productid productname unitprice
--------------- -------- ---------- -------------- ----------
Supplier SWRXU UK NULL NULL NULL
Supplier VHQZD USA NULL NULL NULL
Supplier STUAZ USA NULL NULL NULL
Supplier QOVFD Japan 9 Product AOZBW 97.00
Supplier QOVFD Japan 10 Product YHXGE 31.00
Supplier QOVFD Japan 74 Product BKAZJ 10.00
Supplier EQPNC Spain NULL NULL NULL
Supplier QWUSF Japan 13 Product POXFU 6.00
Supplier QWUSF Japan 14 Product PWCJB 23.25
Supplier QWUSF Japan 15 Product KSZOI 15.50
...
(34 row(s) affected)

```
Now that both predicates appear in the ON clause, both serve a matching purpose. What this means is that all suppliers are returned—even those that aren’t from Japan. But in order to match a product to a supplier, the supplier IDs in both sides need to match, and the supplier country needs to be Japan.

A ```FULL OUTER JOIN``` returns the matched rows, which are normally returned from an inner join; plus rows from the left that don’t have matches in the right, with NULLs used as placeholders in the right side; plus rows from the right that don’t have matches in the left, with NULLs used as placeholders in the left side. It’s not common to need a full outer join because most relationships between tables allow only one of the sides to have rows that don’t have matches in the other, in which case, a one-sided outer join is needed.

### Queries with composite JOINs and NULLs in JOIN columns

Some joins can be a bit tricky to handle, for instance when the join columns can have NULLs, or when you have multiple join columns—what’s known as a composite join. 

When you need to join tables that are related based on multiple columns, the join is called a composite join and the ON clause typically consists of a conjunction of predicates (predicates separated by AND operators) that match the corresponding columns from the two sides. Sometimes you need more complex predicates, especially when NULLs are involved. I’ll demonstrate this by using a pair of tables. One table is called EmpLocations and it holds employee locations and the number of employees in each location. Another table is called CustLocations and it holds customer locations and the number of customers in each location. 

Run the following code to create these tables and populate them with sample data:

```
DROP TABLE IF EXISTS dbo.EmpLocations;
SELECT country, region, city, COUNT(*) AS numemps
INTO dbo.EmpLocations
FROM HR.Employees
GROUP BY country, region, city;
ALTER TABLE dbo.EmpLocations ADD CONSTRAINT UNQ_EmpLocations
 UNIQUE CLUSTERED(country, region, city);
 
DROP TABLE IF EXISTS dbo.CustLocations;
SELECT country, region, city, COUNT(*) AS numcusts
INTO dbo.CustLocations
FROM Sales.Customers
GROUP BY country, region, city;
ALTER TABLE dbo.CustLocations ADD CONSTRAINT UNQ_CustLocations
 UNIQUE CLUSTERED(country, region, city);
```

There’s a key defined in both tables on the location attributes: country, region, and city. Instead of using a primary key constraint I used a unique constraint to enforce the key because the region attribute allows NULLs, and between the two types of constraints, only the latter allows NULLs. I also specified the CLUSTERED keyword in the unique constraint definitions to have SQL Server create a clustered index type to enforce the constraint’s uniqueness property. This index will be beneficial in supporting joins between the tables based on the location attributes as well filters based on those attributes.

```
SELECT country, region, city, numemps
FROM dbo.EmpLocations;
```
```
country region city numemps
--------------- --------------- --------------- -----------
UK NULL London 4
USA WA Kirkland 1
USA WA Redmond 1
USA WA Seattle 2
USA WA Tacoma 1
```
```
SELECT country, region, city, numcusts
FROM dbo.CustLocations;
```
```
country region city numcusts
--------------- --------------- --------------- -----------
Argentina NULL Buenos Aires 3
Austria NULL Graz 1
Austria NULL Salzburg 1
Belgium NULL Bruxelles 1
Belgium NULL Charleroi 1
Brazil RJ Rio de Janeiro 3
Brazil SP Campinas 1
Brazil SP Resende 1
Brazil SP Sao Paulo 4
Canada BC Tsawassen 1
...
(69 row(s) affected)
```

Join the two tables returning only matched locations, with both the employee and customer counts returned along with the location attributes. Your first attempt might be to write a composite join with an ON clause that has a conjunction of simple equality predicates as follows:
```
SELECT EL.country, EL.region, EL.city, EL.numemps, CL.numcusts
FROM dbo.EmpLocations AS EL
 INNER JOIN dbo.CustLocations AS CL
 ON EL.country = CL.country
 AND EL.region = CL.region
 AND EL.city = CL.city;
```
```
country region city numemps numcusts
--------------- --------------- --------------- ----------- -----------
USA WA Kirkland 1 1
USA WA Seattle 2 1
```

The problem is that the region column supports NULLs representing cases where the region is irrelevant (missing but inapplicable) and when you compare NULLs with an equalitybased predicate the result is the logical value unknown, in which case the row is discarded. For instance, the location UK, NULL, London appears in both tables, and therefore you expect to see it in the result of the join, but you don’t. A common way for people to resolve this problem is to use the ISNULL or COALESCE functions to substitute a NULL in both sides with a value that can’t normally appear in the data, and this way when both sides are NULL you get a true back from the comparison. Here’s an example for implementing this solution using the ISNULL function:
```
SELECT EL.country, EL.region, EL.city, EL.numemps, CL.numcusts
FROM dbo.EmpLocations AS EL
 INNER JOIN dbo.CustLocations AS CL
 ON EL.country = CL.country
 AND ISNULL(EL.region, N'<N/A>') = ISNULL(CL.region, N'<N/A>')
 AND EL.city = CL.city;
```
```
country region city numemps numcusts
--------------- --------------- --------------- ----------- -----------
UK NULL London 4 6
USA WA Kirkland 1 1
USA WA Seattle 2 1
```

You can handle NULLs in a manner that gives you the desired logical meaning and that at the same time is considered order preserving by the optimizer using the predicate: (EL.region = CL.region OR (EL.region IS NULL AND CL.region IS NULL)). Here’s the complete solution 
query: (p. 62)
```
SELECT EL.country, EL.region, EL.city, EL.numemps, CL.numcusts
FROM dbo.EmpLocations AS EL
 INNER JOIN dbo.CustLocations AS CL
 ON EL.country = CL.country
 AND (EL.region = CL.region OR (EL.region IS NULL AND CL.region IS NULL))
 AND EL.city = CL.city;
```

The optimizer can rely on index order when using the merge join algorithm. To demonstrate this, again, force this algorithm by adding the MERGE join hint as follows: (p. 63)
```
SELECT EL.country, EL.region, EL.city, EL.numemps, CL.numcusts
FROM dbo.EmpLocations AS EL
 INNER MERGE JOIN dbo.CustLocations AS CL
 ON EL.country = CL.country
 AND (EL.region = CL.region OR (EL.region IS NULL AND CL.region IS NULL))
 AND EL.city = CL.city;
```

Recall that when set operators combine query results they compare corresponding attributes using distinctness and not equality, producing true when comparing two NULLs. However, one drawback that set operators have is that they compare complete rows. Unlike joins, which allow comparing a subset of the attributes and return additional ones in the result, set operators must compare all attributes from the two input queries. But in T-SQL, you can combine joins and set operators to benefit from the advantages of both tools. Namely, rely on the distinctness-based comparison of set operators and the ability of joins to return additional attributes beyond what you compare. In our querying task, the solution looks like this:
```
SELECT EL.country, EL.region, EL.city, EL.numemps, CL.numcusts
FROM dbo.EmpLocations AS EL
 INNER JOIN dbo.CustLocations AS CL
 ON EXISTS (SELECT EL.country, EL.region, EL.city
 INTERSECT 
 SELECT CL.country, CL.region, CL.city);
```

For each row that is evaluated by the join, the set operator performs an intersection of the employee location attributes and customer location attributes using FROM-less SELECT 
statements, each producing one row. If the locations intersect, the result is one row, in which case the EXISTS predicate returns true, and the evaluated row is considered a match. If the locations don’t intersect, the result is an empty set, in which case the EXISTS predicate returns false, and the evaluated row is not considered a match. Remarkably, Microsoft added logic to the optimizer to consider this form order-preserving. The plan for this query is the same as 
the one shown earlier in Figure 1-13.Use the following code to force the merge algorithm in the query:
```
SELECT EL.country, EL.region, EL.city, EL.numemps, CL.numcusts
FROM dbo.EmpLocations AS EL
 INNER MERGE JOIN dbo.CustLocations AS CL
 ON EXISTS (SELECT EL.country, EL.region, EL.city
           INTERSECT 
           SELECT CL.country, CL.region, CL.city);
```

Also here the ordering property of the data is preserved and you get the plan shown earlier in Figure 1-14, where the clustered indexes of both sides are scanned in order and there’s no need for explicit sorting prior to performing the merge join. When you’re done, run the following code for cleanup:
```
DROP TABLE IF EXISTS dbo.CustLocations;
DROP TABLE IF EXISTS dbo.EmpLocations;
```

### Multi-join queries

```
SELECT
 S.companyname AS supplier, S.country,
 P.productid, P.productname, P.unitprice,
 C.categoryname
FROM Production.Suppliers AS S
 LEFT OUTER JOIN Production.Products AS P
 ON S.supplierid = P.supplierid
 INNER JOIN Production.Categories AS C
 ON C.categoryid = P.categoryid
WHERE S.country = N'Japan';
```

```
supplier country productid productname unitprice categoryname
--------------- ------- ---------- -------------- ---------- -------------
Supplier QOVFD Japan 9 Product AOZBW 97.00 Meat/Poultry
Supplier QOVFD Japan 10 Product YHXGE 31.00 Seafood
Supplier QWUSF Japan 13 Product POXFU 6.00 Seafood
Supplier QWUSF Japan 14 Product PWCJB 23.25 Produce
Supplier QWUSF Japan 15 Product KSZOI 15.50 Condiments
Supplier QOVFD Japan 74 Product BKAZJ 10.00 Produce
```

Supplier XYZ from Japan was discarded. Can you explain why?
Conceptually, the first join included outer rows (suppliers with no products) but produced NULLs as placeholders in the product attributes in those rows. Then the join to Production.Categories compared the NULLs in the categoryid column in the outer rows to categoryid values in Production.Categories, and discarded those rows. In short, the inner join that followed the outer join nullified the outer part of the join. In fact, if you look at the query plan for this query, you will find that the optimizer didn’t even bother to process the join between Production.Suppliers and Production.Products as an outer join. It detected the contradiction between the outer join and the subsequent inner join, and converted the first join to an inner join too.

There are a number of ways to address this problem. One is to use a ```LEFT OUTER in both joins```, like so:
```
SELECT
 S.companyname AS supplier, S.country,
 P.productid, P.productname, P.unitprice,
 C.categoryname
FROM Production.Suppliers AS S
 LEFT OUTER JOIN Production.Products AS P
 ON S.supplierid = P.supplierid
 LEFT OUTER JOIN Production.Categories AS C
 ON C.categoryid = P.categoryid
WHERE S.country = N'Japan';
```
Another option is to use an interesting capability in the language—separate some of the joins to their own independent logical phase. What you’re after is a left outer join between Production.Suppliers and the result of the inner join between Production.Products and Production.Categories. You can phrase your query exactly like this:
```
SELECT
 S.companyname AS supplier, S.country,
 P.productid, P.productname, P.unitprice,
 C.categoryname
FROM Production.Suppliers AS S
 LEFT OUTER JOIN 
 (Production.Products AS P
 INNER JOIN Production.Categories AS C
 ON C.categoryid = P.categoryid)
 ON S.supplierid = P.supplierid
WHERE S.country = N'Japan';
```

With both fixes, the query generates the correct result, including suppliers who currently don’t have related products:
```
supplier country productid productname unitprice categoryname
--------------- -------- ---------- -------------- ---------- -------------
Supplier QOVFD Japan 9 Product AOZBW 97.00 Meat/Poultry
Supplier QOVFD Japan 10 Product YHXGE 31.00 Seafood
Supplier QOVFD Japan 74 Product BKAZJ 10.00 Produce
Supplier QWUSF Japan 13 Product POXFU 6.00 Seafood
Supplier QWUSF Japan 14 Product PWCJB 23.25 Produce
Supplier QWUSF Japan 15 Product KSZOI 15.50 Condiments
Supplier XYZ Japan NULL NULL NULL NULL
```

Note that the important change that made the difference is the arrangement of the ON clauses with respect to the joined tables. The ON clause ordering is what defines the logical join ordering. Each ON clause must appear right below the two units that it joins. By specifying the ON clause that matches attributes from Production.Products and Production.Categories first, you set this inner join to be logically evaluated first. Then the second ON clause handles the left outer join by matching an attribute from Production.Suppliers with an attribute from the result of the inner join. Curiously, T-SQL doesn’t really require the parentheses that I added to the query; remove those and rerun the query, and you will see that it runs successfully. However, it’s recommended to use those for clarity.

*`EXAM TIP`*

Multi join queries that mix different join types are very common in practice and therefore there’s a high likelihood that questions about those will show up in the exam. Make sure that you understand the pitfalls in mixing join types, especially when an outer join is subsequently followed by an inner join, which discards the outer rows that were produced by the outer join. When you’re done, run the following code to delete the supplier row that you added at the beginning of this skill:

```
DELETE FROM Production.Suppliers WHERE supplierid > 29
```

## 1.3 Implement functions & aggregate data

### Type conversion functions 

The two fundamental functions that T-SQL supports for conversion purposes are CAST and CONVERT. 

The CAST function’s syntax is ```CAST(source_expression AS target_type```. For example, ```CAST(‘100’ AS INT)``` converts the source character string constant to the target integer value 100. 

The CONVERT function is handy when you need to specify a style for the conversion. Its syntax is ```CONVERT(target_type, source_expression [, style_number])```. You can find the supported style numbers and their meaning at https://msdn.microsoft.com/en-us/library/ms187928.aspx. For instance, when converting a character string to a date and time type or the other way around, you can specify the style number to avoid ambiguity in case the form you use is considered language dependent. As an example, the expression ```CONVERT(DATE, ‘01/02/2017’, 101)``` converts the input string to a date using the U.S. style, meaning January 2, 2017. The expression ```CONVERT(DATE, ‘01/02/2017’, 103)``` uses the British/French style, meaning February 1, 2017.

The PARSE function is an alternative to CONVERT when you want to parse a character string input to a target type, but instead of using cryptic style numbers, it uses a more user-friendly .NET culture name. For instance, the expression ```PARSE(‘01/02/2017’ AS DATE USING ‘en-US’)``` uses the English US culture, parsing the input as a date meaning January 2, 2017. The expression ```PARSE(‘01/02/2017’ AS DATE USING ‘en-GB’)``` uses the English Great Britain culture, parsing the input as a date meaning February 1, 2017. Note though that this function is significantly slower than CONVERT, so I personally stay away from using it. One of the problems with CAST, CONVERT, and PARSE is that if the function fails to convert a value within a query, the whole query fails and stops processing. As an alternative to these functions, T-SQL supports the TRY_CAST, TRY_CONVERT, and TRY_PARSE functions, which behave the same as their counterparts when the conversion is valid, but return a NULL instead of failing when the conversion isn’t valid. As an example, run the following code to try and convert two strings to dates using the CONVERT function:
```
SELECT CONVERT(DATE, '14/02/2017', 101) AS col1, 
 CONVERT(DATE, '02/14/2017', 101) AS col2;
```
The first value doesn’t convert successfully and therefore this code fails with the following error:
```
Msg 241, Level 16, State 1, Line 26
```
Conversion failed when converting date and/or time from character string. Use the TRY_CONVERT function instead of CONVERT, like so:
```
SELECT TRY_CONVERT(DATE, '14/02/2017', 101) AS col1, 
 TRY_CONVERT(DATE, '02/14/2017', 101) AS col2
```

This time the code doesn’t fail, but the first expression returns a NULL, as the following output shows:
```
col1 col2
---------- ----------
NULL 2017-02-14
```

Lastly, the FORMAT function is an alternative to the CONVERT function when you want to format an input expression of some type as a character string, but instead of using cryptic style numbers, you specify a .NET format string and culture, if relevant. For instance, to format an input date and time value, such as now, as a character string using the form ‘yyyy-MM-dd’, use the expression: ```FORMAT(SYSDATETIME(), ‘yyyy-MM-dd’)```. You can use any format string supported by the .NET Framework. (For details, see the topics “FORMAT (Transact-SQL)” and “Formatting Types in the .NET Framework” at https://msdn.microsoft.com/en-us/library/hh213505.aspx and http://msdn.microsoft.com/en-us/library/26etazsy.aspx.). Note that like PARSE, the FORMAT function is also quite slow, so when you need to format a large number of values in a query, you typically get much better performance with alternative built-in functions.

### Date and time functions

T-SQL supports a number of date and time functions that allow you to manipulate your date and time data. This section covers some of the important functions supported by T-SQL and provides some examples. For the full list, as well as the technical details and syntax, see the T-SQL documentation for the topic at https://msdn.microsoft.com/en-us/library/ms186724.aspx. 

#### Current date and time 

One important category of functions is the category that returns the current date and time. The functions in this category are GETDATE, CURRENT_TIMESTAMP, GETUTCDATE, SYSDATETIME, SYSUTCDATETIME, and SYSDATETIMEOFFSET. 

```GETDATE``` is T-SQL–specific, returning the current date and time in the SQL Server instance you’re connected to as a DATETIME data type. 

```CURRENT_TIMESTAMP``` is the same, only it’s standard, and hence the recommended one to use. 

```SYSDATETIME``` and ```SYSDATETIMEOFFSET``` are similar, only returning the values as the more precise ```DATETIME2``` and ```DATETIMEOFFSET``` types (including the time zone offset from UTC), respectively. 

**Note that there are no built-in functions to return the current date and the current time. To get such information, simply cast the SYSDATETIME function to DATE or TIME, respectively.**For example, to get the current date, use C```AST(SYSDATETIME() AS DATE)```. The ```GETUTCDATE``` function returns the current date and time in UTC terms as a DATETIME type, and ```SYSUTCDATETIME``` does the same, only returning the result as the more precise DATETIME2 type.

#### Date and time parts

Using the DATEPART function, you can extract from an input date and time value a desired part, such as a year, minute, or nanosecond, and return the extracted part as an integer. For example, the expression DATEPART(month, ‘20170212’) returns 2. T-SQL provides the functions YEAR, MONTH, and DAY as abbreviations to DATEPART, not requiring you to specify the part. The DATENAME function is similar to DATEPART, only it returns the name of the part as a character string, as opposed to the integer value. Note that the function is language dependent. That is, if the effective language in your session is us_english, the expression DATENAME(month, ‘20170212’) returns ‘February’, but for Italian, it returns ‘febbraio.'

T-SQL provides a set of functions that construct a desired date and time value from its numeric parts. You have such a function for each of the six available date and time types: DATEFROMPARTS, DATETIME2FROMPARTS, DATETIMEFROMPARTS, DATETIMEOFFSETFROMPARTS, SMALLDATETIMEFROMPARTS, and TIMEFROMPARTS. For example, to build a DATE value from 
its parts, you would use an expression such as DATEFROMPARTS(2017, 02, 12).

Finally, the EOMONTH function computes the respective end of month date for the input date and time value. For example, suppose that today was February 12, 2017. The expression 
EOMONTH(SYSDATETIME()) would then return the date ‘2017-02-29’. This function supports a second optional input indicating how many months to add to the result (or subtract if negative).

#### Add and diff functions

T-SQL supports addition and difference date and time functions called DATEADD and DATEDIFF.

DATEADD is a very commonly used function. With it, you can add a requested number of units of a specified part to a specified date and time value. For example, the expression 
```DATEADD(year, 1, ‘20170212’)``` adds one year to the input date February 12, 2017.

DATEDIFF is another commonly used function; it returns the difference in terms of a requested part between two date and time values. For example, the expression ```DATEDIFF(day, ‘20160212’, ‘20170212’)``` computes the difference in days between February 12, 2016 and February 12, 2017, returning the value 366. Note that this function looks only at the parts from the requested one and above in the date and time hierarchy—not below. For example, the expression ```DATEDIFF(year, ‘20161231’, ‘20170101’)``` looks only at the year part, and hence returns 1. It doesn’t look at the month and day parts of the values. The DATEDIFF function returns a value of an INT type. If the difference doesn’t fit in a four-byte integer, use the DATEDIFF_BIG function instead. This function returns the result as a BIGINT type.


