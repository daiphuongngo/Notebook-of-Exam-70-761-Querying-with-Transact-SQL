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

