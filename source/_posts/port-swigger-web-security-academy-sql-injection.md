---
title: port swigger web security academy sql injection
date: 2022-06-04 00:18:43
tags:
    - web security
    - sql injection
    - portswigger
---

## Introduction

This article is the note of PortSwigger Web Security Academy's [SQL Injection](https://portswigger.net/web-security/sql-injection). I will take note of it and write some my opinion.

<!-- more -->

## Examples

-   Retrieving hidden data
-   Subverting application logic
-   UNION attack: retrieve data from other databases or tables.
-   Examining the database
-   Blind SQL injection

### Retrieving hidden data

For example, there is a URL:

```
https://insecure-website.com/products?category=Gifts
```

and SQL like:

```SQL
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

Thus, it can be injected by:

```
https://insecure-website.com/products?category=' OR 1=1 --
```

This will results in the SQL query, and show every products:

```SQL
SELECT * FROM products WHERE category = '' OR 1=1 --' AND released = 1
```

### Subverting application logic

It can bypass login or other business logic too.

In case of SQL query like:

```SQL
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
```

We can login as administrator by input username `administrator' --` and left password blank, results in the SQL query:

```SQL
SELECT * FROM users WHERE username = 'administrator' --' AND password = ''
```

### UNION attack

We can use UNION to get other table's data, for example:

```SQL
SELECT name, description FROM products WHERE category = '{input}'
```

and we input:

```
' UNION SELECT username, password FROM users --
```

result in query:

```SQL
SELECT name, description FROM products WHERE category = '' UNION SELECT username, password FROM users --'
```

Then, we can get username and password from other table.

### Examining the database

To explot the database, we need to identify which database is it.

Because every database have unique syntax, function, or variable...(there are some examples below), we can use some [cheat table](https://portswigger.net/web-security/sql-injection/cheat-sheet) to determine it.

#### database-specific factors

-   Syntax for string concatenation
-   Comments
-   Batched or stacked queries
-   Platform-specific APIs
-   Error messages

After we know what kind of database is it, we can grab some informations about databases, tables, and columns.

For example, most database(MSSQL, MySQL, PostgreSQL...) have a database which store there information we need:

```SQL
SELECT * FROM information_schema.tables
```

### Blind SQL injection

When we can see the result of SQL query, we can use UNION to get the informations we need.

But if the application does not return any results, we can still exploit it by following methods:

-   Conditionally change the logic of the query to trigger a detectable difference. For example, trigger an error such as a divide-by-zero.
-   Conditionally make a time delay.
-   Trigger an out-of-band interaction sush as placing the data into a DNS lookup for a domain we control.

## How to detect vulnerabilities

To every entry point in the application, we can try:

-   Submitting `'` and looking for error or other abnormal response.
-   Submitting some SQL-specific syntax or conditions such as `OR 1=1` to change result of the query, and looking for differences in responses.
-   For those cannot see response, submitting payload to trigger time delays and looking for differences in the time taken to respond.

## Second-order SQL Injection

First, we need to talk about `First-order` SQL injection.

`First-order` means the application takes user's input and use it to excute an SQL query.

So, `Second-order`(that is, stored SQL injection) means the application store user's input to database(or somewhere else). Later, the application retrieves the stored data and excute an SQL query with it.

This will happen because developers are aware of SQL injection vulnerabilities, so safely handle the direct input from user. But they forgot that `Second-order` SQL query is also get input from user, so remember "DONT TRUST ANY USER INPUT" when you develop any application.

## How to prevent

So after all, how do we prevent these vulnerabilites?

We can use parameterized query(also known as prepared statements) instead of directly concatenate strings together.

DONT USE:

```php
$query = "SELECT * FROM products WHERE category = '"+ $input + "'";
```

USE:

```php
$sql = "SELECT * FROM products WHERE category = :category";
$sth = $dbh->prepare($sql, array(PDO::ATTR_CURSOR => PDO::CURSOR_FWDONLY));
$sth->execute(array(':category' => 'shoes'));
```

## Summary

I think the most important lesson that SQLi bring us is "NEVER TRUST USER".

As a developer, we need to make sure we know the meaning of every single line of code that we are writting. And trust no one.

## References

-   <https://portswigger.net/web-security/sql-injection>
-   <https://portswigger.net/web-security/sql-injection/cheat-sheet>
