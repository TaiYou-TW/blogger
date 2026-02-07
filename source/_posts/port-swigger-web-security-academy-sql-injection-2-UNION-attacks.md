---
title: port swigger web security academy sql injection 2 - UNION attacks
date: 2022-06-05 16:49:59
tags:
    - web security
    - sql injection
    - portswigger
---

## Introduction

This article is the sequel of Port Swigger Web Security Academy, you can find previous article [here](https://fongyehong.top/blog/2022/06/03/port-swigger-web-security-academy-sql-injection/).

And this time we will take a deep look about [`UNION` attacks](https://portswigger.net/web-security/sql-injection/union-attacks), let's start.

When we could get responses of query, `UNION` can be used to retrieve more data from other tables. For example:

```sql
SELECT a, b FROM table1 UNION SELECT c, d FROM table2
```

<!-- more -->

And there are two requirements must be met:

1. Two query must return the same number of columns.
2. Two query must have compatible data types in each column.

## Determining the number of columns

There are two methods to reach it:

1. `ORDER BY`
2. `UNION SELECT NULL`

### ORDER BY

Because we can put column name or column order after `ORDER BY`, so we can try below SQL:

```sql
' ORDER BY 1--
```

If there is no error, we know the column number of the first query is at least 1. And we can try:

```sql
' ORDER BY 2--
```

Until we get some error such as:

```
The ORDER BY position number 3 is out of range of the number of items in the select list.
```

Than we know the number of first query is 2.

### UNION SELECT NULL

Because we can directly `SELECT` value by `UNION`, we can try:

```sql
' UNION SELECT NULL--
```

> But why `NULL`? Because we mentioned that `Two query must have compatible data types in each column.` before, and `NULL` is compatible with any type of data.

And if your number of `NULL` is not same with the first query's column number, you will probably get some error like:

```
All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.
```

So keep trying until there is no error!

## Determining the type of columns

After we get the number, we can continue to determine the type of columns. Thus, we can use those columns to hold our interested data. For example:

```sql
' UNION SELECT 'a',NULL,NULL --
```

If you get error like:

```sql
Conversion failed when converting the varchar value 'a' to data type int.
```

Switch position of string to find out which column is string type. Or you can use integer or other type to instead it.

## Retrieving multiple values

We can concatenate the values together to show multiple values in one column, for example in Oracle:

```sql
' UNION SELECT username || '~' || password FROM users--
```

## References

-   <https://portswigger.net/web-security/sql-injection/union-attacks>
